# 6.ネームスペース

Symbolブロックチェーンではネームスペースをレンタルしてアドレスやモザイクに視認性の高い単語をリンクさせることができます。
ネームスペースは最大64文字、利用可能な文字は a, b, c, …, z, 0, 1, 2, …, 9, _ , - です。

## 手数料の計算

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

出力例
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
出力例
```js
> 10000000 //10XYM
```

サブネームスペースに期間指定はありません。ルートネームスペースをレンタルしている限り使用できます。

## レンタル

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


出力例
```js
> Tue Mar 29 2022 18:17:06 GMT+0900 (日本標準時)
```
## リンク

アカウントへのリンクします。
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

モザイクへリンクします。
```js
namespaceId = new sym.NamespaceId("xembook.tomato");
mosaicId = new sym.MosaicId("6BED913FA20223F8");
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

## 未解決で使用

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
XYMをネームスペースで使用する場合は以下のように指定します。

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


```js
namespaceId = new sym.NamespaceId("symbol.xym");

> NamespaceId {fullName: 'symbol.xym', id: Id}
    fullName: "symbol.xym"
    id: Id {lower: 1106554862, higher: 3880491450}
```

Idは内部ではUint64と呼ばれる数値 `{lower: 1106554862, higher: 3880491450}` で保持されています。

## 参照

アドレスへリンクしたネームスペースの参照します
```js
nsRepo = repo.createNamespaceRepository();

namespaceInfo = await nsRepo.getNamespace(new sym.NamespaceId("xembook")).toPromise();
console.log(namespaceInfo);
```

出力例
```js
> NamespaceInfo
	> alias: AddressAlias
		> address: Address
			address: "NCESRRSDSXQW7LTYWMHZOCXAESNNBNNVXHPB6WY"
		type: 2
		mosaicId: undefined
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

出力例
```js
> NamespaceInfo 
	> alias: MosaicAlias
		> mosaicId: MosaicId
			id: Id {lower: 2316569883, higher: 822311105}
		type: 1
		address: undefined

```

### 逆引き

アドレスに紐づけられたネームスペースを全て調べます。
```js
nsRepo = repo.createNamespaceRepository();

accountNames = await nsRepo.getAccountsNames(
  [sym.Address.createFromRawAddress("NBVHIH5E25AFIRQUYOEMZ35FKEOI275O36YMLZI")]
).toPromise();

namespaceIds = accountNames[0].names.map(name=>{
  return name.namespaceId;
});
console.log(namespaceIds);

```

### レシートの参照

トランザクションに使用されたネームスペースをブロックチェーン側がどう解決したかを確認します。

```js
state = await receiptRepo.searchAddressResolutionStatements({height:179401}).toPromise();
```

出力例
```js
data: Array(1)
  0: ResolutionStatement
    height: UInt64 {lower: 179401, higher: 0}
    resolutionEntries: Array(1)
      0: ResolutionEntry
        resolved: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
        source: ReceiptSource {primaryId: 1, secondaryId: 0}
    resolutionType: 0
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

## 現場で使えるヒント

(現在執筆中)
