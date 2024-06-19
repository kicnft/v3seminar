# 7.メタデータ

Symbolは、カスタムデータをアカウント、モザイク、またはネームスペースにトランザクションで関連付けるオプションを提供します。

メタデータの最も一般的な使用例は以下の通りです：
- 資産に関連する情報を添付する。
- 資産に関連付けられた価値を検証し、ユーザーがオフチェーンのアクションを実行できるようにする。

メタデータは、{署名者、ターゲットID、メタデータキー}のタプルによって一意に識別されます。この複合識別子に署名者が含まれていることで、複数のアカウントが同じメタデータを競合なく指定できるようになります。
識別子にリンクされた値は最大1024文字の文字列です。

https://docs.symbol.dev/concepts/metadata.html

## スクリプト
```js
//AccountMetadataTransaction
function acntmetatx(targetAddress,key,sizeDeltaAndValueMap){
    const descriptor = new sym.descriptors.AccountMetadataTransactionV1Descriptor(
      targetAddress,  // ターゲットアドレス
      key,            // キー
      sizeDeltaAndValueMap.delta,      // サイズ差分
      sizeDeltaAndValueMap.value           // 値
    )
    return descriptor
}

//MosaicMetadataTransaction
function mosmetatx(targetAddress,key,targetMosaic,sizeDeltaAndValueMap){
    const descriptor = new sym.descriptors.MosaicMetadataTransactionV1Descriptor(
        targetAddress,  // ターゲットアドレス
        key,            // キー
        new sym.models.UnresolvedMosaicId(BigInt(`0x${targetMosaic}`)),  // メタデータ記録先モザイク
        sizeDeltaAndValueMap.delta,      // サイズ差分
        sizeDeltaAndValueMap.value           // 値
    )
    return descriptor
}

function nstohex(name){
    const namespaceNumber = sym.generateNamespacePath(name).pop()
    return  namespaceNumber.toString(16).toUpperCase()  
}

//NamespaceMetadataTransaction
function nsmetatx(targetAddress,key,targetNamespace,sizeDeltaAndValueMap){
    const descriptor = new sym.descriptors.NamespaceMetadataTransactionV1Descriptor(
        targetAddress,  // ターゲットアドレス
        key,            // キー
        new sym.models.NamespaceId(BigInt(`0x${targetNamespace}`)),  // メタデータ記録先モザイク
        sizeDeltaAndValueMap.delta,      // サイズ差分
        sizeDeltaAndValueMap.value           // 値
    )
    return descriptor
}

function updvalue(info,value){

    const encodedValue = new TextEncoder().encode(value)
    const delta = encodedValue.length
    const res = {}
    if (info.data.length > 0) {
        res.delta = delta - info.data[0].metadataEntry.valueSize
        res.value = sym.metadataUpdateValue(core.utils.hexToUint8(info.data[0].metadataEntry.value), encodedValue )
    }else{
        res.delta = delta
        res.value = encodedValue
    }
    return res
}

function metaquery(addressOrId,sourceAddress,scopedMetadataKey,metadataType){
    let query;
    if(metadataType === 0){

        //Address
        query = new URLSearchParams({
          "targetAddress": addressOrId.toString(),
          "sourceAddress": sourceAddress.toString(),
          "scopedMetadataKey": key.toString(16).toUpperCase(),
          "metadataType": metadataType
        });
    }else{

        //Mosaic or Namespace
        query = new URLSearchParams({
            "targetId": addressOrId.toString(16).toUpperCase(),
            "sourceAddress": sourceAddress.toString(),
            "scopedMetadataKey": key.toString(16).toUpperCase(),
            "metadataType": metadataType
        });
    }
    return query;
}
console.log("Import Metadata Script")
```
    
## 演習

### アドレスに登録
```js
tgtadr = alice.address;  // メタデータ記録先アドレス
srcadr = alice.address;  // メタデータ作成者アドレス

key = sym.metadataGenerateKey("key_account")
value = "test";
query = metaquery(tgtadr,srcadr,key,0)
info = await api("/metadata?" + query.toString())

dtvalue = updvalue(info,value);
//AccountMetadataTransaction
tx = acntmetatx(tgtadr,key,dtvalue);
txes = [
    embed(tx,alice.publicKey),
];

aggtx = aggcptx(txes,alice.publicKey,0);
hash = await sigcosan(aggtx,alice,[])
clog(hash);
```
##### Script
- metaquery ( addressOrId, sourceAddress, scopedMetadataKey, metadataType )
- updvalue( info, value )
- acntmetatx ( targetAddress,key,sizeDeltaAndValueMap )

##### API
- /metadata
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Metadata-routes/operation/getMetadataMerkle

##### SDK
- metadataGenerateKey
    -  https://symbol.github.io/symbol/sdk/javascript/functions/symbol.metadataGenerateKey.html

#### 確認
```js
info = await api("/metadata?" + query.toString())
value = core.utils.hexToUint8(info.data[0].metadataEntry.value)
td = new TextDecoder()
td.decode(value))
```

##### API
- /metadata
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Metadata-routes/operation/getMetadataMerkle

##### SDK
- hexToUint8
    - https://symbol.github.io/symbol/sdk/javascript/functions/index.utils.hexToUint8.html

### モザイクに登録
```js

//所有MOSAIC一覧表示
acntinfo = await api("/accounts/" + alice.address)
for(mos of acntinfo.account.mosaics){
    mosinfo = await api("/mosaics/" + mos.id)
    console.log(mosinfo.mosaic)
}

//ターゲットモザイクの決定
mosid = "71261F9C04C09144"
tgtadr = alice.address;  // メタデータ記録先アドレス
srcadr = alice.address;  // メタデータ作成者アドレス
key = sym.metadataGenerateKey("key_mosaic")
query = metaquery(mosid,srcadr,key,1)
info = await api("/metadata?" + query.toString())
value = "test";

dtvalue = updvalue(info,value);
tx = mosmetatx(tgtadr,key,mosid,dtvalue);
txes = [
    embed(tx,alice.publicKey),
];

aggtx = aggcptx(txes,alice.publicKey,0);
hash = await sigcosan(aggtx,alice,[])
clog(hash);
```

##### Script
- metaquery ( addressOrId, sourceAddress, scopedMetadataKey, metadataType )
- updvalue ( info, value )
- mosmetatx ( targetAddress, key, targetMosaic, sizeDeltaAndValueMap )

##### API
- /metadata
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Metadata-routes/operation/getMetadataMerkle

##### SDK
- metadataGenerateKey
    -  https://symbol.github.io/symbol/sdk/javascript/functions/symbol.metadataGenerateKey.html


#### 確認
```js
info = await api("/metadata?" + query.toString())
value = core.utils.hexToUint8(info.data[0].metadataEntry.value)
td = new TextDecoder()
td.decode(value))
```
##### API
- /metadata
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Metadata-routes/operation/getMetadataMerkle

##### SDK
- hexToUint8
    - https://symbol.github.io/symbol/sdk/javascript/functions/index.utils.hexToUint8.html

### ネームスペースに登録

```js

nstohex("kicnft_test")
await api("/namespaces/" + nstohex("kicnft_test"))

//ターゲットネームスペースの決定
nsid = "86C2182788284A58"
tgtadr = alice.address;  // メタデータ記録先アドレス
srcadr = alice.address;  // メタデータ作成者アドレス
key = sym.metadataGenerateKey("key_namespace")
query = metaquery(nsid,srcadr,key,2)
info = await api("/metadata?" + query.toString())
value = "update test";

dtvalue = updvalue(info,value);
tx = nsmetatx(tgtadr,key,nsid,dtvalue);
txes = [
    embed(tx,alice.publicKey),
];

aggtx = aggcptx(txes,alice.publicKey,0);
hash = await sigcosan(aggtx,alice,[])
clog(hash);
```
##### Script
- metaquery ( addressOrId, sourceAddress, scopedMetadataKey, metadataType )
- updvalue ( info, value )
- nsmetatx ( targetAddress, key, targetNamespace, sizeDeltaAndValueMap )

##### API
- /metadata
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Metadata-routes/operation/getMetadataMerkle

##### SDK
- metadataGenerateKey
    -  https://symbol.github.io/symbol/sdk/javascript/functions/symbol.metadataGenerateKey.html


#### 確認
```js
info = await api("/metadata?" + query.toString())
value = core.utils.hexToUint8(info.data[0].metadataEntry.value)
td = new TextDecoder()
td.decode(value))
```
##### API
- /metadata
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Metadata-routes/operation/getMetadataMerkle

##### SDK
- hexToUint8
    - https://symbol.github.io/symbol/sdk/javascript/functions/index.utils.hexToUint8.html
