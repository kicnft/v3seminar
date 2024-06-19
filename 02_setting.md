# 2.環境設定

## SDK
- Symbol-SDK
    - npm
        - https://www.npmjs.com/package/symbol-sdk
    - unpkg bundle
        - https://www.unpkg.com/symbol-sdk@3.2.2/dist/bundle.web.js

```js
bundle = await import("https://www.unpkg.com/symbol-sdk@3.2.2/dist/bundle.web.js");
```

## リファレンス
- Catapult REST Endpoints (1.0.3)
    - https://symbol.github.io/symbol-openapi/v1.0.3/
- symbol-sdk - v3.2.2
    - https://symbol.github.io/symbol/sdk/javascript/index.html

## ノード選択
- テストネット
    - https://symbolnodes.org/nodes_testnet/
- メインネット
    - https://symbolnodes.org/nodes/

## 共通スクリプト
Chromeブラウザなどで開発者コンソールを開き以下の共通スクリプトをコピーペーストしてください。
共通スクリプトで使用する関数名はSDKで提供される関数と区別しやすいように、大文字なしの省略形で構成しています。

```js
//NETWORK
node = window.origin;
json = await api("/network/properties");
generationHash = json.network.generationHashSeed;
networkId = json.network.identifier;
xymhex = json.chain.currencyMosaicId.split("'").join('').slice(2);
xymid =  BigInt(`0x${xymhex}`);
epochAdjustment = Number(json.network.epochAdjustment.replace("s",""));

json = await api("/network/fees/transaction");
feeMultiplier = json.medianFeeMultiplier;
add2Hours = 60 * 60 * 2;

netstat();

//SDK
bundle = await import("https://www.unpkg.com/symbol-sdk@3.2.2/dist/bundle.web.js");
core = bundle.core;
sym  = bundle.symbol;
chain = new sym.SymbolFacade(networkId);
m = sym.models;
d = sym.descriptors;
u = core.utils;
async function api(path,method,body){
    if(method === "PUT" || method === "POST"){

        const res = await fetch(
          node + path,
          {
            method: method,
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(body),
          }
        );
        return await res.json();
    }else{

        return await (await fetch(node + path)).json();
    }
}

//ログ出力 console log
function clog(signedHash){
    console.log(node + "/transactionStatus/" + signedHash);
    console.log(node + "/transactions/confirmed/" + signedHash);
    console.log("https://symbol.fyi/transactions/" + signedHash);
    console.log("https://testnet.symbol.fyi/transactions/" + signedHash);
}

//トランザクション　ステータス transaction status
function txstat(tx){
    console.log(`== TRANSACTION STATUS ==`);
    console.log(`[ 名称: ${tx.constructor.name} ]`);
    console.log(`[ 有効期限: ${new Date(epochAdjustment * 1000 + Number(tx.deadline))} ]`);
    const useFee = tx.size / 1000000 * 100;
    console.log(`[ 手数料係数: 100, データサイズ: ${tx.size}Byte, 想定手数料: ${useFee}XYM ]`);
    console.log(`[ 署名アドレス: ${chain.network.publicKeyToAddress(tx.signerPublicKey)} ]`);
}

//ネットワーク　ステータス　network status
function netstat(){
    console.log(`== NETWORK STATUS ==`);
    console.log(`[ node: ${node} ]`);
    console.log(`[ generationHash: ${generationHash} ]`);
    console.log(`[ networkId: ${networkId} ]`);
    console.log(`[ xymid: ${xymhex} ]`);
    console.log(`[ epochAdjustment: ${epochAdjustment} ]`);
    console.log(`[ feeMultiplier: ${feeMultiplier} ]`);
}

//アカウント　ステータス account status
async function acntstat(account){
    console.log(`== ACCOUNT STATUS ==`);
    console.log(`[アドレス: ${account.address} ]`);
    console.log(`[公開鍵: ${account.publicKey} ]`);
    try{
        json = await api(`/accounts/${account.address.toString()}`);
        console.log(`[所有モザイク: ID, 数量]`);
        for(mosaic of json.account.mosaics){
            console.log(`[ ${mosaic.id}, ${mosaic.amount} ]`);

        }
    }catch{
        console.log("ResourceNotFound")
    }
}

//アカウント新規作成　new account
function newacnt(){
    const prikey = core.PrivateKey.random()
    return chain.createAccount(prikey)
}

//アカウント復元 account from private key string
function acnt(privateKeyString){
    const prikey = new core.PrivateKey(privateKeyString)
    return chain.createAccount(prikey)
}

function decadr(hexEncodedAddress){
    return sym.Address.fromDecodedAddressHexString(hexEncodedAddress).toString()
}

//転送トランザクション transfer transaction
function trftx(address,mosaics,message){
    messageData = new Uint8Array([
        0x00, //現行アプリケーション対応 00:平文, 01:暗号化
        ...new TextEncoder().encode(message),
    ]);
    
    return new sym.descriptors.TransferTransactionV1Descriptor(address,mosaics,messageData);
}

//署名＆通知 sign and announce
async function sigan(transaction,signer){
    const tx = chain.createTransactionFromTypedDescriptor(transaction,signer.publicKey,feeMultiplier,add2Hours);
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
function embed(transaction,publicKey){
    return chain.createEmbeddedTransactionFromTypedDescriptor(transaction,publicKey);
}

//アグリゲートコンプリートトランザクション aggregate complete transaction
function aggcptx(transactions,initPublicKey,cosignatureCount){
    const transactionsHash = sym.SymbolFacade.hashEmbeddedTransactions(transactions);
    const desc = new sym.descriptors.AggregateCompleteTransactionV2Descriptor(transactionsHash,transactions,[]);
    const tx = chain.createTransactionFromTypedDescriptor(desc,initPublicKey,feeMultiplier,add2Hours,cosignatureCount);
    return tx;
}

//署名＆連署＆通知 sign and cosign and announce
async function sigcosan(aggregateTransaction,signer,cosigners){

    //署名
    const signature = signer.signTransaction(aggregateTransaction);
    sym.SymbolTransactionFactory.attachSignature(aggregateTransaction, signature);

    //連署
    for(cosigner of cosigners){
        const cosignature = cosigner.cosignTransaction(aggregateTransaction);
        tx.cosignatures.push(cosignature);
    }
    txstat(aggregateTransaction);

    //通知
    const requestBody = sym.SymbolTransactionFactory.attachSignature(aggregateTransaction, aggregateTransaction.signature);
    const hash = chain.hashTransaction(aggregateTransaction).toString();
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

//モザイクトークン（hexID,数量）
function mosaic(mosaicId,amount){
    const mosaicNumber = typeof mosaicId === 'string' ? BigInt("0x" + mosaicId) : mosaicId;
    return new sym.descriptors.UnresolvedMosaicDescriptor(
        new sym.models.UnresolvedMosaicId(mosaicNumber), 
        new sym.models.Amount(amount)
    )
}

alice = acnt("C9E741092833FEA79D7DB7DC839506BBEB6704631DB9B4D59C60A02BF6B0200C");
console.log("Good Luck!");
```



