# 08.ロック

Symbolブロックチェーンにはハッシュロックとシークレットロックの２種類のロック機構があります。
ハッシュロックはトランザクションのロックで、必要な署名が全て揃うまでトランザクションをロックしてブロック承認を保留し、
シークレットロックはモザイクのロックで、受信者が共通パスワードの所有を証明するまで指定された送信者のモザイク送信量をロックして保留します。

前者のハッシュロックは、複数のトランザクションを1ブロックで完結するために複数人の署名を待つアグリゲートトランザクションを実現し、
後者のシークレットロックは異なるチェーン間でのアトミックスワップを実現します。

## ハッシュロック

### アグリゲートボンデッドトランザクション
```js
tx1 = sym.TransferTransaction.create(
    undefined,
    bob.address, 
    [networkCurrency.createRelative(1)],
    sym.EmptyMessage,
    0
);

tx2 = sym.TransferTransaction.create(
    undefined,
    alice.address, 
    [networkCurrency.createRelative(2)],
    sym.PlainMessage.create('test'),
    0
);

aggregateArray = [
    tx1.toAggregate(alice.publicAccount),
    tx2.toAggregate(bob.publicAccount),
]

aggregateTx = sym.AggregateTransaction.createBonded(
    sym.Deadline.create(epochAdjustment),
    aggregateArray,
    networkType,
    [],
).setMaxFeeForAggregate(100, 1);

signedAggregateTx = alice.sign(aggregateTx, generationHash);

hashLockTx = sym.HashLockTransaction.create(
  sym.Deadline.create(epochAdjustment),
	networkCurrency.createRelative(10),//固定値
	sym.UInt64.fromUint(480),
	signedAggregateTx,
	networkType
).setMaxFee(100);

signedLockTx = alice.sign(hashLockTx, generationHash);

//先にハッシュロックをアナウンス
txRepo.announce(signedLockTx);
//ハッシュロックトランザクションが承認された後、
txRepo.announceAggregateBonded(signedTx);
```

## シークレットロック

```js
carol = sym.Account.generateNewAccount(networkType);
console.log(carol.address);

random = sym.Crypto.randomBytes(20);
proof = random.toString('hex'); //秘密の合言葉
hash = sha3_256.create();
secret = hash.update(random).hex();//ダイジェスト


lockTx = sym.SecretLockTransaction.create(
    sym.Deadline.create(epochAdjustment),
    networkCurrency.createRelative(1),
    sym.UInt64.fromUint(5),
    sym.LockHashAlgorithm.Op_Sha3_256,
    secret,
    carol.address,
    networkType
).setMaxFee(100);

signedLockTx = alice.sign(lockTx,generationHash);
txRepo.announce(signedLockTx).subscribe(x=>console.log(x));

//ロックされていることを確認

proofTx = sym.SecretProofTransaction.create(
    sym.Deadline.create(epochAdjustment),
    sym.LockHashAlgorithm.Op_Sha3_256,
    secret,
    carol.address,
    proof,
    networkType
).setMaxFee(100);

signedProofTx = carol.sign(proofTx,generationHash);
txRepo.announce(signedProofTx).subscribe(x=>console.log(x));

```


## 今日から現場で使えるヒント
### 手数料代払い

ハッシュロックは誰が発行しても問題ありません。
ただし、ロックされたアグリゲートトランザクションの内部にハッシュロックしたアカウントが含まれている必要があります。
ユーザがXYMを所有しない場合、ハッシュロック費用とネットワーク手数料をサービス提供者が負担することで送信を実現することができます。

### タイマー送金

シークレットロックは指定ブロック数を経過すると元のアカウントへ払い戻されます。
この原理を利用して、シークレットロックしたアカウントにたいしてロック分の費用をサービス提供者が充足しておけば、
期限が過ぎた後ユーザ側がロック分のトークン所有量が増加することになります。
一方で、期限が過ぎる前にシークレット証明トランザクションをアナウンスすると、送信が完了し、サービス提供者に充当戻るためキャンセル扱いとなります。

### テスト環境からのアトミックスワップ
シークレットロックは他のチェーンとのアトミックスワップを実現します。
とりあえずテスト環境で始めたトークン運用を、評判の確認や法制度の対応をしてからメインチェーンにアトミックスワップすることができます。
テスト環境は年に数回リセットされる可能性があります。
