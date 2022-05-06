# 5.モザイク

本章ではモザイクの設定とその生成方法について解説します。
Symbolではトークンのことをモザイクと表現します。

> Wikipediaによると、トークンとは「紀元前8000年頃から紀元前3000年までのメソポタミアの地層から出土する直径が1cm前後の粘土で作られたさまざまな形状の物体」のことを指します。一方でモザイクとは「小片を寄せあわせ埋め込んで、絵（図像）や模様を表す装飾美術の技法。石、陶磁器（モザイクタイル）、有色無色のガラス、貝殻、木などが使用され、建築物の床や壁面、あるいは工芸品の装飾のために施される。」とあります。SymbolにおいてモザイクとはSymbolが作りなすエコシステムの様相を表すさまざまな構成要素、と考えることができます。

## 5.1 モザイク生成

モザイク生成には
作成するモザイクを定義します。
```js
supplyMutable = true; //供給量変更の可否
transferable = false; //第三者への譲渡可否
restrictable = true; //制限設定の可否
revokable = true; //発行者からの還収可否

//モザイク定義
nonce = sym.MosaicNonce.createRandom();
mosaicDefTx = sym.MosaicDefinitionTransaction.create(
    undefined, 
    nonce,
    sym.MosaicId.createFromNonce(nonce, alice.address), //モザイクID
    sym.MosaicFlags.create(supplyMutable, transferable, restrictable, revokable),
    2,//divisibility:可分性
    sym.UInt64.fromUint(0), //duration:有効期限
    networkType
);
```

MosaicFlagsは以下の通りです。

```js
MosaicFlags {
  supplyMutable: false, transferable: false, restrictable: false, revokable: false
}
```
数量変更、第三者への譲渡、モザイクグローバル制限の適用、発行者からの還収の可否について指定します。
この項目は後で変更することはできません。

#### divisibility:可分性

可分性は小数点第何位まで数量の単位とするかを決めます。データは整数値として保持されます。

divisibility:0 = 1  
divisibility:1 = 1.0  
divisibility:2 = 1.00  

#### duration:有効期限

0を指定した場合、無期限に使用することができます。
モザイク有効期限を設定した場合、期限が切れた後も消滅することはなくデータとしては残ります。
アカウント1つにつき1000までしか所有することはできませんのでご注意ください。


次に数量を変更します
```js
//モザイク変更
mosaicChangeTx = sym.MosaicSupplyChangeTransaction.create(
    undefined,
    mosaicDefTx.mosaicId,
    sym.MosaicSupplyChangeAction.Increase,
    sym.UInt64.fromUint(1000000),
    networkType
);
```
supplyMutable:falseの場合、全モザイクが発行者にある場合だけ数量の変更が可能です。
divisibility > 0 の場合は、最小単位を1として整数値で定義してください。
（divisibility:2 で 1.00 作成したい場合は100と指定）

MosaicSupplyChangeActionは以下の通りです。
```js
{0: 'Decrease', 1: 'Increase'}
```
増やしたい場合はIncreaseを指定します。
上記2つのトランザクションをまとめてアグリゲートトランザクションを作成します。

```js
aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [
      mosaicDefTx.toAggregate(alice.publicAccount),
      mosaicChangeTx.toAggregate(alice.publicAccount),
    ],
    networkType,[],
).setMaxFeeForAggregate(100, 0);

signedTx = alice.sign(aggregateTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

アグリゲートトランザクションの特徴として、
まだ存在していないモザイクの数量を変更しようとしている点に注目してください。
配列化した時に、矛盾点がなければ1つのブロック内で問題なく処理することができます。


## 5.2 モザイク送信

作成したモザイクを送信します。
よく、ブロックチェーンに初めて触れる方は、
モザイク送信について「クライアント端末に保存されたモザイクを別のクライアント端末へ送信」することとイメージされている人がいますが、
モザイク情報はすべてのノードで常に共有・同期化されており、送信先に未知のモザイク情報を届けることではありません。
正確にはブロックチェーンへ「トランザクションを送信」することにより、アカウント間でのトークン残量を組み替える操作のことを言います。

```js
//受信アカウント作成
bob = sym.Account.generateNewAccount(networkType);

tx = sym.TransferTransaction.create(
    sym.Deadline.create(epochAdjustment),
    bob.address,  //送信先アドレス
    // 送信モザイクリスト
    [ 
      new sym.Mosaic(
        new sym.MosaicId("3A8416DB2D53B6C8"), //テストネットXYM
        sym.UInt64.fromUint(1000000) //1XYM(divisibility:6)
      ),
      new sym.Mosaic(
        mosaicDefTx.mosaicId, // 5.1 で作成したモザイク
        sym.UInt64.fromUint(1)  // 数量:0.01(divisibility:2 の場合)
      )
    ],
    sym.EmptyMessage,
    networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();

```



##### 送信モザイクリスト

複数のモザイクを一度に送信できます。
XYMを送信するには以下のモザイクIDを指定します。
- メインネット：6BED913FA20223F8
- テストネット：3A8416DB2D53B6C8

#### 送信量
小数点もすべて整数にして指定します。
XYMは可分性6なので、1XYM=1000000で指定します。

### 送信確認

```js
txStatus = await tsRepo.getTransactionStatus(signedTx.hash).toPromise();
console.log(txStatus);

txInfo = await txRepo.getTransaction(signedTx.hash,sym.TransactionGroup.Confirmed).toPromise();
console.log(txInfo); 
```
###### 出力例
```js
> TransactionStatus
    code: "Success"
    group: "confirmed"
    deadline: Deadline {adjustedValue: 12776690385}
    hash: "DE479C001E9736976BDA55E560AB1A5DE526236D9E1BCE24941CF8ED8884289E"
    height: UInt64 {lower: 326922, higher: 0}

> TransferTransaction
    deadline: Deadline {adjustedValue: 12776690385}
    maxFee: UInt64 {lower: 19200, higher: 0}
    message: RawMessage {type: -1, payload: ''}
  > mosaics: Array(2)
      > 0: Mosaic
            amount: UInt64 {lower: 1, higher: 0}
          > id: MosaicId
                id: Id {lower: 207493124, higher: 890137608}
      > 1: Mosaic
            amount: UInt64 {lower: 1000000, higher: 0}
          > id: MosaicId
                id: Id {lower: 760461000, higher: 981735131}
    networkType: 152
    payloadSize: 192
    recipientAddress: Address {address: 'TAR6ERCSTDJJ7KCN4BJNJTK7LBBL5JPPVSHUNGY', networkType: 152}
    signature: "7C4E9E80D250C6D09352FB8EC80175719D59787DE67446896A73AABCFE6C420AF7DD707E6D4D2B2987B8BAD775F2989DCB6F738D39C48C1239FC8CC900A6740D"
    signer: PublicAccount {publicKey: '0E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26', address: Address}
  > transactionInfo: TransactionInfo
        hash: "DE479C001E9736976BDA55E560AB1A5DE526236D9E1BCE24941CF8ED8884289E"
        height: UInt64 {lower: 326922, higher: 0}
        id: "626270069F1D5202A10AE93E"
        index: 0
        merkleComponentHash: "DE479C001E9736976BDA55E560AB1A5DE526236D9E1BCE24941CF8ED8884289E"
    type: 16724
    version: 1
```

TransactionStatusには `code: "Success"` 、`group: "confirmed"`と表示されているので、トランザクションはブロックに承認されています。
TransferTransactionのmosaicsに2種類のモザイクが送信されていることが確認できます。また、TransactionInfoに承認されたブロックの情報が記載されています。

## 5.3 現場で使えるヒント

### 所有証明

前章でトランザクションによる存在証明について説明しました。
アカウントの作成した送信指示が消せない形で残せるので、絶対につじつまの合う台帳を作ることができます。
絶対に消せないすべてのアカウントの送信指示の蓄積結果として、各アカウントは自分のモザイク所有を証明することができます。
（本ドキュメントでは所有を「自分の意思で手放すことができる状態」とします。少し話題がそれますが、法律的にはデジタルデータに所有権が認められていないのも、一度知ってしまったデータは自分の意志では忘れたことを他人に証明することができないという点に注目すると納得がいくかもしれません。ブロックチェーンによりそのデータの放棄を明確に示すことができるのですが、詳しくは法律の専門の方にお任せします。）

#### NFT(non fungible token)

発行枚数を1に限定し、supplyMutableをtrueに設定することで、1つだけしか存在しないトークンを発行できます。
モザイクを作成したアカウントアドレスを改ざんできない情報として保有しているので、
そのアカウントの送信トランザクションをメタ情報として利用できます。

処理概要は以下の通りです。
```js
supplyMutable = false; //供給量変更の可否

//モザイク定義
mosaicDefTx = sym.MosaicDefinitionTransaction.create(
    undefined, nonce,mosaicId
    sym.MosaicFlags.create(supplyMutable, transferable, restrictable, revokable),
    0,//divisibility:可分性
    sym.UInt64.fromUint(0), //duration:無期限
);

//モザイク数量固定
mosaicChangeTx = sym.MosaicSupplyChangeTransaction.create(
    undefined,mosaicId,
    sym.MosaicSupplyChangeAction.Increase, //増やす
    sym.UInt64.fromUint(1), //数量1
);

//NFTデータ
nftTx  = sym.TransferTransaction.create(
    undefined, //Deadline:有効期限
    alice.address, 
    [],
    sym.PlainMessage.create("Hello Symbol!"), //NFTデータ実体
)

//モザイクの生成とNFTデータをアグリゲートしてブロックに登録
aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [
      mosaicDefTx.toAggregate(alice.publicAccount),
      mosaicChangeTx.toAggregate(alice.publicAccount),
      nftTx.toAggregate(alice.publicAccount)
    ],
    networkType,[],
).setMaxFeeForAggregate(100, 0);
```

モザイク生成時のブロック高と作成アカウントはモザイク情報に含まれるので同ブロック内のトランザクションを検索することにより、
紐づけられたNFTデータを取得することができます。

#### 回収可能なポイント運用

transferableをtrueに設定することで転売が制限されるため、資金決済法の影響を受けにくいポイントを定義することができます。
またrevokableをtrueに設定することで、ユーザ側が秘密鍵を管理しなくても使用分を回収できるようなポイント運用を行うことができます。

```js
transferable = false; //第三者への譲渡可否
revokable = true; //発行者からの還収可否
```




