ブロックチェーンの情報を検証する方法について説明します。

## 13.1 トランザクションの検証

トランザクションがブロックに含まれていることを検証します。

### 検証するペイロード

このペイロードの示すトランザクションが指定したブロック高(59639)でブロックチェーンに記録されていることを確認します。

```js
payload = 'C00200000000000093B0B985101C1BDD1BC2BF30D72F35E34265B3F381ECA464733E147A4F0A6B9353547E2E08189EF37E50D271BEB5F09B81CE5816BB34A153D2268520AF630A0A0E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26000000000198414140770200000000002A769FB40000000076B455CFAE2CCDA9C282BF8556D3E9C9C0DE18B0CBE6660ACCF86EB54AC51B33B001000000000000DB000000000000000E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26000000000198544198205C1A4CE06C45B3A896B1B2360E03633B9F36BF7F22338B000000000000000066653465353435393833444430383935303645394533424446434235313637433046394232384135344536463032413837364535303734423641303337414643414233303344383841303630353343353345354235413835323835443639434132364235343233343032364244444331443133343139464435353438323930334242453038423832304100000000006800000000000000B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D000000000198444198205C1A4CE06C45B3A896B1B2360E03633B9F36BF7F2233BC089179EBBE01A81400140035383435344434373631364336433635373237396800000000000000B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D000000000198444198205C1A4CE06C45B3A896B1B2360E03633B9F36BF7F223345ECB996EDDB9BEB1400140035383435344434373631364336433635373237390000000000000000B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D5A71EBA9C924EFA146897BE6C9BB3DACEFA26A07D687AC4A83C9B03087640E2D1DDAE952E9DDBC33312E2C8D021B4CC0435852C0756B1EBD983FCE221A981D02';
height = 59639;
```


### マークルコンポーネントハッシュの計算

トランザクションのハッシュ値には連署者の情報が含まれいません。  
一方でブロックヘッダーに格納されるマークルルートはトランザクションのハッシュに連署者の情報が含まれたものが格納されます。  


```js
Buffer = require("/node_modules/buffer").Buffer;
hash = sym.Transaction.createTransactionHash(payload,sym.Convert.hexToUint8(generationHash));
tx = sym.TransactionMapping.createFromPayload(payload);

var merkleComponentHash = hash;
if( tx.cosignatures !== undefined && tx.cosignatures.length > 0){
  
  const hasher = sha3_256.create();
  hasher.update(Buffer.from(hash, 'hex'));
  for (cosignature of tx.cosignatures ){

    hasher.update(Buffer.from(cosignature.signer.publicKey, 'hex'));
  }
  merkleComponentHash = hasher.hex().toUpperCase();
}
console.log(merkleComponentHash);

> C8D1335F07DE05832B702CACB85B8EDAC2F3086543C76C9F56F99A0861E8F235

```


### InBlockの検証

マークルコンポーネントハッシュからブロックから取得したマークルツリーをたどってブロックヘッダーのマークルルートへ着けるか検証します。

```js
function validateTransactionInBlock(leaf,HRoot,merkleProof){

  if (merkleProof.length === 0) {
    // There is a single item in the tree, so HRoot' = leaf.
    return leaf.toUpperCase() === HRoot.toUpperCase();
  }

  const HRoot0 = merkleProof.reduce((proofHash, pathItem) => {
    const hasher = sha3_256.create();
    if (pathItem.position === sym.MerklePosition.Left) {
      return hasher.update(Buffer.from(pathItem.hash + proofHash, 'hex')).hex();
    } else {
      return hasher.update(Buffer.from(proofHash + pathItem.hash, 'hex')).hex();
    }
  }, leaf);
  return HRoot.toUpperCase() === HRoot0.toUpperCase();
}

blockRepo = repo.createBlockRepository();

//トランザクションから計算
leaf = merkleComponentHash.toLowerCase();//merkleComponentHash

//ノードから取得
HRoot = (await blockRepo.getBlockByHeight(height).toPromise()).blockTransactionsHash;
merkleProof = (await blockRepo.getMerkleTransaction(height, leaf).toPromise()).merklePath;

result = validateTransactionInBlock(leaf,HRoot,merkleProof);
console.log(result);

> true
```



## 13.2 ブロックヘッダーの検証

ブロック高1からファイナライズブロックまで、すべてのノードが同じブロックヘッダーを共有しています。  
ファイナライズブロックのハッシュ値計算に含まれる `前ブロックハッシュ値` を改ざんすることは困難なため、  
前ブロックハッシュ値を計算結果とするブロックヘッダーヘッダーの存在が証明できます。  
その仕組みでn番目のブロックからn-1番目のブロックヘッダーの存在を証明していくと、  
特定のトランザクションのマークルルートを含むブロックヘッダーの存在が証明できます。  
これにより特定のトランザクションがブロックチェーンに記録されていることが確認できます。  



###### normalブロックの検証

```js
block = await blockRepo.getBlockByHeight(height).toPromise();
previousBlock = await blockRepo.getBlockByHeight(targetBlockHeight - 1).toPromise();
if(block.type ===  sym.BlockType.NormalBlock){
    
  hasher = sha3_256.create();
  hasher.update(sym.Convert.hexToUint8(block.signature)); //signature
  hasher.update(sym.Convert.hexToUint8(block.signer.publicKey)); //publicKey
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.version, 1));
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.networkType, 1));
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.type, 2));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.height.lower    ,block.height.higher]));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.timestamp.lower ,block.timestamp.higher]));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.difficulty.lower,block.difficulty.higher]));
  hasher.update(sym.Convert.hexToUint8(block.proofGamma));
  hasher.update(sym.Convert.hexToUint8(block.proofVerificationHash));
  hasher.update(sym.Convert.hexToUint8(block.proofScalar));
  hasher.update(sym.Convert.hexToUint8(previousBlock.hash));
  hasher.update(sym.Convert.hexToUint8(block.blockTransactionsHash));
  hasher.update(sym.Convert.hexToUint8(block.blockReceiptsHash));
  hasher.update(sym.Convert.hexToUint8(block.stateHash));
  hasher.update(sym.RawAddress.stringToAddress(block.beneficiaryAddress.address));
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.feeMultiplier, 4));
  hash = hasher.hex().toUpperCase();
  console.log(hash === block.hash);
}
```

###### importanceブロックの検証

```js
block = await blockRepo.getBlockByHeight(height).toPromise();
previousBlock = await blockRepo.getBlockByHeight(targetBlockHeight - 1).toPromise();
if(block.type ===  sym.BlockType.ImportanceBlock){

  hasher = sha3_256.create();
  hasher.update(sym.Convert.hexToUint8(block.signature)); //signature
  hasher.update(sym.Convert.hexToUint8(block.signer.publicKey)); //publicKey
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.version, 1));
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.networkType, 1));
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.type, 2));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.height.lower    ,block.height.higher]));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.timestamp.lower ,block.timestamp.higher]));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.difficulty.lower,block.difficulty.higher]));
  hasher.update(sym.Convert.hexToUint8(block.proofGamma));
  hasher.update(sym.Convert.hexToUint8(block.proofVerificationHash));
  hasher.update(sym.Convert.hexToUint8(block.proofScalar));
  hasher.update(sym.Convert.hexToUint8(previousBlock.hash));
  hasher.update(sym.Convert.hexToUint8(block.blockTransactionsHash));
  hasher.update(sym.Convert.hexToUint8(block.blockReceiptsHash));
  hasher.update(sym.Convert.hexToUint8(block.stateHash));
  hasher.update(sym.RawAddress.stringToAddress(block.beneficiaryAddress.address));
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.feeMultiplier, 4));
  hasher.update(cat.GeneratorUtils.uintToBuffer(block.votingEligibleAccountsCount,4));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.harvestingEligibleAccountsCount.lower,block.harvestingEligibleAccountsCount.higher]));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.totalVotingBalance.lower,block.totalVotingBalance.higher]));
  hasher.update(sym.Convert.hexToUint8(block.previousImportanceBlockHash));

  hash = hasher.hex().toUpperCase();
  console.log(hash === block.hash);
}
```

importanceBlockは、importance値の再計算が行われるブロックです。NormalBlockに加えてvotingEligibleAccountsCount,harvestingEligibleAccountsCount,totalVotingBalance,previousImportanceBlockHashの情報が追加されています。

###### stateHashの検証
```js
hasher = sha3_256.create();

hasher.update(sym.Convert.hexToUint8("4578D33DD0ED5B8563440DA88F627BBC95A174C183191C15EE1672C5033E0572"));
hasher.update(sym.Convert.hexToUint8("2C76DAD84E4830021BE7D4CF661218973BA467741A1FC4663B54B5982053C606"));
hasher.update(sym.Convert.hexToUint8("259FB9565C546BAD0833AD2B5249AA54FE3BC45C9A0C64101888AC123A156D04"));
hasher.update(sym.Convert.hexToUint8("58D777F0AA670440D71FA859FB51F8981AF1164474840C71C1BEB4F7801F1B27"));
hasher.update(sym.Convert.hexToUint8("C9092F0652273166991FA24E8B115ACCBBD39814B8820A94BFBBE3C433E01733"));
hasher.update(sym.Convert.hexToUint8("4B53B8B0E5EE1EEAD6C1498CCC1D839044B3AE5F85DD8C522A4376C2C92D8324"));
hasher.update(sym.Convert.hexToUint8("132324AF5536EC9AA85B2C1697F6B357F05EAFC130894B210946567E4D4E9519"));
hasher.update(sym.Convert.hexToUint8("8374F46FBC759049F73667265394BD47642577F16E0076CBB7B0B9A92AAE0F8E"));
hasher.update(sym.Convert.hexToUint8("45F6AC48E072992343254F440450EF4E840D8386102AD161B817E9791ABC6F7F"));

hash = hasher.hex().toUpperCase();
```
'9D6801C49FE0C31ADE5C1BB71019883378016FA35230B9813CA6BB98F7572758'

## 13.3 アカウント・メタデータの検証

トランザクションに紐づくアカウントやメタデータの存在を検証します。  
トークンの存在や所有についてはトランザクションが承認されることで証明されますが、  
そのトークンを送信するアカウントに紐づく情報、トークンやアカウントに紐づくメタデータに紐づく情報には別途検証が必要です。  
トークンだけではなく、トークンを所有していたアカウントやメタデータに付随する価値が存在する場合、  
トークンを交換する前に検証しておく必要があります。  


### マークルパトリシアツリー

検証にはマークルパトリシアツリーを用います。  
トランザクションの承認の結果として蓄積されたデータの存在をツリー状に格納するためのデータ構造です。  
マークルパトリシアツリーが、データから枝をたどり周知のブロックに帰着することが出来れば  


### 検証用共通関数

```js
//葉のハッシュ値取得関数
function getLeafHash(encodedPath, leafValue){
    const hasher = sha3_256.create();
    return hasher.update(sym.Convert.hexToUint8(encodedPath + leafValue)).hex().toUpperCase();
}

//枝のハッシュ値取得関数
function getBranchHash(encodedPath, links){
    const branchLinks = Array(16).fill(sym.Convert.uint8ToHex(new Uint8Array(32)));
    links.forEach((link) => {
        branchLinks[parseInt(`0x${link.bit}`, 16)] = link.link;
    });
    const hasher = sha3_256.create();
    const bHash = hasher.update(sym.Convert.hexToUint8(encodedPath + branchLinks.join(''))).hex().toUpperCase();
    return bHash;
}

//ワールドステートの検証
function checkState(stateProof,stateHash,pathHash,rootHash){

  const merkleLeaf = stateProof.merkleTree.leaf;
  const merkleBranches = stateProof.merkleTree.branches.reverse();
  const pathArray = [...merkleBranches[0].path];
  const leafHash = getLeafHash(merkleLeaf.encodedPath,stateHash);

  let linkHash = leafHash; //最初のlinkHashはleafHash
  let bit="";
  for(let i = 0; i < merkleBranches.length; i++){
      const branch = merkleBranches[i];
      const branchLink = branch.links.find(x=>x.link === linkHash)
      linkHash = getBranchHash(branch.encodedPath,branch.links);
      bit =  branchLink.bit + bit;
      if(i == 0 && pathArray.length == 2){
        bit = pathArray[0] + bit;
      }
  }
  let leafPath = merkleLeaf.path;
  if(pathArray.length == 2){
    if(merkleBranches.length % 2 == 0){
      leafPath = leafPath.slice( 0, -1 );
    }
  }else if(merkleBranches.length % 2 == 1){
    leafPath = leafPath.slice( 0, -1 );
  }

  const treeRootHash = linkHash; //最後のlinkHashはrootHash
  const treePathHash = bit + leafPath;
  
  //検証
  console.log(treeRootHash === rootHash);
  console.log(treePathHash === pathHash);
//  console.log(treePathHash.indexOf(pathHash) >= 0);
}
```


### アカウント情報の検証


```js
aliceAddress = sym.Address.createFromRawAddress("NCESRRSDSXQW7LTYWMHZOCXAESNNBNNVXHPB6WY");


hasher = sha3_256.create();
alicePathHash = hasher.update(
  sym.RawAddress.stringToAddress(aliceAddress.plain())
).hex().toUpperCase();

hasher = sha3_256.create();
aliceInfo = await accountRepo.getAccountInfo(aliceAddress).toPromise();
aliceStateHash = hasher.update(aliceInfo.serialize()).hex().toUpperCase();

//サービス提供者以外のノードから取得
blockInfo = await blockRepo.search({order:"desc"}).toPromise();
rootHash = blockInfo.data[0].stateHashSubCacheMerkleRoots[0];

//サービス提供者を含む任意のノードから取得
stateProof = await stateProofService.accountById(aliceAddress).toPromise();
checkState(stateProof,aliceStateHash,alicePathHash,rootHash);
```

### メタデータの検証

###### モザイクへ登録したメタデータの検証
```js

srcAddress = sym.Convert.hexToUint8(
    sym.Address.createFromRawAddress("NBWKWT2HMH7ADCZO2GDZAXEXXDISOKQOFSD3YWA").encoded()
)

targetAddress = sym.Convert.hexToUint8(
    sym.Address.createFromRawAddress("NBWKWT2HMH7ADCZO2GDZAXEXXDISOKQOFSD3YWA").encoded()
)

hasher = sha3_256.create();    
hasher.update(srcAddress);
hasher.update(targetAddress);
hasher.update(sym.Convert.hexToUint8Reverse("DA030AA7795EBE75")); // scopeKey
hasher.update(sym.Convert.hexToUint8Reverse("5A4A9E5BF88D7552")); // targetId
hasher.update(Uint8Array.from([sym.MetadataType.Mosaic])); // type: Account 0
compositeHash = hasher.hex();

hasher = sha3_256.create();   
hasher.update(sym.Convert.hexToUint8(compositeHash));
pathHash = hasher.hex().toUpperCase();

//stateHash(Value値)
hasher = sha3_256.create(); 
hasher.update(cat.GeneratorUtils.uintToBuffer(1, 2)); //version
hasher.update(srcAddress);
hasher.update(targetAddress);
hasher.update(sym.Convert.hexToUint8Reverse("DA030AA7795EBE75")); // scopeKey
hasher.update(sym.Convert.hexToUint8Reverse("5A4A9E5BF88D7552")); // targetId
hasher.update(Uint8Array.from([sym.MetadataType.Mosaic])); //account

value = sym.Convert.utf8ToUint8("{\"version\":\"comsa-nft-1.0\",\"name\":\"48415A455F4B5553414B414245\",\"title\":\"4B5553414B414245C39767696D7265\",\"hash\":\"d9e7638f65f622683f92b4c23e71a3d1ee34bd6e0707e34b1df91904e558989e\",\"type\":\"NFT\",\"mime_type\":\"image/jpeg\",\"media\":\"image\",\"address\":\"NCRG57OBWNFCHUCCFOGVURJUGDL5CSCHQNRSI5Y\",\"mosaic\":\"5A4A9E5BF88D7552\",\"endorser\":\"N/A\"}");

hasher.update(cat.GeneratorUtils.uintToBuffer(value.length, 2)); 
hasher.update(value); 
stateHash = hasher.hex();

//サービス提供者以外のノードから取得
blockInfo = await blockRepo.search({order:"desc"}).toPromise();
rootHash = blockInfo.data[0].stateHashSubCacheMerkleRoots[8];

//サービス提供者を含む任意のノードから取得
stateProof = await stateProofService.metadataById(compositeHash).toPromise();
checkState(stateProof,stateHash,pathHash,rootHash);
```

###### アカウントへ登録したメタデータの検証
- Key:F24C633A596D66E7
- Value:546869732069732058454D426F6F6B2066756E64206163636F756E742E


```js
srcAddress = sym.Convert.hexToUint8(
    sym.Address.createFromRawAddress("NCESRRSDSXQW7LTYWMHZOCXAESNNBNNVXHPB6WY").encoded()
)

targetAddress = sym.Convert.hexToUint8(
    sym.Address.createFromRawAddress("NBVHIH5E25AFIRQUYOEMZ35FKEOI275O36YMLZI").encoded()
)

//compositePathHash(Key値)
hasher = sha3_256.create();    
hasher.update(srcAddress);
hasher.update(targetAddress);
hasher.update(sym.Convert.hexToUint8Reverse("F24C633A596D66E7")); // scopeKey
hasher.update(sym.Convert.hexToUint8Reverse("0000000000000000")); // targetId
hasher.update(Uint8Array.from([sym.MetadataType.Account])); // type: Account 0
compositeHash = hasher.hex();

hasher = sha3_256.create();   
hasher.update(sym.Convert.hexToUint8(compositeHash));
pathHash = hasher.hex().toUpperCase();

//stateHash(Value値)
hasher = sha3_256.create(); 
hasher.update(cat.GeneratorUtils.uintToBuffer(1, 2)); //version
hasher.update(srcAddress);
hasher.update(targetAddress);
hasher.update(sym.Convert.hexToUint8Reverse("F24C633A596D66E7")); // scopeKey
hasher.update(sym.Convert.hexToUint8Reverse("0000000000000000")); // targetId
hasher.update(Uint8Array.from([sym.MetadataType.Account])); //account
value = sym.Convert.utf8ToUint8("546869732069732058454D426F6F6B2066756E64206163636F756E742E");
hasher.update(cat.GeneratorUtils.uintToBuffer(value.length, 2)); 
hasher.update(value); 
stateHash = hasher.hex();

//サービス提供者以外のノードから取得
blockInfo = await blockRepo.search({order:"desc"}).toPromise();
rootHash = blockInfo.data[0].stateHashSubCacheMerkleRoots[8];

//サービス提供者を含む任意のノードから取得
stateProof = await stateProofService.metadataById(compositeHash).toPromise();
checkState(stateProof,stateHash,pathHash,rootHash);
```

## 現場で使えるヒント

ノードへのデータ更新は、最終的にはブロックチェーン全体で検証されますが、  
ノードからのデータ抽出は、ブロックチェーン全体からではなく一つのノードからのデータを取得したに過ぎないため、  
自身が管理したノードから取得して完全に信頼できるブラウザで表示したものでない限り、厳密には検証する必要があります。  

- トランザクションが意図したアカウントに署名されたかを検証。
- そのトランザクションが意図したブロックに含まれているかを検証。
- トランザクションが指定するモザイクのメタデータがブロックに含まれるかを検証。
- そのブロックから既知のファイナライズブロックに辿れるかを検証。

すべての検証に成功すれば、メタデータに重要な証明が記されたモザイクの送信が間違いなくブロックチェーンに記録されていることが確認できます。  
