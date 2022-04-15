# 12.オフライン署名

ロック機構の章で、アナウンスしたトランザクションをハッシュ値指定でロックして、複数の署名（オンライン署名）を集めるアグリゲートトランザクションを紹介しました。
この章では、トランザクションを事前に署名を集めてノードにアナウンスするオフライン署名について説明します。

### 手順

Aliceが起案者となりトランザクションを作成します。
...

###### トランザクション作成
```js
const bob = sym.Account.generateNewAccount(networkType);

innerTx1 = sym.TransferTransaction.create(
    undefined,
    bob.address, 
    [],
    sym.PlainMessage.create("tx1"),
    0
);

innerTx2 = sym.TransferTransaction.create(
    undefined,
    alice.address, 
    [],
    sym.PlainMessage.create("tx2"),
    0
);

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [
      innerTx1.toAggregate(alice.publicAccount),
      innerTx2.toAggregate(bob.publicAccount)
    ],
    networkType,
    [],
    sym.UInt64.fromUint(1000000)
);

signedTx = alice.sign(aggregateTx,generationHash);
txRepo.announce(signedTx).subscribe(x=>console.log(x));
```



```js
//Aliceで署名
bobSignedTx = sym.CosignatureTransaction.signTransactionPayload(bob, signedPayload, generationHash);
bobSignedTxSignature = bobSignedTx.signature;
bobSignedTxSignerPublicKey = bobSignedTx.signerPublicKey;
```


```js
//BobがAliceの署名を添付
cosignSignedTxs = [
    new sym.CosignatureSignedTransaction(signedHash,bobSignedTxSignature,bobSignedTxSignerPublicKey)
];
recreatedTx = sym.TransactionMapping.createFromPayload(signedPayload);
signedRecreatedTx = recreatedTx.signTransactionGivenSignatures(bob, cosignSignedTxs, generationHash);

//ネットワークへ通知
listener.open().then(() => {
    transactionService.announce(signedTx,listener)
    .subscribe(x=>console.log(x))
});

//エクスプローラーで確認
console.log("http://explorer-0.10.0.x-01.symboldev.network/transactions/" + signedTx.hash);
```






