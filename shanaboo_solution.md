 ```diff
--- a/src/rpc/client.cpp
+++ b/src/rpc/client.cpp
@@ -123,6 +123,7 @@ static const CRPCConvertParam vRPCConvertParams[] =
     { "importmulti", 1, "options" },
     { "importmulti", 2, "range" },
     { "importdescriptors", 0, "requests" },
+    { "viewforkaddresses", 0, "addresses" },
 };
 // clang-format on
 
--- a/src/rpc/misc.cpp
+++ b/src/rpc/misc.cpp
@@ -20,6 +20,7 @@
 #include <util/system.h>
 #include <util/strencodings.h>
 #include <warnings.h>
+#include <key_io.h>
 
 #include <stdint.h>
 
@@ -32,6 +33,7 @@
 #include <boost/algorithm/string.hpp>
 
 #include <univalue.h>
+#include <hash.h>
 
 static CRPCValueTable::value_type convertValueType(const UniValue& v)
 {
@@ -618,6 +620,134 @@ static UniValue verifymessage(const JSONRPCRequest& request)
     return (pubkey.GetID() == *keyID);
 }
 
+static UniValue viewforkaddresses(const JSONRPCRequest& request)
+{
+    std::shared_ptr<CWallet> const wallet = GetWalletForJSONRPCRequest(request);
+    CWallet* const pwallet = wallet.get();
+
+    if (!EnsureWalletIsAvailable(pwallet, request.fHelp)) {
+        return NullUniValue;
+    }
+
+    if (request.fHelp || request.params.size() < 1 || request.params.size() > 2)
+        throw std::runtime_error(
+            RPCHelpMan{"viewforkaddresses",
+                "\nView the corresponding BTC and/or ZCL addresses for a given set of BTCP addresses.\n"
+                "This is a view-only utility and does not use or expose any private keys.\n"
+                "The addresses shown are derived from the public information of the BTCP addresses.\n",
+                {
+                    {"addresses", RPCArg::Type::ARR, RPCArg::Optional::NO, "A json array of BTCP addresses",
+                        {
+                            {"address", RPCArg::Type::STR, RPCArg::Optional::OMITTED, "BTCP address"},
+                       },
+                    },
+                    {"options", RPCArg::Type::OBJ, RPCArg::Optional::OMITTED_NAMED_ARG, "Options for viewing fork addresses",
+                        {
+                            {"includebtc", RPCArg::Type::BOOL, /* default */ "true", "Include corresponding BTC addresses"},
+                            {"includezcl", RPCArg::Type::BOOL, /* default */ "true", "Include corresponding ZCL addresses"},
+                            {"includeutxolinks", RPCArg::Type::BOOL, /* default */ "false", "Include blockchain explorer links for UTXO lookup"},
+                        },
+                    },
+                },
+                RPCResult{
+                    "{                             (json object)\n"
+                    "  \"btcp_address\" : {          (json object) The original BTCP address\n"
+                    "    \"btc multisig_address\" : \"address\",  (string, optional) Corresponding BTC address (if applicable)\n"
+                    "    \"zcl_address\" : \"address\",           (string, optional) Corresponding ZCL address (if applicable)\n"
+                    "    \"btc_utxo_link\" : \"url\",             (string, optional) BTC blockchain explorer link\n"
+                    "    \"zcl_utxo_link\" : \"url\"              (string, optional) ZCL blockchain explorer link\n"
+                    "  },\n"
+                    "  ...\n"
+                    "}\n"
+                },
+                RPCExamples{
+                    HelpExampleCli("viewforkaddresses", "\"[\\\"btcpaddress1\\\",\\\"btcpaddress2\\\"]\"")
+                    + HelpExampleCli("viewforkaddresses", "\"[\\\"btcpaddress1\\\"]\" \"{\\\"includebtc\\\":true,\\\"includezcl\\\":true,\\\"includeutxolinks\\\":true}\"")
+                    + HelpExampleRpc("viewforkaddresses", "[\"btcpaddress1\",\"btcpaddress2\"]")
+                },
+            }.ToString());
+
+    RPCTypeCheck(request.params, {UniValue::VARR, UniValue::VOBJ}, true);
+
+    UniValue addresses = request.params[0].get_array();
+
+    bool includeBtc = true;
+    bool includeZcl = true;
+    bool includeUtxoLinks = false;
+
+    if (!request.params[1].isNull()) {
+        const UniValue& options = request.params[1].get_obj();
+        RPCTypeCheckObj(options,
+            {
+                {"includebtc", UniValue::VBOOL},
+                {"includezcl", UniValue::VBOOL},
+                {"includeutxolinks", UniValue::VBOOL},
+            },
+            true, true);
+
+        if (options.exists("includebtc"))
+            includeBtc = options["includebtc"].get_bool();
+        if (options.exists("includezcl"))
+            includeZcl = options["includezcl"].get_bool();
+        if (options.exists("includeutxolinks"))
+            includeUtxoLinks = options["includeutxolinks"].get_bool();
+    }
+
+    UniValue ret(UniValue::VOBJ);
+
+    for (unsigned int idx = 0; idx < addresses.size(); idx++) {
+        const std::string& strAddr = addresses[idx].get_str();
+        CTxDestination dest = DecodeDestination(strAddr);
+        if (!IsValidDestination(dest)) {
+            throw JSONRPCError(RPC_INVALID_ADDRESS_OR_KEY, "Invalid BTCP address: " + strAddr);
+        }
+
+        UniValue addressResult(UniValue::VOBJ);
+
+        // Get the public key hash or script hash from the destination
+        uint160 hash;
+        if (std::get_if<PKHash>(&dest)) {
+            hash = uint160(std::get<PKHash>(dest));
+        } else if (std::get_if<ScriptHash>(&dest)) {
+            hash = uint160(std::get<ScriptHash>(dest));
+        } else {
+            // For other types, we can't derive fork addresses
+            addressResult.pushKV("error", "Unsupported address type for fork address derivation");
+            ret.pushKV(strAddr, addressResult);
+            continue;
+        }
+
+        if (includeBtc) {
+            // BTC uses base58 with version byte 0 (P2PKH) or 5 (P2SH)
+            CTxDestination btcDest = dest;
+            std::string btcAddr = EncodeDestination(btcDest);
+            // Note: In a real implementation, this would need proper BTC address encoding
+           