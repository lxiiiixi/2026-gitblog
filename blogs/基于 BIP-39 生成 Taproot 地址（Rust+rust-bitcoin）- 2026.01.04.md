# 基于 BIP-39 生成 Taproot 地址（Rust+rust-bitcoin）

> 本篇文章会使用 Rust 以及 rust-bitcoin crate 来实现 Taproot 标准地址的创建（BIP-39 → Seed → BIP-32 → BIP-86 → Taproot）。
>
> ```
> BIP-39 助记词 (12 words)
> ↓
> Seed (64 bytes, PBKDF2-HMAC-SHA512)
> ↓
> BIP-32 Master Key (xprv)
> ↓
> BIP-86 路径派生 (m/86'/1'/0'/0/0)
> ↓
> 子私钥 (32 bytes)
> ↓
> Internal Keypair (secp256k1)
> ↓
> Internal x-only pubkey (32 bytes)
> ↓
> Taproot Tweak (TaggedHash)
> ↓
> Output Key (32 bytes)
> ↓
> Taproot 地址 (tb1p..., bech32m)
> ```

## 核心代码

依赖版本：

```toml
[dependencies]
bitcoin = { version = "0.32.8", features = ["std","rand-std"] }
bip39 = { version = "2.2.2", features = ["rand"] } # generate_in 方法需要启用
secp256k1 = { version = "0.28", features = ["rand"] }
```

核心代码：

```rust
use bip39::{Language, Mnemonic};
use bitcoin::{
    Address, Network, PrivateKey, XOnlyPublicKey,
    bip32::{DerivationPath, Xpriv},
    key::{Keypair, Secp256k1, TapTweak, TweakedKeypair},
    taproot::TaprootSpendInfo,
};

use crate::env_config::ENV_CONFIGS;

pub struct TaprootWallet {
    /// Taproot internal key（root identity）
    internal_keypair: Keypair,

    /// Taproot output key（用于签名）
    tweaked_keypair: TweakedKeypair,

    /// Internal x-only pubkey（构造地址 / script tree）
    internal_xonly: XOnlyPublicKey,

    /// 默认 key-path 地址（无 script tree）
    /// 用于接受转账等
    internal_address: Address,
    // Tweaked key-path 地址（有 script tree）
    // tweaked_address: Address,
}

/// 创建 Taproot 钱包
/// 创建 Taproot 钱包（BIP86, testnet: m/86'/1'/0'/0/0）
pub fn create_taproot_wallet(
    secp: &Secp256k1<bitcoin::secp256k1::All>,
) -> Result<TaprootWallet, Box<dyn std::error::Error>> {
    // 创建新的助记词
    let mnemonic = Mnemonic::generate_in(Language::English, 12).unwrap();
    // 如果使用现有的助记词
    // let mnemonic = Mnemonic::parse_in_normalized(Language::English, &ENV_CONFIGS.mnemonic)?;

  	println!("  Mnemonic: {}", mnemonic.to_string());

    // mnemonic -> seed bytes (64 bytes)
    let seed = mnemonic.to_seed_normalized("");

    // seed -> master xprv (bitcoin::bip32)
    let master_xprv = Xpriv::new_master(Network::Testnet, &seed)?;

    // BIP86 路径
    let path: DerivationPath = "m/86'/1'/0'/0/0".parse()?; // 这里是测试网 testnet3 的路径
    let child_xprv = master_xprv.derive_priv(secp, &path)?;

    // bitcoin 中 private_key 就是 secp256k1::SecretKey
    let secret_key = child_xprv.private_key;

    // SecretKey -> Keypair（internal key）
    // 主要作用是：派生 Taproot 地址、构造 script tree、生成 tweaked key，作为钱包主身份
    // 一般不用来：直接签名
    let internal_keypair = Keypair::from_secret_key(secp, &secret_key);

    // Taproot 地址（使用 internal key）
    let (internal_xonly, _) = internal_keypair.x_only_public_key();
    println!("  Internal XOnly: {}", internal_xonly.to_string());
    let internal_address = Address::p2tr(secp, internal_xonly, None, Network::Testnet);

    // Taproot key-path tweak（无 script tree）
    // 这里的 None 表示没有 script tree，只有 internal key
    let tweaked_keypair: TweakedKeypair = internal_keypair.tap_tweak(secp, None);

    let tweaked_address = Address::p2tr(
        secp,
        tweaked_keypair.to_keypair().x_only_public_key().0,
        None,
        Network::Testnet,
    );

    println!("  Internal key address: {}", internal_address.to_string());
    println!(
        "  Tweaked key address(Never should be used): {:?}",
        tweaked_address
    );

    Ok(TaprootWallet {
        internal_xonly,
        tweaked_keypair,
        internal_keypair,
        internal_address,
    })
}

impl TaprootWallet {
    /// 用于所有 key-path 签名
    pub fn sign_keypath(
        &self,
        secp: &Secp256k1<bitcoin::secp256k1::All>,
        msg: &bitcoin::secp256k1::Message,
    ) -> bitcoin::secp256k1::schnorr::Signature {
        secp.sign_schnorr(msg, &self.tweaked_keypair.to_keypair())
    }

    /// 用于 tapscript（script-path）里显式放入的 x-only pubkey 的签名。
    /// 注意：这不是 output key（tweaked key），而是脚本里用到的 internal key。
    pub fn sign_internal(
        &self,
        secp: &Secp256k1<bitcoin::secp256k1::All>,
        msg: &bitcoin::secp256k1::Message,
    ) -> bitcoin::secp256k1::schnorr::Signature {
        secp.sign_schnorr(msg, &self.internal_keypair)
    }

    pub fn get_commit_address_with_script_tree(
        &self,
        secp: &Secp256k1<bitcoin::secp256k1::All>,
        script_tree: &TaprootSpendInfo,
    ) -> Address {
        Address::p2tr(
            secp,
            self.internal_xonly(),
            script_tree.merkle_root(),
            Network::Testnet,
        )
    }

    pub fn get_internal_address(&self) -> Address {
        self.internal_address.clone()
    }

    /// 用于构造 script tree
    pub fn internal_xonly(&self) -> bitcoin::secp256k1::XOnlyPublicKey {
        self.internal_xonly
    }
}
```

`Cargo run` 执行：

```rust
async fn main() {
    let secp = Secp256k1::<bitcoin::secp256k1::All>::new();
    wallets_copy::create_taproot_wallet(&secp).unwrap();
}
```

得到结果：

```
  Mnemonic: mammal search wrong another armed sniff congress promote tent practice impose mix
  Internal XOnly: b2ae389a503ab6256bb856858ab1c8c7b92f0f48bdbec556637841d7bfe00fc5
  Internal key address: tb1pmxdzkk59l8rpa372l6d3vtu48846j8maw0gd67spnglpdzgas25sjhhy22
  Tweaked key address(Never should be used): tb1p9qq3skjzjkzr32k2agrr75g6w2dygn9h2ypxnrezvh9yw9c6nquq0u482a
```

其中 Internal key address 就是一个可以用于接收转帐的 taproot 地址，可以去 [btc testnet faucet](https://coinfaucet.eu/en/btc-testnet/) 领一些测试币用于后续的开发。

## 代码逐行解析

```rust
let secp = Secp256k1::<bitcoin::secp256k1::All>::new();
```

这行代码创建一个 secp256k1 上下文对象，后续的密钥生成、签名和验证操作需要依赖这个执行器做相关的运算。后续被传入 `Keypair::from_secret_key`、`sign_schnorr`、`tap_tweak` 等函数中，用于确保所有椭圆曲线运算都在同一套规则和能力约束下完成。

这里的 `bitcoin::secp256k1::All` 表示这个 secp256k1 上下文同时具备签名（Signing）和 验证（Verification）两种能力，写在这里属于 Rust 在编译期的操作约束。

除此之外还有：

-   `Secp256k1::<Signing>::new()`
    -   只能签名，不能验签，适合“只负责出签名、不负责验证”的场景（比如硬件钱包、HSM）。
-   `Secp256k1::<Verification>::new()`
    -   只能验证签名，不能签名，适合区块验证、轻节点、索引器等场景。
-   `Secp256k1::<All>::new()`
    -   签名 + 验证都可以，最通用，也最常用。

---

```rust
let mnemonic = Mnemonic::generate_in(Language::English, 12).unwrap();
// mnemonic -> seed bytes (64 bytes)
let seed = mnemonic.to_seed_normalized("");
```

生成一组符合 BIP-39 标准的助记词（mnemonic phrase），这里的 12 表示 12 个单词的助记词，对应着 **128 位熵 + 4 位校验和**（总共 132 位，再按 11 位一组映射到词表），`Language::English` 指定使用 **英文词表（2048 个固定单词）**；最后生成的 mnemonic 是一个 `Mnemonic` 结构体，通过 `to_seed_normalized` 方法生成 seed，通常会配合一个可选的 **passphrase**（BIP-39 的“第 25 个词”）一起生成 seed 用于提高安全性。

这里的助记词在安全性上等价于主私钥，它是整个密钥派生树的起点，一旦泄漏则整个后续派生的钱包都不安全。

> #### BIP-32
>
> -   概述：定义了一种从主密钥确定性地派生出一整颗公私钥树的方法（HD 钱包）
> -   理解：在 BIP-32 之前一直都是一个私钥对应一个地址，一个人如果想要多个地址就要自己保管备份多个私钥。对于钱包项目方来说，客户端如果要生成 100 个地址就对应要管理 100 个私钥。BIP-32 之后可以通过一个 seed 生成无限个地址，可以结构化管理。
>
> #### BIP-39
>
> -   概述：把复杂的私钥翻译成一组常见的单词，定义了一种把随机的 seed 编码成人类可以备份的助记词的方法。
> -   理解：BIP-39 之前保存私钥需要保存很大一长串的字符，容易记错抄错，BIP-39 用随机的 12/24 个单词作为助记词来表示生成 seed，更方便抄写记忆。

> #### 助记词是如何生成的
>
> seed 与助记词的关系：助记词本身就是随机熵的“编码形式”，就是将已有的随机熵转换成一组单词
>
> -   助记词 ≠ seed
> -   助记词 →（通过算法）→ seed
>
> ```
> 随机熵（Entropy）
>   ↓
> 加校验位（Checksum）
>   ↓
> 按 11 bit 一组
>   ↓
> 映射到 2048 个单词表(2¹¹)
>   ↓
> 助记词
> ```
>
> 每个单词 = 11 bit 信息
> 12 个单词 = 132 bit（128 bit 熵 + 4 bit 校验）
>
> -   1. 生成随机熵 entropy
>
>     -   12 个词需要 128bit
>     -   比如生成一个随机的二进制数 `0000 0000 0000 0001 0000 0010 0000 0011 ...`
>
> -   2. 计算校验和 Checksum
>
>     -   如何计算？对 entropy 做一次 SHA-256 取 hash 的前 entropy/32 个 bit（比如对于 128bit 熵，校验和就是 4bit）
>
> -   3. 拼接熵 + 校验和
>
>     -   `[ entropy (128 bit) ][ checksum (4 bit) ]`
>
> -   4. 按 11bit 分组
>
>     -   每一个单词是 11bit，对应 12 个单词。每一组 11bit 都是 0-2047 之间的整数，这个整数就代表了单词表的索引。
>
> -   5. 映射到单词表
>
>     -   分组后的整数找到索引对应的单词，就组成了一套助记词。
>
> #### 钱包生成重复助记词的可能性
>
> 生成一套重复的助记词的概率，也即是说生成两个完全相同的随机熵的概率，也就是假设我现在希望生成两个完全相同的 12 个单词的助记词，我就要生成一个完全相同的 128 位随机熵，可能性数量是 N=2^128，也就是说概率是 1/2^128。
>
> 为什么从助记词的角度直接想到可能性为 2048¹² 是不对的，因为 BIP39 的 12 个词中并不是 12×11 = 132 bit 都是自由的，最后的**4 bit 是校验位**，而校验位由前 128 bit 决定，也就是说：在所有 2048¹² 个“词序列”中，**只有一小部分是合法助记词**，校验和反而缩小了这个范围，是不够准确的。

---

```rust
let path: DerivationPath = "m/86'/1'/0'/0/0".parse()?; // 这里是测试网 testnet3 的路径
```

定义 BIP-86 规范下的分层确定性（HD）派生路径，用于从助记词种子确定性地派生出某一个具体私钥。路径从左到右，每一层都在**限定派生语义**，最终唯一确定一个密钥。

其中层级语义为：

-   `m` 表示 主节点（master extended private key, xprv），后面的所有层级，都是从这个主节点继续派生。

-   `86'` 表示 **purpose**， `'` 表示 **Hardened 派生**，为了防止子公钥反推出父私钥（安全边界）。

    | Purpose | 含义                           |
    | ------- | ------------------------------ |
    | `44'`   | BIP-44（Legacy P2PKH）         |
    | `49'`   | BIP-49（P2SH-P2WPKH）          |
    | `84'`   | BIP-84（Native SegWit P2WPKH） |
    | `86'`   | **BIP-86（Taproot P2TR）**     |

-   `1'` 表示 Coin Type

    | Coin Type | 网络             |
    | --------- | ---------------- |
    | `0'`      | Bitcoin Mainnet  |
    | `1'`      | Bitcoin Testnet  |
    | `2'`      | Litecoin（示例） |

-   `0'` 表示 Account，比如 `0'` 表示 第 0 个账户，一个助记词可以拥有多个逻辑帐户。

-   下一个 `0` 表示 Change，语义：

    | 值  | 含义                               |
    | --- | ---------------------------------- |
    | `0` | **接收地址（external / receive）** |
    | `1` | **找零地址（internal / change）**  |

-   下一个 `0` 表示 Address Index（地址索引），每加 1 就是一个全新地址，所有地址都拥有不同的私钥

所以 `"m/86'/1'/0'/0/0"` 所表示的语义就是：从主私钥开始，在测试网络，使用第 0 个帐户，作为接受地址，生成这个分支下的第 0 个地址。

---

```rust
let master_xprv = Xpriv::new_master(Network::Testnet, &seed)?; // 把 64 字节的 seed 转换为主扩展私钥 master xprv
let child_xprv = master_xprv.derive_priv(secp, &path)?; // 按照路径从主私钥派生出一个具体的扩展私钥（派生过程中包含椭圆曲线运算所以需要传入 secp context）
let secret_key = child_xprv.private_key; // 取出可以用于签名的一个 32 字节的 secp256k1 私钥（只能控制这一个地址）
```

`Xpriv` ≠ 私钥

`Xpriv` = 私钥 + chain code + 派生上下文

---

```rust
let (internal_xonly, _) = internal_keypair.x_only_public_key();
```

从一个 secp256k1 `Keypair` 中取出 Taproot 所需的 x-only 公钥（XOnlyPublicKey），用于构造 P2TR 地址和后续的 Taproot tweak。

这里的 internal_xonly 是 P2TR 地址的基础，也是 script tree tweak 的输入

---

```rust
let internal_address = Address::p2tr(
    secp,               // 椭圆曲线运算上下文
    internal_xonly,     // Taproot internal key（x-only）
    None,               // merkle_root：None 表示无脚本树
    Network::Testnet,   // 网络（决定 bech32m HRP：tb1p…）
)
```

使用 internal key 构造一个无脚本（第三个参数 None ）的 Taproot 地址，这种情况下输出公钥 output 为 `output_key = internal_key + H(internal_key || 0) * G`

internal address 一般用于接收普通转帐。

---

```rust
let tweaked_keypair: TweakedKeypair = internal_keypair.tap_tweak(secp, None);
```

把 internal keypair 按 Taproot 规则做一次 tweak，得到真正能用于 key-path 花费（签名）的私钥/公钥对。

> **internal key**：从 BIP-32 派生出来的 x-only 公钥（上一步派生出来的），用来“定义身份/承诺”。
>
> **output key**：`internal key + tweak` 得到的公钥，实际用于链上锁定 UTXO。

---

```

let tweaked_address = Address::p2tr(
    secp,
    tweaked_keypair.to_keypair().x_only_public_key().0,
    None,
    Network::Testnet,
);
```

这是一个错误示例,这段代码展示了 **double tweak 错误**：

1. `tweaked_keypair.to_keypair().x_only_public_key().0` 已经是 **output key**（经过一次 tweak）
2. `Address::p2tr` 会把传入的公钥当作 **internal key**，再次进行 tweak
3. 结果是 `output_key' = output_key + H(output_key || 0) * G`，与链上实际锁定的 output key 不一致

正确做法是始终使用 `internal_xonly` 生成地址，让 `Address::p2tr` 内部完成 tweak。
