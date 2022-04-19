# 09.マルチシグ化
他のチェーンで提供される一般的なマルチシグと異なり、連署者がオプトインすることで既存のアカウントのマルチシグ化を行います。
一度作成したマルチシグアカウントも、条件を満たす署名をそろえることができれば構成を自由に変更することが可能です。


### 注意事項

マルチシグに登録できる連署者の数は、です。
一つのアカウントが複数のマルチシグの連署者になになる場合、その最大値は、です。

## マルチシグの登録

```js
multisigTx = nem.MultisigAccountModificationTransaction.create(
    nem.Deadline.create(epochAdjustment), 2,3,
    [
        recovery1.address,recovery2.address,
        user1.address,user2.address,user3.address
    ],[],
    networkType
);

aggregateTx = nem.AggregateTransaction.createComplete(
    nem.Deadline.create(epochAdjustment),
    [
        multisigTx.toAggregate(asset.publicAccount),
    ],
    networkType,
    [],
    nem.UInt64.fromUint(1000000)
);

signedTx =  aggregateTx.signTransactionWithCosignatories(
    asset,
    [recovery1,recovery2,user1,user2,user3],
    generationHash,
);
signedTx.hash

txRepo.announce(signedTx).subscribe(_ => console.log(_), err => console.error(err));

aggregateArray = [distTx(bob.publicAccount,200).toAggregate(alice.publicAccount)]
.concat(distTx(alice.publicAccount,0).toAggregate(bob.publicAccount))


```

### マルチシグを確認


#### マルチシグアカウントの確認
```js
console.log(multisigInfo);
> MultisigAccountInfo
  > accountAddress: Address {address: 'NCESRRSDSXQW7LTYWMHZOCXAESNNBNNVXHPB6WY', networkType: 104}
  > cosignatoryAddresses: Array(1)
    > 0: Address {address: 'NBVHIH5E25AFIRQUYOEMZ35FKEOI275O36YMLZI', networkType: 104}
  > multisigAddresses: []
  minApproval: 1
  minRemoval: 1
  version: 1
```

cosignatoryAddressesが連署者として登録されていることがわかります。
また、minApproval:1 によりトランザクションが成立するために必要な署名数１
minRemoval: 1により連署者を取り外すために必要な署名者数は1であることがわかります。

次にに連署者アドレスでマルチシグ情報を調べてみます。


#### 連署者の確認
```js
multisigInfo = await msigRepo.getMultisigAccountInfo(bobAddress).toPromise();
console.log(multisigInfo);
> MultisigAccountInfo
  > accountAddress: Address {address: 'NBVHIH5E25AFIRQUYOEMZ35FKEOI275O36YMLZI', networkType: 104}
  > cosignatoryAddresses: []
  > multisigAddresses: Array(1)
      > 0: Address {address: 'NCESRRSDSXQW7LTYWMHZOCXAESNNBNNVXHPB6WY', networkType: 104}
    minApproval: 0
    minRemoval: 0
    version: 1
```

multisigAddresses に対して連署する権利を持っていることが分かります。

### マルチシグで署名

```js
tx = sym.TransferTransaction.create(
    undefined,
    bobAddress, 
    [networkCurrency.createRelative(1)],
    sym.PlainMessage.create('test'),
    0
);

aggregateTx = sym.AggregateTransaction.createBonded(
    sym.Deadline.create(epochAdjustment),
     [tx.toAggregate(carol.publicAccount)],
    networkType,
    [],
).setMaxFeeForAggregate(100, 1);

signedAggregateTx = alice.signTransactionWithCosignatories(
  aggregateTx,[dave],generationHash
);

txHttp.announce(signedTx).subscribe(x => console.log(x));

```



## 現場で使えるTIPS
### 組織でアカウントを運用する

電子署名の秘密鍵が原則個人でしか使用できないことは説明しました。一度知ってしまったデータを知らなかった、にすることはできないのです。
組織で電子署名を扱う業務を引き継ぐ場合は、元の所有者が保存したデータを0x00などで上書きしたり、外部に委託している場合は契約で交わしたりすることで鍵の流出を防ぐそうです。
どうしてもスター型のデータアクセス構造しか取ることができない状態です。
Symbolの構成変更可能なマルチレベルマルチシグを使えばCarolの署名権を簡単にAliceからDaveに譲渡することが可能です。
