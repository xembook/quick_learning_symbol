# 10.監視
SymbolのノードはREST APIの他にWebSocket通信で情報を取得することが可能です。
WebSocketを利用することでブロックチェーンの状態をアプリケーション側でキャッシュしておく必要なく状態変化を検知することが可能です。

## 10.1 受信検知
```js
listener.open().then(() => {
  listener.confirmed(alice.address)
  .subscribe(tx=>{
  });
	listener.unconfirmedAdded(alice.address)
	.subscribe(tx=>{
  });
});
```
## 10.2 ブロック監視
```js
listener.open().then(() => {
    listener.newBlock()
    .subscribe(block=>console.log(block));
});
```

## 10.3 署名要求
```js
listener.open().then(() => {
    listener.aggregateBondedAdded(alice.address)
    .subscribe(async tx=>console.log(tx));
});
```


## 10.4 現場で使えるヒント
### 常時コネクション

一覧からランダムに選択し、接続を試みます。

- ノードの正常起動
- パラメータ取得可能
  - ノードが起動していても
- WebSocket対応


```js

//ノード一覧
NODES = ["https://node.com:3001",...];

//ランダムにノード接続
function connectNode(nodes,aNode){

	const node = nodes[Math.floor(Math.random() * nodes.length)] ;
	$.ajax({url:  node + "/node/health" ,type: 'GET',timeout: 1000})
	.then(res => {
		if(res.status.apiNode == "up" && res.status.db == "up"){
			console.log(node);
			return aNode.resolve(node);
		}
		return connectNode(nodes,aNode);
	})
	.catch(res =>connectNode(nodes,aNode));
	return aNode.promise();
}

//レポジトリ生成
async function createRepo(aRepo,nodes){

	const awaitNode = $.Deferred();
	const node = await connectNode(nodes,awaitNode);
	const repo = new sym.RepositoryFactoryHttp(node);

	try{
		networkType = await repo.getNetworkType().toPromise();
		generationHash = await repo.getGenerationHash().toPromise();
		epochAdjustment = await repo.getEpochAdjustment().toPromise();
		aRepo.resolve(repo);

	}catch(error){
		console.log(error);
		createRepo(aRepo,nodes);
	}
	return aRepo.promise();
}

//リスナー接続維持
async function listenerKeepOpening(repo){

	wsEndpoint = repo.url.replace('http', 'ws') + "/ws";
	const nsRepo = repo.createNamespaceRepository();
	const lner = new sym.Listener(wsEndpoint,nsRepo,WebSocket);
	await lner.open();

	lner.webSocket.onclose = async function(){
		console.log("listener onclose");
		const awaitRepo = $.Deferred();
		const repo = await createRepo(awaitRepo,NODES);
		await listenerKeepOpening(repo);
	}
  return lner;
}

let networkType;
let generationHash;
let epochAdjustment;


const awaitRepo = $.Deferred();
const repo = await createRepo(awaitRepo,NODES);
const listener = await listenerKeepOpening(repo);

const awaitRepo2 = $.Deferred();
const repo2 = await createRepo(awaitRepo2,NODES);
const listener2 = await listenerKeepOpening(repo2);

listener.newBlock().subscribe(x=>console.log(x));
listener2.newBlock().subscribe(x=>console.log(x));


```

### 未署名トランザクション検知

自分が署名しなければいけないトランザクションを検知して通知するプログラム例です。
初期画面表示時と画面閲覧中の受信と２パターンの検知が必要です。


- フィルタ条件
  - 未署名であること
  - アグリゲートトランザクションのうち自分が送信者であること



```js
//アグリゲートトランザクション検知
const bondedListener = listener.aggregateBondedAdded(accountInfo.address);
const bondedHttp = txRepo.search({address:accountInfo.address,group:sym.TransactionGroup.Partial})
.pipe(
	op.delay(2000),
	op.mergeMap(page => page.data)
);

const bondedSubscribe = function(observer){
	observer.pipe(

		//すでに署名済みでない場合
		op.filter(_ => {
			return !_.signedByAccount(sym.PublicAccount.createFromPublicKey(accountInfo.publicKey ,networkType));
		})
	).subscribe(_=>{

		txRepo.getTransactionsById([_.transactionInfo.hash],sym.TransactionGroup.Partial)
		.pipe(
			op.filter(aggTx => aggTx.length > 0)
		)
		.subscribe(aggTx =>{

			//インナートランザクションの署名者に自分が指定されている場合
			if(aggTx[0].innerTransactions.find((inTx) => inTx.signer.equals(accountInfo.publicAccount))!= undefined){

        listener.aggregateBondedAdded(alice.address)
        .subscribe(async tx=>{

          //Aliceのトランザクションで署名
          cosignatureTx = sym.CosignatureTransaction.create(tx);
          signedTx = bob.signCosignatureTransaction(cosignatureTx);
          txRepo.announceAggregateBondedCosignature(signedTx)
          .subscribe(aggTx=> statusChanged(address,aggTx.transactionInfo.hash));
        });

      }
		});
	});
}

bondedSubscribe(bondedListener);
bondedSubscribe(bondedHttp);

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
```
