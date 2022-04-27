# 6.ネームスペース

Symbolブロックチェーンではネームスペースをレンタルしてアドレスやモザイクに視認性の高い単語をリンクさせることができます。
ネームスペースは最大64文字、利用可能な文字は a, b, c, …, z, 0, 1, 2, …, 9, _ , - です。

## 6.1 手数料の計算

ネームスペースのレンタルにはネットワーク手数料とは別にレンタル手数料が発生します。
ネットワークの活性度に比例して価格が変動しますので、取得前に確認するようにしてください。

ルートネームスペースを365日レンタルする場合の手数料を計算します。

```js
nwRepo = repo.createNetworkRepository();

rentalFees = await nwRepo.getRentalFees().toPromise();
rootNsperBlock = rentalFees.effectiveRootNamespaceRentalFeePerBlock.compact();
rentalDays = 365;
rentalBlock = rentalDays * 24 * 60 * 60 / 30;
rootNsRenatalFeeTotal = rentalBlock * rootNsperBlock;
console.log("rentalBlock:" + rentalBlock);
console.log("rootNsRenatalFeeTotal:" + rootNsRenatalFeeTotal);
```
###### 出力例
```js
> rentalBlock:1051200
> rootNsRenatalFeeTotal:210240000 //約210XYM
```

期間はブロック数で指定します。1ブロックを30秒として計算しました。
最低で30日分はレンタルする必要があります。

サブネームスペースの取得手数料を計算します。

```js
childNamespaceRentalFee = rentalFees.effectiveChildNamespaceRentalFee.compact()
console.log(childNamespaceRentalFee);
```
###### 出力例
```js
> 10000000 //10XYM
```

サブネームスペースに期間指定はありません。ルートネームスペースをレンタルしている限り使用できます。

## 6.2 レンタル

ルートネームスペースをレンタルします(例:xembook)
```js

tx = sym.NamespaceRegistrationTransaction.createRootNamespace(
    sym.Deadline.create(epochAdjustment),
    "xembook",
    sym.UInt64.fromUint(86400),
    networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

サブネームスペースをレンタルします(例:xembook.tomato)
```js
subNamespaceTx = sym.NamespaceRegistrationTransaction.createSubNamespace(
    sym.Deadline.create(epochAdjustment),
    "tomato",  //作成するサブネームスペース
    "xembook", //紐づけたいルートネームスペース
    networkType,
).setMaxFee(100);
signedTx = alice.sign(subNamespaceTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

2階層目のサブネームスペースを作成したい場合は
例えば、xembook.tomato.morningを定義したい場合は以下のようにします。

```js
subNamespaceTx = sym.NamespaceRegistrationTransaction.createSubNamespace(
    ,
    "morning",  //作成するサブネームスペース
    "xembook.tomato", //紐づけたいルートネームスペース
    ,
)
```


### 有効期限の計算

レンタル済みルートネームスペースの有効期限を計算します。

```js
nsRepo = repo.createNamespaceRepository();
chainRepo = repo.createChainRepository();
blockRepo = repo.createBlockRepository();

namespaceId = new sym.NamespaceId("xembook");
nsInfo = await nsRepo.getNamespace(namespaceId).toPromise();
lastHeight = (await chainRepo.getChainInfo().toPromise()).height;
lastBlock = await blockRepo.getBlockByHeight(lastHeight).toPromise();
remainHeight = nsInfo.endHeight.compact() - lastHeight.compact();

endDate = new Date(lastBlock.timestamp.compact() + remainHeight * 30000 + epochAdjustment * 1000)
console.log(endDate);
```

ネームスペース情報の終了ブロックを取得し、現在のブロック高から差し引いた残ブロック数に
30秒(平均ブロック生成間隔)を掛け合わせた日時を出力します。

###### 出力例
```js
> Tue Mar 29 2022 18:17:06 GMT+0900 (日本標準時)
```
## 6.3 リンク

### アカウントへのリンク
```js
namespaceId = new sym.NamespaceId("xembook");
address = sym.Address.createFromRawAddress("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ");
tx = sym.AliasTransaction.createForAddress(
    sym.Deadline.create(epochAdjustment),
    sym.AliasAction.Link,
    namespaceId,
    address,
    networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```
リンク先のアドレスは自分が所有していなくても問題ありません。

### モザイクへリンク
```js
namespaceId = new sym.NamespaceId("xembook.tomato");
mosaicId = new sym.MosaicId("3A8416DB2D53xxxx");
tx = sym.AliasTransaction.createForMosaic(
    sym.Deadline.create(epochAdjustment),
    sym.AliasAction.Link,
    namespaceId,
    mosaicId,
    networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

モザイクを作成したアドレスと同一の場合のみリンクできるようです。


## 6.4 未解決で使用

送信先にUnresolvedAccountとして指定して、アドレスを特定しないままトランザクションを署名・アナウンスします。
チェーンがわで解決されたアカウントに対してそうしんが実施されます。
```js
namespaceId = new sym.NamespaceId("xembook");
tx = sym.TransferTransaction.create(
    sym.Deadline.create(epochAdjustment),
    namespaceId, //UnresolvedAccount:未解決アカウントアドレス
    [],
    sym.EmptyMessage,
    networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```
送信モザイクにUnresolvedMosaicとして指定して、モザイクIDを特定しないままトランザクションを署名・アナウンスします。

```js
namespaceId = new sym.NamespaceId("xembook.tomato");
tx = sym.TransferTransaction.create(
    sym.Deadline.create(epochAdjustment),
    address, 
    [
        new sym.Mosaic(
          namespaceId,//UnresolvedMosaic:未解決モザイク
          sym.UInt64.fromUint(1) //送信量
        )
    ],
    sym.EmptyMessage,
    networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

XYMをネームスペースで使用する場合は以下のように指定します。

```js
namespaceId = new sym.NamespaceId("symbol.xym");
```
```js
> NamespaceId {fullName: 'symbol.xym', id: Id}
    fullName: "symbol.xym"
    id: Id {lower: 1106554862, higher: 3880491450}
```

Idは内部ではUint64と呼ばれる数値 `{lower: 1106554862, higher: 3880491450}` で保持されています。

## 6.5 参照

アドレスへリンクしたネームスペースの参照します
```js
nsRepo = repo.createNamespaceRepository();

namespaceInfo = await nsRepo.getNamespace(new sym.NamespaceId("xembook")).toPromise();
console.log(namespaceInfo);
```
###### 出力例
```js
NamespaceInfo
    active: true
  > alias: AddressAlias
        address: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
        mosaicId: undefined
        type: 2 //AliasType
    depth: 1
    endHeight: UInt64 {lower: 500545, higher: 0}
    index: 1
    levels: [NamespaceId]
    ownerAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
    parentId: NamespaceId {id: Id}
    registrationType: 0 //NamespaceRegistrationType
    startHeight: UInt64 {lower: 324865, higher: 0}
```

AliasTypeは以下の通りです。
```js
{0: 'None', 1: 'Mosaic', 2: 'Address'}
```

NamespaceRegistrationTypeは以下の通りです。
```js
{0: 'RootNamespace', 1: 'SubNamespace'}
```

モザイクへリンクしたネームスペースを参照します。
```js
nsRepo = repo.createNamespaceRepository();

namespaceInfo = await nsRepo.getNamespace(new sym.NamespaceId("xembook.tomato")).toPromise();
console.log(namespaceInfo);
```
###### 出力例
```js
NamespaceInfo
  > active: true
    alias: MosaicAlias
        address: undefined
        mosaicId: MosaicId
        id: Id {lower: 1360892257, higher: 309702839}
        type: 1 //AliasType
    depth: 2
    endHeight: UInt64 {lower: 500545, higher: 0}
    index: 1
    levels: (2) [NamespaceId, NamespaceId]
    ownerAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
    parentId: NamespaceId {id: Id}
    registrationType: 1 //NamespaceRegistrationType
    startHeight: UInt64 {lower: 324865, higher: 0}
```

### 逆引き

アドレスに紐づけられたネームスペースを全て調べます。
```js
nsRepo = repo.createNamespaceRepository();

accountNames = await nsRepo.getAccountsNames(
  [sym.Address.createFromRawAddress("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ")]
).toPromise();

namespaceIds = accountNames[0].names.map(name=>{
  return name.namespaceId;
});
console.log(namespaceIds);

```

### レシートの参照

トランザクションに使用されたネームスペースをブロックチェーン側がどう解決したかを確認します。

```js
receiptRepo = repo.createReceiptRepository();
state = await receiptRepo.searchAddressResolutionStatements({height:179401}).toPromise();
```
###### 出力例
```js
data: Array(1)
  0: ResolutionStatement
    height: UInt64 {lower: 179401, higher: 0}
    resolutionEntries: Array(1)
      0: ResolutionEntry
        resolved: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
        source: ReceiptSource {primaryId: 1, secondaryId: 0}
    resolutionType: 0 //ResolutionType
    unresolved: NamespaceId
      id: Id {lower: 646738821, higher: 2754876907}
```

ResolutionTypeは以下の通りです。
```js
{0: 'Address', 1: 'Mosaic'}
```

#### 注意事項
ネームスペースはレンタル制のため、過去のトランザクションで使用したネームスペースのリンク先と
現在のネームスペースのリンク先が異なる可能性があります。
過去のデータを参照する際などに、その時どのアカウントにリンクしていたかなどを知りたい場合は
必ずレシートを参照するようにしてください。

## 6.6 現場で使えるヒント

### 外部ドメインとの相互リンク

ネームスペースは重複取得がプロトコル上制限されているため、
インターネットドメインや実世界で周知されている商標名と同一ネームスペースを取得し、
外部(公式サイトや印刷物など)からネームスペース存在の認知を公表することで、
Symbol上のアカウントのブランド価値を構築することができます。
(法的な効力については調整が必要です)
外部ドメイン側のハッキングあるいは、Symbol側でのネームスペース更新忘れにはご注意ください。


#### ネームスペースを取得するアカウントについての注意
ネームスペースはレンタル期限という概念をもつ機能です。
今のところ、取得したネームスペース放棄か延長の選択肢しかありません。
運用譲渡などが発生する可能性のあるシステムでネームスペース活用を検討する場合は
マルチシグ化(9章)したアカウントでネームスペースを取得することをおすすめします。

