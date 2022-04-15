# 12.オフライン署名
ロック機構の章で、アナウンスしたトランザクションをハッシュ値指定でロックして、複数の署名（オンライン署名）を集めるアグリゲートトランザクションを紹介しました。  
この章では、トランザクションを事前に署名を集めてノードにアナウンスするオフライン署名について説明します。  

### 手順

Aliceが起案者となりトランザクションを作成し、署名します。  
次にBobが署名してAliceに返します。  
最後にAliceがトランザクションを結合してネットワークにアナウンスします。  


### 12.1 トランザクション作成
```js
bob = sym.Account.generateNewAccount(networkType);

innerTx1 = sym.TransferTransaction.create(
    undefined,
    bob.address, 
    [],
    sym.PlainMessage.create("tx1"),
    networkType
);

innerTx2 = sym.TransferTransaction.create(
    undefined,
    alice.address, 
    [],
    sym.PlainMessage.create("tx2"),
    networkType
);

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [
      innerTx1.toAggregate(alice.publicAccount),
      innerTx2.toAggregate(bob.publicAccount)
    ],
    networkType,
    [],
).setMaxFeeForAggregate(100, 1);

signedTx =  alice.sign(aggregateTx,generationHash);
signedHash = signedTx.hash;
signedPayload = signedTx.payload;
```

署名を行い、signedHash,signedPayloadを出力します。  
signedPayloadをBobに渡して署名を促します。  

### 12.2 Bobによる連署

BobはAliceからsignedPayloadを受け取ります。

```js
//Bobで署名
bobSignedTx = sym.CosignatureTransaction.signTransactionPayload(bob, signedPayload, generationHash);
bobSignedTxSignature = bobSignedTx.signature;
bobSignedTxSignerPublicKey = bobSignedTx.signerPublicKey;
```

CosignatureTransactionで署名を行い、bobSignedTxSignature,bobSignedTxSignerPublicKeyを出力しAliceに返却します。  
BobがAliceの作成したsignedHashを知っている場合はBobがアナウンスすることも可能です。  

### 12.3 Aliceによるアナウンス

AliceはBobからbobSignedTxSignature,bobSignedTxSignerPublicKeyを受け取ります。  
また事前に自分が作成したsignedHash,signedPayloadを用意します。  

```js
//BobがAliceの署名を添付
cosignSignedTxs = [
    new sym.CosignatureSignedTransaction(signedHash,bobSignedTxSignature,bobSignedTxSignerPublicKey)
];

recreatedTx = sym.TransactionMapping.createFromPayload(signedPayload);

cosignSignedTxs.forEach((cosignedTx) => {
    signedPayload += cosignedTx.version.toHex() + cosignedTx.signerPublicKey + cosignedTx.signature;
});

// Calculate new size
size = `00000000${(signedPayload.length / 2).toString(16)}`;
formatedSize = size.substr(size.length - 8, size.length);
littleEndianSize = formatedSize.substr(6, 2) + formatedSize.substr(4, 2) + formatedSize.substr(2, 2) + formatedSize.substr(0, 2);

signedPayload = littleEndianSize + signedPayload.substr(8, signedPayload.length - 8);
signedTx = new sym.SignedTransaction(signedPayload, signedHash, alice.publicKey, recreatedTx.type, recreatedTx.networkType);

await txRepo.announce(signedTx,listener).toPromise();
