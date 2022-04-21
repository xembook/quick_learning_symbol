# 7.メタデータ

アカウント・モザイク・ネームスペースに対して付加的なデータをKey-Value形式で登録することができます。
Valueの最大値は1024バイトです。


### アカウントに登録

ターゲットとなるアカウントに対して、Key値・ソースアカウントの複合キーでValue値を登録します。

```js
metaRepo = repo.createMetadataRepository();
metaService = new sym.MetadataTransactionService(metaRepo);

accountKey = sym.KeyGenerator.generateUInt64Key("key_account");
value = "test";

metaTx = await metaService.createAccountMetadataTransaction(
    undefined,
    networkType,
    alice.address,//target
    accountKey,
    value,
    alice.address // sender
).toPromise();

aggregateTx = sym.AggregateTransaction.createComplete(
  sym.Deadline.create(epochAdjustment),
  [metaTx.toAggregate(alice.publicAccount)],
  networkType,[]
).setMaxFeeForAggregate(100, 0);

signedTx = alice.sign(aggregateTx,generationHash);
await txRepo.announce(signedTx).toPromise();

```

登録にはターゲットアカウントの署名が必要です。また、ターゲットとソースアカウントが同一でもアグリゲートトランザクションにする必要があります。




### モザイクに登録


ターゲットとなるモザイクに対して、Key値・ソースアカウントの複合キーでValue値を登録します。
なお、登録にはターゲットアカウントの署名が必要です。ターゲットとソースアカウントが同一でも必要です。



```js
metaRepo = repo.createMetadataRepository();
metaService = new sym.MetadataTransactionService(metaRepo);

key = sym.KeyGenerator.generateUInt64Key('key_mosaic');
value = 'test';

metaTx = await metaService.createMosaicMetadataTransaction(
  undefined,
  networkType,
  alice.address,
  new sym.MosaicId("1275B0B7511D9161"),
  key,value,
  alice.address
).toPromise();


aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [metaTx.toAggregate(alice.publicAccount)],
    networkType,[]
).setMaxFeeForAggregate(100, 0);

signedTx = alice.sign(aggregateTx,generationHash);
await txRepo.announce(signedTx).toPromise();

```



### ネームスペースに登録


```js
metaRepo = repo.createMetadataRepository();
metaService = new sym.MetadataTransactionService(metaRepo);

key = sym.KeyGenerator.generateUInt64Key('key_namespace');
value = 'test';

metaTx = await metaService.createNamespaceMetadataTransaction(
    undefined,networkType,
    alice.address,//ネームスペースの作成者
    new sym.NamespaceId("xembook"),
    key,value,
    alice.address //メタデータの登録者
).toPromise();

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [metaTx.toAggregate(alice.publicAccount)],
    networkType,[]
).setMaxFeeForAggregate(100, 0);

signedTx = alice.sign(aggregateTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

## メタデータ参照
登録したメタデータを参照します。

### アカウントに登録したメタデータを検索
```js
key = sym.KeyGenerator.generateUInt64Key("key_account");
targetAddress = sym.Address.createFromRawAddress("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ")
sourceAddress = sym.Address.createFromRawAddress("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ")
result = await metaRepo.search({
  scopedMetadataKey:key.toHex(),
  targetAddress:targetAddress,
  sourceAddress:sourceAddress,
  metadataType:sym.MetadataType.Account}
).toPromise();
console.log(result.data);
```

```js
> [Metadata]
  > 0: Metadata
      id: "62471DD29AFFBDFF8D494103"
      > metadataEntry: MetadataEntry
          compositeHash: "617B0F9208753A1080F93C1CEE1A35ED740603CE7CFC21FBAE3859B7707A9063"
          metadataType: 0
          scopedMetadataKey: UInt64 {lower: 92350423, higher: 2540877595}
          sourceAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
          targetAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
          targetId: undefined
          value: "test"
```

###### MetadataType
```js
sym.MetadataType
{0: 'Account', 1: 'Mosaic', 2: 'Namespace'}
```

## 現場で使えるヒント

(現在執筆中)
