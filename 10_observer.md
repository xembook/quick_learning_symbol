# 10.監視
SymbolのノードはWebSocket通信でブロックチェーンの状態変化を監視することが可能です。  

## 10.1 リスナー設定

WebSocketを生成してリスナーの設定を行います。

```js
nsRepo = repo.createNamespaceRepository();
wsEndpoint = NODE.replace('http', 'ws') + "/ws";
listener = new sym.Listener(wsEndpoint,nsRepo,WebSocket);
listener.open();
```

エンドポイントのフォーマットは以下の通りです。
- wss://{node url}:3001/ws

何も通信が無ければ、listenerは1分で切断されます。

## 10.2 受信検知

アカウントが受信したトランザクションを検知します。

```js
listener.open().then(() => {

    //承認トランザクションの検知
    listener.confirmed(alice.address)
    .subscribe(tx=>{
        //受信後の処理を記述
        console.log(tx);
    });

    //未承認トランザクションの検知
    listener.unconfirmedAdded(alice.address)
    .subscribe(tx=>{
        //受信後の処理を記述
        console.log(tx);
    });
});
```
上記リスナーを実行後、aliceへの送信トランザクションをアナウンスしてください。

###### 出力例
```js
> Promise {<pending>}
> TransferTransaction {type: 16724, networkType: 152, version: 1, deadline: Deadline, maxFee: UInt64, …}
    deadline: Deadline {adjustedValue: 12449258375}
    maxFee: UInt64 {lower: 32000, higher: 0}
    message: RawMessage {type: -1, payload: ''}
    mosaics: []
    networkType: 152
    payloadSize: undefined
    recipientAddress: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
    signature: "914B625F3013635FA9C99B2F138C47CD75F6E1DF7BDDA291E449390178EB461AA389522FA126D506405163CC8BA51FA9019E0522E3FA9FED7C2F857F11FBCC09"
    signer: PublicAccount {publicKey: 'D4933FC1E4C56F9DF9314E9E0533173E1AB727BDB2A04B59F048124E93BEFBD2', address: Address}
    transactionInfo: TransactionInfo
        hash: "3B21D8842EB70A780A662CCA19B8B030E2D5C7FB4C54BDA8B3C3760F0B35FECE"
        height: UInt64 {lower: 316771, higher: 0}
        id: undefined
        index: undefined
        merkleComponentHash: "3B21D8842EB70A780A662CCA19B8B030E2D5C7FB4C54BDA8B3C3760F0B35FECE"
    type: 16724
    version: 1
```

未承認トランザクションは transactionInfo.height=0　で受信します。

## 10.3 ブロック監視

新規に生成されたブロックを検知します。

```js
listener.open().then(() => {

    //ブロック生成の検知
    listener.newBlock()
    .subscribe(block=>console.log(block));
});
```
###### 出力例
```js
> Promise {<pending>}
> NewBlock
    beneficiaryAddress: Address {address: 'TAKATV2VSYBH3RX4JVCCILITWANT6JRANZI2AUQ', networkType: 152}
    blockReceiptsHash: "ABDDB66A03A270E4815C256A8125B70FC3B7EFC4B95FF5ECAD517CB1AB5F5334"
    blockTransactionsHash: "0000000000000000000000000000000000000000000000000000000000000000"
    difficulty: UInt64 {lower: 1316134912, higher: 2328}
    feeMultiplier: 0
    generationHash: "5B4F32D3F2CDD17917D530A6A967927D93F73F2B52CC590A64E3E94408D8CE96"
    hash: "E8294BDDDAE32E17242DF655805EC0FCAB3B628A331824B87A3CA7578683B09C"
    height: UInt64 {lower: 316759, higher: 0}
    networkType: 152
    previousBlockHash: "38382D616772682321D58046511DD942F36A463155C5B7FB0A2CBEE8E29B253C"
    proofGamma: "37187F1C8BD8C87CB4F000F353ACE5717D988BC220EFBCC25E2F40B1FB9B7D7A"
    proofScalar: "AD91A572E5D81EA92FE313CA00915E5A497F60315C63023A52E292E55345F705"
    proofVerificationHash: "EF58228B3EB3C422289626935DADEF11"
    signature: "A9481E5976EDA86B74433E8BCC8495788BA2B9BE0A50F9435AD90A14D1E362D934BA26069182C373783F835E55D7F3681817716295EC1EFB5F2375B6DE302801"
    signer: PublicAccount {publicKey: 'F2195B3FAFBA3DF8C31CFBD9D5BE95BB3F3A04BDB877C59EFB9D1C54ED2DC50E', address: Address}
    stateHash: "4A1C828B34DE47759C2D717845830BA14287A4EC7220B75494BDC31E9539FCB5"
    timestamp: UInt64 {lower: 3851456497, higher: 2}
    type: 33091
    version: 1
```

listener.newBlock()をしておくと、約30秒ごとに通信が発生するのでWebSocketの切断が起こりにくくなります。  
まれに、ブロック生成が1分を超える場合があるのでその場合はリスナーを再接続する必要があります。
（その他の事象で切断される可能性もあるので、万全を期したい場合は後述するoncloseで補足しましょう）

## 10.4 署名要求

署名が必要なトランザクションが発生すると検知します。

```js
listener.open().then(() => {
    //署名が必要なアグリゲートボンデッドトランザクション発生の検知
    listener.aggregateBondedAdded(alice.address)
    .subscribe(async tx=>console.log(tx));
});
```
###### 出力例
```js

> AggregateTransaction
    cosignatures: []
    deadline: Deadline {adjustedValue: 12450154608}
  > innerTransactions: Array(2)
        0: TransferTransaction {type: 16724, networkType: 152, version: 1, deadline: Deadline, maxFee: UInt64, …}
        1: TransferTransaction {type: 16724, networkType: 152, version: 1, deadline: Deadline, maxFee: UInt64, …}
    maxFee: UInt64 {lower: 94400, higher: 0}
    networkType: 152
    signature: "972968C5A2FB70C1D644BE206A190C4FCFDA98976F371DBB70D66A3AAEBCFC4B26E7833BCB86C407879C07927F6882C752C7012C265C2357CAA52C29834EFD0F"
    signer: PublicAccount {publicKey: '0E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26', address: Address}
  > transactionInfo: TransactionInfo
        hash: "44B2CD891DA0B788F1DD5D5AB24866A9A172C80C1749DCB6EB62255A2497EA08"
        height: UInt64 {lower: 0, higher: 0}
        id: undefined
        index: undefined
        merkleComponentHash: "0000000000000000000000000000000000000000000000000000000000000000"
    type: 16961
    version: 1

```

指定アドレスが関係するすべてのアグリゲートトランザクションが検知されます。
連署が必要かどうかは別途フィルターして判断します。


## 10.5 現場で使えるヒント
### 常時コネクション

一覧からランダムに選択し、接続を試みます。

##### ノードへの接続
```js
//ノード一覧
NODES = ["https://node.com:3001",...];

function connectNode(nodes) {
    const node = nodes[Math.floor(Math.random() * nodes.length)] ;
    console.log("try:" + node);

    return new Promise((resolve, reject) => {
        let req = new XMLHttpRequest();
        req.timeout = 2000; //タイムアウト値:2秒(=2000ms)
        req.open('GET', node + "/node/health", true);
        req.onload = function() {
            if (req.status === 200) {
                const status = JSON.parse(req.responseText).status;
                if(status.apiNode == "up" && status.db == "up"){
                    return resolve(node);
                }else{
                    console.log("fail node status:" + status);
                    return connectNode(nodes).then(node => resolve(node));
                }
            } else {
                console.log("fail request status:" + req.status)
                return connectNode(nodes).then(node => resolve(node));
            }
        };

        req.onerror = function(e) {
            console.log("onerror:" + e)
            return connectNode(nodes).then(node => resolve(node));
        };

        req.ontimeout = function (e) {
            console.log("ontimeout")
            return connectNode(nodes).then(node => resolve(node));
        };  

    req.send();
    });
}
```

タイムアウト値を設定しておき、応答の悪いノードに接続した場合は選びなおします。
エンドポイント /node/health　を確認してステータス異常の場合はノードを選びなおします。


##### レポジトリの作成
```js
function createRepo(nodes){

    return connectNode(nodes).then(async function onFulfilled(node) {

        const repo = new sym.RepositoryFactoryHttp(node);

        try{
            epochAdjustment = await repo.getEpochAdjustment().toPromise();
        }catch(error){
          console.log("fail createRepo");
          return await createRepo(nodes);
        }
        return await repo;
    });
}
```
まれに /network/properties のエンドポイントが解放されていないノードが存在するため、
getEpochAdjustment() の情報を取得してチェックを行います。取得できない場合は再帰的にcreateRepoを読み込みます。


##### リスナーの常時接続
```js
async function listenerKeepOpening(nodes){

    const repo = await createRepo(NODES);
    let wsEndpoint = repo.url.replace('http', 'ws') + "/ws";
    const nsRepo = repo.createNamespaceRepository();
    const lner = new sym.Listener(wsEndpoint,nsRepo,WebSocket);
    try{
        await lner.open();
        lner.newBlock();
    }catch(e){
        console.log("fail websocket");
        return await listenerKeepOpening(nodes);
    }

    lner.webSocket.onclose = async function(){
        console.log("listener onclose");
        return await listenerKeepOpening(nodes);
    }
  return lner;
}
```

リスナーがcloseした場合は再接続します。

##### リスナー開始
```js
listener = await listenerKeepOpening(NODES);
```

### 未署名トランザクション自動連署

未署名のトランザクションを検知して、署名＆ネットワークにアナウンスします。  
初期画面表示時と画面閲覧中の受信と２パターンの検知が必要です。  

```js
//rxjsの読み込み
op  = require("/node_modules/rxjs/operators");
rxjs = require("/node_modules/rxjs");

//アグリゲートトランザクション検知
bondedListener = listener.aggregateBondedAdded(bob.address);
bondedHttp = txRepo.search({address:bob.address,group:sym.TransactionGroup.Partial})
.pipe(
    op.delay(2000),
    op.mergeMap(page => page.data)
);

//選択中アカウントの完了トランザクション検知リスナー
const statusChanged = function(address,hash){

    const transactionObservable = listener.confirmed(address);
    const errorObservable = listener.status(address, hash);
    return rxjs.merge(transactionObservable, errorObservable).pipe(
        op.first(),
        op.map((errorOrTransaction) => {
            if (errorOrTransaction.constructor.name === "TransactionStatusError") {
                throw new Error(errorOrTransaction.code);
            } else {
                return errorOrTransaction;
            }
        }),
    );
}

//連署実行
function exeAggregateBondedCosignature(tx){

    txRepo.getTransactionsById([tx.transactionInfo.hash],sym.TransactionGroup.Partial)
    .pipe(
        //トランザクションが抽出された場合のみ
        op.filter(aggTx => aggTx.length > 0)
    )
    .subscribe(async aggTx =>{

        //インナートランザクションの署名者に自分が指定されている場合
        if(aggTx[0].innerTransactions.find((inTx) => inTx.signer.equals(bob.publicAccount))!= undefined){
            //Aliceのトランザクションで署名
            const cosignatureTx = sym.CosignatureTransaction.create(aggTx[0]);
            const signedTx = bob.signCosignatureTransaction(cosignatureTx);
            const cosignedAggTx = await txRepo.announceAggregateBondedCosignature(signedTx).toPromise();
            statusChanged(bob.address,signedTx.parentHash).subscribe(res=>{
              console.log(res);
            });
        }
    });
}

bondedSubscribe = function(observer){
    observer.pipe(

        //すでに署名済みでない場合
        op.filter(tx => {
            return !tx.signedByAccount(sym.PublicAccount.createFromPublicKey(bob.publicKey ,networkType));
        })
    ).subscribe(tx=>{
        console.log(tx);
        exeAggregateBondedCosignature(tx);
    });
}

bondedSubscribe(bondedListener);
bondedSubscribe(bondedHttp);
```

##### 注意事項
スキャムトランザクションを自動署名しないように、
送信元のアカウントを確認するなどのチェック処理を必ず実施するようにしてください。
