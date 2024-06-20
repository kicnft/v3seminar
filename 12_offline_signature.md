# 12.オフライン署名

アグリゲートコンプリートトランザクションを使用して、ノードのキャッシュを利用せずに共同署名を追加する方法を紹介します。
これは、セキュリティ上の理由で秘密鍵をオフラインデバイス（コールドウォレット）に保管している場合に便利です。オフラインで共同署名を追加することで、共同署名者は秘密鍵を完全に安全に保ちながら、コールドウォレットからトランザクションを実行できるようになります。さらに、ユーザーがアグリゲートボンデッドトランザクションで資金を不必要にロックすることを避けることもできます。

https://docs.symbol.dev/guides/aggregate/adding-cosginatures-aggregate-complete.html

## スクリプト
```js
function sig(aggregateTransaction,signer){

    const signature = signer.signTransaction(aggregateTransaction);
    const requestBody = sym.SymbolTransactionFactory.attachSignature(aggregateTransaction, signature);
    const hash = chain.hashTransaction(aggregateTransaction).toString();
    return {request:JSON.parse(requestBody),hash:hash}
}

console.log("Import Offline Signature Script")
```

### 事前準備
```js
bob = newacnt()
console.log(`bob address: ${bob.address}`)
console.log(`bob private key: ${bob.keyPair.privateKey}`)
```

### トランザクション作成
```js
//TransferTransaction
tx1 = trftx(alice.address,[],"tx1")
tx2 = trftx(bob.address,[],"tx2")

txes = [
    embed(tx1,bob.publicKey),
    embed(tx2,alice.publicKey)
]

//AggregateCompleteTransaction
aggtx = aggcptx(txes,alice.publicKey,1)
res = sig(aggtx,alice,[])
console.log(`payload:\n${res.request.payload}`)
```

### Bobによる連署
```js
bob = acnt("") //秘密鍵を指定
tx = sym.SymbolTransactionFactory.deserialize(u.hexToUint8("")); //payloadを指定

chain.verifyTransaction(tx,tx.signature)
cosig = bob.cosignTransaction(tx, true);
console.log(`signerPublicKey: ${cosig.signerPublicKey}`);
console.log(`signature: ${cosig.signature}`);
```

##### SDK
- verifyTransaction
  - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.SymbolFacade.html#verifyTransaction
- cosignTransaction
  - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.SymbolAccount.html#cosignTransaction


### 集約&アナウンス
```js
retx = sym.SymbolTransactionFactory.deserialize(u.hexToUint8("")) //payload指定
hash = chain.hashTransaction(retx)

//必要な連署の数だけ繰り返す
cosig = new m.Cosignature()
cosig.parentHash = hash
cosig.version = 0n
cosig.signerPublicKey = new m.PublicKey("") //Bobの公開鍵を指定
cosig.signature = new m.Signature("") //Bobの署名を指定
retx.cosignatures.push(cosig)

payload = u.uint8ToHex(retx.serialize())
res = await api("/transactions","PUT",{"payload": payload})
clog(hash.toString())
```

##### SDK
- deserialize
  - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.SymbolTransactionFactory.html#deserialize
- hashTransaction
  - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.SymbolFacade.html#hashTransaction
- Cosignature
  - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.models.Cosignature.html
