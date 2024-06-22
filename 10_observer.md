# 10.監視

ブロックチェーンでイベントが発生した際にリアルタイム更新を取得するために、SymbolではWebSocketが公開されています。クライアントアプリケーションはWebSocket接続を開き、一意の識別子を取得できます。
この識別子を使用すると、アプリケーションは更新を取得するためにAPIを常時ポーリングする代わりに、利用可能なチャンネルにサブスクライブする資格を得ます。イベントが発生すると、RESTゲートウェイはリアルタイムでサブスクライブしているすべてのクライアントにチャンネルを通じて通知を送信します。
WebSocketのURIはHTTPリクエストのURIと同じホストとポートを共有しますが、ws://プロトコルを使用します。

https://docs.symbol.dev/api.html#websockets

## スクリプト
```js
uid = "";
funcMap = {};

function addcb(channel, callback){
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

function wssend(uid,channel){
  ws.socket.send(JSON.stringify({ uid: uid, subscribe: channel }));

}

console.log("Import Observer Script")
```

### 接続
```js
ws = await connectWebSocket(node);
```
##### Script
- await connectWebSocket ( targetNode )


### ブロック生成
```js
addcb("block", e => console.log(e) )
wssend(uid,"block")
```

##### Script
- addcb ( channel, callback )
- wssend ( uid, channel )

##### API
- channels : block
  - https://docs.symbol.dev/api.html#channels


### 承認トランザクション
```js
//tab2
bob = newacnt()
bob.address.toString()

ch2 = "confirmedAdded/" + bob.address
addcb(ch2 , e => console.log(e) )
wssend(uid,ch2)

//tab1
ch1 = "confirmedAdded/" + alice.address
addcb(ch1 , e => console.log(e) )
wssend(uid,ch1)

bobadr = new sym.Address('')
tx = trftx(bobadr,[],'hello');
hash = await sigan(tx,alice);
clog(hash);
```

##### Script
- addcb ( channel, callback )
- wssend ( uid, channel )

##### API
- channels : confirmedAdded
  - https://docs.symbol.dev/api.html#channels

### 署名要求

[8.ロック](08_lock.md) のスクリプトを実行しておいてください

```js
//tab2
ch2 = "partialAdded/" + bob.address
addcb(ch2 , e => console.log(e) )
wssend(uid,ch2)
bob.publicKey.toString()


//tab1
ch1 = "partialAdded/" + alice.address
addcb(ch1 , e => console.log(e) )
wssend(uid,ch1)

bob = new sym.SymbolPublicAccount(
  chain,
  new core.PublicKey("")
)
tx1 = trftx(alice.address,[],'')
tx2 = trftx(bob.address,[],'')
txes = [
  embed(tx1,bob.publicKey),
  embed(tx2,alice.publicKey)
]
//AggregateBondedTransaction
aggtx = aggbdtx(txes,alice.publicKey,1); //起案者
sigedtx = sig(aggtx,alice)

ch3 = "confirmedAdded/" + alice.address
addcb(ch3 , async e => {
  console.log(e)
  if(e.transaction.type === 16712){
    res = await api("/transactions/partial","PUT",sigedtx.request)
    clog(sigedtx.hash)
  }
})
wssend(uid,ch3)

//HashLockTransaction
hltx = hlocktx(sigedtx.hash);
hlhash = await sigan(hltx,alice);
clog(hlhash);
```

##### Script
- addcb ( channel, callback )
- wssend ( uid, channel )

##### API
- channels : partialAdded
  - https://docs.symbol.dev/api.html#channels
