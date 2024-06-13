# スクリプト
```js
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

```

# 演習

### アカウント生成
```js
alice = newacnt()
```
- newacnt ( )
- SymbolAccount
    - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.SymbolAccount.html

### アドレス表示
```js
alice.address.toString()
`${alice.address}`
```
- Address
    - https://symbol.github.io/symbol/sdk/javascript/classes/nem.models.Address.html

### 秘密鍵表示
```js
alice.keyPair.privateKey.toString()
`${alice.keyPair.privateKey}`
```
- KeyPair
    - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.KeyPair.html

- PrivateKey
    - https://symbol.github.io/symbol/sdk/javascript/classes/index.PrivateKey.html

### 再作成
```js
alice = newacnt()
console.log(alice.address.toString())
console.log(alice.keyPair.privateKey.toString())
```

### アカウント復元
```js
alice = acnt("1BEE48A5F3E34BF5AD55072106C1D2CE2D28C1D6926EE824E010C745AC3B9EF3")
alice.address.toString()
alice.keyPair.privateKey.toString()
```
- acnt ( privateKeyString )

### アカウント検索
```js
bob = await acnt("C9E741092833FEA79D7DB7DC839506BBEB6704631DB9B4D59C60A02BF6B0200C")
json = await api("/accounts/" + bob.address)
json.account.mosaics
```
- await api ( path, method, body )
- Get account information
    - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Account-routes/operation/getAccountInfo

### テキスト暗号化
```js
code = enchex(alice,bob.publicKey,"thisisjustatest")
dechex(bob,alice.publicKey,code)
```
- enchex ( sendPrikey, recvPubkey, message )
- dechex ( recvPrikey, sendPubkey, hex)

- MessageEncoder
    - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.MessageEncoder.html

### 鍵暗号化

```js
cipher = await enckey(alice, password);
cipher.salt
cipher.iv
cipher.ciphertext

plain = await deckey(
    cipher.salt,
    cipher.iv,
    cipher.ciphertext,
    password
);
```

- await enckey ( secretKey, password)
- await deckey ( hexSalt, hexIV, hexCiphertext, password )
