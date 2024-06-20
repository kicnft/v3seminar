# 9.マルチシグ化
Symbolアカウントはマルチシグに変換することができます。マルチシグの共同署名者である他のアカウントがアカウントの管理者となります。
マルチシグアカウントは自分自身でトランザクションを通知することができなくなります。マルチシグの共同署名者がマルチシグを含むトランザクションを提案し、それをアグリゲートトランザクションにラップする必要があります。
トランザクションがブロックに記録されるためには、他の共同署名者が同意する必要があります。

https://docs.symbol.dev/concepts/multisig-account.html

## スクリプト

```js
function msigtx(minRemoval,minApproval,addressAdditions,addressDeletions){
    descriptor = new sym.descriptors.MultisigAccountModificationTransactionV1Descriptor(  // Txタイプ:マルチシグ設定Tx
      minRemoval,  // minRemoval:除名のために必要な最小署名者数増分
      minApproval,  // minApproval:承認のために必要な最小署名者数増分
      addressAdditions,
      addressDeletions  // 除名対象アドレスリスト
    );
    return descriptor
}

//アグリゲートボンデッドトランザクション aggregate bobded transaction
function aggbdtx(transactions,initPublicKey,cosignatureCount){
    const transactionsHash = sym.SymbolFacade.hashEmbeddedTransactions(transactions);
    const desc = new sym.descriptors.AggregateBondedTransactionV2Descriptor(transactionsHash,transactions,[]);
    const tx = chain.createTransactionFromTypedDescriptor(desc,initPublicKey,feeMultiplier,add2Hours,cosignatureCount);
    return tx;
}

async function cosan(aggregateTx,cosigner){

    const cosignature = cosigner.cosignTransaction(aggregateTx, true);
    const body= {
      "parentHash": cosignature.parentHash.toString(),
      "signature": cosignature.signature.toString(),
      "signerPublicKey": cosignature.signerPublicKey.toString(),
      "version": cosignature.version.toString()
    };
    
    const res = await api('/transactions/cosignature',"PUT",body)
    console.log(res)
    return cosignature.parentHash.toString();
}
console.log("Import Multisig Script")
```
## 演習
### 準備
```js
bob = newacnt();
carol1 = newacnt();
carol2 = newacnt();
carol3 = newacnt();
carol4 = newacnt();
carol5 = newacnt();

console.log(bob.keyPair.privateKey.toString())
console.log(carol1.keyPair.privateKey.toString())
console.log(carol2.keyPair.privateKey.toString())
console.log(carol3.keyPair.privateKey.toString())
console.log(carol4.keyPair.privateKey.toString())
console.log(carol5.keyPair.privateKey.toString())

console.log(bob.address.toString())
console.log(carol1.address.toString())
console.log(carol2.address.toString())
console.log(carol3.address.toString())
console.log(carol4.address.toString())
console.log(carol5.address.toString())

tx1 = trftx(bob.address,[mosaic(xymid,20000000)],'');
tx2 = trftx(carol1.address,[mosaic(xymid,20000000)],"");
txes = [
    embed(tx1,alice.publicKey),
    embed(tx2,alice.publicKey)
];
aggtx = aggcptx(txes,alice.publicKey,0);
hash = await sigcosan(aggtx,alice,[])
clog(hash);
```

### マルチシグ化
```js
//MultisigAccountModificationTransaction
tx = msigtx(3,3,[carol1.address,carol2.address,carol3.address,carol4.address],[])
txes = [
    embed(tx,bob.publicKey),
];
aggtx = aggcptx(txes,bob.publicKey,4);
hash = await sigcosan(aggtx,bob,[carol1,carol2,carol3,carol4])
clog(hash)
```
##### Script
- msigtx ( minRemoval, minApproval, addressAdditions, addressDeletions )

#### 確認
```js
info = await api("/account/" + bob.address.toString() + "/multisig")
info.multisig.cosignatoryAddresses.map(x=>decadr(x))
```
##### API
- /account/{address}/multsig
  - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Multisig-routes/operation/getAccountMultisig

### 送信
#### アグリゲートコンプリートトランザクションで送信
```js
tx = trftx(alice.address,[],'');
txes = [embed(tx,bob.publicKey)]; //マルチシグ化したアカウントのアセット
aggtx = aggcptx(txes,carol1.publicKey,2); //起案者
hash = await sigcosan(aggtx,carol1,[carol2,carol3]); //署名
clog(hash);
```

#### アグリゲートボンデッドトランザクションで送信
```js
tx = trftx(alice.address,[],'');
txes = [embed(tx,bob.publicKey)]; //マルチシグ化したアカウントのアセット
//AggregateBondedTransaction
aggtx = aggbdtx(txes,carol1.publicKey,2); //起案者
sigedtx = sig(aggtx,carol1)

//ハッシュロックトランザクション
hltx = hlocktx(sigedtx.hash);
hlhash = await sigan(hltx,alice);
clog(hlhash);

//ブロック承認待ち

res = await api("/transactions/partial","PUT",sigedtx.request)
console.log(res)
clog(sigedtx.hash)

payload = sigedtx.request.payload
```
#### 連署
```js
phash = await cosan(aggtx,carol2)
clog(phash)
phash = await cosan(aggtx,carol3)
clog(phash)
```
##### Script
- cosan ( aggregateTx, cosigner )

#### 確認
```js
info = await api("/transactions/confirmed/" + phash)
info.transaction.transactions[0].transaction.signerPublicKey
```

### 構成変更
#### Carol3の除名
```js
//MultisigAccountModificationTransaction
tx = msigtx(-1,-1,[],[carol3.address])
txes = [embed(tx,bob.publicKey)]
aggtx = aggcptx(txes,carol1.publicKey,2)
hash = await sigcosan(aggtx,carol1,[carol2,carol4])
clog(hash)

console.log(`Carol1:${carol1.address}`)
console.log(`Carol2:${carol2.address}`)
console.log(`Carol4:${carol4.address}`)
```

#### Carol4とCarol5を交代
```js
//MultisigAccountModificationTransaction
tx = msigtx(0,0,[carol5.address],[carol4.address])
txes = [embed(tx,bob.publicKey)]
aggtx = aggcptx(txes,carol1.publicKey,2)
hash = await sigcosan(aggtx,carol1,[carol2,carol5])
clog(hash)

console.log(`Carol1:${carol1.address}`)
console.log(`Carol2:${carol2.address}`)
console.log(`Carol5:${carol5.address}`)
```
