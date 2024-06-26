# 13.検証

Symbolは、ブロックに関連する大量のデータを保存するためにツリー構造を使用しています。このデータはブロックヘッダーから直接取得することはできません。これにより、軽量クライアントが台帳全体の履歴を要求することなく、要素（例：トランザクション、レシートステートメント）が存在するかどうかを検証することができます。

https://docs.symbol.dev/concepts/data-validation.html

## スクリプト
```js
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
  console.log("ルート検証：");
  console.log(treeRootHash === rootHash);
  console.log("パス検証：");
  console.log(treePathHash === pathHash);

}

console.log("Import Verify Script")
```
## 演習

### トランザクションの検証

入手したペイロードがトランザクションとしてブロックチェーンに記録されていることを確認

#### 署名検証
```js
height = 686312;
tx = sym.SymbolTransactionFactory.deserialize(u.hexToUint8("2802000000000000A5151FD55D82351DD488DB5563DD328DA72B2AD25B513C1D0F7F78AFF4D35BA094ABF505C74E6D6BE1FA19F3E5AC60A85E1A4EDC4AC07DECC0E56C59D5D24F0B69A31A837EB7DE323F08CA52495A57BA0A95B52D1BB54CEA9A94C12A87B1CADB0000000002984141A0D70000000000000EEAD6810500000062E78B6170628861B4FC4FCA75210352ACDBD2378AC0A447A3DCF63F969366BB1801000000000000540000000000000069A31A837EB7DE323F08CA52495A57BA0A95B52D1BB54CEA9A94C12A87B1CADB000000000198544198A8D76FEF8382274D472EE377F2FF3393E5B62C08B4329D04000000000000000074783100000000590000000000000069A31A837EB7DE323F08CA52495A57BA0A95B52D1BB54CEA9A94C12A87B1CADB000000000198444198A8D76FEF8382274D472EE377F2FF3393E5B62C08B4329D6668A0DE72812AAE05000500746573743100000000000000590000000000000069A31A837EB7DE323F08CA52495A57BA0A95B52D1BB54CEA9A94C12A87B1CADB000000000198444198A8D76FEF8382274D472EE377F2FF3393E5B62C08B4329DBF85DADBFD54C48D050005007465737432000000000000000000000000000000662CEDF69962B1E0F1BF0C43A510DFB12190128B90F7FE9BA48B1249E8E10DBEEDD3B8A0555B4237505E3E0822B74BCBED8AA3663022413AFDA265BE1C55431ACAE3EA975AF6FD61DEFFA6A16CBA5174A16EF5553AE669D5803A0FA9D1424600"));

res = chain.verifyTransaction(tx, tx.signature);
```
##### SDK
- SymbolTransactionFactory.deserialize
  - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.SymbolTransactionFactory.html#deserialize
- verifyTransaction
  - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.SymbolFacade.html#verifyTransaction

#### マークルハッシュ値の計算
```js
mkhash = chain.hashTransaction(tx);
if (tx.cosignatures !== undefined && tx.cosignatures.length > 0) {
  hasher = sha3_256.create();
  hasher.update(mkhash.bytes);
  for (cos of tx.cosignatures){
    hasher.update(cos.signerPublicKey.bytes);
  }
  mkhash = u.uint8ToHex(hasher.digest());
}
console.log(mkhash);
```

#### 検証
```js
//トランザクションから計算
leaf = new core.Hash256(mkhash);

//ブロックチェーンから取得
binfo = await api("/blocks/" + height)
HRoot = new core.Hash256(binfo.block.transactionsHash)

minfo = await api(`/blocks/${height}/transactions/${leaf}/merkle`)
let paths = [];
minfo.merklePath.forEach(path => paths.push({
    "hash": new core.Hash256(path.hash),
    "isLeft": path.position === "left"
}));
merkleProof = paths;

result = sym.proveMerkle(leaf, merkleProof, HRoot);
console.log(result);
```
##### API
- /blocks/{height}/transactions/{leaf}/merkle
  - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Block-routes/operation/getMerkleTransaction

##### SDK
- Hash256
  - https://symbol.github.io/symbol/sdk/javascript/classes/symbol.models.Hash256.html
- proveMerkle
  - https://symbol.github.io/symbol/sdk/javascript/functions/symbol.proveMerkle.html

### ブロックヘッダーの検証
##### 事前準備
```js
info = await api("/blocks/" + height)
block = info.block;
```
#### 署名検証
```js
buf = new Uint8Array([
...new Uint8Array([block.version]),
...new Uint8Array([block.network]),
...new Uint8Array((new Uint16Array([block.type])).buffer),
...new Uint8Array((new BigInt64Array([BigInt(block.height)])).buffer),
...new Uint8Array((new BigInt64Array([BigInt(block.timestamp)])).buffer),
...new Uint8Array((new BigInt64Array([BigInt(block.difficulty)])).buffer),
...hexToUint8(block.proofGamma),
...hexToUint8(block.proofVerificationHash),
...hexToUint8(block.proofScalar),
...hexToUint8(block.previousBlockHash),
...hexToUint8(block.transactionsHash),
...hexToUint8(block.receiptsHash),
...hexToUint8(block.stateHash),
...hexToUint8(block.beneficiaryAddress),
...new Uint8Array((new Uint32Array([block.feeMultiplier])).buffer)
]);

if(block.type === 33347){//importance block

	buf = new Uint8Array([
		...buf,
	    ...new Uint8Array((new Uint32Array([block.votingEligibleAccountsCount])).buffer), 
	    ...new Uint8Array((new BigInt64Array([BigInt(block.harvestingEligibleAccountsCount)])).buffer),
	    ...new Uint8Array((new BigInt64Array([BigInt(block.totalVotingBalance)])).buffer),
	    ...hexToUint8(block.previousImportanceBlockHash)
	]);
}

v = new sym.Verifier(new core.PublicKey(block.signerPublicKey))
v.verify(buf,new core.Signature(block.signature))
```

##### SDK
- Verifier
	- https://symbol.github.io/symbol/sdk/javascript/classes/symbol.Verifier.html

#### チェーン検証
```js
hasher = sha3_256.create();
hasher.update(hexToUint8(block.signature)); //signature
hasher.update(hexToUint8(block.signerPublicKey)); //publicKey
hasher.update(new Uint8Array([block.version]));
hasher.update(new Uint8Array([block.network]));
hasher.update(new Uint8Array((new Uint16Array([block.type])).buffer));
hasher.update(new Uint8Array((new BigInt64Array([BigInt(block.height)])).buffer));
hasher.update(new Uint8Array((new BigInt64Array([BigInt(block.timestamp)])).buffer));
hasher.update(new Uint8Array((new BigInt64Array([BigInt(block.difficulty)])).buffer));
hasher.update(hexToUint8(block.proofGamma));
hasher.update(hexToUint8(block.proofVerificationHash));
hasher.update(hexToUint8(block.proofScalar));
hasher.update(hexToUint8(block.previousBlockHash));
hasher.update(hexToUint8(block.transactionsHash));
hasher.update(hexToUint8(block.receiptsHash));
hasher.update(hexToUint8(block.stateHash));
hasher.update(hexToUint8(block.beneficiaryAddress));
hasher.update(new Uint8Array((new Uint32Array([block.feeMultiplier])).buffer)); 
if(block.type === 33347){//importance block

    hasher.update(new Uint8Array((new Uint32Array([block.votingEligibleAccountsCount])).buffer)); 
    hasher.update(new Uint8Array((new BigInt64Array([BigInt(block.harvestingEligibleAccountsCount)])).buffer));
    hasher.update(new Uint8Array((new BigInt64Array([BigInt(block.totalVotingBalance)])).buffer));
    hasher.update(hexToUint8(block.previousImportanceBlockHash));
}

hash = uint8ToHex(hasher.digest());
console.log(hash);

pinfo = await api("/blocks/" + (height-1));
block.previousBlockHash === pinfo.meta.hash;
```

##### API
- /blocks/{height}
  - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Block-routes/operation/getBlockByHeight
 
### ステートハッシュの検証
```js
hasher = sha3_256.create();
hasher.update(hexToUint8(info.meta.stateHashSubCacheMerkleRoots[0])); //AccountState
hasher.update(hexToUint8(info.meta.stateHashSubCacheMerkleRoots[1])); //Namespace
hasher.update(hexToUint8(info.meta.stateHashSubCacheMerkleRoots[2])); //Mosaic
hasher.update(hexToUint8(info.meta.stateHashSubCacheMerkleRoots[3])); //Multisig
hasher.update(hexToUint8(info.meta.stateHashSubCacheMerkleRoots[4])); //HashLockInfo
hasher.update(hexToUint8(info.meta.stateHashSubCacheMerkleRoots[5])); //SecretLockInfo
hasher.update(hexToUint8(info.meta.stateHashSubCacheMerkleRoots[6])); //AccountRestriction
hasher.update(hexToUint8(info.meta.stateHashSubCacheMerkleRoots[7])); //MosaicRestriction
hasher.update(hexToUint8(info.meta.stateHashSubCacheMerkleRoots[8])); //Metadata
hash = uint8ToHex(hasher.digest()).toUpperCase();
console.log(block.stateHash)
block.stateHash === hash
```

### アカウントの検証

#### 前準備
```js
aliceAddress = new sym.Address(
  hexToUint8("9850BF0FD1A45FCEE211B57D0FE2B6421EB81979814F6292")
);

//hasher = sha3_256.create();
info = await api('/accounts/' + aliceAddress.toString())
aliceInfo = info.account
```

#### 検証
```js
format = (parseInt(aliceInfo.importance) === 0 || aliceInfo.activityBuckets.length < 5) ? 0x00 : 0x01;
supplementalPublicKeysMask = 0x00;
extractPublicKey = (key) => key ? hexToUint8(key.publicKey) : new Uint8Array([]);
linkedPublicKey = extractPublicKey(aliceInfo.supplementalPublicKeys.linked);
nodePublicKey = extractPublicKey(aliceInfo.supplementalPublicKeys.node);
vrfPublicKey = extractPublicKey(aliceInfo.supplementalPublicKeys.vrf);
if (aliceInfo.supplementalPublicKeys.linked) supplementalPublicKeysMask |= 0x01;
if (aliceInfo.supplementalPublicKeys.node) supplementalPublicKeysMask |= 0x02;
if (aliceInfo.supplementalPublicKeys.vrf) supplementalPublicKeysMask |= 0x04;
votingPublicKeys = new Uint8Array([]);
if (aliceInfo.supplementalPublicKeys.voting) {
  aliceInfo.supplementalPublicKeys.voting.publicKeys.forEach(key => {
    votingPublicKeys = new Uint8Array([...votingPublicKeys, ...hexToUint8(key.publicKey)]);
  });
}
importanceSnapshots = new Uint8Array([]);
if (parseInt(aliceInfo.importance) !== 0) {
  const importance = new Uint8Array((new BigInt64Array([BigInt(aliceInfo.importance)])).buffer);
  const importanceHeight = new Uint8Array((new BigInt64Array([BigInt(aliceInfo.importanceHeight)])).buffer);
  importanceSnapshots = new Uint8Array([...importance, ...importanceHeight]);
}
activityBuckets = new Uint8Array([]);
if (aliceInfo.importance > 0) {
  aliceInfo.activityBuckets.slice(0, 5).forEach(bucket => {
    const startHeight = new Uint8Array((new BigInt64Array([BigInt(bucket.startHeight)])).buffer);
    const totalFeesPaid = new Uint8Array((new BigInt64Array([BigInt(bucket.totalFeesPaid)])).buffer);
    const beneficiaryCount = new Uint8Array((new BigInt64Array([BigInt(bucket.beneficiaryCount)])).buffer);
    const rawScore = new Uint8Array((new BigInt64Array([BigInt(bucket.rawScore)])).buffer);
    activityBuckets = new Uint8Array([...activityBuckets, ...startHeight, ...totalFeesPaid, ...beneficiaryCount, ...rawScore]);
  });
}
balances = new Uint8Array([]);
if (aliceInfo.mosaics.length > 0) {
  aliceInfo.mosaics.forEach(mosaic => {
    const mosaicId = hexToUint8(mosaic.id).reverse();
    const amount = new Uint8Array((new BigInt64Array([BigInt(mosaic.amount)])).buffer);
    balances = new Uint8Array([...balances, ...mosaicId, ...amount]);
  });
}

accountInfoBytes = new Uint8Array([
  ...new Uint8Array((new Uint16Array([aliceInfo.version])).buffer),
  ...hexToUint8(aliceInfo.address),
  ...new Uint8Array((new BigInt64Array([BigInt(aliceInfo.addressHeight)])).buffer),
  ...hexToUint8(aliceInfo.publicKey),
  ...new Uint8Array((new BigInt64Array([BigInt(aliceInfo.publicKeyHeight)])).buffer),
  ...new Uint8Array([aliceInfo.accountType]),
  ...new Uint8Array([format]),
  ...new Uint8Array([supplementalPublicKeysMask]),
  ...new Uint8Array([votingPublicKeys.length]),
  ...linkedPublicKey, ...nodePublicKey, ...vrfPublicKey, ...votingPublicKeys,
  ...importanceSnapshots, ...activityBuckets,
  ...new Uint8Array((new Uint16Array([aliceInfo.mosaics.length])).buffer), ...balances
]);

hasher = sha3_256.create();
aliceStateHash = uint8ToHex(hasher.update(accountInfoBytes).digest());

hasher = sha3_256.create();
alicePathHash = uint8ToHex(hasher.update(aliceAddress.bytes).digest());

blockInfo = await api("/blocks?order=desc");
rootHash = blockInfo.data[0].meta.stateHashSubCacheMerkleRoots[0];

stateProof = await api(`/accounts/${aliceAddress}/merkle` )

checkState(stateProof,aliceStateHash,alicePathHash,rootHash);
```

##### Script
- checkState ( stateProof, stateHash, pathHash, rootHash )

##### API
- /accounts/{accountId}/merkle
  - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Account-routes/operation/getAccountInfoMerkle

### メタデータの検証

- 検証対象
  - https://testnet.symbol.fyi/accounts/TA6EAATQMNJM4QV4ARM3DVWGSZULWBDB7HGEVPY


```js
srcAddress = (new sym.Address("TA6EAATQMNJM4QV4ARM3DVWGSZULWBDB7HGEVPY")).bytes;
targetAddress = (new sym.Address("TA6EAATQMNJM4QV4ARM3DVWGSZULWBDB7HGEVPY")).bytes;

//compositePathHash(Key値)
hasher = sha3_256.create();    
hasher.update(srcAddress);
hasher.update(targetAddress);
hasher.update(hexToUint8("9772B71B058127D7").reverse()); // scopeKey, メタデータキーを指定
hasher.update(hexToUint8("0000000000000000").reverse()); // targetId
hasher.update(Uint8Array.from([0])); // type: Account 0
compositeHash = uint8ToHex(hasher.digest());

hasher = sha3_256.create();
bytes = new Uint8Array(hexToUint8(compositeHash));
pathHash = uint8ToHex(hasher.update(bytes).digest());

//stateHash(Value値)
hasher = sha3_256.create();
version = 1;

hasher.update(new Uint8Array((new Uint16Array([version])).buffer)); //version
hasher.update(srcAddress);
hasher.update(targetAddress);
hasher.update(hexToUint8("9772B71B058127D7").reverse());  // scopeKey, メタデータキーを指定
hasher.update(hexToUint8("0000000000000000").reverse());  // targetId
hasher.update(Uint8Array.from([0]));                      // account
value = new TextEncoder().encode("test");          // メタデータの値を指定
hasher.update(new Uint8Array((new Uint16Array([value.length])).buffer));
hasher.update(value); 
stateHash = uint8ToHex(hasher.digest());

blockInfo = await api("/blocks?order=desc");
rootHash = blockInfo.data[0].meta.stateHashSubCacheMerkleRoots[8];

hasher = sha3_256.create();
secretStateHash = uint8ToHex(hasher.update(bytes).digest());
stateProof = await api(`/metadata/${compositeHash}/merkle` )

//検証
checkState(stateProof,stateHash,pathHash,rootHash);
```
##### Script
- checkState ( stateProof, stateHash, pathHash, rootHash )

##### API
- /metadata/{compositeHash}/merkle
  - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Metadata-routes/operation/getMetadataMerkle


### シークレットロックの検証

#### 事前準備
[8.ロック](08_lock.md)のスクリプトを事前に実行しておいてください。

```js
proof = crypto.getRandomValues(new Uint8Array(20))
hash = sha3_256.create();
hash.update(proof);
secret = hash.digest();
//SecretLockTransaction
tx = slocktx(alice.address,secret,mosaic(xymhex,1000000))
hash = await sigan(tx,alice);
clog(hash)

info = await api("/lock/secret?address=" + alice.address)
linfo = info.data[0].lock
```

#### 検証
```js
bytes = new Uint8Array([
...hexToUint8(linfo.secret),
...hexToUint8(linfo.recipientAddress)
]);

hasher = sha3_256.create();
compositeHash = uint8ToHex(hasher.update(bytes).digest());

bytes = new Uint8Array([
...new Uint8Array((new Uint16Array([linfo.version])).buffer),
...hexToUint8(linfo.ownerAddress),
...hexToUint8(linfo.mosaicId).reverse(),
...new Uint8Array((new BigInt64Array([BigInt(linfo.amount)])).buffer),
...new Uint8Array((new BigInt64Array([BigInt(linfo.endHeight)])).buffer),
...new Uint8Array([linfo.status]),
...new Uint8Array([linfo.hashAlgorithm]),
...hexToUint8(linfo.secret),
...hexToUint8(linfo.recipientAddress)
]);

hasher = sha3_256.create();
secretStateHash = uint8ToHex(hasher.update(bytes).digest());

hasher = sha3_256.create();
bytes = new Uint8Array(hexToUint8(compositeHash));
secretPathHash = uint8ToHex(hasher.update(bytes).digest());

blockInfo = await api("/blocks?order=desc");
rootHash = blockInfo.data[0].meta.stateHashSubCacheMerkleRoots[5];

stateProof = await api(`/lock/secret/${compositeHash}/merkle` )

checkState(stateProof,secretStateHash,secretPathHash,rootHash);
```
##### Script
- checkState ( stateProof, stateHash, pathHash, rootHash )

##### API
- /lock/secret/{compositeHash}/merkle
  - https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Secret-Lock-routes/operation/getSecretLockMerkle
