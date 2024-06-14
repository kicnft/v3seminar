# 10.監視

ブロックチェーンでイベントが発生した際にリアルタイム更新を取得するために、SymbolではWebSocketを公開されています。クライアントアプリケーションはWebSocket接続を開き、一意の識別子を取得できます。
この識別子を使用すると、アプリケーションは更新を取得するためにAPIを常にポーリングする代わりに、利用可能なチャンネルにサブスクライブする資格を得ます。チャンネルでイベントが発生すると、RESTゲートウェイはリアルタイムでサブスクライブしているすべてのクライアントに通知を送信します。
WebSocketのURIはHTTPリクエストのURIと同じホストとポートを共有しますが、ws://プロトコルを使用します。

## 演習内容
- 接続
- ブロック生成受信
- 承認トランザクション受信
- 署名要求受信

## スクリプト
```js
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
```

### 接続
```js
ws = await connectWebSocket(node);
```
##### Script
- await connectWebSocket ( targetNode )


### ブロック生成
```js
addCallback("block", (block) => {
  console.log(block);
});
ws.socket.send(JSON.stringify({ uid: uid, subscribe: "block" }));
```

##### Script
- addCallback ( channel, callback )

##### API
- channels : block
  - https://docs.symbol.dev/api.html#channels


### 承認トランザクション
```js
//tab1
ch1 = `confirmedAdded/${alice.address}`
addCallback(ch1 , (tx) => {
  console.log(tx);
});
ws.socket.send(JSON.stringify({ uid: uid, subscribe: ch1 }));

//tab2
bob = newacnt()
ch2 = `confirmedAdded/${bob.address}`
addCallback(ch2 , (tx) => {
  console.log(tx);
});
ws.socket.send(JSON.stringify({ uid: uid, subscribe: ch2 }));

tx = trftx(bob.address,[],'hello');
hash = await sigan(tx,alice);
clog(hash);
```

##### Script
- addCallback ( channel, callback )

##### API
- channels : confirmedAdded
  - https://docs.symbol.dev/api.html#channels

### 署名要求
```js
//tab1
ch1 = `partialAdded/TCF7OVMXVNZC6GCMYBNMDSNS56CTOH6FA5H4W7I`
addCallback(ch1 , (tx) => {
  console.log(tx);
});
ws.socket.send(JSON.stringify({ uid: uid, subscribe: ch1 }));

//tab2
ch2 = `partialAdded/TBP6WMTKDVSSYYSS4LLHA6T6AMNO2XKBPU73BNY`
addCallback(ch2 , (tx) => {
  console.log(tx);
});
ws.socket.send(JSON.stringify({ uid: uid, subscribe: ch2 }));

//tab3
ch3 = `partialAdded/TCXIDAWKVRXS4XWWKXN2J5XCHXRPRUDGSTUO3BY`
addCallback(ch3 , (tx) => {
  console.log(tx);
});
ws.socket.send(JSON.stringify({ uid: uid, subscribe: ch3 }));
```

##### Script
- addCallback ( channel, callback )

##### API
- channels : partialAdded
  - https://docs.symbol.dev/api.html#channels
