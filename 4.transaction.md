# 4.トランザクション

前章で作成したアカウントはトランザクションに署名することでブロックチェーンの更新を行います。
この章では、もっとも基本となる転送トランザクションを通じて、トランザクションの作成から署名・ネットワークへの通知までを学びます
なお、前章で作成したアカウントは引き続きこの後の章でも使用するので学習が終わるまで秘密鍵を保管しておいてください。

## スクリプト
```js
//転送トランザクション transfer transaction
function trftx(address,mosaics,message){
    messageData = new Uint8Array([
        0x00, //現行アプリケーション対応 00:平文, 01:暗号化
        ...new TextEncoder().encode(message),
    ]);
    
    return new sym.descriptors.TransferTransactionV1Descriptor(address,mosaics,messageData);
}

//署名＆通知 sign and announce
async function sigan(desc,signer){
    const tx = chain.createTransactionFromTypedDescriptor(desc,signer.publicKey,feeMultiplier,add2Hours);
    txstat(tx);

    const signature = signer.signTransaction(tx);
    const requestBody = sym.SymbolTransactionFactory.attachSignature(tx, signature);
    const hash = chain.hashTransaction(tx).toString();
    const res = await fetch(
      new URL('/transactions', node),
      {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: requestBody,
      }
    );
    console.log(res);
    return hash;
}

//埋め込みトランザクション
function embed(tx,pubkey){
    return chain.createEmbeddedTransactionFromTypedDescriptor(tx,pubkey);
}

//アグリゲートコンプリートトランザクション aggregate complete transaction
function aggcptx(transactions,initPublicKey,cosignatureCount){
    const transactionsHash = sym.SymbolFacade.hashEmbeddedTransactions(transactions);
    const desc = new sym.descriptors.AggregateCompleteTransactionV2Descriptor(transactionsHash,transactions,[]);
    const tx = chain.createTransactionFromTypedDescriptor(desc,initPublicKey,feeMultiplier,add2Hours,cosignatureCount);
    return tx;
}

//署名＆連署＆通知 sign and cosign and announce
async function sigcosan(tx,signer,cosigners){

    //署名
    const signature = signer.signTransaction(tx);
    sym.SymbolTransactionFactory.attachSignature(tx, signature);

    //連署
    for(cosigner of cosigners){
        const cosignature = cosigner.cosignTransaction(tx);
        tx.cosignatures.push(cosignature);
    }
    txstat(tx);

    //通知
    const requestBody = sym.SymbolTransactionFactory.attachSignature(tx, tx.signature);
    const hash = chain.hashTransaction(tx).toString();
    const res = await fetch(
      new URL('/transactions', node),
      {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: requestBody,
      }
    );
    console.log(res);
    return hash;
}


```

## 演習

### 転送トランザクション
```js
alice = acnt("C9E741092833FEA79D7DB7DC839506BBEB6704631DB9B4D59C60A02BF6B0200C");
bob = newacnt();

tx = trftx(bob.address,[],'hello');
hash = await sigan(tx,alice);
clog(hash);
```

- **trftx ( address, mosaics, message)**
    - TransferTransactionV1Descriptor
        - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.descriptors.TransferTransactionV1Descriptor.html
- **sigan ( desc, signer )**
    - createTransactionFromTypedDescriptor
        - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.SymbolFacade.html#createTransactionFromTypedDescriptor
    - attachSignature
        - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.SymbolTransactionFactory.html#attachSignature
    - hashTransaction
        - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.SymbolFacade.html#hashTransaction
- **clog ( transactionHash )**
    - API /transactionStatus
        - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Transaction-status-routes
            - group
                - unconfirmed: 未承認トランザクション
                - confirmed: 承認済みトランザクション
                - failed: 失敗
            - code
                - Success: 成功
                - Failure_Core_Insufficient_Balance: 残高不足エラー
    - API /transactions/confirmed
        - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Transaction-routes/operation/searchConfirmedTransactions
            - {"meta":...,"transaction":...}
            - code
                - ResourceNotFound
    - EXPLORER mainnet
        - https://symbol.fyi/transactions/
    - EXPLORER testnet
        - https://testnet.symbol.fyi/transactions/

#### ハッシュ値検索
```js
json = await api("/transactions/confirmed/" + hash)
meta = json.meta
tx = json.transaction
console.log(meta)
console.log(tx)
```

#### トランザクションタイプ

```js
tx.type.value
sym.models.TransactionType.valueToKey(tx.type.value)
sym.models.TransactionType.TRANSFER
```

- TransactionType
    - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.models.TransactionType.html



#### タイムスタンプ
```js
meta.timestamp
new Date(epochAdjustment * 1000 + Number(meta.timestamp))
```
#### アドレス変換

```js
tx.recipientAddress
sym.Address.fromDecodedAddressHexString(tx.recipientAddress).toString()
```

#### メッセージ変換

```js
tx.message
core.utils.hexToUint8(tx.message)
core.utils.hexToUint8(tx.message).slice(1)
tdec = new TextDecoder()
tdec.decode(core.utils.hexToUint8(tx.message).slice(1))
```

### アグリゲートトランザクション
```js
alice = acnt("C9E741092833FEA79D7DB7DC839506BBEB6704631DB9B4D59C60A02BF6B0200C");
bob = newacnt();
carol = newacnt();

tx1 = trftx(bob.address,[],'tx1');
tx2 = trftx(carol.address,[],"tx2");

txes = [
    embed(tx1,alice.publicKey),
    embed(tx2,alice.publicKey)
];

aggtx = aggcptx(txes,alice.publicKey,0);
hash = await sigcosan(aggtx,alice,[])
clog(hash);
```
