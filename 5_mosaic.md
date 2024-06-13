# 5.モザイク
## スクリプト


```js
//モザイクトークン（hexID,数量）
function mosaic(mosaicId,amount){
    const mosaicNumber = typeof value === 'string' ? BigInt("0x" + mosaicId) : mosaicId;
    return new sym.descriptors.UnresolvedMosaicDescriptor(
        new sym.models.UnresolvedMosaicId(mosaicNumber), 
        new sym.models.Amount(amount)
    )
}

function nonce() {
    const buffer = new ArrayBuffer(sym.models.MosaicNonce.SIZE);
    const view = new DataView(buffer);
    crypto.getRandomValues(new Uint8Array(buffer));
    return view.getUint32(0, true);
}

//MosaicDefinitionTransaction
function mosdftx(mosaicId,duration,mosaicNonce,mosaicFlags,divisibility){

    const desc = new sym.descriptors.MosaicDefinitionTransactionV1Descriptor(
        new sym.models.MosaicId(mosaicId),
        new sym.models.BlockDuration(duration),
        new sym.models.MosaicNonce(mosaicNonce),
        new sym.models.MosaicFlags(mosaicFlags),
        divisibility
    )
    return desc;
}

//MosaicSupplyChangeTransaction
function mossctx(mosaicId,amount,mosaicSupplyChangeAction){

    const desc = new sym.descriptors.MosaicSupplyChangeTransactionV1Descriptor(
        new sym.models.UnresolvedMosaicId(mosaicId),
        new sym.models.Amount(amount),
        mosaicSupplyChangeAction
    )
    return desc
}
```

## 演習
```js
mosnc = nonce()
mosid = sym.generateMosaicId(alice.address, mosnc)
mosflg = 0 + 1 + 2 + 4 + 8
//1:SUPPLY_MUTABLE 供給量変更の可否
//2:TRANSFERABLE 第三者への譲渡可否
//4:RESTRICTABLE 制限設定の可否
//8:REVOKABLE  発行者からの還収可否

//MosaicDefinitionTransaction
tx1 = mosdftx(mosid,0,mosnc,mosflg,3)
//dulation:0 有効期限(0:無期限 INFINITY)
//divisivility:3 可分性(3 = 0.000)

//MosaicSupplyChangeTransaction
tx2 = mossctx(mosid,1000,sym.models.MosaicSupplyChangeAction.INCREASE)
//MosaicSupplyChangeAction
//INCREASE:1
//DECREASE:0

txes = [
    embed(tx1,alice.publicKey),
    embed(tx2,alice.publicKey)
];

aggtx = aggcptx(txes,alice.publicKey,0);
hash = await sigcosan(aggtx,alice,[])
clog(hash);
```
##### Script
- nonce ( )
- mosdftx ( mosaicId, duration, mosaicNonce, mosaicFlags, divisibility )
- mossctx ( mosaicId, amount, mosaicSupplyChangeAction )

##### SDK
- MosaicSupplyChangeAction
    - https://symbol.github.io/symbol/sdk/javascript/classes/nem.models.MosaicSupplyChangeAction.html 

### モザイクID一覧
```js
acntinfo = await api("/accounts/" + alice.address)
for(mos of acntinfo.account.mosaics){
    console.log(mos.id)
}
```


### モザイクID変換
```js
mosid
mosid.toString(16)
mosid.toString(16).toUpperCase()
```


### モザイク一覧
```js
acntinfo = await api("/accounts/" + alice.address)
for(mos of acntinfo.account.mosaics){
    mosinfo = await api("/mosaics/" + mos.id)
    console.log(mosinfo.mosaic)
}
```
##### API
- /mosaics/{mosaicId}
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Mosaic-routes/operation/getMosaic 


#### フラグの確認
```js
acntinfo = await api("/accounts/" + alice.address)
mosinfo = await api("/mosaics/" + acntinfo.account.mosaics[0].id)
console.log(mosinfo)

console.log(mosinfo.mosaic.flags & sym.models.MosaicFlags.SUPPLY_MUTABLE.value)
console.log(mosinfo.mosaic.flags & sym.models.MosaicFlags.TRANSFERABLE.value)
console.log(mosinfo.mosaic.flags & sym.models.MosaicFlags.RESTRICTABLE.value)
console.log(mosinfo.mosaic.flags & sym.models.MosaicFlags.REVOKABLE.value)
```
##### SDK
- MosaicFlags
    - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.models.MosaicFlags.html
 
### モザイク送信
```js
tx = trftx(alice.address,[mosaic("71261F9C04C09144",1)],'');
hash = await sigan(tx,alice);
clog(hash);
```

##### Script

- mosaic ( mosaicId, amount )
