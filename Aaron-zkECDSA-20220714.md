| Item            | Description                              |
| --------------- | ---------------------------------------- |
| Name            | Aaron Chen |
| Discord Account | Aaron Chen#9344 |
| Study Group     | team3 |
| Assignment      | `zkSNARKs Application Survey Report`|
| GitHub Rep      | https://github.com/aaronisme/PrivacyIN-Assignment |

# ZKECDSA的应用研究报告

## 概述
最近10年是区块链技术蓬勃发展的10年，比特币和以太坊的出现和发展，让自主金融，自主主权的概念成为可能。这些都是建立在密码学的基础之上的。现在如何人都可以借助非对称加密算法，来生成一堆公私钥对，来保存自己的数字资产。公钥，地址这些越来越多的成为一个个人的ID。

然而，目前密码学的这套公私体系，没法支撑的起一套完备的身份体系，主要的一个原因就是缺乏隐私性。因为区块链的特性，所有的数据都是公开可访问的。也就是说如果一个地址被标记出来，那么这个地址上的所有资产以及所有的链上行为都会被标记出来。尝试想下如下场景：
1. 我有个一个高价值的NFT，但是我不想明确的告诉你具体是哪一个。有没有什么办法可以在不泄漏我隐私的前提下进行证明。
2. 在一个DAO在是否能在不暴露具体信息的前提下，对于某个提议我有了足够多的投票。

解决这些问题的一种潜在方法就是使用zkSNARKs。zkSNARKs是一种新的密码学方法可以在不暴露隐私数据的前提下，向外界提供证明。在本报告中，主要调研了两个将ZKP应用到ECDSA的应用。下文将分别进行介绍。

## circom-ecdsa
[circom-ecdsa](https://github.com/0xPARC/circom-ecdsa) 是 [0xPARC](https://github.com/0xPARC) 团队使用circom来实现ECDSA的一种方法。目前这个项目还是POC的阶段，主要的目的是用于验证想法。在zkSNARKs中实现ECDSA还是有一定的挑选的。zkSNARKs proof中的“寄存器大小”最多只有254bits，而ECDSA算法中的计算是256bit的。于是`circom-ecdsa`提供了一种解决的方法。

### circom-ecdsa 电路实现
`circom-ecdsa`中实现了如下的ZK 电路：
- bigInt：BigInteger的电路实现
- Speck256K1: Speck256K1的相关运算电路：
    - Secp256k1AddUnequal： 加法电路
    - Secp256k1Double： Double一个点电路
    - Secp256k1ScalarMult：乘法电路
    - Secp256k1PointOnCurve：验证一个点在椭圆曲线上
- ecdsa：ECDSA算法的电路实现
    - ECDSAPrivToPub：私钥计算公钥
    - ECDSAVerifyNoPubkeyCheck：验证给定消息的签名和公钥是合法的
- eth_addr: 以太坊地址的计算电路
    - PrivKeyToAddr：计算私钥对应的以太坊地址

### Benchmarks
`circom-ecdsa`同样提供了测试性能的Benchmarks，如下表：
该数据在AWS c5.4x large 的机器上测得（16-core 3.0GHz, 32G RAM）
||pubkeygen|eth_addr|groupsig|verify|
|---|---|---|---|---|
|Constraints                          |95444 |247380 |250938 |1508136 |
|Circuit compilation                  |21s   |47s    |48s    |72s     |
|Witness generation                   |11s   |11s    |12s    |175s    |
|Trusted setup phase 2 key generation |71s   |94s    |98s    |841s    |
|Trusted setup phase 2 contribution   |9s    |20s    |19s    |149s    |
|Proving key size                     |62M   |132M   |134M   |934M    |
|Proving key verification             |61s   |81s    |80s    |738s    |
|Proving time                         |3s    |7s     |6s     |45s     |
|Proof verification time              |1s    |<1s    |1s     |1s      |


### 小结
[0xPARC](https://github.com/0xPARC) 团队提出的[circom-ecdsa](https://github.com/0xPARC/circom-ecdsa)是使用circom对于ECDSA算法的一种zkSNARKs的电路实现，基于此实现可以将zKSNARK引入到区块链中广泛使用的ecdsa中。例如这样的实现可以在实现例如身份确权的功能，使用zkSNARKs 的proof来替换掉Signature，可以达到再不泄露隐私的情况下，证明自己的身份。目前该实现还是在POC阶段，没有经过第三方的审计和测试。


## zkp-ecdsa
[zkp-ecdsa](https://github.com/cloudflare/zkp-ecdsa) 是对于[ZKAttest: Ring and Group Signatures for Existing ECDSA Keys](https://link.springer.com/chapter/10.1007/978-3-030-99277-4_4)的一种软件实现，坦白来说我不确定这是不是zkSNARKs的典型的应用，但是直观上感觉依旧是利用零知识证明来解决隐私问题的一种方法，因此也放到研究报告中。

ZKAttest是使用环签名的方式将公钥隐藏在一组公钥环中，同时提供一个zkAttestProof，这样验证者就可以在不知道公钥的情况下，验证签名。下边是一个ZKAttest过程的最小化例子。

### 示例

1. 第一步生成Keypair和签名数据。

```ts
// Message to be signed.
const msg = new TextEncoder().encode('Hello ZKAttest');

// Generate a keypair for signing.
const keyPair = await crypto.subtle.generateKey(
    { name: 'ECDSA', namedCurve: 'P-256' },
    true, [ 'sign', 'verify'],
);

// Sign a message as usual.
const msgHash = new Uint8Array(await crypto.subtle.digest('SHA-256', msg));
const signature = new Uint8Array(
    await crypto.subtle.sign(
        { name: 'ECDSA', hash: 'SHA-256' },
        keyPair.privateKey, msgHash,
    )
);
```

2. 将公钥插入到公钥环中，这样就可以将公钥隐在公钥环中，达到隐藏隐私的目的。

```ts
import { keyToInt } from '@cloudflare/zkp-ecdsa'

// Add the public key to an existing ring of keys,
const listKeys = [BigInt(4), BigInt(5), BigInt(6), BigInt(7), BigInt(8)];
listKeys.unshift(await keyToInt(keyPair.publicKey));
```

3. 生成 ZKAttest proof of knowledge,这个proof可以达到两个目的
 - 签名由于私钥产生
 - 私钥对应的公钥在公钥环中

```ts
import { generateParamsList, proveSignatureList } from '@cloudflare/zkp-ecdsa'

// Create a zero-knowledge proof about the signature.
const params = generateParamsList();
const zkAttestProof = await proveSignatureList(
    params,
    msgHash,
    signature,
    keyPair.publicKey,
    0, // position of the public key in the list.
    listKeys
);
```

4. 以上的数据可以让所有人都可以验证proof是valid，这意味着消息是有私钥所签名，并可以不暴露公钥，这样就达到了不暴露隐私的条件下，可以用于验证签名的效果。

```ts
import { verifySignatureList } from '@cloudflare/zkp-ecdsa'
// Verify that zero-knowledge proof is valid.
const valid = await verifySignatureList(params, msgHash, listKeys, zkAttestProof)
console.assert(valid == true)
```

### 小结
zkp-ecdsa并没有使用zkSNARKs电路的方式来实现ECDSA，但也提供了一种在不暴露个人信息（public key）的情况下，证明数据签名的方法，不太确定是否zkp-ecdsa是zkSNARKs应用中的一种，但也把它列在这里，希望能得到老师的指导。


## 总结
本报告总结了zkSNARKs中在zkECDSA应用的现状，介绍了`circom-ecdsa`和`zkp-ecdsa`这两种应用的基本情况和使用例子。ECDSA作为一种广泛使用在区块链中的密码学算法，如果可以和zkSNARKs向结合和使用，将会大大的提高区块链应用的上隐私性。
