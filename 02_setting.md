# 2.環境構築

本書の読み進め方について解説します。

## 2.1 使用言語

JavaScriptを使用します。

### SDK
symbol-sdk-typescript-javascript v2.0.0  
https://github.com/symbol/symbol-sdk-typescript-javascript

上記SDKをbrowserify化したものをブラウザの開発者コンソールに読み込ませて使用します。  
https://github.com/xembook/nem2-browserify

##### 注意
現在 symbol-sdk v3.0.0がアルファ版としてリリースされており、v 2.0.0はdeprecatedです。  
v3ではrxjsに依存した多くの機能が削除されるため、REST APIへの直接アクセスが推奨されます。  
 
### リファレンス
Symbol SDK for TypeScript and JavaScript  
https://symbol.github.io/symbol-sdk-typescript-javascript/1.0.3/

Catapult REST Endpoints (1.0.3)  
https://symbol.github.io/symbol-openapi/v1.0.3/

## 2.2 サンプルソースコード

### 変数宣言
console上で何度も書き直して動作検証をして欲しいため、あえてconst宣言を行いません。  
アプリケーション開発時はconst宣言するなどしてセキュリティを確保してください。

### 出力値確認
console.log()を変数の内容を出力します。好みに応じた出力関数に読み替えてお試しください。  
出力内容は `>` 以下に記述しています。サンプルを実行する場合はこの部分を含まずに試してください。

### 同期・非同期
他言語に慣れた開発者の方には非同期処理の書き方に抵抗がある人もいると思うので、特に問題が無い限り非同期処理を使わずに解説します。


### アカウント
#### Alice
本書では主にAliceアカウントを中心として解説します。  
3章で作成したAliceをその後の章でも引き続き使いますので、十分なXYMを送信した状態でお読みください。

#### Bob
Aliceとの送受信用のアカウントとして各章で必要に応じて作成します。その他、マルチシグの章などでCarolなどを使用します。

### 手数料
本書で紹介するトランザクションの手数料乗数は100でトランザクションを作成します。


## 2.3 事前準備
ノード一覧より任意のノードのページをChromeブラウザなどで開きます。本書ではテストネットを前提として解説しています。

- テストネット
    - https://symbolnodes.org/nodes_testnet/
- メインネット
    - https://symbolnodes.org/nodes/

F12キーを押して開発者コンソールを開き、以下のスクリプトを入力します。

```js
(script = document.createElement('script')).src = 'https://xembook.github.io/nem2-browserify/symbol-sdk-pack-2.0.0.js';
document.getElementsByTagName('head')[0].appendChild(script);
```

続いて、ほぼすべての章で利用する共通ロジック部分を実行しておきます。

```js
NODE = window.origin; //現在開いているページのURLがここに入ります
sym = require("/node_modules/symbol-sdk");
repo = new sym.RepositoryFactoryHttp(NODE);
txRepo = repo.createTransactionRepository();
(async() =>{
    networkType = await repo.getNetworkType().toPromise();
    generationHash = await repo.getGenerationHash().toPromise();
    epochAdjustment = await repo.getEpochAdjustment().toPromise();
})();

function clog(signedTx){
    console.log(NODE + "/transactionStatus/" + signedTx.hash);
    console.log(NODE + "/transactions/confirmed/" + signedTx.hash);
    console.log("https://symbol.fyi/transactions/" + signedTx.hash);
    console.log("https://testnet.symbol.fyi/transactions/" + signedTx.hash);
}
```

これで準備完了です。  

本ドキュメントの内容が少し分かりにくい場合はQiita等の記事もご参考ください。

[Symbolブロックチェーンのテストネットで送金を体験する](https://qiita.com/nem_takanobu/items/e2b1f0aafe7a2df0fe1b)
