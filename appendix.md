# All Scripts

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
function acnt(prikeyString){
    const prikey = new core.PrivateKey(prikeyString)
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

//モザイクトークン（hexID,数量）
function mosaic(mosaicId,amount){
    const mosaicNumber = typeof value === 'string' ? BigInt("0x" + mosaicId) : mosaicId;
    return new sym.descriptors.UnresolvedMosaicDescriptor(
        new sym.models.UnresolvedMosaicId(mosaicNumber), 
        new sym.models.Amount(amount)
    )
}

//アカウント新規作成　new account
function newacnt(){
    const prikey = core.PrivateKey.random()
    return chain.createAccount(prikey)
}

//アカウント復元 account from private key string
function acnt(prikeyString){
    const prikey = new core.PrivateKey(prikeyString)
    return chain.createAccount(prikey)
}

//テキスト暗号化 encode to hex from text string
function enchex(sendPrikey,recvPubkey,message){
    return core.utils.uint8ToHex(sendPrikey.messageEncoder().encode(recvPubkey,new TextEncoder().encode(message)))
}

//テキスト復号化 decode from hex to text string
function dechex(recvPrikey,sendPubkey,hex){

    msgDecoded = recvPrikey.messageEncoder().tryDecode(
        sendPubkey,
        core.utils.hexToUint8(hex)
    )
    const textDecoder = new TextDecoder();
    return textDecoder.decode(msgDecoded.message);
}


async function generateSalt(length) {
    const array = new Uint8Array(length);
    window.crypto.getRandomValues(array);
    return array;
}

async function deriveKey(password, salt, iterations, keyLength) {
    const enc = new TextEncoder();
    const keyMaterial = await window.crypto.subtle.importKey(
        'raw',
        enc.encode(password),
        { name: 'PBKDF2' },
        false,
        ['deriveKey']
    );
    return window.crypto.subtle.deriveKey(
        {
            name: 'PBKDF2',
            salt: salt,
            iterations: iterations,
            hash: 'SHA-512'
        },
        keyMaterial,
        { name: 'AES-GCM', length: keyLength },
        false,
        ['encrypt', 'decrypt']
    );
}

function generateIV() {
    return window.crypto.getRandomValues(new Uint8Array(12)); // AES-GCM では 12 バイトの IV を使用
}

async function enckey(account, password) {
    const secretKey = account.keyPair.privateKey
    const salt = await generateSalt(16);
    const iterations = 310000; // 推奨される最低回数
    const keyLength = 256; // 256ビット
    const derivedKey = await deriveKey(password, salt, iterations, keyLength);

    const iv = generateIV();
    const enc = new TextEncoder();
    const encrypted = await window.crypto.subtle.encrypt(
        {
            name: 'AES-GCM',
            iv: iv
        },
        derivedKey,
        enc.encode(secretKey)
    );
    return {
        salt: core.utils.uint8ToHex(salt),
        iv: core.utils.uint8ToHex(iv),
        ciphertext: core.utils.uint8ToHex(new Uint8Array(encrypted))
    };

}

async function deckey(hexSalt, hexIV, hexCiphertext, password) {

    const salt = core.utils.hexToUint8(hexSalt)
    const iv =	core.utils.hexToUint8(hexIV)
    const ciphertext = 	core.utils.hexToUint8(hexCiphertext)
    const iterations = 310000; // 推奨される最低回数
    const keyLength = 256; // 256ビット
    const derivedKey = await deriveKey(password, salt, iterations, keyLength);

    try {
        const decrypted = await window.crypto.subtle.decrypt(
            {
                name: 'AES-GCM',
                iv: iv
            },
            derivedKey,
            ciphertext
        );

        const dec = new TextDecoder();
        return dec.decode(decrypted);
    } catch (error) {
        console.error("Decryption failed:", error);
        throw new Error("Decryption failed");
    }
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

//AccountMetadataTransaction
function acntmetatx(targetAddress,key,sizeDeltaAndValueMap){
    const descriptor = new sym.descriptors.AccountMetadataTransactionV1Descriptor(
      targetAddress,  // ターゲットアドレス
      key,            // キー
      sizeDeltaAndValueMap.delta,      // サイズ差分
      sizeDeltaAndValueMap.value           // 値
    )
    return descriptor
}

//MosaicMetadataTransaction
function mosmetatx(targetAddress,key,targetMosaic,sizeDeltaAndValueMap){
    const descriptor = new sym.descriptors.MosaicMetadataTransactionV1Descriptor(
        targetAddress,  // ターゲットアドレス
        key,            // キー
        new sym.models.UnresolvedMosaicId(BigInt(`0x${targetMosaic}`)),  // メタデータ記録先モザイク
        sizeDeltaAndValueMap.delta,      // サイズ差分
        sizeDeltaAndValueMap.value           // 値
    )
    return descriptor
}

//NamespaceMetadataTransaction
function nsmetatx(targetAddress,key,targetNamespace,sizeDeltaAndValueMap){
console.log("nsmetatx")
    const descriptor = new sym.descriptors.NamespaceMetadataTransactionV1Descriptor(
        targetAddress,  // ターゲットアドレス
        key,            // キー
        new sym.models.NamespaceId(BigInt(`0x${targetNamespace}`)),  // メタデータ記録先モザイク
        sizeDeltaAndValueMap.delta,      // サイズ差分
        sizeDeltaAndValueMap.value           // 値
    )
    return descriptor
}

function updvalue(info,value){

    const encodedValue = new TextEncoder().encode(value)
    const delta = encodedValue.length
    const res = {}
    if (info.data.length > 0) {
        res.delta = delta - info.data[0].metadataEntry.valueSize
        res.value = sym.metadataUpdateValue(core.utils.hexToUint8(info.data[0].metadataEntry.value), encodedValue )
    }else{
        res.delta = delta
        res.value = encodedValue
    }
    return res
}

function metaquery(addressOrId,sourceAddress,scopedMetadataKey,metadataType){
    let query;
    if(metadataType === 0){

        //Address
        query = new URLSearchParams({
          "targetAddress": addressOrId.toString(),
          "sourceAddress": sourceAddress.toString(),
          "scopedMetadataKey": scopedMetadataKey.toString(16).toUpperCase(),
          "metadataType": metadataType
        });
    }else{

        //Mosaic or Namespace
        query = new URLSearchParams({
            "targetId": addressOrId.toString(16).toUpperCase(),
            "sourceAddress": sourceAddress.toString(),
            "scopedMetadataKey": scopedMetadataKey.toString(16).toUpperCase(),
            "metadataType": metadataType
        });
    }
    return query;
}

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

function msigtx(minRemoval,minApproval,addressAdditions,addressDeletions){
    descriptor = new sym.descriptors.MultisigAccountModificationTransactionV1Descriptor(  // Txタイプ:マルチシグ設定Tx
      minRemoval,  // minRemoval:除名のために必要な最小署名者数増分
      minApproval,  // minApproval:承認のために必要な最小署名者数増分
      addressAdditions,
      addressDeletions  // 除名対象アドレスリスト
    );
    return descriptor
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

uid = "";
funcMap = {};

function addCallback(channel, callback){
  if (!funcMap.hasOwnProperty(channel)) {
    funcMap[channel] = [];
  }
  funcMap[channel].push(callback);
};

function connectWebSocket(targetNode) {
    return new Promise((resolve, reject) => {
        const wsEndpoint = targetNode.replace("http", "ws") + "/ws";
        const socket = new WebSocket(wsEndpoint);

        socket.onmessage = function (event) {
            const response = JSON.parse(event.data);
            if(response.uid){
                uid = response.uid
                resolve({socket:socket,uid:response.uid});
            }else{
                if (funcMap.hasOwnProperty(response.topic)) {
                    funcMap[response.topic].forEach(f => {
                        f(response.data);
                    });
                }
            }
        };

        socket.onclose = async function () {
            uid = "";
            funcMap = {};
            console.log("WebSocket connection closed.");
        };
        socket.onopen = function () {console.log("WebSocket connected");};
        socket.onerror = function () {reject(new Error("Failed to connect to the WebSocket"));};
    });
}

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

function sig(aggregateTx,signer){

    const signature = signer.signTransaction(aggregateTx);
    const requestBody = sym.SymbolTransactionFactory.attachSignature(aggregateTx, signature);
    const hash = chain.hashTransaction(aggregateTx).toString();
    return {request:JSON.parse(requestBody),hash:hash}
}

bob = newacnt()
console.log(`bob address: ${bob.address}`)
console.log(`bob private key: ${bob.keyPair.privateKey}`)

function hexToUint8(hex) {
  if (hex.length % 2 !== 0) {
    throw new Error('Hex string must have an even length');
  }
  const byteArray = new Uint8Array(hex.length / 2);
  for (let i = 0; i < hex.length; i += 2) {
    byteArray[i / 2] = parseInt(hex.slice(i, i + 2), 16);
  }
  return byteArray;
}

// Uint8ArrayをHEX文字列に変換するヘルパー関数
function uint8ToHex(uint8Array) {
  return Array.from(uint8Array)
    .map(byte => byte.toString(16).padStart(2, '0'))
    .join('').toUpperCase();
}

sha3_256 = (await import('https://cdn.skypack.dev/@noble/hashes/sha3')).sha3_256;

//葉のハッシュ値取得関数
function getLeafHash(encodedPath, leafValue){
    const hasher = sha3_256.create();
    return uint8ToHex(hasher.update(hexToUint8(encodedPath + leafValue)).digest());
}

//枝のハッシュ値取得関数
function getBranchHash(encodedPath, links){
    const branchLinks = Array(16).fill(uint8ToHex(new Uint8Array(32)));
    links.forEach((link) => {
        branchLinks[parseInt(`0x${link.bit}`, 16)] = link.link;
    });
    const hasher = sha3_256.create();
    const bHash = uint8ToHex(hasher.update(hexToUint8(encodedPath + branchLinks.join(''))).digest());
    return bHash;
}

//ワールドステートの検証
function checkState(stateProof,stateHash,pathHash,rootHash){
  merkleLeaf = undefined;
  merkleBranches = [];
  stateProof.tree.forEach(n => {
    if (n.type === 255) {
      merkleLeaf = n;
    } else {
      merkleBranches.push(n);
    }
  });
  merkleBranches.reverse();

  const leafHash = getLeafHash(merkleLeaf.encodedPath,stateHash);

  let linkHash = leafHash; //最初のlinkHashはleafHash
  let bit="";
  for(let i = 0; i < merkleBranches.length; i++){
      const branch = merkleBranches[i];
      const branchLink = branch.links.find(x=>x.link === linkHash)
      linkHash = getBranchHash(branch.encodedPath,branch.links);
      bit = merkleBranches[i].path.slice(0,merkleBranches[i].nibbleCount) + branchLink.bit + bit ;
  }

  const treeRootHash = linkHash; //最後のlinkHashはrootHash
  let treePathHash = bit + merkleLeaf.path;

  if(treePathHash.length % 2 == 1){
    treePathHash = treePathHash.slice( 0, -1 );
  }
 
  //検証
  console.log(treeRootHash === rootHash);
  console.log(treePathHash === pathHash);
}


alice = acnt("C9E741092833FEA79D7DB7DC839506BBEB6704631DB9B4D59C60A02BF6B0200C");
console.log("Good Luck!");

```
