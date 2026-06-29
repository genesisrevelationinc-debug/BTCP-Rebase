 ```diff
--- a/src/wallet/rpcwallet.cpp
+++ b/src/wallet/rpcwallet.cpp
@@ -1,4 +1,5 @@
 // Copyright (c) 2010 Satoshi Nakamoto
+// Copyright (c) 2018 The Bitcoin Private developers
 // Copyright (c) 2009-2018 The Bitcoin Core developers
 // Distributed under the MIT software license, see the accompanying
 // file COPYING or http://www.opensource.org/licenses/mit-license.php.
@@ -7,6 +8,7 @@
 #include <core_io.h>
 #include <init.h>
 #include <key_io.h>
+#include <pubkey.h>
 #include <net.h>
 #include <outputtype.h>
 #include <policy/fees.h>
@@ -15,6 +17,7 @@
 #include <rpc/server.h>
 #include <rpc/util.h>
 #include <script/descriptor.h>
+#include <script/standard.h>
 #include <timedata.h>
 #include <util/system.h>
 #include <util/moneystr.h>
@@ -22,6 +25,7 @@
 #include <wallet/coincontrol.h>
 #include <wallet/feebumper.h>
 #include <wallet/rpcwallet.h>
+#include <wallet/wallet.h>
 #include <wallet/walletutil.h>
 #include <wallet/coinselection.h>
 #include <warnings.h>
@@ -29,6 +33,7 @@
 #include <stdint.h>
 
 #include <univalue.h>
+#include <boost/algorithm/string.hpp>
 
 
 static const std::string WALLET_ENDPOINT_BASE = "/wallet/";
@@ -36,6 +41,12 @@ static const std::string WALLET_ENDPOINT_BASE = "/wallet/";
 static std::string urlDecode(const std::string &s)
 {
     std::string ret;
+    unsigned int i;
+    for (i = 0; i < s.length(); i++) {
+        if (s[i] == '%' && i + 2 < s.length()) {
+            int val = 0;
+            int ii;
+            for (ii = 1; ii <= 2; ii++) {
+                val *= 16;
+                if (s[i + ii] >= '0' && s[i + ii] <= '9') val += s[i + ii] - '0';
+                else if (s[i + ii] >= 'A' && s[i + ii] <= 'F') val += s[i + ii] - 'A' + 10;
+                else if (s[i + ii] >= 'a' && s[i + ii] <= 'f') val += s[i + ii] - 'a' + 10;
+            }
+            ret += (char)val;
+            i += 2;
+        } else {
+            ret += s[i];
+        }
+    }
+    return ret;
+}
+
+static std::string GetExplorerLink(const std::string& address, const std::string& chain)
+{
+    if (chain == "btc") {
+        return "https://blockchain.info/address/" + address;
+    } else if (chain == "zcl") {
+        return "https://explorer.zcl Dustin.com/address/" + address;
+    }
+    return "";
+}
+
+UniValue viewpreforkaddresses(const JSONRPCRequest& request)
+{
+    std::shared_ptr<CWallet> const wallet = GetWalletForJSONRPCRequest(request);
+    CWallet* const pwallet = wallet.get();
+
+    if (!EnsureWalletIsAvailable(pwallet, request.fHelp)) {
+        return NullUniValue;
+    }
+
+    if (request.fHelp || request.params.size() > 1)
+        throw std::runtime_error(
+            "viewpreforkaddresses ( \"type\" )\n"
+            "\nView the pre-fork BTC and ZCL addresses that correspond to this wallet's BTCP addresses.\n"
+            "\nThis is a view-only utility that does NOT use or expose any private keys.\n"
+            "It simply shows what addresses would exist on other chains for the same public keys.\n"
+            "\nArguments:\n"
+            "1. \"type\"        (string, optional) The type of addresses to view: \"btc\", \"zcl\", or \"all\" (default: \"all\")\n"
+            "\nResult:\n"
+            "{\n"
+            "  \"btcp_addresses\" : [\n"
+            "    {\n"
+            "      \"btcp\"       : \"btcp_address\",\n"
+            "      \"btc\"        : \"btc_address\",\n"
+            "      \"zcl\"        : \"zcl_address\",\n"
+            "      \"btc_explorer\" : \"https://...\",\n"
+            "      \"zcl_explorer\" : \"https://...\"\n"
+            "    },\n"
+            "    ...\n"
+            "  ]\n"
+            "}\n"
+            "\nExamples:\n"
+            + HelpExampleCli("viewpreforkaddresses", "")
+            + HelpExampleCli("viewpreforkaddresses", "\"btc\"")
+            + HelpExampleRpc("viewpreforkaddresses", "\"zcl\"")
+        );
+
+    LOCK2(cs_main, pwallet->cs_wallet);
+
+    std::string type = "all";
+    if (!request.params.empty()) {
+        type = request.params[0].get_str();
+        boost::to_lower(type);
+        if (type != "btc" && type != "zcl" && type != "all")
+            throw JSONRPCError(RPC_INVALID_PARAMETER, "Invalid type. Must be \"btc\", \"zcl\", or \"all\"");
+    }
+
+    UniValue result(UniValue::VOBJ);
+    UniValue addressArray(UniValue::VARR);
+
+    std::vector<CTxDestination> vDestinations;
+    pwallet->GetAddresses(vDestinations);
+
+    for (const auto& dest : vDestinations) {
+        UniValue entry(UniValue::VOBJ);
+        std::string btcpAddr = EncodeDestination(dest);
+        entry.pushKV("btcp", btcpAddr);
+
+        CTxDestination btcDest;
+        CTxDestination zclDest;
+        bool hasBtc = false;
+        bool hasZcl = false;
+
+        // Try to get the public key hash from the destination
+        CKeyID keyID;
+        if (ExtractDestination(GetScriptForDestination(dest), keyID)) {
+            // BTC address (P2PKH)
+            if (type == "btc" || type == "all") {
+                // BTC P2PKH address version byte is 0x00
+                std::vector<unsigned char> btcAddrData;
+                btcAddrData.push_back(