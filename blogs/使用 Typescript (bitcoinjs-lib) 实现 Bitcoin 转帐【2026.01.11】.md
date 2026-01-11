使用 Nodejs 手动构建一个才能支持 Native SegWit (BIP84) 的比特币钱包，并实现从 UTXO 选择到交易广播的完整转账流程。

> 完整的代码可参考 https://github.com/lxiiiixi/bitcoin-txs 仓库。

## 申请 blockcypher API

首先去 blockcypher 申请一个免费的 API token，接下来的操作会根据 blockcypher 的接口进行查询和上链操作。

定义相关方法：

```ts
import fetch from "node-fetch";
import * as dotenv from "dotenv";

dotenv.config();

export interface UTXO {
    tx_hash: string;
    tx_output_n: number;
    value: bigint;
    script: string;
}

export enum Network {
    TESTNET = "test3",
    MAINNET = "main",
}

export const BLOCKCYPHER_TOKEN = process.env.BC_TOKEN as string;

export async function fetchUTXOs(address: string) {
    const url = `https://api.blockcypher.com/v1/btc/test3/addrs/${address}?unspentOnly=true&includeScript=true&token=${BLOCKCYPHER_TOKEN}`;
    const res = await fetch(url);
    if (!res.ok) throw new Error(`fetch utxo failed ${res.status}`);
    const j = await res.json();
    console.log(`[BlockCypher] fetchUTXOs response of ${address}:`, JSON.stringify(j, null, 2));
    // BlockCypher 返回：txrefs 数组（也可能是 empty），字段: tx_hash, tx_output_n, value, script
    return (j as any).txrefs || ([] as UTXO[]);
}

export async function broadcastTransaction(txHex: string, network: Network = Network.TESTNET) {
    const url = `https://api.blockcypher.com/v1/btc/${network}/txs/push?token=${BLOCKCYPHER_TOKEN}`;
    const res = await fetch(url, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ tx: txHex }),
    });
    if (!res.ok) throw new Error(`broadcast transaction failed ${res.status}`);
    const j = await res.json();
    return j;
}

export async function getBalance(address: string, network: Network = Network.TESTNET) {
    const url = `https://api.blockcypher.com/v1/btc/${network}/addrs/${address}/balance?token=${BLOCKCYPHER_TOKEN}`;
    const res = await fetch(url);
    if (!res.ok) throw new Error(`get balance failed ${res.status}`);
    const j = await res.json();
    return j;
}

/**
 * 查询某个特定地址相关的所有信息
 * curl "https://api.blockcypher.com/v1/btc/test3/addrs/tb1qu63netywsfh7x6u56ka58wjua4d2qscdyj65xv"
 */
export async function getAddressInfo(address: string, network: Network = Network.TESTNET) {
    const url = `https://api.blockcypher.com/v1/btc/${network}/addrs/${address}?token=${BLOCKCYPHER_TOKEN}`;
    const res = await fetch(url);
    if (!res.ok) throw new Error(`get address info failed ${res.status}`);
    const j = await res.json();
    return j;
}
```

## HD 钱包与 BIP84 地址派生

首先来创建一个 HD 钱包

`hd_wallets.ts`：

```ts
import * as bitcoin from "bitcoinjs-lib"; // 处理比特币交易的库
import * as bip39 from "bip39"; // 负责生成助记词
import * as ecc from "tiny-secp256k1"; // 椭圆曲线算法
import BIP32Factory from "bip32"; // 负责 BIP32 HD 派生路径
import * as dotenv from "dotenv";

dotenv.config();

bitcoin.initEccLib(ecc); // 告诉 bitcoinjs-lib 使用 tiny-secp256k1 椭圆曲线算法（绑定）
const bip32 = BIP32Factory(ecc); // BIP32Factory 表示用 ecc 作为底层加密模块，构造一个 BIP32 HD 钱包工厂
const network = bitcoin.networks.testnet;

// 1. 生成助记词
const mnemonic = bip39.generateMnemonic();
console.log("New generated mnemonic:", mnemonic);
// const mnemonic = process.env.MNEMONIC as string; // 如果已经创建了

// 2. 助记词 -> seed
const seed = bip39.mnemonicToSeedSync(mnemonic);
// 什么是 seed？助记词（mnemonic） → 通过 PBKDF2 → 生成一串 512bit 的随机数据，这就是 seed，seed 就是整个钱包的根。

// 3. 种子 -> root (含私钥)
export const root = bip32.fromSeed(seed, network);
// BIP32 的 root node，包含一个私钥，包含一个可以用于派生子私钥的 chain code，可以 derivePath 派生子节点。是整个钱包的核心私钥。

// 方法 1：从 xpub 派生地址
export async function deriveFromXpub() {
    // 先用 root 派生到帐户层
    // 生成 BIP84 account: m/84'/1'/0'
    const account = root.derivePath("m/84'/1'/0'"); // 测试网路径
    // 路径解释：
    // m   =>  主私钥（master private key）
    // 84' =>  purpose，表示使用 BIP84 标准
    // 1'  =>  coin_type=1（测试网）主网是 0'
    // 0' =>  account，表示使用第一个账户

    // 从帐户层导出 xpub（可以保存到服务器端）
    const xpub = account.neutered().toBase58();
    // neutered() = 去掉私钥，变成扩展公钥（xpub）。
    // xpub 包含：公钥、chain code（用于派生子公钥）、网络参数、深度信息
    console.log("Account XPUB:", xpub);

    const accountNode = bip32.fromBase58(xpub, network);

    for (let i = 0; i < 5; i++) {
        const child = accountNode.derive(0).derive(i);

        const { address } = bitcoin.payments.p2wpkh({
            pubkey: child.publicKey,
            network,
        });
        // 通过 xpub 派生出来得不到私钥

        console.log(`index=${i} 地址=${address}`);
    }
}

// 方法 2：从助记词派 => 子私钥 => 公钥 => 地址
export async function deriveFromMnemonic() {
    for (let i = 0; i < 5; i++) {
        const path = `m/84'/1'/0'/0/${i}`; // 硬化路径只能从根节点派生
        // m             - master private key
        // 84'           - BIP84 (native segwit)
        // 1'            - testnet coin type
        // 0'            - account 0
        // 0             - external chain (收款地址)
        // i             - index

        // Q 什么是硬化路径？
        // 带 ' 的叫硬化路径，如 84'，只能从 私钥（xprv） 派生，保护私钥不被暴露。
        const child = root.derivePath(path);

        const { address } = bitcoin.payments.p2wpkh({
            pubkey: child.publicKey,
            network,
        });

        console.log(`index=${i} 地址=${address} 私钥=${child.toWIF()}`);
    }
}

```

比如我这里执行得到：

```
bitcoin-txs on  main [!] is 📦 1.0.0 via ⬢ v23.4.0 
➜ tsx ./scripts/create_wallets.ts
[dotenv@17.2.3] injecting env (2) from .env -- tip: 🔑 add access controls to secrets: https://dotenvx.com/ops
New generated mnemonic: crouch wool feel maximum estate adjust aerobic dumb salt fan unusual utility

======== 从帐户层直接派生出地址 =========
Account XPUB: tpubDCq5CvqawQ94Z9jZ4VfP6FAuAA7RS3K5RHQssT8p4ZTDguCzHXwFctY3XNKPa4tw7d3QRRmynQDTTS3fBz6d6jCJ585F3vhzke7MnJ2qLTg
index=0 地址=tb1qtk4h6f2jj2tfgnwk4dcqxlkm7ga0jehn4kg2dh
index=1 地址=tb1qsgu43zvs52vwpucsj0eka9xy5ka0y6pr25jrjl
index=2 地址=tb1qlxtetd4a3zrmmxlsh2zwv52pdmensf4zjd7szj
index=3 地址=tb1q92he5cp35wnlgzdljmeq4vduxvv6nw8cu44r9f
index=4 地址=tb1q3sdk553v6kwruy0njhpwhw48cmmyu0s0a7r2nm

======== 从助记词派 => 子私钥 => 公钥 => 地址 ================
index=0 地址=tb1qtk4h6f2jj2tfgnwk4dcqxlkm7ga0jehn4kg2dh 私钥=cVJRUEhTovwDoYGw8T86XPFmUrgpdU1tMqPwJqGSErgFSRhSL9LC
index=1 地址=tb1qsgu43zvs52vwpucsj0eka9xy5ka0y6pr25jrjl 私钥=cRJZcZWDS7dcojVxBcz4SUEfdoksr34Uqg4YCRc4xMr17CWG8iiE
index=2 地址=tb1qlxtetd4a3zrmmxlsh2zwv52pdmensf4zjd7szj 私钥=cNRDFA2NLsQira3V3Zm7ZD2CBbRWtKaQSo9uQpXSZWCVZPUEj28U
index=3 地址=tb1q92he5cp35wnlgzdljmeq4vduxvv6nw8cu44r9f 私钥=cVYS79PvuZHJPx6rvCGXAzwDZcDUmFuvTm1XDW3n3WnVbv3Nvg19
index=4 地址=tb1q3sdk553v6kwruy0njhpwhw48cmmyu0s0a7r2nm 私钥=cSXf8yXWAMDBb7beAwkLPEBFwbHFvBMbfX1Rh3sW2w1t8dH2TGa9
```

这里展示了两种通过同一个助记词派生的方式，但是生成的地址都是一样的。

> 但是通过公钥派生地址的好处是除了在第一次创建助记词的时候需要承担保留助记词和私钥的风险，后续继续生产其他的地址都可以只是通过相对安全的公钥了，对于一些项目中的场景是有用的。

复制这个助记词，接下来会放在环境变量中基于这些钱包来进行转账。

复制 index 0 的地址去 [faucet](https://coinfaucet.eu/en/btc-testnet/) 领点测试 btc。比如我[这笔交易](https://mempool.space/testnet/tx/a1032e96c8c636dad6b7eaddf9d4137d37af97fcc5effea6a49b71d951059820)就是收到 0.00142448 BTC 转账的交易。

## 转帐实现

接下来用 index=0 的地址给 index=1 的地址转 300 sats。代码如下：

```ts
import * as bitcoin from "bitcoinjs-lib";
import { broadcastTransaction, fetchUTXOs, UTXO } from "./blockcypher";
import * as dotenv from "dotenv";
import { root } from "./hd_wallets";

dotenv.config();

const network = bitcoin.networks.testnet;

// 根据 index 获取 keyPair
async function getKeyPair(index: number) {
    const path = `m/84'/1'/0'/0/${index}`;
    const child = root.derivePath(path);
    return child;
}

export function selectUTXOs(
    utxos: UTXO[],
    targetPlusFee: bigint
): { chosen: UTXO[]; sum: bigint } | null {
    // 简单贪心选币（从大到小），生产环境用更好策略
    utxos.sort((a, b) => {
        if (a.value > b.value) return -1;
        if (a.value < b.value) return 1;
        return 0;
    });
    const chosen: UTXO[] = [];
    let sum: bigint = BigInt(0);
    for (const u of utxos) {
        chosen.push(u);
        sum += BigInt(u.value);
        if (sum >= targetPlusFee) break;
    }
    if (sum < targetPlusFee) return null;
    return { chosen, sum };
}

async function transferByBlockcypher(
    account: { index: number; address: string },
    amountSat: bigint,
    feeSat: bigint,
    toAddress: string
) {
    const utxos = await fetchUTXOs(account.address);
    console.log("utxos:", utxos);

    if (!utxos.length) throw new Error("没有可用 UTXO");

    const need: bigint = amountSat + feeSat;
    const pick = selectUTXOs(utxos, need);

    console.log("pick:", pick);

    if (!pick) throw new Error("UTXO 不足");

    const psbt = new bitcoin.Psbt({ network });
    for (const utxo of pick.chosen) {
        psbt.addInput({
            hash: utxo.tx_hash,
            index: utxo.tx_output_n,
            //  witnessUtxo 只能用于 SegWit 类输入。
            witnessUtxo: {
                script: Buffer.from(utxo.script, "hex"),
                value: BigInt(utxo.value),
            },
        });
    }

    // 输出：主接收方
    psbt.addOutput({
        address: toAddress,
        value: amountSat,
    });

    // 找零回到 FROM_ADDRESS（如果有多余）
    // 不找零的话会造成财产丢失
    const change: bigint = pick.sum - amountSat - feeSat;
    if (change > BigInt(0)) {
        psbt.addOutput({
            address: account.address,
            value: change,
        });
    }

    const keyPair = await getKeyPair(account.index);
    for (let i = 0; i < pick.chosen.length; i++) {
        // 这里的顺序的确有可能是不对的
        // 要根据 utxo 所属的地址的 index 来确定 keyPair
        psbt.signInput(i, keyPair);
    }

    psbt.finalizeAllInputs();

    const rawTx = psbt.extractTransaction().toHex();
    console.log("\n📦 原始交易 hex:");
    console.log(rawTx);

    // 广播交易
    console.log("\n📡 广播交易中...");
    const response = await broadcastTransaction(rawTx);

    console.log("\n🚀 广播成功！");
    console.log("🔗 交易详情:", JSON.stringify(response, null, 2));
}

const toAcconut = "tb1qwzyf62ew0cc09aly597ky0weyqz6e4qx46hh0n";

transferByBlockcypher(
    {
        index: 0,
        address: "tb1qhtp56txkkc8vzcla9e4pmgfgqgp5nawthyx98w",
    },
    BigInt(300),
    BigInt(200),
    toAcconut
);

```

> ### 比特币的找零机制
>
> 在构建交易的过程中如果自己计算相关的金，要明确的知道如果不找零相当于给矿工发红包。
> 当你花费一个 UTXO 时，这个 UTXO 会被**彻底消耗**掉，不能只花掉其中一部分。
> 在构建交易时，必须明确指定找零的去处。一般可以原路返回，但是这样一定程度上损害隐私性，钱包往往自动派生新的地址接收找零，通常来说：
>
> - 收款路径：`m/84'/1'/0'/0/i`（`0` 代表外部链，用于收款）
> - 找零路径： `m/84'/1'/0'/1/i`（`1` 代表内部链，用于找零）

> ### 使用 blockcypher api 过程中的一个小插曲
>
> 发现无论如何都查询不到一些地址新获取的 UTXO 并且交易也无法上链，以为是我使用 api 的免费额度超了，用另外的邮箱去重新注册申请了发现也是同样的结果，获取到的都是以前的数据，说明不是这个原因。
>
> ```
> ➜ curl https://api.blockcypher.com/v1/btc/test3
> {
>   "name": "BTC.test3",
>   "height": 4786130,
>   "hash": "00000000000aa760fffa5f1e1336f1ee9450eed5c25a6b7fe3b3d9d01655e364",
>   "time": "2026-01-09T18:01:34.373845068Z",
>   "latest_url": "https://api.blockcypher.com/v1/btc/test3/blocks/00000000000aa760fffa5f1e1336f1ee9450eed5c25a6b7fe3b3d9d01655e364",
>   "previous_hash": "000000000474c9ec4976e9aaad6b4b58811799dd77ec08879abb756e2c8b7e87",
>   "previous_url": "https://api.blockcypher.com/v1/btc/test3/blocks/000000000474c9ec4976e9aaad6b4b58811799dd77ec08879abb756e2c8b7e87",
>   "peer_count": 162,
>   "unconfirmed_count": 0,
>   "high_fee_per_kb": 23991,
>   "medium_fee_per_kb": 13142,
>   "low_fee_per_kb": 7553,
>   "last_fork_height": 4786122,
>   "last_fork_hash": "0000000008988f607ec81c80952d559f34fecfc1cb938969509ed144bcf8a86e"
> }
> ```
>
> 请求了链相关的 API，发现其中有一个参数 **height**（文档中对字段的定义是 The current height of the blockchain; i.e., the number of blocks in the blockchain.）为 `4786130`，但是截止到目前我的时间 testnet3 的区块高度是 `4812008`，也就是说他们的节点落后了很多区块，自然找不到数据。
>
> 所以在选择区块链 API 时，务必检查其 `height` 是否与主网一致。

那就只能换一个 API 了，改用 alchemy 的 api：

```ts
import fetch from "node-fetch";
import * as dotenv from "dotenv";

dotenv.config();

const ALCHEMY_API_URL = process.env.ALCHEMY_API_URL as string;

export enum Network {
    TESTNET = "test3",
    MAINNET = "main",
}

//     curl -X POST https://bitcoin-testnet.g.alchemy.com/v2/docs-demo \
//      -H "Content-Type: application/json" \
//      -d '{
//   "jsonrpc": "2.0",
//   "method": "sendrawtransaction",
//   "params": [
//     "0200000000010153fc6712e0c6cbfd15e56743f2a16bba3c0b17837d4fd33d68d2d930739e2b130000000000ffffffff01c0c62d0000000000160014c24b61118d4a2b36257b65e1ea7f15f85e41ff0402483045022100ac32e935715a57ec1d642a5e178c37f74c013bf8e4edc4cb1c79f5352f136e87022020b0b3192347d1b84e9b89d00a2ecb290f18f9c39e514fa3ef2b7a889e7b6c1b012103ab0b56c7aa6254a80c124e04d2149f7fc376afedfe4623f3c59b87c279eaeb1400000000",
//     "0.1"
//   ],
//   "id": 1
// }'
export async function broadcastTransactionWithAlchemy(txHex: string) {
    const res = await fetch(ALCHEMY_API_URL, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            jsonrpc: "2.0",
            method: "sendrawtransaction",
            params: [txHex, 0.1],
            id: 1,
        }),
    });
    if (!res.ok) throw new Error(`broadcast transaction failed ${res.status}`);
    const j = await res.json();
    return j;
}

// curl -X POST https://bitcoin-testnet.g.alchemy.com/v2/docs-demo \
//      -H "Content-Type: application/json" \
//      -d '{
//   "jsonrpc": "2.0",
//   "method": "gettxout",
//   "params": [
//     "546263a196ce5cf674d5002afc0231ab417c2e971fd4ed1735c7a4c63f44720b",
//     0,
//     true
//   ],
//   "id": 1
// }'

// {
//     bestblock: '0000000085d19511a71fce474b98f82c444b567a0ca1061146a0c1bb6c53f1f1',
//     confirmations: 10,
//     value: 0.00193745,
//     scriptPubKey: {
//       asm: '0 bac34d2cd6b60ec163fd2e6a1da128020349f5cb',
//       desc: 'addr(tb1qhtp56txkkc8vzcla9e4pmgfgqgp5nawthyx98w)#xhsj3fmx',
//       hex: '0014bac34d2cd6b60ec163fd2e6a1da128020349f5cb',
//       address: 'tb1qhtp56txkkc8vzcla9e4pmgfgqgp5nawthyx98w',
//       type: 'witness_v0_keyhash'
//     },
//     coinbase: false
//   }
interface UTXOWithAlchemy {
    bestblock: string;
    confirmations: number;
    value: number;
    scriptPubKey: {
        asm: string;
        desc: string;
        hex: string;
        address: string;
        type: string;
    };
    coinbase: boolean;
}
export async function fetchUTXOsWithAlchemy(
    txHash: string,
    voutIndex: number
): Promise<UTXOWithAlchemy> {
    const res = await fetch(ALCHEMY_API_URL, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            jsonrpc: "2.0",
            method: "gettxout",
            params: [txHash, voutIndex, true],
            id: 1,
        }),
    });
    if (!res.ok) throw new Error(`fetch utxo failed ${res.status}`);
    const j = (await res.json()) as { result: UTXOWithAlchemy };
    return j.result as UTXOWithAlchemy;
}
```

但是 alchemy 并没有像是 blockcypher 那样根据地址直接查询 UTXO 的接口，而是通过某个特定的 txid 以及接下来要花费的 UTXO 所处的 output index。比如说如下[这笔交易](https://mempool.space/testnet/tx/a1032e96c8c636dad6b7eaddf9d4137d37af97fcc5effea6a49b71d951059820)是我从水龙头领取的交易， [tb1qtk4h6f2jj2tfgnwk4dcqxlkm7ga0je](https://mempool.space/testnet/address/tb1qtk4h6f2jj2tfgnwk4dcqxlkm7ga0jehn4kg2dh) 这个是我可以花费的（我可以用我的私钥解锁使用），那么 `fetchUTXOsWithAlchemy` 这个接口对应的 *txHash* 就是这个交易的 txid，*voutIndex* 就是 1。

![image.png| 600](https://raw.githubusercontent.com/lxiiiixi/Image-Hosting/main/Markdown/20260110095450.png)

执行方法：

```ts
async function transferByAlchemy(
    senderAccount: {
        index: number;
        address: string;
    },
    amountSatToSend: bigint,
    feeSat: bigint,
    toAddress: string
) {
    const input_hash = "a1032e96c8c636dad6b7eaddf9d4137d37af97fcc5effea6a49b71d951059820";
    const input_index = 1;
    const inputUtxo = await fetchUTXOsWithAlchemy(input_hash, input_index);
    console.log("inputUtxo:", inputUtxo);
    if (!inputUtxo) throw new Error("UTXO 不存在");
    const utxoBalance = BigInt(inputUtxo.value * 1e8);
    const changeSatAmount = utxoBalance - amountSatToSend - feeSat;

    console.log("   余额:", utxoBalance);
    console.log("   发送金额:", amountSatToSend);
    console.log("   手续费:", feeSat);
    console.log("   找零:", changeSatAmount);
    if (changeSatAmount <= BigInt(0)) {
        throw new Error("UTXO 不足");
    }

    const psbt = new bitcoin.Psbt({ network });
    psbt.addInput({
        hash: input_hash,
        index: input_index,
        witnessUtxo: {
            script: Buffer.from(inputUtxo.scriptPubKey.hex, "hex"),
            value: utxoBalance,
        },
    });
    psbt.addOutput({
        address: toAddress,
        value: amountSatToSend,
    });
    psbt.addOutput({
        address: senderAccount.address,
        value: changeSatAmount,
    });
    // sign
    const keyPair = await getKeyPair(senderAccount.index);
    psbt.signInput(0, keyPair); // 第一个参数表示 PSBT 输入列表的索引
    psbt.finalizeAllInputs();

    const rawTx = psbt.extractTransaction().toHex();
    console.log("\n📦 原始交易 hex:");
    console.log(rawTx);
    const response = await broadcastTransactionWithAlchemy(rawTx);
    console.log("\n🚀 广播成功！");
    console.log("🔗 交易详情:", JSON.stringify(response, null, 2));
}

transferByAlchemy(account0, BigInt(300), BigInt(200), toAcconut);
```

> ### Dust Limit（粉尘限制）
>
> 注意转账的费用是不能小于手续费的，否则会遇到 “dust, tx with dust output must be 0-fee” 的错误，这是节点本身的策略，主要是为了防止花钱制造费用过低的 UTXO 垃圾对 UTXO 集进行污染增加节点负担。

结果：

```
inputUtxo: {
  bestblock: '0000000000000044352ddd9a8ae2094db848609a86ec6b3a7cb8443bb50d60fe',
  confirmations: 90,
  value: 0.00142448,
  scriptPubKey: {
    asm: '0 5dab7d25529296944dd6ab70037edbf23af966f3',
    desc: 'addr(tb1qtk4h6f2jj2tfgnwk4dcqxlkm7ga0jehn4kg2dh)#73ch2j26',
    hex: '00145dab7d25529296944dd6ab70037edbf23af966f3',
    address: 'tb1qtk4h6f2jj2tfgnwk4dcqxlkm7ga0jehn4kg2dh',
    type: 'witness_v0_keyhash'
  },
  coinbase: false
}
   余额: 142448n
   发送金额: 300n
   手续费: 200n
   找零: 141948n

📦 原始交易 hex:
0200000000010120980551d9719ba4a6feefc5fc97af377d13d4f9ddeab7d6da36c6c8962e03a10100000000ffffffff022c010000000000001600148239588990a298e0f31093f36e94c4a5baf268237c2a0200000000001600145dab7d25529296944dd6ab70037edbf23af966f3024730440220257e4961ad0365d2b1a1621184985c41de8317f8a07fb4ffad720601a03e4c280220347f7e8c156a8c1958b5d1089410ee2bc064c6b1422a01cfe45c9a7e37de0f530121020779cc0316602121de5f3c90e64ff4447d7299c6721bc6440f4a245e8e6145a700000000

🚀 广播成功！
🔗 交易详情: {
  "jsonrpc": "2.0",
  "result": "58157f15f16087b3e15e75f11a50b53bf8db60c98baa5b96a1626f8a3a6ffbbc",
  "id": 1
}
```

这笔转账就完成了，给注定 to 地址转了 300sats，余下的找零给自己。

> 注意一定要显式的在交易计算好找零的余额并且构建到 output 中，否则全部会被算作为手续费。

![image.png| 600](https://raw.githubusercontent.com/lxiiiixi/Image-Hosting/main/Markdown/20260110100229.png)

> ### 补充说明 - UTXO 组合的更优策略
>
> 本例是自己构建 UTXO 以及计算找零，在生产环境中，通常会使用 [`coinselect` 库](https://github.com/bitcoinjs/coinselect)来自动处理 UTXO 组合和手续费计算，避免手动计算 `change` 导致的财产丢失风险。



