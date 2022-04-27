# 7.メタデータ

アカウント・モザイク・ネームスペースに対してKey-Value形式のデータを登録することができます。  
Valueの最大値は1024バイトです。
本章ではモザイク・ネームスペースの作成アカウントとメタデータの作成アカウントがどちらもAliceであることを前提に説明します。

本章のサンプルスクリプトを実行する前に以下を実行して必要ライブラリを読み込んでおいてください。
```js
metaRepo = repo.createMetadataRepository();
mosaicRepo = repo.createMosaicRepository();
metaService = new sym.MetadataTransactionService(metaRepo);
```
## 7.1 アカウントに登録

アカウントに対して、Key-Value値を登録します。

```js
key = sym.KeyGenerator.generateUInt64Key("key_account");
value = "test";

tx = await metaService.createAccountMetadataTransaction(
    undefined,
    networkType,
    alice.address, //メタデータ記録先アドレス
    key,value, //Key-Value値
    alice.address //メタデータ作成者アドレス
).toPromise();

aggregateTx = sym.AggregateTransaction.createComplete(
  sym.Deadline.create(epochAdjustment),
  [tx.toAggregate(alice.publicAccount)],
  networkType,[]
).setMaxFeeForAggregate(100, 0);

signedTx = alice.sign(aggregateTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

メタデータの登録には記録先アカウントが承諾を示す署名が必要です。
また、記録先アカウントと記録者アカウントが同一でもアグリゲートトランザクションにする必要があります。

異なるアカウントのメタデータに登録する場合は署名時に
signTransactionWithCosignatoriesを使用します。

```js
tx = await metaService.createAccountMetadataTransaction(
    undefined,
    networkType,
    bob.address, //メタデータ記録先アドレス
    key,value, //Key-Value値
    alice.address //メタデータ作成者アドレス
).toPromise();

aggregateTx = sym.AggregateTransaction.createComplete(
  sym.Deadline.create(epochAdjustment),
  [tx.toAggregate(alice.publicAccount)],
  networkType,[]
).setMaxFeeForAggregate(100, 1); // 第二引数に連署者の数:1

signedTx = aggregateTx.signTransactionWithCosignatories(
  alice,[bob],generationHash,// 第二引数に連署者
);
```

bobの秘密鍵が分からない場合はこの後の章で説明する
アグリゲートボンデッドトランザクション、あるいはオフライン署名を使用する必要があります。

## 7.2 モザイクに登録

ターゲットとなるモザイクに対して、Key値・ソースアカウントの複合キーでValue値を登録します。
登録・更新にはモザイクを作成したアカウントの署名が必要です。

```js
mosaicId = new sym.MosaicId("1275B0B7511D9161");
mosaicInfo = await mosaicRepo.getMosaic(mosaicId).toPromise();

key = sym.KeyGenerator.generateUInt64Key('key_mosaic');
value = 'test';

tx = await metaService.createMosaicMetadataTransaction(
  undefined,
  networkType,
  mosaicInfo.ownerAddress, //モザイク作成者アドレス
  mosaicId,
  key,value, //Key-Value値
  alice.address
).toPromise();

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [tx.toAggregate(alice.publicAccount)],
    networkType,[]
).setMaxFeeForAggregate(100, 0);

signedTx = alice.sign(aggregateTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

## 7.3 ネームスペースに登録

ネームスペースに対して、Key-Value値を登録します。
登録・更新にはネームスペースを作成したアカウントの署名が必要です。

```js
namespaceId = new sym.NamespaceId("xembook");
namespaceInfo = await nsRepo.getNamespace(namespaceId).toPromise();

key = sym.KeyGenerator.generateUInt64Key('key_namespace');
value = 'test';

tx = await metaService.createNamespaceMetadataTransaction(
    undefined,networkType,
    namespaceInfo.ownerAddress, //ネームスペースの作成者アドレス
    namespaceId,
    key,value, //Key-Value値
    alice.address //メタデータの登録者
).toPromise();

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [tx.toAggregate(alice.publicAccount)],
    networkType,[]
).setMaxFeeForAggregate(100, 0);

signedTx = alice.sign(aggregateTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

## 7.4 確認
登録したメタデータを確認します。

```js
res = await metaRepo.search({
  targetAddress:alice.address,
  sourceAddress:alice.address}
).toPromise();
console.log(res);
```
###### 出力例
```js
data: Array(3)
  0: Metadata
    id: "62471DD2BF42F221DFD309D9"
    metadataEntry: MetadataEntry
      compositeHash: "617B0F9208753A1080F93C1CEE1A35ED740603CE7CFC21FBAE3859B7707A9063"
      metadataType: 0
      scopedMetadataKey: UInt64 {lower: 92350423, higher: 2540877595}
      sourceAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
      targetAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
      targetId: undefined
      value: "test"
  1: Metadata
    id: "62471F87BF42F221DFD30CC8"
    metadataEntry: MetadataEntry
      compositeHash: "D9E2019D7BD5BA58245320392A68B51752E35A35DA349B08E141DCE99AC3655A"
      metadataType: 1
      scopedMetadataKey: UInt64 {lower: 1789141730, higher: 3475078673}
      sourceAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
      targetAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
      targetId: MosaicId
      id: Id {lower: 1360892257, higher: 309702839}
      value: "test"
  3: Metadata
    id: "62616372BF42F221DF00A88C"
    metadataEntry: MetadataEntry
      compositeHash: "D8E597C7B491BF7F9990367C1798B5C993E1D893222F6FC199F98915339D92D5"
      metadataType: 2
      scopedMetadataKey: UInt64 {lower: 141807833, higher: 2339015223}
      sourceAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
      targetAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
      targetId: NamespaceId
      id: Id {lower: 646738821, higher: 2754876907}
      value: "test"
```
metadataTypeは以下の通りです。
```js
sym.MetadataType
{0: 'Account', 1: 'Mosaic', 2: 'Namespace'}
```


## 7.5 現場で使えるヒント

### 有資格証明

モザイクの章で所有証明、ネームスペースの章でドメインリンクの説明をしました。
実社会で信頼性の高いドメインからリンクされたアカウントが発行したメタデータの付与を受けることで
そのドメイン内での有資格情報の所有を証明することができます。

#### DID

分散型アイデンティティと呼ばれます。
エコシステムは発行者、所有者、検証者に分かれ、例えば大学が発行した卒業証書を学生が所有し、
企業は学生から提示された証明書を大学が公表している公開鍵をもとに検証します。
このやりとりにプラットフォームに依存する情報はありません。
メタデータを活用することで、大学は学生の所有するアカウントにメタデータを発行することができ、
企業は大学の公開鍵と学生のモザイク(アカウント)所有証明でメタデータに記載された卒業証明を検証することができます。

