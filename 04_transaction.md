# 4.トランザクション

トランザクションは、一般的にデータベースシステム内の作業単位を表します。ブロックチェーンの場合、アカウントの署名やネットワークへの通知操作によって、ブロックチェーンの状態が変化することをさします。

https://docs.symbol.dev/concepts/transaction.html

## スクリプト

3.アカウントの章で作成したアカウントを復元してください。
```js
alice = acnt("C9E741092833FEA79D7DB7DC839506BBEB6704631DB9B4D59C60A02BF6B0200C");
"https://testnet.symbol.tools?recipient=" + alice.address
"https://testnet.symbol.fyi/accounts/" + alice.address

bob = newacnt();
```

> [!NOTE]
> 以後の章でも演習に入る前に同様の手順でアカウントを復元しておいてください

## 演習


### 転送トランザクション
```js
//TransferTransaction
tx = trftx(bob.address,[],'hello');
hash = await sigan(tx,alice);
clog(hash);
```

##### Script
- trftx ( address, mosaics, message )
- await sigan ( desc, signer )
- clog ( transactionHash )

##### API
- /transactionStatus
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Transaction-status-routes/operation/getTransactionStatus
        - group
            - unconfirmed: 未承認トランザクション
            - confirmed: 承認済みトランザクション
            - failed: 失敗
        - code
            - Success: 成功
            - Failure_Core_Insufficient_Balance: 残高不足エラー
- /transactions/confirmed/{transactionId}
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Transaction-routes/operation/getConfirmedTransaction
        - {"meta":...,"transaction":...}
        - code
            - ResourceNotFound

### トランザクション検索
#### ハッシュ値で検索
```js
info = await api("/transactions/confirmed/" + hash)
meta = info.meta
tx = info.transaction
```

#### トランザクションタイプ
```js
tx.type.value
sym.models.TransactionType.valueToKey(tx.type.value)
sym.models.TransactionType.TRANSFER
```
##### SDK
- TransactionType
    - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.models.TransactionType.html

#### タイムスタンプ
```js
meta.timestamp
new Date(epochAdjustment * 1000)
new Date(epochAdjustment * 1000 + Number(meta.timestamp))
```
##### Script
- epochAdjustment

#### メッセージ変換
```js
tx.message
u.hexToUint8(tx.message)
u.hexToUint8(tx.message).slice(1)
td = new TextDecoder()
td.decode(u.hexToUint8(txinfo.message).slice(1))
```
##### SDK
- utils.hexToUint8
  - https://symbol.github.io/symbol/sdk/javascript/functions/index.utils.hexToUint8.html 

### アグリゲートトランザクション
```js
alice = acnt("C9E741092833FEA79D7DB7DC839506BBEB6704631DB9B4D59C60A02BF6B0200C");
bob = newacnt();
carol = newacnt();

//TransferTransaction
tx1 = trftx(bob.address,[],'tx1');
tx2 = trftx(carol.address,[],"tx2");

txes = [
    embed(tx1,alice.publicKey),
    embed(tx2,alice.publicKey)
];

//AggregateCompleteTransaction
aggtx = aggcptx(txes,alice.publicKey,0);
hash = await sigcosan(aggtx,alice,[])
clog(hash);
```

##### Script
- embed( transaction, publicKey)
- aggcptx( transactions[], initPublicKey, cosignaturesCount )
- await sigcosan( transaction, signer, cosigners[] )
