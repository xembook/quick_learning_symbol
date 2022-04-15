# 11.制限

アカウントに対する制限、モザイクのグローバルな振る舞いを制限する方法を紹介します。

## アカウント制限

###### 指定アドレスからの受信制限・指定アドレスへの送信制限
```js
tx = sym.AccountRestrictionTransaction.createAddressRestrictionModificationTransaction(
  sym.Deadline.create(epochAdjustment),
  sym.AddressRestrictionFlag.AllowIncomingAddress,
  [targetAddress],
  [],
  networkType
);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

###### AddressRestrictionFlag
```json
{1: 'AllowIncomingAddress', 16385: 'AllowOutgoingAddress', 32769: 'BlockIncomingAddress', 49153: 'BlockOutgoingAddress'}
```

AddressRestrictionFlagにはAllowIncomingAddressのほか、上記のようなフラグが使用できます。
- AllowIncomingAddress：指定アドレスからのみ受信許可
- AllowOutgoingAddress：指定アドレス宛のみ送信許可
- BlockIncomingAddress：指定アドレスからの受信受拒否
- BlockOutgoingAddress：指定アドレス宛への送信禁止

###### 指定モザイクの受信制限
```js
tx = sym.AccountRestrictionTransaction.createMosaicRestrictionModificationTransaction(
  sym.Deadline.create(epochAdjustment),
  sym.MosaicRestrictionFlag.AllowMosaic,
  [targetMosaicId],
  [],
  networkType
);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

###### MosaicRestrictionFlag
```json
{2: 'AllowMosaic', 32770: 'BlockMosaic'}
```

MosaicRestrictionFlagにはAllowMosaicとBlockMosaicが使用できます。

- AllowMosaic：指定モザイクを含むトランザクションのみ受信許可
- BlockMosaic：指定モザイクを含むトランザクションを受信拒否となります。
モザイク送信の制限機能はありません。

###### 指定トランザクションの送信制限

```js
tx = sym.AccountRestrictionTransaction.createOperationRestrictionModificationTransaction(
  sym.Deadline.create(epochAdjustment),
  sym.OperationRestrictionFlag.AllowOutgoingTransactionType,
  [sym.TransactionType.TRANSFER],
  [],
  networkType
);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```


###### OperationRestrictionFlag
```json
{16388: 'AllowOutgoingTransactionType', 49156: 'BlockOutgoingTransactionType'}
```
OperationRestrictionFlagにはAllowOutgoingTransactionTypeとBlockOutgoingTransactionTypeが使用できます。
- AllowOutgoingTransactionType：指定トランザクションの送信のみ許可
- BlockOutgoingTransactionType：指定トランザクションの送信を禁止

指定できるオペレーションは以下の通りです。トランザクション受信の制限機能はありません。

###### TransactionType
```json
{0: 'RESERVED', 16705: 'AGGREGATE_COMPLETE', 16707: 'VOTING_KEY_LINK', 16708: 'ACCOUNT_METADATA', 16712: 'HASH_LOCK', 16716: 'ACCOUNT_KEY_LINK', 16717: 'MOSAIC_DEFINITION', 16718: 'NAMESPACE_REGISTRATION', 16720: 'ACCOUNT_ADDRESS_RESTRICTION', 16721: 'MOSAIC_GLOBAL_RESTRICTION', 16722: 'SECRET_LOCK', 16724: 'TRANSFER', 16725: 'MULTISIG_ACCOUNT_MODIFICATION', 16961: 'AGGREGATE_BONDED', 16963: 'VRF_KEY_LINK', 16964: 'MOSAIC_METADATA', 16972: 'NODE_KEY_LINK', 16973: 'MOSAIC_SUPPLY_CHANGE', 16974: 'ADDRESS_ALIAS', 16976: 'ACCOUNT_MOSAIC_RESTRICTION', 16977: 'MOSAIC_ADDRESS_RESTRICTION', 16978: 'SECRET_PROOF', 17220: 'NAMESPACE_METADATA', 17229: 'MOSAIC_SUPPLY_REVOCATION', 17230: 'MOSAIC_ALIAS', 17232: 'ACCOUNT_OPERATION_RESTRICTION'}
```

###### MosaicRestrictionType

```json
{0: 'NONE', 1: 'EQ', 2: 'NE', 3: 'LT', 4: 'LE', 5: 'GT', 6: 'GE'}
```


## グローバルモザイク制限

グローバルモザイク制限はアカウントに対してではなく、モザイクに対して制限を儲けます。

###### RestrictionFlag

```json
{1: 'Address', 2: 'Mosaic', 4: 'TransactionType', 16384: 'Outgoing', 32768: 'Block'}
```

## 今日から現場で使えるTIPS

### アカウントバーン

AllowIncomingAddressによって指定アドレスからのみ受信可能にしておいて、
手数料に必要なXYM全額を出金すると、秘密鍵を持っていても操作困難なアカウントを明示的に作成することができます。
（秘密鍵を持っている場合は最小手数料を0に設定したノードによって承認される可能性はあります。）


### トークンロック
トランザクションデータの追記
