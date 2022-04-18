# 11.制限
アカウントに対する制限、モザイクのグローバル制限についての方法を紹介します。

## 11.1 アカウント制限

### 指定アドレスからの受信制限・指定アドレスへの送信制限
```js
bob = sym.Account.generateNewAccount(networkType);

tx = sym.AccountRestrictionTransaction.createAddressRestrictionModificationTransaction(
  sym.Deadline.create(epochAdjustment),
  sym.AddressRestrictionFlag.BlockOutgoingAddress,
  [bob.address],
  [],
  networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

##### AddressRestrictionFlag
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
mosaicId = new sym.MosaicId("1275B0B7511D9161");
tx = sym.AccountRestrictionTransaction.createMosaicRestrictionModificationTransaction(
  sym.Deadline.create(epochAdjustment),
  sym.MosaicRestrictionFlag.BlockMosaic,
  [mosaicId],
  [],
  networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

##### MosaicRestrictionFlag
```js
{2: 'AllowMosaic', 32770: 'BlockMosaic'}
```

- AllowMosaic：指定モザイクを含むトランザクションのみ受信許可
- BlockMosaic：指定モザイクを含むトランザクションを受信拒否となります。

モザイク送信の制限機能はありません。

### 指定トランザクションの送信制限

```js
tx = sym.AccountRestrictionTransaction.createOperationRestrictionModificationTransaction(
  sym.Deadline.create(epochAdjustment),
  sym.OperationRestrictionFlag.AllowOutgoingTransactionType,
  [sym.TransactionType.TRANSFER],
  [],
  networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

##### OperationRestrictionFlag
```js
{16388: 'AllowOutgoingTransactionType', 49156: 'BlockOutgoingTransactionType'}
```

- AllowOutgoingTransactionType：指定トランザクションの送信のみ許可
- BlockOutgoingTransactionType：指定トランザクションの送信を禁止

トランザクション受信の制限機能はありません。指定できるオペレーションは以下の通りです。

##### TransactionType
```js
{16705: 'AGGREGATE_COMPLETE', 16707: 'VOTING_KEY_LINK', 16708: 'ACCOUNT_METADATA', 16712: 'HASH_LOCK', 16716: 'ACCOUNT_KEY_LINK', 16717: 'MOSAIC_DEFINITION', 16718: 'NAMESPACE_REGISTRATION', 16720: 'ACCOUNT_ADDRESS_RESTRICTION', 16721: 'MOSAIC_GLOBAL_RESTRICTION', 16722: 'SECRET_LOCK', 16724: 'TRANSFER', 16725: 'MULTISIG_ACCOUNT_MODIFICATION', 16961: 'AGGREGATE_BONDED', 16963: 'VRF_KEY_LINK', 16964: 'MOSAIC_METADATA', 16972: 'NODE_KEY_LINK', 16973: 'MOSAIC_SUPPLY_CHANGE', 16974: 'ADDRESS_ALIAS', 16976: 'ACCOUNT_MOSAIC_RESTRICTION', 16977: 'MOSAIC_ADDRESS_RESTRICTION', 16978: 'SECRET_PROOF', 17220: 'NAMESPACE_METADATA', 17229: 'MOSAIC_SUPPLY_REVOCATION', 17230: 'MOSAIC_ALIAS'}

17232: 'ACCOUNT_OPERATION_RESTRICTION' の制限は許可されていません
```

### 確認

設定した制限情報を確認します

```js
res = await resAccountRepo.getAccountRestrictions(alice.address).toPromise();
console.log(res);

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
送信アカウント、受信アカウント相互に条件を満たした場合のみ、該当モザイクを送信することができます。  


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
    sym.MosaicId.createFromNonce(nonce, alice.address),
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
      mosaicDefTx.toAggregate(alice.publicAccount),
      mosaicChangeTx.toAggregate(alice.publicAccount),
      mosaicGlobalResTx.toAggregate(alice.publicAccount)
    ],
    networkType,[],
).setMaxFeeForAggregate(100, 0);

signedTx = alice.sign(aggregateTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

##### MosaicRestrictionType

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

Alice,Bobに対してグローバル制限モザイクに対しての適格情報を追加します。  
送信・受信についてかかる制限なので、すでに所有しているモザイク量についての制限はありません。  
送信を成功させるためには、送信者・受信者双方が条件をクリアしている必要があります。  
モザイク作成者の秘密鍵があればどのアカウントに対しても許可なく制限をつけることができます。  

```js
//Aliceに適用
aliceMosaicAddressResTx =  sym.MosaicAddressRestrictionTransaction.create(
    sym.Deadline.create(epochAdjustment),
    mosaicDefTx.mosaicId, // mosaicId
    sym.KeyGenerator.generateUInt64Key("KYC"), // restrictionKey
    alice.address, // address
    sym.UInt64.fromUint(1), // newRestrictionValue
    networkType,
    sym.UInt64.fromHex('FFFFFFFFFFFFFFFF') //previousRestrictionValue
).setMaxFee(100);
signedTx = alice.sign(aliceMosaicAddressResTx,generationHash);
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
signedTx = alice.sign(bobMosaicAddressResTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

### 制限状態確認

ノードに問い合わせて限状態を確認します。

```js
res = await resMosaicRepo.search({mosaicId:mosaicDefTx.mosaicId}).toPromise();
console.log(res);

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
signedTx = alice.sign(trTx,generationHash);
await txRepo.announce(signedTx).toPromise();

//失敗
carol = sym.Account.generateNewAccount(networkType);
trTx = sym.TransferTransaction.create(
        sym.Deadline.create(epochAdjustment),
        carol.address, 
        [new sym.Mosaic(mosaicDefTx.mosaicId, sym.UInt64.fromUint(1))],
        sym.PlainMessage.create(""),
        networkType
      ).setMaxFee(100);
signedTx = alice.sign(trTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

失敗した場合以下のようなエラーステータスになります。

```js
{"hash":"E3402FB7AE21A6A64838DDD0722420EC67E61206C148A73B0DFD7F8C098062FA","code":"Failure_RestrictionMosaic_Account_Unauthorized","deadline":"12371602742","group":"failed"}
```

## 11.3 今日から現場で使えるTIPS

### アカウントバーン

AllowIncomingAddressによって指定アドレスからのみ受信可能にしておいて、  
手数料に使用するトークンを全量送信すると、秘密鍵を持っていても操作困難なアカウントを明示的に作成することができます。  
（最小手数料を0に設定したノードによって承認される可能性はあります。）  

### モザイクロック
譲渡不可設定のモザイクを配布し、配布者側のアカウントで受け取り拒否を行うと動かすことのできないモザイクを作ることができます。

### 独自経済圏
KYC済みのアカウント間でのみ流通可能なモザイクを作成することができます。

### オプトイン署名不要なメタデータ
MosaicAddressRestrictionを利用して任意のアカウントに対し、許可なしで数値のメタデータを付与することができます。
