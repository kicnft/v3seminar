# 11.制限

#### アカウント制限
アカウントは、一連の制限に基づいてトランザクションのアナウンスや受信をブロックするためのスマートルールを設定することができます。
アカウントの所有者（マルチシグアカウントの場合は複数人）は、後で特定のAccountAddressRestrictionTransactionをアナウンスすることにより、アカウントの制限を編集できます。
Symbolのパブリックネットワークでは、アカウントごとおよび制限タイプごとに最大512の制限を定義でき、このパラメータはネットワークごとに設定可能です。

#### グローバルモザイク制限
モザイクの制限により、モザイク作成者はどのアカウントがその資産と取引できるかを決定することができます。
この機能は特にセキュリティ・トークン・オファリング（STO）向けに特化されています。ICOで導入された規制されていないトークンとは対照的に、セキュリティトークンは価値のブロックチェーン上の表現であり、証券法の規制対象となるため、ブロックチェーンの自律性を回避する方法が必要です。
特定のネットワークのすべてのモザイクがモザイク制限の対象になるわけではありません。この機能は、発行者が作成時に明示的に制限可能なプロパティを追加したモザイクにのみ影響します。このプロパティは、パブリックネットワーク通貨のような自律的なトークンには望ましくないため、デフォルトでは無効になっています。

## 演習内容
- アカウント制限
- グローバルモザイク制限
  
## スクリプト
```js
function acntrestx(restrictionFlags,restrictionAdditions,restrictionDeletions){
    descriptor = new sym.descriptors.AccountAddressRestrictionTransactionV1Descriptor(  // Txタイプ:アドレス制限設定Tx
      flags,restrictionAdditions,restrictionDeletions
    );
    return descriptor
}

function mosrestx(restrictionFlags,restrictionAdditions,restrictionDeletions){
    descriptor = new sym.descriptors.AccountMosaicRestrictionTransactionV1Descriptor(  // Txタイプ:アドレス制限設定Tx
      flags,restrictionAdditions,restrictionDeletions
    );
    return descriptor
}

function operestx(flags,restrictionAdditions,restrictionDeletions){
    descriptor = new sym.descriptors.AccountOperationRestrictionTransactionV1Descriptor(
        flags,restrictionAdditions,restrictionDeletions
    );
    return descriptor
}

function mosglorestx(
    mosaicId,referenceMosaicId,restrictionKey,
    previousRestrictionValue,newRestrictionValue,
    previousRestrictionType,newRestrictionType
){
    descriptor = new sym.descriptors.MosaicGlobalRestrictionTransactionV1Descriptor(
        mosaicId,referenceMosaicId,restrictionKey,
        previousRestrictionValue,newRestrictionValue,
        previousRestrictionType,newRestrictionType
    )
    return descriptor
}

function mosadrrestx(
    mosaicId,restrictionKey,previousRestrictionValue,newRestrictionValue,targetAddress
){
    descriptor = new sym.descriptors.MosaicAddressRestrictionTransactionV1Descriptor(
        mosaicId,restrictionKey,previousRestrictionValue,newRestrictionValue,targetAddress
    )
    return descriptor
}

function moshex(moshex){
    return new sym.models.UnresolvedMosaicId(BigInt("0x" + moshex))
}

negkey = 0xFFFFFFFFFFFFFFFFn
sha3_256 = (await import('https://cdn.skypack.dev/@noble/hashes/sha3')).sha3_256;
```

## 演習
```js
dave = newacnt()
tx = trftx(dave.address,[mosaic(xymid,100_000000n)],'');
hash = await sigan(tx,alice);
clog(hash);


```

```js
bob = newacnt()

f = 1 + 32768 //BlockIncomingAddress
flags = new sym.models.AccountRestrictionFlags(f);
//    1:AllowIncomingAddress: 0_allow     + 0_incoming     + 1_adddress
//16375:AllowOutgoingAddress: 0_allow     + 16384_outgoing + 1_address 
//32769:BlockIncomingAddress: 32768_block + 0_incoming     + 1_address
//49153:BlockOutgoingAddress: 32768_block + 16384_outgoing + 1_address

tx = acntrestx(flags,[bob.address],[])
hash = await sigan(tx,dave)
clog(hash)
```

```js
f = 2 + 32768 //BlockMosaicId
flags = new sym.models.AccountRestrictionFlags(f);
//    2:AllowMosaic: 0_allow     + 2_mosaic_id
//32770:BlockMosaic: 32768_block + 2_mosaic_id

tx = mosrestx(flags,[moshex("71261F9C04C09144")],[])
hash = await sigan(tx,dave)
clog(hash)
```

```js
f = 4 + 16384 //AllowOutgoingTransactionType
flags = new sym.models.AccountRestrictionFlags(f);
//16388:AllowOutgoingAddress: 0_allow     + 16384_outgoing + 4_transaction_type 
//49156:BlockOutgoingAddress: 32768_block + 16384_outgoing + 4_transaction_type 

tx = operestx(flags,[m.TransactionType.ACCOUNT_OPERATION_RESTRICTION.value],[])
hash = await sigan(tx,dave)
clog(hash)
```


- 

### 確認
```js
await api(`/restrictions/account/${dave.address}`)
```

### 解除

- 5.モザイク のスクリプトを実行しておいてください。

### グローバルモザイク制限
```js

ellen = newacnt()
tx = trftx(ellen.address,[mosaic(xymid,60_000000n)],"")// 52XYM
hash = await sigan(tx,alice)
clog(hash)

```
#### モザイク作成
```js
mosnc = nonce()
mosid = sym.generateMosaicId(ellen.address, mosnc)
mosflg = 4
//4:RESTRICTABLE 制限設定の可否

//MosaicDefinitionTransaction
tx1 = mosdftx(mosid,0,mosnc,mosflg,0)
//dulation:0 有効期限(0:無期限 INFINITY)
//divisivility:3 可分性(3 = 0.000)

//MosaicSupplyChangeTransaction
tx2 = mossctx(mosid,1000,1)
//MosaicSupplyChangeAction
//INCREASE:1
//DECREASE:0

key = sym.metadataGenerateKey("KYC"); // restrictionKey 
tx3 = mosglorestx(mosid,0,key,0n,1n,0,1)

txes = [
    embed(tx1,ellen.publicKey),
    embed(tx2,ellen.publicKey),
    embed(tx3,ellen.publicKey),
];

aggtx = aggcptx(txes,ellen.publicKey,0);
hash = await sigcosan(aggtx,ellen,[])
clog(hash);


```

- mosglorestx
    - (mosaicId, referenceMosaicId, restrictionKey, previousRestrictionValue, newRestrictionValue, previousRestrictionType, newRestrictionType)
    - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.descriptors.MosaicGlobalRestrictionTransactionV1Descriptor.html

- MosaicRestrictionType
    - EQ,GE,GT,LE,LT,NE,NONE
    - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.models.MosaicRestrictionType.html

```js
frank = newacnt()
tx1 = mosadrrestx(mosid,key,negkey,1n,ellen.address)
tx2 = mosadrrestx(mosid,key,negkey,1n,frank.address)

txes = [
    embed(tx1,ellen.publicKey),
    embed(tx2,ellen.publicKey)
]

aggtx = aggcptx(txes,ellen.publicKey,0)
hash = await sigcosan(aggtx,ellen,[])
clog(hash)
```

- mosadrrestx
    - (mosaicId, restrictionKey, previousRestrictionValue, newRestrictionValue, targetAddress)
    - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.descriptors.MosaicAddressRestrictionTransactionV1Descriptor.html


### 確認
```js
hex = mosid.toString(16)
info = await api("/restrictions/mosaic?mosaicId=" + hex)
info.data
```


### 送信テスト

ellen -> aliceに送信：失敗
```js
tx = trftx(alice.address,[mosaic(mosid,1)],"")
hash = await sigan(tx,ellen)
clog(hash)
```
`Failure_RestrictionMosaic_Account_Unauthorized`

ellen->frankに送信：成功
```js
tx = trftx(frank.address,[mosaic(mosid,1)],"")
hash = await sigan(tx,ellen)
clog(hash)
```

