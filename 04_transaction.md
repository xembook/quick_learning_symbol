ブロックチェーンのデータを更新するためには、アカウントの秘密鍵で署名したトランザクションをアナウンスする必要があります。


## トランザクションのライフサイクル

- トランザクション作成
  - ブロックチェーンが受理できるフォーマットでトランザクションを作成します。
- 署名
  - アカウントの秘密鍵でトランザクションを署名します。
- アナウンス
  - 任意のノードに署名済みトランザクションを通知します。
- 未承認トランザクション
  - ノードに受理されたトランザクションは、未承認トランザクションとして全ノードに伝播します
    - トランザクションに設定した最大手数料が、各ノード毎に設定されている最低手数料を満たさない場合はそのノードへは伝播しません。
- 承認済みトランザクション
  - 約30秒に1度ごとに生成されるブロックに未承認トランザクションが取り込まれると、承認済みトランザクションとなります。
- ロールバック
  - ノード間の合意に達することができずロールバックされたブロックに含まれていたトランザクションは、未承認トランザクションに差し戻されます。
    - 有効期限切れや、キャッシュからあふれたトランザクションは切り捨てられます
- ファイナライズ
  - 投票ノードによるファイナライズプロセスによるブロックが確定するとトランザクションはロールバック不可なデータとして扱うことができます。


## トランザクション作成

まずは最も基本的な転送トランザクションを作成してみます。

### Bobへの転送トランザクション
```js
bobAddress = sym.Address.createFromRawAddress("アドレス");
tx1 = sym.TransferTransaction.create(
    sym.Deadline.create(epochAdjustment),
    bobAddress, 
    [],
    sym.PlainMessage.create("Hello Symbol!"),
    networkType
).setMaxFee(100);

```

#### 有効期限
sdkではデフォルトで2時間後に設定されます。
最大48時間まで指定可能です。
```js
sym.Deadline.create(epochAdjustment,48)
```

#### メッセージ
トランザクションに最大1023バイトのメッセージを添付することができます。
バイナリデータであってもrawdataとして送信することが可能です。

##### 空メッセージ
```js
sym.EmptyMessage
```

##### 平文メッセージ
```js
sym.PlainMessage.create("Hello Symbol!")
```

##### 暗号文メッセージ
```js
sym.EncryptedMessage('test');
```

##### 生データ
```js
sym.RawMessage.create(uint8Arrays[i])
```

#### 最大手数料

ネットワーク手数料については、常に少し多めに払っておけば問題はないのですが、最低限の知識は持っておく必要があります。
アカウントはトランザクションを作成するときに、ここまでは手数料として払ってもいいという最大手数料を指定します。
一方で、ノードはその時々で最も高い手数料となるトランザクションのみブロックにまとめて収穫しようとします。
つまり、多く払ってもいいというトランザクションが他に多く存在すると承認されるまでの時間が長くなります。
逆に、より少なく払いたいというトランザクションが多く存在し、その総額が大きい場合は、設定した最大額に満たない手数料額で送信が実現します。

トランザクションサイズ x feeMultiprilerというもので決定されます。
176バイトだった場合 maxFee を100で設定すると 176000μXYM = 0.176XYMを手数料として許容します。
feeMultiprier = 100として指定する方法と
maxFee = 176000 として指定する方法があります。

##### feeMultiprier = 100として指定する方法
```js
tx = sym.TransferTransaction.create(
  ,,,,
  networkType
).setMaxFee(100);
```

##### maxFee = 176000 として指定する方法
```js
tx = sym.TransferTransaction.create(
  ,,,,
  networkType,
  sym.UInt64.fromUint(176000)
);
```

本書では以後、feeMultiprier = 100として指定する方法で統一して説明します。

## 署名とアナウンス

作成したトランザクションを秘密鍵で署名して、任意のノードを通じてアナウンスします。

### 署名
```js
signedTx = alice.sign(tx,generationHash);
console.log(signedTx);

> SignedTransaction
    hash: "3BD00B0AF24DE70C7F1763B3FD64983C9668A370CB96258768B715B117D703C2"
    networkType: 152
    payload:        
"AE00000000000000CFC7A36C17060A937AFE1191BC7D77E33D81F3CC48DF9A0FFE892858DFC08C9911221543D687813ECE3D36836458D2569084298C09223F9899DF6ABD41028D0AD4933FC1E4C56F9DF9314E9E0533173E1AB727BDB2A04B59F048124E93BEFBD20000000001985441F843000000000000879E76C702000000986F4982FE77894ABC3EBFDC16DFD4A5C2C7BC05BFD44ECE0E000000000000000048656C6C6F2053796D626F6C21"
    signerPublicKey: "D4933FC1E4C56F9DF9314E9E0533173E1AB727BDB2A04B59F048124E93BEFBD2"
    type: 16724
```

トランザクションの署名にはAccountクラスとgenerationHash値が必要です。

##### generationHash(テストネット)
```js
'7FCCD304802016BEBBCD342A332F91FF1F3BB5E902988B352697BE245F48E836'
```

##### generationHash(メインネット)
```js
'57F7DA205008026C776CB6AED843393F04CD458E0AA2D9F1D5F31A402072B2D6'
```

generationHash値はそのブロックチェーンネットワークを一意に識別するための値です。
同じ秘密鍵をもつ他のネットワークに使いまわされないようにそのネットワーク個別のハッシュ値を織り交ぜて署名済みトランザクションを作成します。


### アナウンス
```js
res = await txRepo.announce(signedTx).toPromise();
console.log(res);

> TransactionAnnounceResponse {message: 'packet 9 was pushed to the network via /transactions'}
```

上記のスクリプトのように `packet n was pushed to the network` というレスポンスがあれば、トランザクションはノードに受理されたことになります。
これはトランザクションのフォーマット等に異常が無かった程度の意味しかありません。
Symbolではノードの応答速度を極限に高めるため、トランザクションの内容を検証するまえに受信結果の応答を返し接続を切断します。
レスポンス値はこの情報を受け取ったにすぎません。フォーマットに異常があった場合は以下のようなメッセージ応答があります。

##### アナウンスに失敗した場合の応答例
```js
Uncaught Error: {"statusCode":409,"statusMessage":"Unknown Error","body":"{\"code\":\"InvalidArgument\",\"message\":\"payload has an invalid format\"}"}
```

## 確認


### ステータスの確認

ノードに受理されたトランザクションのステータスを確認

```js
transactionStatus = await tsRepo.getTransactionStatus(signedTx.hash).toPromise();
console.log(transactionStatus);

> TransactionStatus
    group: "confirmed"
    code: "Success"
    deadline: Deadline {adjustedValue: 11989512431}
    hash: "661360E61C37E156B0BE18E52C9F3ED1022DCE846A4609D72DF9FA8A5B667747"
    height: undefined
```

承認されると ` group: "confirmed"`となっています。

受理されたものの、エラーが発生していた場合は以下のような出力となります。
トランザクションを書き直して再度アナウンスしてみてください。

```js
> TransactionStatus
    group: "failed"
    code: "Failure_Core_Insufficient_Balance"
    deadline: Deadline {adjustedValue: 11990156766}
    hash: "A82507C6C46DF444E36AC94391EA2D0D7DD1A218948DED465A7A4F9D1B53CA0E"
    height: undefined
```

### 承認確認


トランザクションがブロックに承認されるまでに30秒程度かかります。

### エクスプローラーで確認
signedTx.hash で取得できるハッシュ値を使ってエクスプローラーで検索してみましょう。

```js
console.log(signedTx.hash);

> "661360E61C37E156B0BE18E52C9F3ED1022DCE846A4609D72DF9FA8A5B667747"
```

- メインネット　
  - https://symbol.fyi/transactions/661360E61C37E156B0BE18E52C9F3ED1022DCE846A4609D72DF9FA8A5B667747
- テストネット　
  - https://testnet.symbol.fyi/transactions/661360E61C37E156B0BE18E52C9F3ED1022DCE846A4609D72DF9FA8A5B667747


### 注意点

トランザクションはブロックで承認されたとしても、ロールバックが発生するとトランザクションの承認が取り消される場合があります。
ブロックが承認された後、数ブロックの承認が進むと、ロールバックの発生する確率は減少していきます。
また、Votingノードの投票で実施されるファイナライズブロックを待つことで、記録されたデータは確実なものとなります。

## トランザクション履歴

```js
result = await txRepo.search(
  {
    group:sym.TransactionGroup.Confirmed,
    embedded:true,
    address:alice.address
  }
).toPromise();

txes = result.data;
txes.forEach(tx => {
  console.log(tx);
})

> TransferTransaction
    type: 16724
    networkType: 152
    payloadSize: 176
    deadline: Deadline {adjustedValue: 11905303680}
    maxFee: UInt64 {lower: 200000000, higher: 0}
    recipientAddress: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
    signature: "E5924A1EB653240A7220405A4DD4E221E71E43327B3BA691D267326FEE3F57458E8721907188DB33A3F2A9CB1D0293845B4D0F1D7A93C8A3389262D1603C7108"
    signer: PublicAccount {publicKey: 'BDFAF3B090270920A30460AA943F9D8D4FCFF6741C2CB58798DBF7A2ED6B75AB', address: Address}
  > message: RawMessage
      payload: ""
      type: -1
  > mosaics: Array(1)
      0: Mosaic
        amount: UInt64 {lower: 10000000, higher: 0}
        id: MosaicId
          id: Id {lower: 760461000, higher: 981735131}
  > transactionInfo: TransactionInfo
      hash: "308472D34BE1A58B15A83B9684278010F2D69B59E39127518BE38A4D22EEF31D"
      height: UInt64 {lower: 301717, higher: 0}
      id: "6255242053E0E706653116F9"
      index: 0
      merkleComponentHash: "308472D34BE1A58B15A83B9684278010F2D69B59E39127518BE38A4D22EEF31D"
```

###### TransactionType
```json
{0: 'RESERVED', 16705: 'AGGREGATE_COMPLETE', 16707: 'VOTING_KEY_LINK', 16708: 'ACCOUNT_METADATA', 16712: 'HASH_LOCK', 16716: 'ACCOUNT_KEY_LINK', 16717: 'MOSAIC_DEFINITION', 16718: 'NAMESPACE_REGISTRATION', 16720: 'ACCOUNT_ADDRESS_RESTRICTION', 16721: 'MOSAIC_GLOBAL_RESTRICTION', 16722: 'SECRET_LOCK', 16724: 'TRANSFER', 16725: 'MULTISIG_ACCOUNT_MODIFICATION', 16961: 'AGGREGATE_BONDED', 16963: 'VRF_KEY_LINK', 16964: 'MOSAIC_METADATA', 16972: 'NODE_KEY_LINK', 16973: 'MOSAIC_SUPPLY_CHANGE', 16974: 'ADDRESS_ALIAS', 16976: 'ACCOUNT_MOSAIC_RESTRICTION', 16977: 'MOSAIC_ADDRESS_RESTRICTION', 16978: 'SECRET_PROOF', 17220: 'NAMESPACE_METADATA', 17229: 'MOSAIC_SUPPLY_REVOCATION', 17230: 'MOSAIC_ALIAS', 17232: 'ACCOUNT_OPERATION_RESTRICTION'
```

###### MessageType
```json
{0: 'PlainMessage', 1: 'EncryptedMessage', 254: 'PersistentHarvestingDelegationMessage', -1: 'RawMessage'}
```
## アグリゲートトランザクション

Symbolでは複数のトランザクションをリストにしてまとめて1ブロックで承認
以降の章で扱う内容にアグリゲートトランザクションへの理解が必要な機能が含まれますので、
本章ではアグリゲートトランザクションのうち、簡単なものだけを紹介します。
### 起案者の署名だけが必要な場合

```js
bobAddress = sym.Address.createFromRawAddress("受信アドレス");

innerTx1 = sym.TransferTransaction.create(
    undefined,
    bobAddress, 
    [],
    sym.PlainMessage.create("tx1"),
    0
);

innerTx2 = sym.TransferTransaction.create(
    undefined,
    bobAddress, 
    [],
    sym.PlainMessage.create("tx2"),
    0
);

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [
      innerTx1.toAggregate(alice.publicAccount),
      innerTx2.toAggregate(alice.publicAccount)
    ],
    networkType,
    [],
    sym.UInt64.fromUint(1000000)
);
signedTx = alice.sign(aggregateTx,generationHash);
txRepo.announce(signedTx).subscribe(x=>console.log(x));

```

## 現場で使えるヒント

(現在執筆中)
