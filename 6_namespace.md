# 6.ネームスペース

ネームスペースは、アドレスやモザイクIDの代わりに使用できる人間が読めるテキスト文字列です。
これにより、ブロックチェーン上でプロジェクトや資産のための独自の場所を作成することができます。

ネームスペースはインターネットドメインと同様に機能します。ネームスペースを作成するには、アカウントや資産を参照するために使用する名前を選ぶことから始めます。この名前はネットワーク内で一意である必要があり、最大64文字の長さまで許可されます（使用可能な文字はaからz、0から9、_、および-のみです）。

https://docs.symbol.dev/concepts/namespace.html

## スクリプト
```js
//NamespaceRegistrationTransaction(ROOT)
function nsregtx(name,duration){
    
    const descriptor = new sym.descriptors.NamespaceRegistrationTransactionV1Descriptor(
      new sym.models.NamespaceId(sym.generateNamespaceId(name)),
      sym.models.NamespaceRegistrationType.ROOT,
      new sym.models.BlockDuration(duration),
      undefined,
      name
    )
    return descriptor
}

//NamespaceRegistrationTransaction(CHILD)
function subnsregtx(name,parentName){

    const parentNamespaceNumber = sym.generateNamespacePath(parentName).pop()
    const descriptor = new sym.descriptors.NamespaceRegistrationTransactionV1Descriptor(
      new sym.models.NamespaceId(sym.generateNamespaceId(name)),
      sym.models.NamespaceRegistrationType.CHILD,
      undefined,
      new sym.models.NamespaceId(parentNamespaceNumber),
      name
    )
    return descriptor
}

//AddressAliasTransaction
function adralitx(hexNamespaceId,address,aliasAction){
    const descriptor = new sym.descriptors.AddressAliasTransactionV1Descriptor(
        new sym.models.NamespaceId(BigInt(`0x${hexNamespaceId}`)),
        address,
        aliasAction
    )
    return descriptor
}

//MosaicAliasTransaction
function mosalitx(hexNamespaceId,hexMosaicId,aliasAction){
    const descriptor = new sym.descriptors.MosaicAliasTransactionV1Descriptor(
        new sym.models.NamespaceId(BigInt(`0x${hexNamespaceId}`)),
        new sym.models.MosaicId(BigInt(`0x${hexMosaicId}`)),
        aliasAction
    )
    return descriptor
}

//ネームスペースをアドレスに変換
function nstoadr(name){
    const hex = nstohexadr(name)
    return sym.Address.fromDecodedAddressHexString(hex)
}

//ネームスペースを16進数アドレス文字列に変換
function nstohexadr(name){
    console.log("name")
    const namespaceNumber =  sym.generateNamespacePath(name).pop()
    const address = sym.Address.fromNamespaceId(
      new sym.models.NamespaceId(namespaceNumber),
      chain.network.identifier
    )
    return core.utils.uint8ToHex(address.bytes);
}

//ネームスペースを16進数ネームスペース文字列に変換
function nstohex(name){
    const namespaceNumber = sym.generateNamespacePath(name).pop()
    return  namespaceNumber.toString(16).toUpperCase()  
}

function mosaic(mosaicId,amount){
console.log("mosaic")
    const mosaicNumber = typeof mosaicId === 'string' ? BigInt("0x" + mosaicId) : mosaicId;
    return new sym.descriptors.UnresolvedMosaicDescriptor(
        new sym.models.UnresolvedMosaicId(mosaicNumber), 
        new sym.models.Amount(amount)
    )
}

```

## 演習

### レンタル手数料計算
#### ルートネームスペース
```js
info = await api('/network/fees/rental')

fpb = Number(info.effectiveRootNamespaceRentalFeePerBlock);
blocks = 365 * 24 * 60 * 60 / 30; 
fee = (blocks * fpb) / 1000000;
```

#### サブネームスペース
```js
subfee = Number(info.effectiveChildNamespaceRentalFee) / 1000000;
```

### ネームスペース作成
#### ルートネームスペースの作成
```js
tx1 = nsregtx("kicnft_test",30 * 24 * 60 * 60 / 30)
hash = await sigan(tx1,alice)
clog(hash)
```

#### サブネームスペースの作成
```js
tx2 = subnsregtx("address","kicnft_test")
hash = await sigan(tx2,alice)
clog(hash);
```

```js
tx2 = subnsregtx("mosaic","kicnft_test")
hash = await sigan(tx2,alice)
clog(hash);
```


### リンク
#### アドレスにリンク
```js
nsid = nstohex("kicnft_test.address")
tx3 = adralitx(nsid,alice.address,1)
//AliasAction
//1:LINK
//0:UNLINK

hash = await sigan(tx3,alice)
clog(hash);
```

#### モザイクにリンク
```js

//所有MOSAIC一覧表示
acntinfo = await api("/accounts/" + alice.address)
for(mos of acntinfo.account.mosaics){
    mosinfo = await api("/mosaics/" + mos.id)
    console.log(mosinfo.mosaic)
}

//MOSAIC ID指定
mosid = "71261F9C04C09144"

nsid = nstohex("kicnft_test.mosaic")
tx4 = mosalitx(nsid,mosid,1)
hash = await sigan(tx4,alice)
clog(hash);
```

### 確認
```js
nstohex("kicnft_test")
await api("/namespaces/" + nstohex("kicnft_test"))

nstohex("kicnft_test.address")
await api("/namespaces/" + nstohex("kicnft_test.address"))
```
```js
nstohex("kicnft_test.mosaic")
await api("/namespaces/" + nstohex("kicnft_test.mosaic"))
```

### 未解決アドレスとして使用
```js
adr = nstoadr("kicnft_test.address")
tx5 = trftx(adr,[],'');
hash = await sigan(tx5,alice);
clog(hash);
```

### 未解決モザイクIDとして使用
```js
mosid = nstohex("kicnft_test.mosaic")
tx6 = trftx(alice.address,[mosaic(mosid,1)],'');
hash = await sigan(tx6,alice);
clog(hash);
```

### 逆引き

```js
nsinfo = await api("/namespaces/" + nstohex("kicnft_test.address"))
nsinfo.namespace
nsinfo.namespace.alias
nsinfo.namespace.alias.address
address = sym.Address.fromDecodedAddressHexString(nsinfo.namespace.alias.address).toString()

body = {"addresses": [address]};
res = await api("/namespaces/account/names","POST",body)
res.accountNames
res.accountNames[0].names
```

```js
//モザイクIDを確認
nsinfo = await api("/namespaces/" + nstohex("kicnft_test.mosaic"))
nsinfo.namespace
nsinfo.namespace.alias
mosaicId = nsinfo.namespace.alias.mosaicId

body = {"mosaicIds": [mosaicId]};
res = await api("/namespaces/mosaic/names","POST",body)
res.mosaicNames
res.mosaicNames[0].names
```

### レシート参照

#### アドレス解決
```js
info = await api("/transactions/confirmed/C2E2DCCAAA58176C2076C3E9A8548E57581124D4B749237260110A56C20829C3")
height = info.meta.height
stmt = await api("/statements/resolutions/address?height=" + height)

hex = nstohexadr("kicnft_test.address") //16進数にエンコードされたアドレス文字列
stmt.data.filter(x=>x.statement.unresolved == hex)[0]
stmt.data.filter(x=>x.statement.unresolved == hex)[0].statement
stmt.data.filter(x=>x.statement.unresolved == hex)[0].statement.resolutionEntries[0]
hexadr = stmt.data.filter(x=>x.statement.unresolved == hex)[0].statement.resolutionEntries[0].resolved
adr = sym.Address.fromDecodedAddressHexString(hexadr).toString()
info = await api("/accounts/" + adr);

```

#### モザイク解決
```js
txinfo = await api("/transactions/confirmed/D8121327A6660AB2A1650A857E02A8B94A24AF875CF36E7555D1D6F718F5A823")
height = txinfo.meta.height
stmt = await api("/statements/resolutions/mosaic?height=" + height)

hex = nstohex("kicnft_test.mosaic");
stmt.data.filter(x=>x.statement.unresolved == hex)[0]
stmt.data.filter(x=>x.statement.unresolved == hex)[0].statement
stmt.data.filter(x=>x.statement.unresolved == hex)[0].statement.resolutionEntries[0]
mosid = stmt.data.filter(x=>x.statement.unresolved == hex)[0].statement.resolutionEntries[0].resolved
info = await api("/mosaics/" + mosid)


```


```js
//ネームスペースの作成
tx = subnsregtx("address","kicnft")
hash = await sigan(tx,alice) //署名＆通知

//アドレスにリンク
nsid = nstohex("kicnft.address")
tx = adralitx(nsid,alice.address,1)
hash = await sigan(tx,alice) //署名＆通知

//ネームスペースを使用して送信
adr = nstoadr("kicnft.address")
tx = trftx(adr,[],''); //0XYM ノーコメント
hash = await sigan(tx,alice) //署名＆通知
```
