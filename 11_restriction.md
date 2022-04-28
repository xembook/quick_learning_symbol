# 11.制限

アカウントに対する制限とモザイクのグローバル制限についての方法を紹介します。
本章では、既存アカウントの権限を制限してしまうので、使い捨てのアカウントを新規に作成してお試しください。

```js
//使い捨てアカウントCarolの生成
carol = sym.Account.generateNewAccount(networkType);
console.log(carol.address);

//FAUCET URL出力
console.log("https://testnet.symbol.tools/?recipient=" + carol.address.plain() +"&amount=10");
```
## 11.1 アカウント制限

### 指定アドレスからの受信制限・指定アドレスへの送信制限
```js

bob = sym.Account.generateNewAccount(networkType);

tx = sym.AccountRestrictionTransaction.createAddressRestrictionModificationTransaction(
  sym.Deadline.create(epochAdjustment),
  sym.AddressRestrictionFlag.BlockIncomingAddress,
  [bob.address],//設定アドレス
  [],　　　　　　//解除アドレス
  networkType
).setMaxFee(100);
signedTx = carol.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

AddressRestrictionFlagについては以下の通りです。
```js
{1: 'AllowIncomingAddress', 16385: 'AllowOutgoingAddress', 32769: 'BlockIncomingAddress', 49153: 'BlockOutgoingAddress'}
```

AddressRestrictionFlagにはAllowIncomingAddressのほか、上記のようなフラグが使用できます。
- AllowIncomingAddress：指定アドレスからのみ受信許可
- AllowOutgoingAddress：指定アドレス宛のみ送信許可
- BlockIncomingAddress：指定アドレスからの受信受拒否
- BlockOutgoingAddress：指定アドレス宛への送信禁止

### 指定モザイクの受信制限
```js
mosaicId = new sym.MosaicId("3A8416DB2D53B6C8"); //テストネット XYM
tx = sym.AccountRestrictionTransaction.createMosaicRestrictionModificationTransaction(
  sym.Deadline.create(epochAdjustment),
  sym.MosaicRestrictionFlag.BlockMosaic,
  [mosaicId],//設定モザイク
  [],//解除モザイク
  networkType
).setMaxFee(100);
signedTx = carol.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

MosaicRestrictionFlagについては以下の通りです。
```js
{2: 'AllowMosaic', 32770: 'BlockMosaic'}
```

- AllowMosaic：指定モザイクを含むトランザクションのみ受信許可
- BlockMosaic：指定モザイクを含むトランザクションを受信拒否

モザイク送信の制限機能はありません。

### 指定トランザクションの送信制限

```js
tx = sym.AccountRestrictionTransaction.createOperationRestrictionModificationTransaction(
  sym.Deadline.create(epochAdjustment),
  sym.OperationRestrictionFlag.AllowOutgoingTransactionType,
      [sym.TransactionType.ACCOUNT_OPERATION_RESTRICTION],//設定トランザクション
  [],//解除トランザクション
  networkType
).setMaxFee(100);
signedTx = carol.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

OperationRestrictionFlagについては以下の通りです。
```js
{16388: 'AllowOutgoingTransactionType', 49156: 'BlockOutgoingTransactionType'}
```

- AllowOutgoingTransactionType：指定トランザクションの送信のみ許可
- BlockOutgoingTransactionType：指定トランザクションの送信を禁止

トランザクション受信の制限機能はありません。指定できるオペレーションは以下の通りです。

TransactionTypeについては以下の通りです。
```js
{16705: 'AGGREGATE_COMPLETE', 16707: 'VOTING_KEY_LINK', 16708: 'ACCOUNT_METADATA', 16712: 'HASH_LOCK', 16716: 'ACCOUNT_KEY_LINK', 16717: 'MOSAIC_DEFINITION', 16718: 'NAMESPACE_REGISTRATION', 16720: 'ACCOUNT_ADDRESS_RESTRICTION', 16721: 'MOSAIC_GLOBAL_RESTRICTION', 16722: 'SECRET_LOCK', 16724: 'TRANSFER', 16725: 'MULTISIG_ACCOUNT_MODIFICATION', 16961: 'AGGREGATE_BONDED', 16963: 'VRF_KEY_LINK', 16964: 'MOSAIC_METADATA', 16972: 'NODE_KEY_LINK', 16973: 'MOSAIC_SUPPLY_CHANGE', 16974: 'ADDRESS_ALIAS', 16976: 'ACCOUNT_MOSAIC_RESTRICTION', 16977: 'MOSAIC_ADDRESS_RESTRICTION', 16978: 'SECRET_PROOF', 17220: 'NAMESPACE_METADATA', 17229: 'MOSAIC_SUPPLY_REVOCATION', 17230: 'MOSAIC_ALIAS'}
```

##### 注意事項
17232: 'ACCOUNT_OPERATION_RESTRICTION' の制限は許可されていません。
AllowOutgoingTransactionTypeを指定する場合は、ACCOUNT_OPERATION_RESTRICTIONを必ず含める必要があり、
BlockOutgoingTransactionTypeを指定する場合は、ACCOUNT_OPERATION_RESTRICTIONを含めることはできません。


### 確認

設定した制限情報を確認します

```js
resAccountRepo = repo.createRestrictionAccountRepository();

res = await resAccountRepo.getAccountRestrictions(carol.address).toPromise();
console.log(res);
```
###### 出力例
```js
> AccountRestrictions
    address: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
  > restrictions: Array(2)
      0: AccountRestriction
        restrictionFlags: 32770
        values: Array(1)
          0: MosaicId
            id: Id {lower: 1360892257, higher: 309702839}
      1: AccountRestriction
        restrictionFlags: 49153
        values: Array(1)
          0: Address {address: 'TCW2ZW7LVJMS4LWUQ7W6NROASRE2G2QKSBVCIQY', networkType: 152}
```

## 11.2 グローバルモザイク制限

グローバルモザイク制限はモザイクに対して送信可能な条件を設定します。  
その後、各アカウントに対してグローバルモザイク制限専用の数値メタデータを付与します。  
送信アカウント・受信アカウントの両方が条件を満たした場合のみ、該当モザイクを送信することができます。  

最初に必要ライブラリの設定を行います。
```js
nsRepo = repo.createNamespaceRepository();
resMosaicRepo = repo.createRestrictionMosaicRepository();
mosaicResService = new sym.MosaicRestrictionTransactionService(resMosaicRepo,nsRepo);
```


### グローバル制限機能つきモザイクの作成
```js
upplyMutable = true;
transferable = true;
restrictable = true;
revokable = true;

nonce = sym.MosaicNonce.createRandom();
mosaicDefTx = sym.MosaicDefinitionTransaction.create(
    undefined,
    nonce,
    sym.MosaicId.createFromNonce(nonce, carol.address),
    sym.MosaicFlags.create(upplyMutable, transferable, restrictable, revokable),
    0,//divisibility
    sym.UInt64.fromUint(0), //duration
    networkType
);

//モザイク変更
mosaicChangeTx = sym.MosaicSupplyChangeTransaction.create(
    undefined,
    mosaicDefTx.mosaicId,
    sym.MosaicSupplyChangeAction.Increase,
    sym.UInt64.fromUint(1000000),
    networkType
);

//グローバルモザイク制限
key = sym.KeyGenerator.generateUInt64Key("KYC") // restrictionKey 
mosaicGlobalResTx = await mosaicResService.createMosaicGlobalRestrictionTransaction(
    undefined,
    networkType,
    mosaicDefTx.mosaicId,
    key,
    '1',
    sym.MosaicRestrictionType.EQ,
).toPromise();

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [
      mosaicDefTx.toAggregate(carol.publicAccount),
      mosaicChangeTx.toAggregate(carol.publicAccount),
      mosaicGlobalResTx.toAggregate(carol.publicAccount)
    ],
    networkType,[],
).setMaxFeeForAggregate(100, 0);

signedTx = carol.sign(aggregateTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

MosaicRestrictionTypeについては以下の通りです。

```js
{0: 'NONE', 1: 'EQ', 2: 'NE', 3: 'LT', 4: 'LE', 5: 'GT', 6: 'GE'}
```

| 演算子  | 略称  | 英語  |
|---|---|---|
| =  | EQ  | equal to  |
| !=  | NE  | not equal to  |
| <  | LT  | less than  |
| <=  | LE  | less than or equal to  |
| >  | GT  | greater than  |
| <=  | GE  | greater than or equal to  |


### アカウントへのモザイク制限適用

Carol,Bobに対してグローバル制限モザイクに対しての適格情報を追加します。  
送信・受信についてかかる制限なので、すでに所有しているモザイクについての制限はありません。  
送信を成功させるためには、送信者・受信者双方が条件をクリアしている必要があります。  
モザイク作成者の秘密鍵があればどのアカウントに対しても承諾の署名を必要とせずに制限をつけることができます。  

```js
//Carolに適用
carolMosaicAddressResTx =  sym.MosaicAddressRestrictionTransaction.create(
    sym.Deadline.create(epochAdjustment),
    mosaicDefTx.mosaicId, // mosaicId
    sym.KeyGenerator.generateUInt64Key("KYC"), // restrictionKey
    carol.address, // address
    sym.UInt64.fromUint(1), // newRestrictionValue
    networkType,
    sym.UInt64.fromHex('FFFFFFFFFFFFFFFF') //previousRestrictionValue
).setMaxFee(100);
signedTx = carol.sign(carolMosaicAddressResTx,generationHash);
await txRepo.announce(signedTx).toPromise();

//Bobに適用
bob = sym.Account.generateNewAccount(networkType);
bobMosaicAddressResTx =  sym.MosaicAddressRestrictionTransaction.create(
    sym.Deadline.create(epochAdjustment),
    mosaicDefTx.mosaicId, // mosaicId
    sym.KeyGenerator.generateUInt64Key("KYC"), // restrictionKey
    bob.address, // address
    sym.UInt64.fromUint(1), // newRestrictionValue
    networkType,
    sym.UInt64.fromHex('FFFFFFFFFFFFFFFF') //previousRestrictionValue
).setMaxFee(100);
signedTx = carol.sign(bobMosaicAddressResTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

### 制限状態確認

ノードに問い合わせて制限状態を確認します。

```js
res = await resMosaicRepo.search({mosaicId:mosaicDefTx.mosaicId}).toPromise();
console.log(res);
```

###### 出力例
```js
> data
    > 0: MosaicGlobalRestriction
      compositeHash: "68FBADBAFBD098C157D42A61A7D82E8AF730D3B8C3937B1088456432CDDB8373"
      entryType: 1
    > mosaicId: MosaicId
        id: Id {lower: 2467167064, higher: 973862467}
    > restrictions: Array(1)
        0: MosaicGlobalRestrictionItem
          key: UInt64 {lower: 2424036727, higher: 2165465980}
          restrictionType: 1
          restrictionValue: UInt64 {lower: 1, higher: 0}
    > 1: MosaicAddressRestriction
      compositeHash: "920BFD041B6D30C0799E06585EC5F3916489E2DDF47FF6C30C569B102DB39F4E"
      entryType: 0
    > mosaicId: MosaicId
        id: Id {lower: 2467167064, higher: 973862467}
    > restrictions: Array(1)
        0: MosaicAddressRestrictionItem
          key: UInt64 {lower: 2424036727, higher: 2165465980}
          restrictionValue: UInt64 {lower: 1, higher: 0}
          targetAddress: Address {address: 'TAZCST2RBXDSD3227Y4A6ZP3QHFUB2P7JQVRYEI', networkType: 152}
  > 2: MosaicAddressRestriction
  ...
```

### 送信確認

実際にモザイクを送信してみて、制限状態を確認します。

```js
//成功
trTx = sym.TransferTransaction.create(
        sym.Deadline.create(epochAdjustment),
        bob.address, 
        [new sym.Mosaic(mosaicDefTx.mosaicId, sym.UInt64.fromUint(1))],
        sym.PlainMessage.create(""),
        networkType
      ).setMaxFee(100);
signedTx = carol.sign(trTx,generationHash);
await txRepo.announce(signedTx).toPromise();

//失敗
dave = sym.Account.generateNewAccount(networkType);
//手数料分のXYMを送信後、以下のトランザクションを実行
trTx = sym.TransferTransaction.create(
        sym.Deadline.create(epochAdjustment),
        carol.address, 
        [new sym.Mosaic(mosaicDefTx.mosaicId, sym.UInt64.fromUint(1))],
        sym.PlainMessage.create(""),
        networkType
      ).setMaxFee(100);
signedTx = dave.sign(trTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

失敗した場合以下のようなエラーステータスになります。

```js
{"hash":"E3402FB7AE21A6A64838DDD0722420EC67E61206C148A73B0DFD7F8C098062FA","code":"Failure_RestrictionMosaic_Account_Unauthorized","deadline":"12371602742","group":"failed"}
```

## 11.3 現場で使えるヒント

ブロックチェーンの社会実装などを考えたときに、法律や信頼性の見地から
一つの役割のみを持たせたいアカウント、関係ないアカウントを巻き込みたくないと思うことがあります。
そんな場合にアカウント制限とグローバルモザイク制限を使いこなすことで、
モザイクのふるまいを柔軟にコントロールすることができます。

### アカウントバーン

AllowIncomingAddressによって指定アドレスからのみ受信可能にしておいて、  
XYMを全量送信すると、秘密鍵を持っていても自力では操作困難なアカウントを明示的に作成することができます。  
（最小手数料を0に設定したノードによって承認されることもあり、その可能性はゼロではありません）  

### モザイクロック
譲渡不可設定のモザイクを配布し、配布者側のアカウントで受け取り拒否を行うとモザイクをロックさせることができます。

### 所属証明
モザイクの章で所有の証明について説明しました。グローバルモザイク制限を活用することで、
KYCが済んだアカウント間でのみ所有・流通させることが可能なモザイクを作ることができ、独自経済圏を構築することが可能です。


