# 6.ネームスペース

Symbolブロックチェーンではネームスペースをレンタルしてアドレスやモザイクに視認性の高い単語をリンクさせることができます。
ネームスペースは最大64文字、利用可能な文字は a, b, c, …, z, 0, 1, 2, …, 9, _ , - です。

## 手数料の計算

ネームスペースのレンタルにはネットワーク手数料とは別にレンタル手数料が発生します。
ネットワークの活性度に比例して価格が変動しますので、取得前に確認するようにしてください。

ルートネームスペースを365日レンタルする場合の手数料取得

```js
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
> rootNsRenatalFeeTotal:210240000
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
> 10000000
```

サブネームスペースに期間指定はありません。ルートネームスペースをレンタルしている限り使用できます。

## レンタル

ルートネームスペースをレンタルします
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

サブネームスペースをレンタルします
```js
subNamespaceTx = nem.NamespaceRegistrationTransaction.createSubNamespace(
	sym.Deadline.create(epochAdjustment),
	"tomato",
	"xembook",
	networkType,
).setMaxFee(100);
signedTx = alice.sign(subNamespaceTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

### 有効期限の計算

レンタル済みルートネームスペースの有効期限を計算します。

```js
nsInfo = await nsRepo.getNamespace(nsXembookId).toPromise();
lastHeight = (await chainRepo.getChainInfo().toPromise()).height;
lastBlock = await blockRepo.getBlockByHeight(lastHeight).toPromise();
remainHeight = nsInfo.endHeight.compact() - lastHeight.compact();

new Date(lastBlock.timestamp.compact() + remainHeight * 30000 + epochAdjustment * 1000)

//>Tue Mar 29 2022 18:17:06 GMT+0900 (日本標準時)
```
## リンク

###### アカウントへのリンク
```js
nsXembookId = new sym.NamespaceId("xembook");
address = nem.Address.createFromRawAddress("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ");
addressAliasTx = sym.AliasTransaction.createForAddress(
	sym.Deadline.create(epochAdjustment),
	sym.AliasAction.Link,
	namespaceId,
	address,
	networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();

```

###### モザイクへのリンク
```js
nsTomatoId = new sym.NamespaceId("xembook.tomato");
mosaicId = new sym.MosaicId("6BED913FA20223F8");
mosaicAliasTx = sym.AliasTransaction.createForMosaic(
    sym.Deadline.create(epochAdjustment),
    sym.AliasAction.Link,
    nsTomatoId,
    mosaicId,
    networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();

```

## 未解決で使用
### UnresolvedAccountとして使う

```js
nsXembookId = new sym.NamespaceId("xembook");
tx = sym.TransferTransaction.create(
	sym.Deadline.create(epochAdjustment),
	nsXembook, 
	[],
	sym.EmptyMessage,
	networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

### UnresolvedMosaicとして使う

```js
nsTomatoId = new sym.NamespaceId("xembook.tomato");
tx = sym.TransferTransaction.create(
	sym.Deadline.create(epochAdjustment),
	address, 
	[new sym.Mosaic(nsTomato,sym.UInt64.fromUint(1))],
	sym.EmptyMessage,
	networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

## 参照

###### アドレスへリンクしたネームスペースの参照
```js
namespaceInfo = await nsRepo.getNamespace(new sym.NamespaceId("xembook")).toPromise();
console.log(namespaceInfo);

> NamespaceInfo
	> alias: AddressAlias
		> address: Address
			address: "NCESRRSDSXQW7LTYWMHZOCXAESNNBNNVXHPB6WY"
		type: 2
		mosaicId: undefined
```

###### AliasType
```js
{0: 'None', 1: 'Mosaic', 2: 'Address'}
```

###### NamespaceRegistrationType
```js
{0: 'RootNamespace', 1: 'SubNamespace'}
```

###### モザイクへリンクしたネームスペースの参照
```js
namespaceInfo = await nsRepo.getNamespace(new sym.NamespaceId("xembook.tomato")).toPromise();
console..log(namespaceInfo);

> NamespaceInfo 
	> alias: MosaicAlias
		> mosaicId: MosaicId
			id: Id {lower: 2316569883, higher: 822311105}
		type: 1
		address: undefined

```

### 逆引き

```js
accountNames = await nsRepo.getAccountsNames(
  [sym.Address.createFromRawAddress("NBVHIH5E25AFIRQUYOEMZ35FKEOI275O36YMLZI")]
).toPromise();

namespaceIds = accountNames[0].names.map(name=>{
  return name.namespaceId;
});
console.log(namespaceIds);

```

### レシートの参照

```js
state = await receiptRepo.searchAddressResolutionStatements({height:179401}).toPromise();
```

###### ResolutionType
```js
{0: 'Address', 1: 'Mosaic'}
```

## 現場で使えるヒント

(現在執筆中)

