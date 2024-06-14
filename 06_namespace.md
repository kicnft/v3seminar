# 6.ネームスペース

ネームスペースは、アドレスやモザイクIDの代わりに使用できる人間が読めるテキスト文字列です。
これにより、ブロックチェーン上でプロジェクトや資産のための独自の場所を作成することができます。

ネームスペースはインターネットドメインと同様に機能します。ネームスペースを作成するには、アカウントや資産を参照するために使用する名前を選ぶことから始めます。この名前はネットワーク内で一意である必要があり、最大64文字の長さまで許可されます（使用可能な文字はaからz、0から9、_、および-のみです）。

https://docs.symbol.dev/concepts/namespace.html

## 演習内容
- レンタル手数料計算
- ネームスペース作成
- アドレスやモザイクへのリンク
- 登録内容の確認
- 未解決アドレスとして使用
- 未解決モザイクとして使用
- 逆引き
- 名前解決の確認

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

##### API
- /network/fees/rental
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Network-routes/operation/getRentalFees

#### サブネームスペース
```js
subfee = Number(info.effectiveChildNamespaceRentalFee) / 1000000;
```

### ネームスペース作成
#### ルートネームスペースの作成
```js
//NamespaceRegistrationTransaction
tx = nsregtx("kicnft_test",30 * 24 * 60 * 60 / 30)
hash = await sigan(tx,alice)
clog(hash)
```

##### Script
- neregtx( name, duration )

#### サブネームスペースの作成
```js
//NamespaceRegistrationTransaction
tx = subnsregtx("address","kicnft_test")
hash = await sigan(tx,alice)
clog(hash);
```
```js
//NamespaceRegistrationTransaction
tx = subnsregtx("mosaic","kicnft_test")
hash = await sigan(tx,alice)
clog(hash);
```
##### Script
- subnsregtx ( name, parentName )


### リンク
#### アドレスにリンク
```js
nsid = nstohex("kicnft_test.address")

//AddressAliasTransaction
tx = adralitx(nsid,alice.address,1)
//AliasAction
//1:LINK
//0:UNLINK

hash = await sigan(tx,alice)
clog(hash)
```
##### Script
- nstohex ( name ) //ネームスペースを16進数文字列に変換
- adralitx ( hexNamespaceId, address, aliasAction )

#### モザイクにリンク
```js
//所有MOSAIC一覧表示
acntinfo = await api("/accounts/" + alice.address)
for(mos of acntinfo.account.mosaics){
    mosinfo = await api("/mosaics/" + mos.id)
    console.log(mosinfo.mosaic)
}

//所有モザイクの中から一つ選んで指定
mosid = "71261F9C04C09144"

nsid = nstohex("kicnft_test.mosaic")
tx = mosalitx(nsid,mosid,1)
hash = await sigan(tx,alice)
clog(hash);
```
##### Script
- nstohex ( name ) //ネームスペースを16進数文字列に変換
- mosalitx ( hexNamespaceId, hexMosaicId, aliasAction )

### 確認
```js
nstohex("kicnft_test")
await api("/namespaces/" + nstohex("kicnft_test"))

nstohex("kicnft_test.address")
await api("/namespaces/" + nstohex("kicnft_test.address"))

nstohex("kicnft_test.mosaic")
await api("/namespaces/" + nstohex("kicnft_test.mosaic"))
```

##### Script
- nstohex ( name ) //ネームスペースを16進数文字列に変換


### 未解決アドレスとして使用
```js
adr = nstoadr("kicnft_test.address")
//TransferTransaction
tx = trftx(adr,[],'');
adrhash = await sigan(tx,alice);
clog(adrhash);
```

##### Script
- nstoadr ( name )　//ネームスペースをアドレスに変換

### 未解決モザイクIDとして使用
```js
mosid = nstohex("kicnft_test.mosaic")
//TransferTransaction
tx = trftx(alice.address,[mosaic(mosid,1)],'');
moshash = await sigan(tx,alice);
clog(moshash);
```
##### Script
- nstohex ( name ) //ネームスペースを16進数文字列に変換

### 名前解決の確認

#### アドレス解決
```js
info = await api("/transactions/confirmed/" + adrhash)
height = info.meta.height

stmt = await api("/statements/resolutions/address?height=" + height)

hex = nstohexadr("kicnft_test.address") //16進数にエンコードされたアドレス文字列
hexadr = stmt.data.filter(x=>x.statement.unresolved == hex)[0].statement.resolutionEntries[0].resolved
adr = decadr(hexadr)
info = await api("/accounts/" + adr);
```
##### Script
- nstohexadr ( name ) //ネームスペースを16進数アドレス文字列に変換
- decadr ( hexEncodedAddress )
    

#### モザイク解決
```js
txinfo = await api("/transactions/confirmed/" + moshash)
height = txinfo.meta.height

stmt = await api("/statements/resolutions/mosaic?height=" + height)

hex = nstohex("kicnft_test.mosaic");
mosid = stmt.data.filter(x=>x.statement.unresolved == hex)[0].statement.resolutionEntries[0].resolved
info = await api("/mosaics/" + mosid)
```
##### Script
- nstohex ( name )

##### API
- /statements/resolutions/mosaic
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Receipt-routes/operation/searchMosaicResolutionStatements

### 逆引き

#### アドレスの逆引き
```js
nsinfo = await api("/namespaces/" + nstohex("kicnft_test.address"))
address = decadr(nsinfo.namespace.alias.address)

body = {"addresses": [address]};
res = await api("/namespaces/account/names","POST",body)
res.accountNames[0].names
```
##### Script
- decadr ( hexEncodedAddress )
  
##### API
- /namespaces/account/names
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Namespace-routes/operation/getAccountsNames

```js
//モザイクIDを確認
nsinfo = await api("/namespaces/" + nstohex("kicnft_test.mosaic"))
mosaicId = nsinfo.namespace.alias.mosaicId

body = {"mosaicIds": [mosaicId]};
res = await api("/namespaces/mosaic/names","POST",body)
res.mosaicNames[0].names
```

