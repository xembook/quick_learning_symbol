# 8.ロック

Symbolブロックチェーンにはハッシュロックとシークレットロックの２種類のロック機構があります。  

## 8.1 ハッシュロック

ハッシュロックは後でアナウンスされる予定のトランザクションを事前にハッシュ値で登録しておくことで、
該当トランザクションがアナウンスされた場合に、そのトランザクションをAPIノード上で処理せずにロックさせて、署名が集まってから処理を行うことができます。
アカウントが所有するモザイクを操作できないようにロックするわけではなく、ロックするのはハッシュ値の対象となるトランザクションとなります。
ハッシュロックにかかる費用は10XYM、有効期限は最大約48時間です。ロックしたトランザクションが承認されれば10XYMは返却されます。

### アグリゲートボンデッドトランザクションの作成
```js
bob = sym.Account.generateNewAccount(networkType);

tx1 = sym.TransferTransaction.create(
    undefined,
    bob.address,  //Bobへの送信
    [ //1XYM
      new sym.Mosaic(
        new sym.NamespaceId("symbol.xym"),
        sym.UInt64.fromUint(1000000)
      )
    ],
    sym.EmptyMessage, //メッセージ無し
    networkType
);

tx2 = sym.TransferTransaction.create(
    undefined,
    alice.address,  // Aliceへの送信
    [],
    sym.PlainMessage.create('thank you!'), //メッセージ
    networkType
);

aggregateArray = [
    tx1.toAggregate(alice.publicAccount), //Aliceからの送信
    tx2.toAggregate(bob.publicAccount), // Bobからの送信
]

//アグリゲートボンデッドトランザクション
aggregateTx = sym.AggregateTransaction.createBonded(
    sym.Deadline.create(epochAdjustment),
    aggregateArray,
    networkType,
    [],
).setMaxFeeForAggregate(100, 1);

//署名
signedAggregateTx = alice.sign(aggregateTx, generationHash);
```

### ハッシュロックトランザクションの作成と署名、アナウンス
```js
//ハッシュロックTX作成
hashLockTx = sym.HashLockTransaction.create(
  sym.Deadline.create(epochAdjustment),
    new sym.Mosaic(new sym.NamespaceId("symbol.xym"),sym.UInt64.fromUint(10 * 1000000)), //10xym固定値
    sym.UInt64.fromUint(480), // ロック有効期限
    signedAggregateTx,// このハッシュ値を登録
    networkType
).setMaxFee(100);

//署名
signedLockTx = alice.sign(hashLockTx, generationHash);

//ハッシュロックTXをアナウンス
await txRepo.announce(signedLockTx).toPromise();
```

### アグリゲートボンデッドトランザクションのアナウンス

エクスプローラーなどで確認した後、ボンデッドトランザクションをネットワークにアナウンスします。
```js
await txRepo.announceAggregateBonded(signedAggregateTx).toPromise();
```


### 連署
ロックされたトランザクションを指定されたアカウント(Bob)で連署します。

```js
txInfo = await txRepo.getTransaction(signedAggregateTx.hash,sym.TransactionGroup.Partial).toPromise();
cosignatureTx = sym.CosignatureTransaction.create(txInfo);
signedCosTx = bob.signCosignatureTransaction(cosignatureTx);
await txRepo.announceAggregateBondedCosignature(signedCosTx).toPromise();
```

### 注意点
ハッシュロックトランザクションは誰がアナウンスしても大丈夫ですが、
ロックされたアグリゲートトランザクションのinnerTransactionのsignerに含めるようにしてください。
モザイク送信無し＆メッセージ無しのダミートランザクションでも問題ありません（パフォーマンスに影響が出るための仕様とのことです）


## 8.2 シークレットロック・シークレットプルーフ

シークレットロックは事前に共通パスワードを作成しておき、指定モザイクをロックします。
受信者が有効期限内にパスワードの所有を証明することができればロックされたモザイクを受け取ることができる仕組みです。

ここではAliceが1XYMをロックしてBobが解除することで受信する方法を説明します。

まずはAliceとやり取りするBobアカウントを作成します。
ロック解除にBob側からトランザクションをアナウンスする必要があるのでFAUCETで10XYMほど受信しておきます。

```js
bob = sym.Account.generateNewAccount(networkType);
console.log(bob.address);

//FAUCET URL出力
console.log("https://testnet.symbol.tools/?recipient=" + bob.address.plain() +"&amount=10");
```

### シークレットロック

ロック・解除にかかわる共通暗号を作成します。

```js
sha3_256 = require('/node_modules/js-sha3').sha3_256;

random = sym.Crypto.randomBytes(20);
hash = sha3_256.create();
secret = hash.update(random).hex(); //ロック用キーワード
proof = random.toString('hex'); //解除用キーワード
console.log("secret:" + secret);
console.log("proof:" + proof);
```

###### 出力例
```js
> secret:f260bfb53478f163ee61ee3e5fb7cfcaf7f0b663bc9dd4c537b958d4ce00e240
  proof:7944496ac0f572173c2549baf9ac18f893aab6d0
```

トランザクションを作成・署名・アナウンスします
```js
lockTx = sym.SecretLockTransaction.create(
    sym.Deadline.create(epochAdjustment),
    new sym.Mosaic(
      new sym.NamespaceId("symbol.xym"),
      sym.UInt64.fromUint(1000000) //1XYM
    ),　//ロックするモザイク
    sym.UInt64.fromUint(480), //ロック期間(ブロック数)
    sym.LockHashAlgorithm.Op_Sha3_256, //ロックキーワード生成に使用したアルゴリズム
    secret,　//ロック用キーワード
    bob.address, //解除時の転送先:Bob
    networkType
).setMaxFee(100);

signedLockTx = alice.sign(lockTx,generationHash);
await txRepo.announce(signedLockTx).toPromise();
```

LockHashAlgorithmは以下の通りです。
```js
{0: 'Op_Sha3_256', 1: 'Op_Hash_160', 2: 'Op_Hash_256'}
```

ロック時に解除先を指定するのでBob以外のアカウントが解除することはできません。
ロック期間は最長で365日(ブロック数を日換算)までです。

承認されたトランザクションを確認します。
```js
slRepo = repo.createSecretLockRepository();
res = await slRepo.search({secret:secret}).toPromise();
console.log(res.data[0]);
```
###### 出力例
```js
> SecretLockInfo
    amount: UInt64 {lower: 1000000, higher: 0}
    compositeHash: "770F65CB0CC0CA17370DE961B2AA5B48B8D86D6DB422171AB00DF34D19DEE2F1"
    endHeight: UInt64 {lower: 323495, higher: 0}
    hashAlgorithm: 0
    mosaicId: MosaicId {id: Id}
    ownerAddress: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
    recipientAddress: Address {address: 'TBTWKXCNROT65CJHEBPL7F6DRHX7UKSUPD7EUGA', networkType: 152}
    recordId: "6260A1D3205E94BEA3D9E3E9"
    secret: "F260BFB53478F163EE61EE3E5FB7CFCAF7F0B663BC9DD4C537B958D4CE00E240"
    status: 0
    version: 1
```
ロックしたAliceがownerAddress、受信予定のBobがrecipientAddressに記録されています。
secret情報が公開されていて、これに対応するproofをBobがネットワークに通知します。


### シークレットプルーフ

解除用キーワードを使用してロック解除します。
Bobは事前に解除用キーワードを入手しておく必要があります。

```js
proofTx = sym.SecretProofTransaction.create(
    sym.Deadline.create(epochAdjustment),
    sym.LockHashAlgorithm.Op_Sha3_256, //ロック作成に使用したアルゴリズム
    secret, //ロックキーワード
    bob.address, //解除アカウント（受信アカウント）
    proof, //解除用キーワード
    networkType
).setMaxFee(100);

signedProofTx = bob.sign(proofTx,generationHash);
await txRepo.announce(signedProofTx).toPromise();
```

承認結果を確認します。
```js
txInfo = await txRepo.getTransaction(signedProofTx.hash,sym.TransactionGroup.Confirmed).toPromise();
console.log(txInfo);
```
###### 出力例
```js
> SecretProofTransaction
  > deadline: Deadline {adjustedValue: 12669305546}
    hashAlgorithm: 0
    maxFee: UInt64 {lower: 20700, higher: 0}
    networkType: 152
    payloadSize: 207
    proof: "A6431E74005585779AD5343E2AC5E9DC4FB1C69E"
    recipientAddress: Address {address: 'TBTWKXCNROT65CJHEBPL7F6DRHX7UKSUPD7EUGA', networkType: 152}
    secret: "4C116F32D986371D6BCC44CE64C970B6567686E79850E4A4112AF869580B7C3C"
    signature: "951F440860E8F24F6F3AB8EC670A3D448B12D75AB954012D9DB70030E31DA00B965003D88B7B94381761234D5A66BE989B5A8009BB234716CA3E5847C33F7005"
    signer: PublicAccount {publicKey: '9DC9AE081DF2E76554084DFBCCF2BC992042AA81E8893F26F8504FCED3692CFB', address: Address}
  > transactionInfo: TransactionInfo
        hash: "85044FF702A6966AB13D05DBE4AC4C3A13520C7381F32540429987C207B2056B"
        height: UInt64 {lower: 323805, higher: 0}
        id: "6260CC7F60EE2B0EA10CCEDA"
        merkleComponentHash: "85044FF702A6966AB13D05DBE4AC4C3A13520C7381F32540429987C207B2056B"
    type: 16978
```

SecretProofTransactionにはモザイクの受信量の情報は含まれていません。
ブロック生成時に作成されるレシートで受信量を確認します。
レシートタイプ:LockHash_Completed でBob宛のレシートを検索してみます。

```js
receiptRepo = repo.createReceiptRepository();

receiptInfo = await receiptRepo.searchReceipts({
    receiptType:sym.ReceiptTypeLockHash_Completed,
    targetAddress:bob.address
}).toPromise();
console.log(receiptInfo.data);
```
###### 出力例
```js
> data: Array(1)
  >  0: TransactionStatement
        height: UInt64 {lower: 323805, higher: 0}
     >  receipts: Array(1)
          > 0: BalanceChangeReceipt
                amount: UInt64 {lower: 1000000, higher: 0}
            > mosaicId: MosaicId
                  id: Id {lower: 760461000, higher: 981735131}
              targetAddress: Address {address: 'TBTWKXCNROT65CJHEBPL7F6DRHX7UKSUPD7EUGA', networkType: 152}
              type: 8786
```

ReceiptTypeは以下の通りです。

```js
{4685: 'Mosaic_Rental_Fee', 4942: 'Namespace_Rental_Fee', 8515: 'Harvest_Fee', 8776: 'LockHash_Completed', 8786: 'LockSecret_Completed', 9032: 'LockHash_Expired', 9042: 'LockSecret_Expired', 12616: 'LockHash_Created', 12626: 'LockSecret_Created', 16717: 'Mosaic_Expired', 16718: 'Namespace_Expired', 16974: 'Namespace_Deleted', 20803: 'Inflation', 57667: 'Transaction_Group', 61763: 'Address_Alias_Resolution', 62019: 'Mosaic_Alias_Resolution'}

8786: 'LockSecret_Completed' :ロック解除完了
9042: 'LockSecret_Expired'　：ロック期限切れ
```

## 8.3 現場で使えるヒント


### 手数料代払い

一般的にブロックチェーンはトランザクション送信に手数料を必要とします。
そのため、ブロックチェーンを利用しようとするユーザは事前に手数料を取引所から入手しておく必要があります。
このユーザが企業である場合はその管理方法も加えてさらにハードルの高い問題となります。
アグリゲートトランザクションを使用することでハッシュロック費用とネットワーク手数料をサービス提供者が代理で負担することができます。

### タイマー送信

シークレットロックは指定ブロック数を経過すると元のアカウントへ払い戻されます。
この原理を利用して、シークレットロックしたアカウントにたいしてロック分の費用をサービス提供者が充足しておけば、
期限が過ぎた後ユーザ側がロック分のトークン所有量が増加することになります。
一方で、期限が過ぎる前にシークレット証明トランザクションをアナウンスすると、送信が完了し、サービス提供者に充当戻るためキャンセル扱いとなります。

### アトミックスワップ
シークレットロックを使用して、他のチェーンとのトークン・モザイクの交換を行うことができます。
他のチェーンではハッシュタイムロックコントラクト(HTLC)と呼ばれているためハッシュロックと間違えないようにご注意ください。


