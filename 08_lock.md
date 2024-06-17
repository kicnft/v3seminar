# 8.ロック

### ハッシュロック
AggregateBondedTransactionをアナウンスするために必要なデポジットをロックします。
複数のアカウントによる共同署名が必要なAggregateBondedTransactionは、完全に署名されるまで各ノードのキャッシュに保存されるため、ネットワークリソースを消費します。
そのため、スパム攻撃を防ぐために、HashLockを実施して一定の資金（デフォルトで10 XYM）をロックする必要があります。アグリゲートトランザクションの署名が完了すると、ロックされた資金はハッシュロックを署名したアカウントに返金されます。
アグリゲートトランザクションの署名が完了する前にロックが期限切れになると（デフォルトで48時間）、ロックされた資金はロックが期限切れとなったブロックを収穫したアカウントによって回収されます。

### シークレットロック
Symbolは、資産の分散型交換のために信頼のない環境を作り出すためにHashed TimeLock Contract（HTLC）プロトコルに従います。このプロトコルは、すべての参加者が同意すれば、スワップが行われることを保証します。逆に、参加者の一部がプロセスを完了しないことを決定した場合、各参加者はロックされた資金を元に戻すことができます。HTLCは、ハッシュロックとタイムロックを使用してカウンターパーティリスクを軽減します。トークンの交換に参加するすべての参加者は、完了するために証拠（ハッシュロック）を提示する必要があります。そうしなかった場合、ロックされた資産はタイムロックが期限切れになると元の所有者に戻されます。

## 演習内容
- アグリゲートボンデッドトランザクション
- シークレットロック・シークレットプルーフ

## スクリプト

```js
function mosaic(mosaicId,amount){
console.log("mosaic")
    const mosaicNumber = typeof mosaicId === 'string' ? BigInt("0x" + mosaicId) : mosaicId;
    return new sym.descriptors.UnresolvedMosaicDescriptor(
        new sym.models.UnresolvedMosaicId(mosaicNumber), 
        new sym.models.Amount(amount)
    )
}

//アグリゲートボンデッドトランザクション aggregate bobded transaction
function aggbdtx(transactions,initPublicKey,cosignatureCount){
    const transactionsHash = sym.SymbolFacade.hashEmbeddedTransactions(transactions);
    const desc = new sym.descriptors.AggregateBondedTransactionV2Descriptor(transactionsHash,transactions,[]);
    const tx = chain.createTransactionFromTypedDescriptor(desc,initPublicKey,feeMultiplier,add2Hours,cosignatureCount);
    return tx;
}

function hlocktx(hash){

    const descriptor = new sym.descriptors.HashLockTransactionV1Descriptor( // Txタイプ:ハッシュロックTx
      new sym.descriptors.UnresolvedMosaicDescriptor(
        new sym.models.UnresolvedMosaicId(xymid), // UnresolvedMosaic:未解決モザイク
        new sym.models.Amount(10n * 1000000n)           // 10xym固定値
      ),
      new sym.models.BlockDuration(480n),               // ロック有効期限
      core.utils.hexToUint8(hash)                     // アグリゲートトランザクションのハッシュ値を登録
    );
    return descriptor
}

function slocktx(address,secret,mosaic){

    descriptor = new sym.descriptors.SecretLockTransactionV1Descriptor( // Txタイプ:シークレットロックTx
        address,                                // 解除時の転送先:Bob
        secret,                                     // ロック用キーワード
        mosaic,
        new sym.models.BlockDuration(480n),   // ロック期間(ブロック数)
        sym.models.LockHashAlgorithm.SHA3_256 // ロックキーワード生成に使用したアルゴリズム
    )
    return descriptor
}

function prooftx(address,secret,proof){

    descriptor = new sym.descriptors.SecretProofTransactionV1Descriptor( // Txタイプ:シークレットプルーフTx
        address,                                  // 解除アカウント（受信アカウント）
        secret,                                       // ロックキーワード
        sym.models.LockHashAlgorithm.SHA3_256,  // ロックキーワード生成に使用したアルゴリズム
        proof                                         // 解除用キーワード
    );
    return descriptor
}

function sig(aggregateTx,signer){

    const signature = signer.signTransaction(aggregateTx);
    const requestBody = sym.SymbolTransactionFactory.attachSignature(aggregateTx, signature);
    const hash = chain.hashTransaction(aggregateTx).toString();
    return {request:JSON.parse(requestBody),hash:hash}
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
```

## 演習

### アグリゲートボンデッドトランザクション
```js
bob = newacnt()

tx1 = trftx(bob.address,[mosaic(xymhex,1000000)],'');
tx2 = trftx(alice.address,[],"thank you!");

txes = [
    embed(tx1,alice.publicKey),
    embed(tx2,bob.publicKey)
];

//AggregateBondedTransaction
aggtx = aggbdtx(txes,alice.publicKey,0);
sigedtx = sig(aggtx,alice)

//HashLockTransaction
hltx = hlocktx(sigedtx.hash);
hlhash = await sigan(hltx,alice);
clog(hlhash);

//WAIT for Confirmed

//部分署名のアナウンス
res = await api("/transactions/partial","PUT",sigedtx.request)
console.log(res)
clog(sigedtx.hash)

payload = sigedtx.request.payload
```
##### Script
- aggbdtx ( transactions, initPublicKey, cosignatureCount )
- sig ( aggregateTx, signer )
- hlocktx ( hash ) 

##### API
- /transactions/partial
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Transaction-routes/operation/searchPartialTransactions

### Bobの連署
```js
payload
aggtx = m.TransactionFactory.deserialize(u.hexToUint8(payload))
phash = await cosan(aggtx,bob)
clog(phash)
```
##### Script
- cosan ( aggregateTx, cosigner ) 
##### SDK
- TransactionFactory.deserialize
    - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.SymbolTransactionFactory.html#deserialize

### シークレットロック
```js
sha3_256 = (await import('https://cdn.skypack.dev/@noble/hashes/sha3')).sha3_256

proof = crypto.getRandomValues(new Uint8Array(20))
hash = sha3_256.create();
hash.update(proof);
secret = hash.digest();
//SecretLockTransaction
tx = slocktx(bob.address,secret,mosaic(xymhex,1000000))
hash = await sigan(tx,alice);
clog(hash)
```

##### Script
- slocktx ( address, secret, mosaic )

##### 確認
```js
info = await api("/lock/secret?secret=" + u.uint8ToHex(secret))
linfo = info.data[0].lock
//LockHashAlgorithm
//0: 'Op_Sha3_256'
//1: 'Op_Hash_160'
//2: 'Op_Hash_256'

decadr(linfo.ownerAddress)
decadr(linfo.recipientAddress)
```
##### API
- /lock/secret
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Secret-Lock-routes/operation/searchSecretLock

##### SDK
- LockHashAlgorithm
    - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.models.LockHashAlgorithm.html

### シークレットプルーフ
```js
//SecretProofTransaction
tx = prooftx(bob.address,secret,proof)
hash = await sigan(tx,bob);
clog(hash)
```
##### Script
- prooftx ( address, secret, proof )

##### 確認
```js
await api("/transactions/confirmed/"+ hash)
await api("/statements/transaction?targetAddress=" + bob.address.toString())

//statement.receipts[n].type
//8786: 'LockSecret_Completed' :ロック解除完了
//9042: 'LockSecret_Expired'　：ロック期限切れ
```
