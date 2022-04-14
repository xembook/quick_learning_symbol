アカウントは秘密鍵に紐づく情報が記録されたデータ構造体です。  
アカウントと関連づいた秘密鍵を使って署名することでのみブロックチェーンのデータを更新することができます。

## アカウント生成

アカウントには秘密鍵と公開鍵をセットにしたキーペア、アドレスなどの情報が含まれています。
まずはランダムにアカウントを作成して、それらの情報を確認してみましょう。

### 新規生成
```js
alice = sym.Account.generateNewAccount(networkType);
console.log(alice);

> Account
    address: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
    keyPair: {privateKey: Uint8Array(32), publicKey: Uint8Array(32)}
```

networkTypeは以下の通りです。
```js
{104: 'MAIN_NET', 152: 'TEST_NET'}
```

### 秘密鍵と公開鍵の導出
```js
console.log(alice.privateKey);
console.log(alice.publicKey);
> 1E9139CC1580B4AED6A1FE110085281D4982ED0D89CE07F3380EB83069B1****
> D4933FC1E4C56F9DF9314E9E0533173E1AB727BDB2A04B59F048124E93BEFBD2
```

#### 注意事項
秘密鍵を紛失するとそのアカウントに紐づけられたデータを操作することが出来なくなります。
また、他人は知らないという秘密鍵の性質を利用してデータ操作の署名を行うので、秘密鍵を他人に教えてはいけません。
一般的なWebサービスでは「アカウントID」に対してパスワードが割り振られるため、パスワードの変更が可能ですが、
ブロックチェーンではパスワードにあたる秘密鍵に対して一意に決まるID(アドレス)が割り振られるため、
アカウントに紐づく秘密鍵を変更するということはできません。


### アドレスの導出
```js
aliceRawAddress = alice.address.plain();
console.log(aliceRawAddress);

> TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ
```

これらがブロックチェーンを操作するための最も基本的な情報となります。
また、秘密鍵からアカウントを生成したり、公開鍵やアドレスのみを扱うクラスの生成方法も確認しておきましょう。

### 秘密鍵からアカウント生成
```js
alice = sym.Account.createFromPrivateKey(
  "1E9139CC1580B4AED6A1FE110085281D4982ED0D89CE07F3380EB83069B1****",
  networkType
);
```

### 公開鍵クラスの生成
```js
alicePublicAccount = sym.PublicAccount.createFromPublicKey(
  "D4933FC1E4C56F9DF9314E9E0533173E1AB727BDB2A04B59F048124E93BEFBD2",
  networkType
);
console.log(alicePublicAccount);

> PublicAccount
    address: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
    publicKey: "D4933FC1E4C56F9DF9314E9E0533173E1AB727BDB2A04B59F048124E93BEFBD2"

```

### アドレスクラスの生成
```js
aliceAddress = sym.Address.createFromRawAddress(
  "TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ"
);
console.log(aliceAddress);

> Address
    address: "TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ"
    networkType: 152
```

## アカウントへの送信

アカウントを作成しただけでは、ブロックチェーンにデータを送信することはできません。
パブリックブロックチェーンはリソースを有効活用するためにデータ送信時に手数料を要求します。
Symbolブロックチェーンでは、この手数料をXYMという共通トークンで支払うことになります。
アカウントを生成したら、この後の章から説明するトランザクションを実行するために必要な手数料を送信しておきます。

### フォーセットから送信

テストネットではフォーセット（蛇口）サービスから検証用のXYMを入手することができます。
メインネットの場合は取引所などでXYMを購入するか、投げ銭サービス(NEMLOG,QUEST)などを利用して寄付を募りましょう。

テストネット
- FAUCET(蛇口)
  - https://testnet.symbol.tools/

メインネット
- NEMLOG
  - https://nemlog.nem.social/
- QUEST
  - https://quest-bc.com/



### エクスプローラーで確認

フォーセットから作成したアカウントへ送信が成功したらエクスプローラーで確認してみましょう。

テストネット
  https://testnet.symbol.fyi/
メインネット
  https://symbol.fyi/

## アカウント情報の確認

ノードに保存されているアカウント情報を取得します。

### 所有モザイク一覧の取得

```js
accountRepo = repo.createAccountRepository();
accountInfo = await accountRepo.getAccountInfo(aliceAddress).toPromise();
console.log(accountInfo);

> AccountInfo
    address: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
    publicKey: "0000000000000000000000000000000000000000000000000000000000000000"
  > mosaics: Array(1)
      0: Mosaic
        amount: UInt64 {lower: 10000000, higher: 0}
        id: MosaicId
          id: Id {lower: 760461000, higher: 981735131}
```

#### publicKey
クライアント側で作成しただけで、ブロックチェーンでまだ利用されていないアカウント情報は記録されていません。
宛先として指定されて受信することで初めてアカウント情報が記録され、署名したトランザクションを送信することで公開鍵の情報が記録されます。
そのため、publicKeyは現在`00000...`表記となっています。

#### UInt64
JavaScriptでは大きすぎる数値はあふれてしまうため、idやamountはUInt64というフォーマットで管理されています。
文字列に変換する場合は toString()、数値に変換する場合は compact()、16進数にする場合は toHex() で変換してください。

```js
accountInfo.mosaics.forEach(async mosaic => {
  console.log("id:" + mosaic.id.toHex());
  console.log("amount:" + mosaic.amount.toString());
  mosaicInfo = await mosaicRepo.getMosaic(mosaic.id).toPromise();
});
```

#### 表示桁数の調整

所有するトークンの量は誤差の発生を防ぐため、整数値で扱います。
トークンの定義から可分性を取得することができるので、その値を使って正確な所有量を表示してみます。

```js
mosaicAmount = mosaics[0].amount.toString();
mosaicInfo = await mosaicRepo.getMosaic(mosaics[0].id).toPromise();
divisibility = mosaicInfo.divisibility; //可分性
if(divisibility > 0){
  displayAmount = mosaicAmount.slice(0,mosaicAmount.length-divisibility)  
  + "." + mosaicAmount.slice(-divisibility);
}else{
  displayAmount = mosaicAmount;
}
console.log(displayAmount);
```

## 現場で使えるヒント
### 暗号化と署名

アカウントとして生成した秘密鍵や公開鍵は、そのまま従来の暗号化や電子署名として活用することができます。
信頼性に問題点があるアプリケーションを使用する必要がある場合も、個人間でデータの秘匿性・正当性を検証することができます。



#### 事前準備：対話のためのBobアカウントを生成
```js
bob = sym.Account.generateNewAccount(networkType);
bobPublicAccount = bob.publicAccount;
```

#### 暗号化

Aliceの公開鍵・Bobの公開鍵で暗号化し、Aliceの公開鍵・Bobの秘密鍵で復号します。

```js
message = 'Hello Symol!';
encryptedMessage = alice.encryptMessage(message ,bob.publicAccount);
console.log(encryptedMessage);

> 294C8979156C0D941270BAC191F7C689E93371EDBC36ADD8B920CF494012A97BA2D1A3759F9A6D55D5957E9D
```

#### 復号化
```js
decryptMessage = bob.decryptMessage(
  new nem.EncryptedMessage(
    "294C8979156C0D941270BAC191F7C689E93371EDBC36ADD8B920CF494012A97BA2D1A3759F9A6D55D5957E9D"
  ),
  alice.publicAccount
).payload
console.log(decryptMessage);

>"Hello Symol!"
```

#### 署名

Aliceの秘密鍵でメッセージを署名し、Aliceの公開鍵と署名でメッセージを検証します。

```js
payload = sym.Convert.hexToUint8(sym.Convert.utf8ToHex("Hello Symol!"));
signature = sym.Convert.utf8ToHex(sym.KeyPair.sign(alice.keyPair, payload));
console.log(signature);

> "7FCF7614D60BF4506DEC7D1B55AA1A0972881FA8FC40D0D0FBAAAF77E036E7988270E57631D49FCE253C4FB992E17C9360FA948B6626B2954E7DAB84DA7D6E01"
```

#### 検証
```js
isVerified = sym.KeyPair.verify(
  alice.keyPair.publicKey,
  sym.Convert.hexToUint8(nem.Convert.utf8ToHex("Hello Symbol!")),
  sym.Convert.hexToUint8( 
 "7FCF7614D60BF4506DEC7D1B55AA1A0972881FA8FC40D0D0FBAAAF77E036E7988270E57631D49FCE253C4FB992E17C9360FA948B6626B2954E7DAB84DA7D6E01")
)
console.log(isVerified)

> true
```

### アカウントの保管

アカウントの管理方法について説明しておきます。
秘密鍵はそのままで保存しないようにしてください。
symbol-qr-libraryを利用して秘密鍵をパスフレーズで暗号化して保存する方法を紹介します。

#### 秘密鍵の暗号化

```js
qr = require("/node_modules/symbol-qr-library");

//パスフレーズでロックされたアカウント生成
signerQR = qr.QRCodeGenerator.createExportAccount(
  alice.privateKey, networkType, generationHash, "パスフレーズ"
);

//QRコード表示
signerQR.toBase64().subscribe(x =>{

  //HTML body上にQRコードを表示する例
  (tag= document.createElement('img')).src = x;
  document.getElementsByTagName('body')[0].appendChild(tag);
});

//アカウントを暗号化したJSONデータとして表示
console.log(signerQR.toJSON());

> {"v":3,"type":2,"network_id":152,"chain_id":"7FCCD304802016BEBBCD342A332F91FF1F3BB5E902988B352697BE245F48E836",
    "data":{
      "ciphertext":
        "6d782997d69d293a9205235ad23a5fb13ALz7Mdb7vRYot40WgaCIWMedlN8kCAAjqnzl0po8khczUuzq3+445rOX+1+igZ+YlWD84VCNJqs4m//eKMGMF5ZlOC4gYXWsGOztbGTwyk=",
      "salt":"3df82ffe3378951a8633fed212654ec68bd6ced651121e11d2e77d134148188d"
    }
  }
```

#### 暗号化された秘密鍵の復号

```js
qr = require("/node_modules/symbol-qr-library");
signerQR = qr.AccountQR.fromJSON({暗号化された秘密鍵のJSONデータ},{パスフレーズ});
console.log(signerQR.accountPrivateKey);

> 1E9139CC1580B4AED6A1FE110085281D4982ED0D89CE07F3380EB83069B1****
```

