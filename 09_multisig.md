# 9.マルチシグ化
アカウントのマルチシグ化について説明します。

### 注意事項

一つのマルチシグアカウントに登録できる連署者の数は25個です。
一つのアカウントは最大25個のマルチシグの連署者になれます。
マルチシグは最大3階層まで構成できます。
本書では1階層のマルチシグのみ解説します。

## 9.0 アカウントの準備
この章のサンプルソースコードで使用するアカウントを準備しておきます。

```js
bob = sym.Account.generateNewAccount(networkType);
carol1 = sym.Account.generateNewAccount(networkType);
carol2 = sym.Account.generateNewAccount(networkType);
carol3 = sym.Account.generateNewAccount(networkType);
carol4 = sym.Account.generateNewAccount(networkType);
carol5 = sym.Account.generateNewAccount(networkType);
```

テストネットの場合はFAUCETでネットワーク手数料分をbobとcarol1に補給しておきます。

- Faucet
    - https://testnet.symbol.tools/

##### URL出力
```js
console.log("https://testnet.symbol.tools/?recipient=" + bob.address.plain() +"&amount=10");
console.log("https://testnet.symbol.tools/?recipient=" + carol1.address.plain() +"&amount=10");
```

## 9.1 マルチシグの登録

Symbolではマルチシグアカウントを新規に作成するのではなく、既存アカウントについて連署者を指定してマルチシグ化します。
マルチシグ化には連署者に指定されたアカウントの承諾署名が必要なため、アグリゲートトランザクションを使用します。

```js
multisigTx = sym.MultisigAccountModificationTransaction.create(
    undefined, 
    3, //minApproval:承認のために必要な最小署名者数増分
    3, //minRemoval:除名のために必要な最小署名者数増分
    [
        carol1.address,carol2.address,carol3.address
    ], //追加対象アドレスリスト
    [],//除名対象アドレスリスト
    networkType
);

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [//マルチシグ化したいアカウントの公開鍵を指定
      multisigTx.toAggregate(bob.publicAccount),
    ],
    networkType,[]
).setMaxFeeForAggregate(100, 3);

signedTx =  aggregateTx.signTransactionWithCosignatories(
    bob, //マルチシグ化したいアカウント
    [carol1,carol2,carol3], //追加・除外対象として指定したアカウント
    generationHash,
);
await txRepo.announce(signedTx).toPromise();
```

## 9.2 確認

### マルチシグ化したアカウントの確認
```js
multisigInfo = await msigRepo.getMultisigAccountInfo(bob.address).toPromise();
console.log(multisigInfo);

> MultisigAccountInfo 
    accountAddress: Address {address: 'TCOMA5VG67TZH4X55HGZOXOFP7S232CYEQMOS7Q', networkType: 152}
  > cosignatoryAddresses: Array(3)
        0: Address {address: 'TBAFGZOCB7OHZCCYYV64F2IFZL7SOOXNDHFS5NY', networkType: 152}
        1: Address {address: 'TB3XP4GQK6XH2SSA2E2U6UWCESNACK566DS4COY', networkType: 152}
        2: Address {address: 'TCV67BMTD2JMDQOJUDQHBFJHQPG4DAKVKST3YJI', networkType: 152}
    minApproval: 3
    minRemoval: 3
    multisigAddresses: []
```

cosignatoryAddressesが連署者として登録されていることがわかります。
また、minApproval:1 によりトランザクションが成立するために必要な署名数１
minRemoval: 1により連署者を取り外すために必要な署名者数は1であることがわかります。

### 連署者アカウントの確認
```js
multisigInfo = await msigRepo.getMultisigAccountInfo(carol1.address).toPromise();
console.log(multisigInfo);

> MultisigAccountInfo
    accountAddress: Address {address: 'TCV67BMTD2JMDQOJUDQHBFJHQPG4DAKVKST3YJI', networkType: 152}
    cosignatoryAddresses: []
    minApproval: 0
    minRemoval: 0
  > multisigAddresses: Array(1)
        0: Address {address: 'TCOMA5VG67TZH4X55HGZOXOFP7S232CYEQMOS7Q', networkType: 152}
```

multisigAddresses に対して連署する権利を持っていることが分かります。

## 9.3 マルチシグ署名

マルチシグ化したアカウントからモザイクを送信します。

### アグリゲートコンプリートトランザクションで送信

```js
tx = sym.TransferTransaction.create(
    undefined,
    alice.address, 
    [networkCurrency.createRelative(1)],
    sym.PlainMessage.create('test'),
    networkType
);

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
     [//マルチシグ化したアカウントの公開鍵を指定
       tx.toAggregate(bob.publicAccount)
     ],
    networkType,[],
).setMaxFeeForAggregate(100, 2);

signedTx =  aggregateTx.signTransactionWithCosignatories(
    carol1, //起案者
    [carol2,carol3],　//連署者
    generationHash,
);
await txRepo.announce(signedTx).toPromise();
```

### アグリゲートボンデッドトランザクションで送信

アグリゲートボンデッドトランザクションの場合は連署者を指定せずにアナウンスできます。

```js
tx = sym.TransferTransaction.create(
    undefined,
    alice.address, //Aliceへの送信
    [networkCurrency.createRelative(1)], //1XYM
    sym.PlainMessage.create('test'),
    networkType
);

aggregateTx = sym.AggregateTransaction.createBonded(
    sym.Deadline.create(epochAdjustment),
     [ //マルチシグ化したアカウントの公開鍵を指定
       tx.toAggregate(bob.publicAccount)
     ],
    networkType,[],
).setMaxFeeForAggregate(100, 2);

signedAggregateTx = carol1.sign(aggregateTx, generationHash);

hashLockTx = sym.HashLockTransaction.create(
  sym.Deadline.create(epochAdjustment),
	networkCurrency.createRelative(10), //固定値:10XYM
	sym.UInt64.fromUint(480),
	signedAggregateTx,
	networkType
).setMaxFee(100);

signedLockTx = carol1.sign(hashLockTx, generationHash);

//ハッシュロックTXをアナウンス
txRepo.announce(signedLockTx);
//ハッシュロック承認後、ボンデッドTXをアナウンス
txRepo.announceAggregateBonded(signedAggregateTx);
```


## 9.4 マルチシグ送信の確認

マルチシグで行った送信トランザクションの結果を確認してみます。

```js
txInfo = await txRepo.getTransaction(signedTx.hash,sym.TransactionGroup.Confirmed).toPromise();
console.log(txInfo);

> AggregateTransaction
  > cosignatures: Array(2)
		0: AggregateTransactionCosignature
			signature: "554F3C7017C32FD4FE67C1E5E35DD21D395D44742B43BD1EF99BC8E9576845CDC087B923C69DB2D86680279253F2C8A450F97CC7D3BCD6E86FE4E70135D44B06"
			signer: PublicAccount
				address: Address {address: 'TB3XP4GQK6XH2SSA2E2U6UWCESNACK566DS4COY', networkType: 152}
				publicKey: "A1BA266B56B21DC997D637BCC539CCFFA563ABCB34EAA52CF90005429F5CB39C"
		1: AggregateTransactionCosignature
			signature: "AD753E23D3D3A4150092C13A410D5AB373B871CA74D1A723798332D70AD4598EC656F580CB281DB3EB5B9A7A1826BAAA6E060EEA3CC5F93644136E9B52006C05"
			signer: PublicAccount
				address: Address {address: 'TBAFGZOCB7OHZCCYYV64F2IFZL7SOOXNDHFS5NY', networkType: 152}
				publicKey: "B00721EDD76B24E3DDCA13555F86FC4BDA89D413625465B1BD7F347F74B82FF0"
	deadline: Deadline {adjustedValue: 12619660047}
  > innerTransactions: Array(1)
	  >	0: TransferTransaction
			deadline: Deadline {adjustedValue: 12619660047}
			maxFee: UInt64 {lower: 48000, higher: 0}
			message: PlainMessage {type: 0, payload: 'test'}
			mosaics: [Mosaic]
			networkType: 152
			payloadSize: undefined
			recipientAddress: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
			signature: "670EA8CFA4E35604DEE20877A6FC95C2786D748A8449CE7EEA7CB941FE5EC181175B0D6A08AF9E99955640C872DAD0AA68A37065C866EE1B651C3CE28BA95404"
			signer: PublicAccount
				address: Address {address: 'TCOMA5VG67TZH4X55HGZOXOFP7S232CYEQMOS7Q', networkType: 152}
				publicKey: "4667BC99B68B6CA0878CD499CE89CDEB7AAE2EE8EB96E0E8656386DECF0AD657"
			transactionInfo: AggregateTransactionInfo {height: UInt64, index: 0, id: '62600A8C0A21EB5CD28679A4', hash: undefined, merkleComponentHash: undefined, …}
			type: 16724
	maxFee: UInt64 {lower: 48000, higher: 0}
	networkType: 152
	payloadSize: 480
	signature: "670EA8CFA4E35604DEE20877A6FC95C2786D748A8449CE7EEA7CB941FE5EC181175B0D6A08AF9E99955640C872DAD0AA68A37065C866EE1B651C3CE28BA95404"
  >	signer: PublicAccount
		address: Address {address: 'TCV67BMTD2JMDQOJUDQHBFJHQPG4DAKVKST3YJI', networkType: 152}
		publicKey: "FF9595FDCD983F46FF9AE0F7D86D94E9B164E385BD125202CF16528F53298656"
  >	transactionInfo: 
		hash: "AA99F8F4000F989E6F135228829DB66AEB3B3C4B1F06BA77D373D042EAA4C8DA"
		height: UInt64 {lower: 322376, higher: 0}
		id: "62600A8C0A21EB5CD28679A3"
		merkleComponentHash: "1FD6340BCFEEA138CC6305137566B0B1E98DEDE70E79CC933665FE93E10E0E3E"
	type: 16705
```

- マルチシグアカウント
    - Bob
        - AggregateTransaction.innerTransactions[0].signer.address
            - TCOMA5VG67TZH4X55HGZOXOFP7S232CYEQMOS7Q
- 起案者アカウント
    - Carol1
        - AggregateTransaction.signer.address
            - TCV67BMTD2JMDQOJUDQHBFJHQPG4DAKVKST3YJI
- 連署者アカウント
    - Carol2
        - AggregateTransaction.cosignatures[0].signer.address
            - TB3XP4GQK6XH2SSA2E2U6UWCESNACK566DS4COY
    - Carol3
        - AggregateTransaction.cosignatures[1].signer.address
            - TBAFGZOCB7OHZCCYYV64F2IFZL7SOOXNDHFS5NY

## 9.5 マルチシグ構成変更

### マルチシグ構成の縮小

連署者を減らすには除名対象アドレスに指定するとともに最小署名者数を連署者数が超えてしまわないように調整してトランザクションをアナウンスします。
除名対象者を連署者に含む必要はありません。

```js
multisigTx = sym.MultisigAccountModificationTransaction.create(
    undefined, 
    -1, //承認のために必要な最小署名者数増分
    -1, //除名のために必要な最小署名者数増分
    [], //追加対象アドレス
    [carol3.address],//除名対象アドレス
    networkType
);

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [ //構成変更したいマルチシグアカウントの公開鍵を指定
      multisigTx.toAggregate(bob.publicAccount),
    ],
    networkType,[]    
).setMaxFeeForAggregate(100, 1);

signedTx =  aggregateTx.signTransactionWithCosignatories(
    carol1,
    [carol2],
    generationHash,
);
await txRepo.announce(signedTx).toPromise();
```

### 連署者構成の差替え

連署者を差し替えるには、追加対象アドレスと除名対象アドレスを指定します。
新たに追加指定するアカウントの連署は必ず必要です。

```js
multisigTx = sym.MultisigAccountModificationTransaction.create(
    undefined, 
    0, //承認のために必要な最小署名者数増分
    0, //除名のために必要な最小署名者数増分
    [carol4.address], //追加対象アドレス
    [carol3.address], //除名対象アドレス
    networkType
);

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [ //構成変更したいマルチシグアカウントの公開鍵を指定
      multisigTx.toAggregate(bob.publicAccount),
    ],
    networkType,[]    
).setMaxFeeForAggregate(100, 2);

signedTx =  aggregateTx.signTransactionWithCosignatories(
    carol1, //起案者
    [carol2,carol4], //連署者+承諾アカウント
    generationHash,
);
await txRepo.announce(signedTx).toPromise();
```

## 9.6 現場で使えるヒント

後日執筆予定
