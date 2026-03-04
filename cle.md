

# 技术设计文档：基于 Celestia 的去中心化战争迷雾卡牌游戏

**FogCard Protocol — Technical Design Document**

**版本：** v1.0  
**日期：** 2025年7月  
**状态：** Draft / RFC  

---

## 目录

1. [摘要 (Abstract)](#1-摘要)
2. [项目背景与动机 (Background & Motivation)](#2-项目背景与动机)
3. [核心愿景 (The Vision)](#3-核心愿景)
4. [术语表 (Glossary)](#4-术语表)
5. [系统架构：三层结构 (Architecture)](#5-系统架构三层结构)
6. [密码学原语 (Cryptographic Primitives)](#6-密码学原语)
7. [关键机制设计 (Core Mechanisms)](#7-关键机制设计)
8. [详细业务流程 (User Flow)](#8-详细业务流程)
9. [协议状态机 (Protocol State Machine)](#9-协议状态机)
10. [数据结构与编码 (Data Structures)](#10-数据结构与编码)
11. [安全性分析 (Security Analysis)](#11-安全性分析)
12. [为什么选择 Celestia Private Blockspace？](#12-为什么选择-celestia-private-blockspace)
13. [性能与 Gas 估算 (Performance & Cost)](#13-性能与-gas-估算)
14. [开发路线图 (Roadmap)](#14-开发路线图)
15. [参考文献 (References)](#15-参考文献)

---

## 1. 摘要

本文档描述了 **FogCard Protocol** 的完整技术设计——一款运行在 Celestia 数据可用层之上的去中心化卡牌对战游戏。该游戏利用 **Private Blockspace（私有区块空间）** 与 **Verifiable Encryption（可验证加密，VE）** 技术，在完全无需中心化服务器的前提下，实现了：

- **战争迷雾（Fog of War）**：玩家的卡牌位置、手牌内容对对手不可见；
- **可验证公平（Verifiable Fairness）**：每一步操作都附带密码学证明，确保无人作弊；
- **资产安全（Asset Safety）**：即使对手离线、恶意中断，玩家仍可单方面结算。

本文档面向工程师、研究人员及潜在合作者，旨在提供从高层架构到底层数据结构的全景描述。

---

## 2. 项目背景与动机

### 2.1 链上游戏的困境

全链游戏（Fully On-chain Games, FOCG）在 2023–2025 年间获得显著发展。以 MUD、Dojo 等框架为代表，开发者已经能够在 EVM 或 StarkNet 上构建复杂的游戏逻辑。然而，这些游戏天然面临一个致命问题：

> **区块链是透明的。**

所有链上状态对所有人可见，这意味着：

| 传统游戏机制 | 链上实现困难度 | 原因 |
|---|---|---|
| 战争迷雾 | ★★★★★ | 所有位置坐标公开可读 |
| 暗牌 / 手牌 | ★★★★★ | 合约存储全局可查 |
| 伏击 / 埋伏 | ★★★★★ | 任何人可预读下一步操作 |
| 随机抽卡 | ★★★★ | 链上随机数可被矿工操控 |

为了解决透明性问题，业界尝试过以下方案：

1. **Commit-Reveal 模式**：玩家先提交操作的哈希（Commit），随后再揭示（Reveal）。问题在于 Reveal 阶段可以被恶意拒绝，且无法验证 Commit 内容的合法性。
2. **链下服务器 + 链上结算**：引入中心化服务器管理私密状态。问题在于服务器可以作弊、宕机、审查。
3. **纯 ZK 方案**：利用零知识证明隐藏状态。问题在于 ZKP 的生成开销极大，尤其是复杂游戏逻辑场景下延迟不可接受。

### 2.2 Celestia 带来的新范式

Celestia 在 2025 年提出的 **Private Blockspace** 概念为上述困境提供了全新解法。其核心思路是：

> 将加密后的数据发布到 Celestia 的数据可用层（DA Layer），配合可验证加密（VE）证明，实现"数据对外不可读，但可证明其合法性"。

这一范式的独特之处在于：

- **数据始终在线**：不依赖对手或服务器的配合即可获取数据；
- **可验证性**：VE 证明确保加密数据符合预定义规则；
- **选择性披露**：仅在特定条件触发时，解密对应字段。

FogCard Protocol 正是基于这一范式构建的首个战争迷雾卡牌游戏。

---

## 3. 核心愿景

**一句话总结：**

> 利用 Celestia 的 Private Blockspace 和 Verifiable Encryption 技术，打破传统链上游戏"全透明"的局限，在不依赖中心化服务器的前提下，实现具备信息不对称（战争迷雾）、公平可验证、且资产高度安全的卡牌对战游戏。

### 3.1 设计原则

| 原则 | 描述 |
|---|---|
| **No Trusted Third Party** | 游戏全程无需任何中心化服务器或中间人 |
| **Verifiable Fairness** | 每步操作均可被对手及任意第三方验证合法性 |
| **Information Asymmetry** | 玩家仅获得"应知"信息，实现真正的战争迷雾 |
| **Liveness Guarantee** | 即使对手下线，游戏仍可继续推进并结算 |
| **Asset Sovereignty** | 游戏资产（卡牌 NFT、代币）完全由玩家自托管 |

---

## 4. 术语表

| 术语 | 缩写 | 定义 |
|---|---|---|
| Verifiable Encryption | VE | 一种密码学方案，允许证明者证明密文中的明文满足特定性质，而不暴露明文本身 |
| Private Blockspace | PB | Celestia 提出的概念，指在 DA 层中存储加密数据的区块空间 |
| Zero-Knowledge Proof | ZKP | 零知识证明，允许一方向另一方证明某陈述为真而不透露额外信息 |
| Data Availability | DA | 数据可用性，保证区块数据已被发布且任何人可获取 |
| Selective Disclosure | SD | 选择性披露，在满足条件时仅解密特定字段 |
| zkVM | — | 零知识虚拟机，在虚拟机中执行程序并生成执行正确性的 ZK 证明 |
| TEE | — | 可信执行环境 (Trusted Execution Environment) |
| Namespace | NS | Celestia 中用于隔离不同应用数据的逻辑分区 |
| Blob | — | Celestia 上发布的数据单元 |
| Anchor | — | VE 锚点，用于标记加密数据已发布且可用的证明 |
| Commit-Reveal | CR | 传统的两阶段承诺揭示方案 |

---

## 5. 系统架构：三层结构

FogCard Protocol 采用三层解耦架构，各层职责清晰、可独立升级。

```
┌──────────────────────────────────────────────────────────────┐
│                     执行层 (Execution Layer)                   │
│                                                              │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐   │
│   │ 游戏客户端   │   │  zkVM 引擎   │   │  本地密钥管理器  │   │
│   │  (Unity/Web) │   │ (RiscZero/  │   │  (Keystore)     │   │
│   │             │   │  SP1)       │   │                 │   │
│   └──────┬──────┘   └──────┬──────┘   └────────┬────────┘   │
│          │                 │                    │            │
│          └─────────────────┼────────────────────┘            │
│                            │                                 │
│                    加密数据 + VE 证明                          │
└────────────────────────────┼─────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────┐
│                  数据可用层 (DA Layer — Celestia)              │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐    │
│   │              Private Blockspace (Namespace)          │    │
│   │                                                     │    │
│   │   Blob_1: [encrypted_hand_A, VE_proof_A]            │    │
│   │   Blob_2: [encrypted_move_A_turn1, VE_proof_move]   │    │
│   │   Blob_3: [encrypted_hand_B, VE_proof_B]            │    │
│   │   ...                                               │    │
│   └─────────────────────────────────────────────────────┘    │
│                                                              │
│   特性: 数据不可篡改 · 全球可获取 · 按 Namespace 隔离          │
└────────────────────────────┬─────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────┐
│                    验证层 (Verification Layer)                 │
│                                                              │
│   ┌──────────────────┐   ┌──────────────────────────────┐    │
│   │  结算智能合约     │   │  轻节点 / 链下验证器          │    │
│   │  (EVM / SVM /    │   │  (验证 VE 证明、裁决争议)    │    │
│   │   Move)          │   │                              │    │
│   └──────────────────┘   └──────────────────────────────┘    │
│                                                              │
│   职责: VE 证明验证 · 冲突裁决 · 资产结算 · 超时惩罚          │
└──────────────────────────────────────────────────────────────┘
```

### 5.1 执行层 (Execution Layer)

执行层运行在玩家本地设备上，是玩家与游戏交互的唯一入口。

**组件描述：**

| 组件 | 技术选型 | 职责 |
|---|---|---|
| 游戏客户端 | Unity (WebGL) / React + Canvas | 渲染游戏界面、接受玩家输入、展示已破雾区域 |
| zkVM 引擎 | RiscZero (RISC-V) / SP1 (Succinct) | 执行游戏规则逻辑、生成移动合法性的 ZK 证明 |
| VE 证明生成器 | 自定义电路 (基于 Groth16 / Halo2) | 生成可验证加密证明 |
| 本地密钥管理器 | BIP-39 派生 + AES-256-GCM | 管理玩家的加密/解密密钥对 |
| P2P 通信模块 | libp2p / WebRTC | 玩家间直接通信（可选，用于低延迟场景） |

**关键设计决策：**

- **证明生成在客户端本地完成**：不依赖任何远程 Prover。为了保证用户体验，我们将采用增量证明（Incremental Proving）策略——在玩家"思考"时预计算部分证明。
- **密钥层级派生**：从一个主密钥派生出多个子密钥，每个子密钥对应一场游戏的一个维度（手牌密钥、位置密钥、属性密钥），支持细粒度的选择性披露。

```
Master Key (BIP-39 Seed)
├── Game Session Key
│   ├── Hand Encryption Key (HEK)
│   ├── Position Encryption Key (PEK)
│   ├── Attribute Encryption Key (AEK)
│   └── Reveal Key (RK) — 用于选择性披露
└── Asset Signing Key (用于链上资产操作)
```

### 5.2 数据可用层 (DA Layer — Celestia)

Celestia 作为 DA 层，承担所有游戏状态的持久化存储。

**为什么不用链上合约存储？**

| 对比维度 | 链上合约存储 (e.g., EVM storage) | Celestia DA |
|---|---|---|
| 成本 | 极高 (SSTORE ~20,000 gas) | 极低 (按 Blob 大小计费) |
| 吞吐量 | 受限于区块 Gas Limit | 理论可达 1 Tb/s (未来路线图) |
| 隐私 | 合约 storage 全局可读 | 支持加密 Blob |
| 数据可用性保证 | 需要全节点 | DAS (数据可用性采样) |

**Namespace 设计：**

每场游戏创建一个独立的 Celestia Namespace，格式如下：

```
Namespace = SHA256("fogcard" || game_id || version)[0:29]
```

在该 Namespace 下，所有 Blob 按时间顺序排列，构成完整的游戏状态历史。

**Blob 类型定义：**

| Blob 类型 | 标识符 | 内容 | 发布时机 |
|---|---|---|---|
| `INIT_HAND` | `0x01` | 加密后的初始手牌 + VE 证明 | 开局阶段 |
| `MOVE_ACTION` | `0x02` | 加密后的目标坐标 + VE 证明 | 每回合移动时 |
| `ATTACK_DECLARE` | `0x03` | 攻击意图 + 目标区域 | 战斗阶段 |
| `REVEAL_RESPONSE` | `0x04` | 选择性解密密钥 + 属性明文 | 迷雾破除时 |
| `STATE_CHECKPOINT` | `0x05` | 双方确认的状态快照 | 每 N 回合 |
| `DISPUTE` | `0x06` | 争议数据 + 证据 | 争议发生时 |
| `FORFEIT / TIMEOUT` | `0x07` | 超时声明 | 对手超时时 |

### 5.3 验证层 (Verification Layer)

验证层负责仲裁和结算，是游戏公平性的最终保障。

**双模式验证：**

1. **乐观模式 (Optimistic)**：默认情况下，双方客户端互相验证对方的 VE 证明。如果双方对结果无异议，游戏顺利进行。仅在出现争议时才需要链上合约介入。

2. **强制模式 (Forced)**：当一方检测到对方作弊或对方超时，可向结算合约提交争议（Dispute）。合约将：
   - 从 Celestia 拉取对应 Namespace 的完整 Blob 历史；
   - 验证所有 VE 证明；
   - 按规则裁决胜负；
   - 执行资产转移。

**结算合约部署选项：**

| 部署链 | 优势 | 劣势 |
|---|---|---|
| Ethereum L1 | 最高安全性 | 高 Gas、低速 |
| Arbitrum / OP Stack | 低成本、EVM 兼容 | 需信任 Sequencer |
| Celestia Rollup (自建) | 最高集成度 | 开发成本高 |
| Eclipse (SVM on Celestia) | 高性能 + 原生 Celestia DA | 生态较新 |

**推荐初期方案：** 部署在 Arbitrum 上，利用其低成本和成熟工具链，同时通过 Blobstream 桥接 Celestia 数据证明。

---

## 6. 密码学原语

本章详细介绍 FogCard Protocol 所依赖的密码学组件。

### 6.1 可验证加密 (Verifiable Encryption, VE)

**定义：**

可验证加密是一种密码学方案，允许加密者在生成密文的同时，附带一个证明 $\pi$，使得验证者可以确认：

$$
\text{Verify}(\pi, C, \text{stmt}) = \text{true} \iff \exists\, m \text{ s.t. } C = \text{Enc}(pk, m) \land P(m) = \text{true}
$$

其中：
- $C$ 是密文
- $m$ 是明文
- $pk$ 是加密公钥
- $P(m)$ 是关于明文的谓词（如"明文是合法卡牌"、"坐标在移动范围内"等）

**在 FogCard 中的应用：**

| 场景 | 明文 $m$ | 谓词 $P(m)$ |
|---|---|---|
| 初始手牌 | 10 张卡牌 ID 列表 | 所有 ID ∈ 合法卡池 ∧ 无重复 ∧ 费用 ≤ 预算 |
| 移动操作 | 目标坐标 $(x', y')$ | $\text{dist}((x,y), (x',y')) \le \text{MoveRange}$ |
| 属性声明 | 攻击力 / 防御力 | 属性值 ∈ 该卡牌的合法属性范围 |

**实现方案选择：**

| 方案 | 原理 | 优劣 |
|---|---|---|
| **ElGamal + Sigma Protocol** | 基于离散对数，使用 Chaum-Pedersen 协议证明加密内容的性质 | 简单，但表达能力有限 |
| **Paillier + Range Proof** | 同态加密 + Bulletproofs 范围证明 | 支持算术约束，但证明较大 |
| **SNARK over Encryption** | 在 ZK-SNARK 电路中嵌入加密逻辑 | 最灵活，证明紧凑，但电路复杂 |
| **Hybrid: AES-GCM + ZKP** | 用 AES-GCM 加密数据，用独立的 ZKP 证明明文性质 | 工程友好，性能可控 |

**推荐方案：Hybrid（AES-GCM + ZKP）**

```
VE_Proof = {
    ciphertext: AES-GCM-Encrypt(key, plaintext),
    commitment: Poseidon(plaintext),
    zk_proof:   SNARK.Prove(
                    circuit,
                    public_input  = {commitment, game_rules},
                    private_input = {plaintext, key}
                )
}
```

工作流程：
1. 玩家用 AES-GCM 加密明文（如手牌列表）。
2. 玩家计算明文的 Poseidon 哈希承诺。
3. 玩家在 ZK 电路中证明：
   - "我知道一个明文 $m$ 和一个密钥 $k$"；
   - "$\text{Poseidon}(m) = \text{commitment}$"；
   - "$\text{AES}(k, m) = \text{ciphertext}$"（可选，取决于是否需要绑定加密关系）；
   - "$P(m) = \text{true}$"（明文满足游戏规则）。

### 6.2 密钥管理与选择性披露

**密钥层级：**

```
┌─────────────────────────────────────┐
│         Master Seed (BIP-39)         │
└──────────────────┬──────────────────┘
                   │
         ┌─────────┴─────────┐
         │                   │
    Game Key (GK)       Asset Key (AK)
    (每局游戏独立)      (链上资产签名)
         │
    ┌────┼────┬────┐
    │    │    │    │
   HEK  PEK  AEK  RK
```

| 密钥 | 用途 | 生命周期 |
|---|---|---|
| HEK (Hand Encryption Key) | 加密手牌内容 | 整局游戏 |
| PEK (Position Encryption Key) | 加密单位坐标 | 每回合轮换 |
| AEK (Attribute Encryption Key) | 加密卡牌属性 | 整局游戏 |
| RK (Reveal Key) | 用于选择性披露的派生密钥 | 按需生成 |

**选择性披露协议：**

当需要破雾时（例如两个单位相遇），执行以下协议：

```
Step 1: Player A 声明探测区域 R
        → Publish(ATTACK_DECLARE, R)

Step 2: Player B 的客户端检测到自己的单位 U_b ∈ R
        → 需要披露 U_b 的位置和基础属性

Step 3: Player B 计算 Reveal Key:
        RK_specific = KDF(PEK_b, turn_number || unit_id)

Step 4: Player B 发布:
        Publish(REVEAL_RESPONSE, {
            RK_specific,
            unit_id,
            zk_proof: "此 RK 确实能解密之前发布的位置密文"
        })

Step 5: Player A 使用 RK_specific 解密 B 的位置
        → 迷雾退去
```

### 6.3 承诺方案 (Commitment Scheme)

使用 **Poseidon Hash** 作为承诺函数，原因如下：

- ZK-friendly：在 SNARK 电路中的约束数仅约 200-300 个（相比 SHA-256 的约 25,000 个）；
- 碰撞抗性：128-bit 安全性；
- 广泛支持：RiscZero、SP1、Circom 均有成熟实现。

```
Commitment = Poseidon(value || salt)
```

---

## 7. 关键机制设计

### 7.1 初始化的"暗牌"机制 (Hidden Hand Initialization)

**目标：** 确保每个玩家的初始手牌是合法的，且对对手不可见。

**详细流程：**

```
┌─────────────┐                          ┌─────────────┐
│  Player A   │                          │  Player B   │
└──────┬──────┘                          └──────┬──────┘
       │                                        │
       │  1. 本地选择/抽取 5 张卡牌 (3单位+2陷阱) │
       │     hand_A = [c1, c2, ..., c5]         │
       │                                        │
       │  2. 加密手牌                             │
       │     ct_A = AES-GCM(HEK_A, hand_A)     │
       │                                        │
       │  3. 计算承诺                             │
       │     com_A = Poseidon(hand_A || salt_A)  │
       │                                        │
       │  4. 生成 VE 证明 π_A                    │
       │     证明:                               │
       │     - hand_A 中所有卡牌 ∈ 合法卡池       │
       │     - 无重复卡牌                         │
       │     - 总费用 ≤ 预算上限                   │
       │     - Poseidon(hand_A||salt_A) = com_A  │
       │                                        │
       │  5. 发布至 Celestia                      │
       │     Blob(INIT_HAND, {ct_A, com_A, π_A})│
       │────────────────────────────────────────→│
       │                                        │
       │      6. B 从 Celestia 拉取 Blob         │
       │      7. B 验证 π_A                      │
       │         Verify(π_A, com_A, rules) = ✓   │
       │      8. B 无法解密 ct_A, 但确认 A 合法   │
       │                                        │
       │←────────────────────────────────────────│
       │  B 同样执行 1-5 步                       │
       │  A 验证 B 的证明                          │
       │                                        │
       │  ═══════════════════════════════════    │
       │     双方确认，游戏正式开始 (GAME START)   │
       │  ═══════════════════════════════════    │
```

**ZK 电路伪代码 (Hand Validation Circuit)：**

```rust
// 伪代码：手牌验证电路
fn verify_hand(
    // Public inputs
    commitment: Field,
    rules: GameRules,
    
    // Private inputs (witness)
    hand: [CardId; 5],
    salt: Field,
) -> bool {
    // 1. 验证承诺
    assert!(poseidon_hash(hand, salt) == commitment);
    
    // 2. 验证所有卡牌在合法卡池中
    for card in hand.iter() {
        assert!(rules.legal_card_pool.contains(card));
    }
    
    // 3. 验证无重复
    for i in 0..5 {
        for j in (i+1)..5 {
            assert!(hand[i] != hand[j]);
        }
    }
    
    // 4. 验证总费用不超预算
    let total_cost: u32 = hand.iter()
        .map(|c| rules.card_cost(c))
        .sum();
    assert!(total_cost <= rules.max_budget);
    
    true
}
```

### 7.2 战争迷雾中的移动 (Fog-of-War Movement)

**目标：** 玩家可以合法移动单位，但对手无法得知新位置。

**核心约束：**

$$
\text{distance}(\text{pos}_{old}, \text{pos}_{new}) \le \text{MoveRange}(unit)
$$

使用曼哈顿距离（适合网格地图）：

$$
|x_{new} - x_{old}| + |y_{new} - y_{old}| \le R
$$

或欧几里得距离（适合自由移动）：

$$
(x_{new} - x_{old})^2 + (y_{new} - y_{old})^2 \le R^2
$$

**详细流程：**

```
Player A 的回合：

1. 选择单位 U 和目标位置 (x', y')

2. 加密新位置：
   ct_pos = AES-GCM(PEK_A_turn_n, (x', y'))

3. 计算位置承诺：
   com_pos = Poseidon(x', y', turn_n, unit_id, salt)

4. 生成移动合法性的 ZK 证明 π_move：
   公开输入: com_pos_old (上一回合的位置承诺), com_pos_new, unit_type
   私有输入: (x_old, y_old), (x_new, y_new), salts
   
   证明:
   - com_pos_old 确实是上一回合发布的承诺
   - com_pos_new = Poseidon(x_new, y_new, ...)
   - distance((x_old,y_old), (x_new,y_new)) ≤ MoveRange(unit_type)
   - (x_new, y_new) 在地图边界内
   - (x_new, y_new) 不在已知障碍物上

5. 发布至 Celestia：
   Blob(MOVE_ACTION, {ct_pos, com_pos_new, π_move, turn_n, unit_id})
```

**对手视角：**

```
Player B 收到 Blob 后：

1. 验证 π_move → 确认 A 的移动合法
2. 无法解密 ct_pos → 不知道 A 的新位置
3. 更新本地状态：A 的单位 U 已移动，位置未知
4. 客户端界面：A 的单位在 B 的屏幕上消失在迷雾中
```

**ZK 电路伪代码 (Movement Circuit)：**

```rust
fn verify_movement(
    // Public inputs
    com_pos_old: Field,
    com_pos_new: Field,
    unit_type: UnitType,
    map_config: MapConfig,
    
    // Private inputs
    x_old: u32, y_old: u32,
    x_new: u32, y_new: u32,
    salt_old: Field, salt_new: Field,
) -> bool {
    // 1. 验证旧位置承诺
    assert!(poseidon(x_old, y_old, salt_old) == com_pos_old);
    
    // 2. 验证新位置承诺
    assert!(poseidon(x_new, y_new, salt_new) == com_pos_new);
    
    // 3. 验证移动距离
    let move_range = get_move_range(unit_type);
    let dx = abs_diff(x_new, x_old);
    let dy = abs_diff(y_new, y_old);
    assert!(dx + dy <= move_range); // 曼哈顿距离
    
    // 4. 验证边界
    assert!(x_new < map_config.width);
    assert!(y_new < map_config.height);
    
    // 5. 验证不在障碍物上
    assert!(!map_config.is_obstacle(x_new, y_new));
    
    true
}
```

### 7.3 "相遇即破雾" (Encounter & Selective Disclosure)

**目标：** 当两个玩家的单位距离足够近时，自动触发信息披露。

**挑战：** 在双方位置都加密的情况下，如何判断"距离足够近"？

**解决方案：区域探测协议（Area Probe Protocol）**

我们采用主动探测机制：每个单位有一个"视野范围"（Vision Range），玩家每回合可以声明探测一个区域。

```
Phase 1: 探测声明 (Probe Declaration)
────────────────────────────────────

Player A 声明探测区域:
  probe_region = {center: (px, py), radius: V}
  
  其中 V = Vision_Range(unit_type_A)
  (px, py) 必须在 A 的某个单位的视野范围内

A 需要证明:
  ∃ unit_A s.t. distance(unit_A.pos, (px, py)) ≤ V

发布:
  Blob(PROBE_DECLARE, {
      probe_region,     // 公开：探测区域
      unit_A_id,        // 公开：哪个单位在探测
      π_probe           // 证明：该单位确实在探测范围内
  })


Phase 2: 存在性检测 (Presence Check)
────────────────────────────────────

Player B 的客户端接收到探测声明后:
  
  For each unit_B in B.units:
      if distance(unit_B.pos, probe_region.center) ≤ probe_region.radius:
          // 被发现！需要披露
          MUST respond with REVEAL_RESPONSE
      else:
          // 安全，无需响应
          NO-OP

问题：B 可能撒谎（明明在范围内却不响应）


Phase 3: 防作弊保障 (Anti-cheat Guarantee)
────────────────────────────────────────────

方案 A: 强制披露证明 (Forced Disclosure Proof)

  B 必须对每个单位发布"不在区域内"的 ZK 证明：
  
  π_absence: "我的单位 U_b 的位置 (x_b, y_b) 满足
              distance((x_b, y_b), probe_center) > probe_radius"
  
  如果 B 无法提供此证明（因为实际上在范围内），
  则被视为"被探测到"，必须执行 Reveal。

方案 B: 超时强制揭示 (Timeout Forced Reveal)

  如果 B 在 T 个区块内未响应（既未提供 absence 证明也未 reveal），
  则 A 可向结算合约申请强制解密。
  合约从 Celestia 拉取 B 的加密位置数据，
  利用预存的条件解密密钥（Conditional Decryption Key）强制解密。


Phase 4: 信息披露 (Information Reveal)
──────────────────────────────────────

如果 B 的单位被探测到:

  B 发布:
  Blob(REVEAL_RESPONSE, {
      unit_b_id,
      position: (x_b, y_b),          // 明文位置
      basic_attributes: {card_type, ...}, // 基础可见属性
      reveal_key: RK_specific,         // 仅限该单位的解密密钥
      π_reveal: "被 reveal 的数据与之前加密发布的数据一致"
  })
```

**视觉呈现：**

```
探测前 (Player A 视角):        探测后 (Player A 视角):

  ┌─────────────────┐          ┌─────────────────┐
  │ . . . ░ ░ ░ ░ . │          │ . . . ░ ░ ░ ░ . │
  │ . . ░ ░ ░ ░ . . │          │ . . ░ ░ ░ ░ . . │
  │ . ░ ░ A ░ ░ . . │          │ . ░ ░ A ░ ░ . . │
  │ . . ░ ░ ░ ░ . . │          │ . . ░ ░ B ░ . . │  ← B 出现！
  │ . . . ░ ░ . . . │          │ . . . ░ ░ . . . │
  │ . . . . . . . . │          │ . . . . . . . . │
  └─────────────────┘          └─────────────────┘
  
  ░ = 已探索区域    . = 迷雾    A/B = 单位
```

### 7.4 战斗与结算 (Combat Resolution)

**触发条件：** 当双方单位在同一格或相邻格时，进入战斗阶段。

**战斗协议：**

```
Step 1: 双方同时提交战斗承诺
────────────────────────────
  A: com_battle_A = Poseidon(attack_A, defense_A, skill_A, salt_A)
  B: com_battle_B = Poseidon(attack_B, defense_B, skill_B, salt_B)
  
  双方将承诺发布至 Celestia

Step 2: 双方同时揭示战斗属性
────────────────────────────
  A: Reveal(attack_A, defense_A, skill_A, salt_A)
  B: Reveal(attack_B, defense_B, skill_B, salt_B)
  
  附带证明: 揭示的属性与该卡牌的注册属性一致

Step 3: 确定性战斗计算
────────────────────────────
  damage_to_B = max(0, attack_A - defense_B) * skill_modifier_A
  damage_to_A = max(0, attack_B - defense_A) * skill_modifier_B
  
  new_hp_A = hp_A - damage_to_A
  new_hp_B = hp_B - damage_to_B

Step 4: 更新加密状态
────────────────────────────
  双方将新的 HP 加密后发布至 Celestia
  如果某单位 HP ≤ 0, 标记为阵亡（公开信息）
```

**同时提交保障：**

为防止一方看到对方的战斗属性后再决定自己的行动，采用 **时间锁 + 哈希锁** 双重保障：

1. 双方必须在同一个 Celestia 区块（或相邻区块）内提交 `com_battle`；
2. 提交后，等待 1 个区块确认；
3. 在下一个区块中提交 Reveal；
4. 若一方未在规定时间内 Reveal，按弃权处理。

### 7.5 公平随机数生成 (Fair Randomness)

游戏中的随机事件（如暴击判定、随机抽卡）需要一个双方都信任的随机源。

**方案：Commit-Reveal 联合随机**

```
1. A 生成随机数 r_A, 提交 com_r_A = Hash(r_A)
2. B 生成随机数 r_B, 提交 com_r_B = Hash(r_B)
3. 双方揭示 r_A 和 r_B
4. 最终随机数 R = Hash(r_A || r_B)
```

任何一方都无法单独操控结果，因为最终随机数依赖于双方的输入。

**增强方案：VDF (Verifiable Delay Function)**

在 Step 4 中加入 VDF 处理：$R = VDF(Hash(r_A \| r_B), T)$，确保即使在 Reveal 阶段存在时序差异，也无法利用时间差计算对自己有利的结果。

---

## 8. 详细业务流程

### 8.1 全局流程图

```
╔══════════════════════════════════════════════════════════════╗
║                      GAME LIFECYCLE                         ║
╠══════════════════════════════════════════════════════════════╣
║                                                             ║
║  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐  ║
║  │ 匹配阶段 │──→│ 开局阶段 │──→│ 游戏回合 │──→│ 结算阶段│  ║
║  │ MATCHING │   │  SETUP   │   │  TURNS   │   │  SETTLE │  ║
║  └──────────┘   └──────────┘   └────┬─────┘   └─────────┘  ║
║                                     │                       ║
║                                     ↓                       ║
║                              ┌──────────────┐               ║
║                              │   回合循环    │               ║
║                              │              │               ║
║                              │ ┌──────────┐ │               ║
║                              │ │ 移动阶段 │ │               ║
║                              │ └────┬─────┘ │               ║
║                              │      ↓       │               ║
║                              │ ┌──────────┐ │               ║
║                              │ │ 探测阶段 │ │               ║
║                              │ └────┬─────┘ │               ║
║                              │      ↓       │               ║
║                              │ ┌──────────┐ │               ║
║                              │ │ 战斗阶段 │ │               ║
║                              │ └────┬─────┘ │               ║
║                              │      ↓       │               ║
║                              │ ┌──────────┐ │               ║
║                              │ │ 结算检查 │─┼─→ 胜负判定    ║
║                              │ └────┬─────┘ │               ║
║                              │      │       │               ║
║                              │      ↓       │               ║
║                              │  下一回合    │               ║
║                              └──────────────┘               ║
║                                                             ║
╚══════════════════════════════════════════════════════════════╝
```

### 8.2 第一阶段：匹配 (Matching)

```
1. Player A 创建游戏房间：
   - 生成 game_id = random_uuid()
   - 创建 Celestia Namespace
   - 质押赌注 (可选): stake_A → 结算合约
   - 发布 Blob(GAME_CREATE, {game_id, rules, stake_amount})

2. Player B 加入房间：
   - 扫描链上/Celestia 上的公开游戏列表
   - 选择加入
   - 质押赌注 (可选): stake_B → 结算合约
   - 发布 Blob(GAME_JOIN, {game_id, player_B_pubkey})

3. 双方交换公钥：
   - A 获得 B 的 VE 公钥
   - B 获得 A 的 VE 公钥
   - 用于后续的选择性披露
```

### 8.3 第二阶段：开局 (Setup)

```
1. 地图生成:
   - 使用 game_id 作为种子确定性生成地图
   - 地图参数（尺寸、障碍物位置）双方一致
   - map = DeterministicMapGen(Hash(game_id))

2. 阵容部署:
   - 各自在本地选择卡牌阵容 (如 5 张: 3 英雄 + 2 陷阱)
   - 部署在己方区域的初始位置
   
   encrypted_formation_A = {
       cards: AES(HEK_A, [card_1, card_2, ..., card_5]),
       positions: AES(PEK_A, [(x1,y1), (x2,y2), ..., (x5,y5)]),
       commitment: Poseidon(cards || positions || salt),
       ve_proof: π_formation  // 证明合法性
   }

3. 双方发布至 Celestia 并互相验证

4. VE 锚点确认 (Anchor Confirmation):
   - A: "我确认 B 的 Blob 已在 Celestia 上可用"
   - B: "我确认 A 的 Blob 已在 Celestia 上可用"
   - 双方签名确认 → 游戏正式开始
```

### 8.4 第三阶段：游戏回合 (Turns)

每回合包含以下子阶段：

```
╔═══════════════════════════════════════════════════╗
║                  TURN N (A's Turn)                 ║
╠═══════════════════════════════════════════════════╣
║                                                   ║
║  1. MOVE PHASE                                    ║
║     ├── A 选择要移动的单位                         ║
║     ├── A 选择目标坐标                             ║
║     ├── 客户端生成 VE 证明                          ║
║     ├── 发布加密移动到 Celestia                     ║
║     └── B 验证证明（但不知道具体位置）              ║
║                                                   ║
║  2. PROBE PHASE                                   ║
║     ├── A 选择探测区域                              ║
║     ├── 发布探测声明                                ║
║     ├── B 检查自己的单位是否在区域内                ║
║     ├── IF 被发现 → B 发送 REVEAL                  ║
║     └── IF 未发现 → B 发送 ABSENCE_PROOF           ║
║                                                   ║
║  3. ACTION PHASE                                  ║
║     ├── A 选择攻击/使用技能/待机                    ║
║     ├── IF 攻击 → 进入战斗子协议                   ║
║     └── IF 技能 → 加密效果发布至 Celestia           ║
║                                                   ║
║  4. END PHASE                                     ║
║     ├── 更新加密状态                                ║
║     ├── 双方状态同步                                ║
║     └── 切换到 B 的回合                             ║
║                                                   ║
╚═══════════════════════════════════════════════════╝
```

### 8.5 第四阶段：结算 (Settlement)

**正常结算：**

```
当满足胜利条件时（如对方所有单位阵亡、对方主基地被摧毁等）：

1. 胜利方提交 VICTORY_CLAIM:
   - 附带对方 HP ≤ 0 的证据
   - 或对方认输信号

2. 败方确认 (ACK) 或提出争议 (DISPUTE)

3. 若确认:
   - 结算合约执行资产转移
   - 释放质押
   
4. 若争议:
   - 进入争议解决流程（见 7.5 节）
```

**异常结算（对手掉线/超时）：**

```
1. 检测超时:
   - 如果对手超过 T 个 Celestia 区块未提交任何 Blob
   - 当前玩家发布 TIMEOUT_CLAIM

2. 宽限期:
   - 对手有额外 T' 个区块的时间恢复
   - 如果恢复，游戏继续

3. 强制结算:
   - 如果宽限期结束仍无响应
   - 当前玩家向结算合约提交所有 Blob 历史
   - 合约验证完整性后判定超时方负
   - 执行资产转移
```

---

## 9. 协议状态机

### 9.1 游戏全局状态机

```
                    ┌──────────┐
                    │  IDLE    │
                    └────┬─────┘
                         │ create_game()
                         ▼
                    ┌──────────┐
                    │ WAITING  │ ←── 等待对手加入
                    └────┬─────┘
                         │ player_join()
                         ▼
                    ┌──────────┐
                    │  SETUP   │ ←── 双方部署阵容
                    └────┬─────┘
                         │ both_ready()
                         ▼
                    ┌──────────┐
              ┌────→│IN_PROGRESS│←────┐
              │     └────┬─────┘     │
              │          │           │
              │    ┌─────┴──────┐    │
              │    │            │    │
              │    ▼            ▼    │
              │ ┌──────┐  ┌──────┐  │
              │ │TURN_A│  │TURN_B│  │
              │ └──┬───┘  └──┬───┘  │
              │    │         │      │
              │    └────┬────┘      │
              │         │           │
              │   next_turn()       │
              └─────────┘           │
                                    │
                    ┌───────────┐    │ victory/timeout
                    │  DISPUTE  │←───┤
                    └─────┬─────┘    │
                          │          │
                    ┌─────┴──────┐   │
                    │  SETTLING  │←──┘
                    └─────┬──────┘
                          │
                    ┌─────┴──────┐
                    │ COMPLETED  │
                    └────────────┘
```

### 9.2 回合内状态机

```
                    ┌──────────────┐
                    │ TURN_START   │
                    └──────┬───────┘
                           │
                    ┌──────┴───────┐
                    │  MOVE_PHASE  │
                    └──────┬───────┘
                           │ submit_move() or skip_move()
                    ┌──────┴───────┐
                    │ PROBE_PHASE  │
                    └──────┬───────┘
                           │ submit_probe()
                    ┌──────┴───────┐
                    │REVEAL_PHASE  │ ←── 等待对方响应
                    └──────┬───────┘
                           │ reveal() or absence_proof() or timeout()
                    ┌──────┴───────┐
                    │ACTION_PHASE  │
                    └──────┬───────┘
                           │ attack() or skill() or wait()
                    ┌──────┴───────┐
                    │COMBAT_RESOLVE│ ←── 仅在有战斗时
                    └──────┬───────┘
                           │
                    ┌──────┴───────┐
                    │  TURN_END    │
                    └──────────────┘
```

---

## 10. 数据结构与编码

### 10.1 Blob 格式

所有发布到 Celestia 的 Blob 遵循统一格式：

```
┌─────────────────────────────────────────────────┐
│                  FogCard Blob                    │
├────────────┬────────────────────────────────────┤
│ Field      │ Description                        │
├────────────┼────────────────────────────────────┤
│ version    │ u8  — 协议版本 (当前: 0x01)        │
│ blob_type  │ u8  — Blob 类型标识符              │
│ game_id    │ [u8; 32] — 游戏唯一标识            │
│ sender     │ [u8; 32] — 发送者公钥              │
│ turn_num   │ u32 — 回合编号                     │
│ timestamp  │ u64 — Unix 时间戳                  │
│ payload    │ bytes — 具体内容（因类型而异）      │
│ signature  │ [u8; 64] — Ed25519 签名            │
└────────────┴────────────────────────────────────┘
```

### 10.2 Payload 结构定义

**INIT_HAND Payload：**

```protobuf
message InitHandPayload {
    bytes encrypted_hand = 1;         // AES-GCM(HEK, hand_data)
    bytes hand_commitment = 2;        // Poseidon(hand || salt)
    bytes ve_proof = 3;               // ZK proof of hand validity
    bytes encryption_pubkey = 4;      // Player's VE public key
}
```

**MOVE_ACTION Payload：**

```protobuf
message MoveActionPayload {
    uint32 unit_id = 1;               // 移动的单位 ID
    bytes encrypted_new_pos = 2;      // AES-GCM(PEK, (x', y'))
    bytes pos_commitment_old = 3;     // 上一回合的位置承诺
    bytes pos_commitment_new = 4;     // 新位置承诺
    bytes ve_proof = 5;               // ZK proof: 移动合法
}
```

**PROBE_DECLARE Payload：**

```protobuf
message ProbeDeclarePayload {
    uint32 prober_unit_id = 1;        // 执行探测的单位 ID
    int32 probe_center_x = 2;         // 探测区域中心 X
    int32 probe_center_y = 3;         // 探测区域中心 Y
    uint32 probe_radius = 4;          // 探测半径
    bytes ve_proof = 5;               // ZK proof: 探测范围合法
}
```

**REVEAL_RESPONSE Payload：**

```protobuf
message RevealResponsePayload {
    uint32 unit_id = 1;               // 被发现的单位 ID
    int32 pos_x = 2;                  // 明文位置 X
    int32 pos_y = 3;                  // 明文位置 Y
    CardInfo card_info = 4;           // 卡牌基础信息
    bytes reveal_key = 5;             // 选择性解密密钥
    bytes consistency_proof = 6;      // 证明与之前加密数据一致
}

message CardInfo {
    uint32 card_type_id = 1;
    string card_name = 2;
    uint32 attack = 3;
    uint32 defense = 4;
    uint32 hp = 5;
    uint32 move_range = 6;
    uint32 vision_range = 7;
}
```

**STATE_CHECKPOINT Payload：**

```protobuf
message StateCheckpointPayload {
    uint32 turn_number = 1;
    bytes state_root = 2;             // Merkle root of game state
    bytes player_a_signature = 3;     // A 对状态的签名
    bytes player_b_signature = 4;     // B 对状态的签名
    repeated UnitState units = 5;     // 所有单位的加密状态
}

message UnitState {
    uint32 unit_id = 1;
    bytes encrypted_position = 2;     // 加密位置
    bytes encrypted_hp = 3;           // 加密 HP
    bytes position_commitment = 4;    // 位置承诺
    bool is_alive = 5;               // 是否存活 (公开)
    bool is_revealed = 6;            // 是否已被发现
}
```

### 10.3 游戏状态数据结构

```rust
/// 完整游戏状态 (每个玩家在本地维护)
struct GameState {
    game_id: [u8; 32],
    current_turn: u32,
    phase: TurnPhase,
    
    // 自己的完整状态 (明文)
    my_units: Vec<Unit>,
    my_hand: Vec<Card>,
    
    // 对手的部分状态 (只有被 reveal 的部分是明文)
    opponent_units: Vec<OpponentUnit>,
    
    // Celestia 数据引用
    blob_history: Vec<BlobReference>,
    
    // 密钥
    keys: PlayerKeys,
}

struct Unit {
    id: u32,
    card: Card,
    position: (u32, u32),
    hp: u32,
    is_alive: bool,
    pos_commitment: [u8; 32],       // 当前位置的承诺
    pos_salt: [u8; 32],             // 位置承诺的盐
}

struct OpponentUnit {
    id: u32,
    is_alive: bool,                  // 公开
    is_revealed: bool,               // 是否已破雾
    revealed_position: Option<(u32, u32)>,  // 如果破雾则有值
    revealed_card: Option<Card>,     // 如果破雾则有值
    last_known_commitment: [u8; 32], // 最近一次的位置承诺
}

struct Card {
    type_id: u32,
    name: String,
    attack: u32,
    defense: u32,
    hp: u32,
    move_range: u32,
    vision_range: u32,
    skills: Vec<Skill>,
    cost: u32,
}

enum TurnPhase {
    MovePhase,
    ProbePhase,
    RevealPhase,
    ActionPhase,
    CombatResolve,
    TurnEnd,
}
```

---

## 11. 安全性分析

### 11.1 威胁模型

| 威胁 | 描述 | 影响级别 |
|---|---|---|
| **T1: 位置窥探** | 攻击者尝试解密对手的位置密文 | 高 |
| **T2: 手牌窥探** | 攻击者尝试得知对手的手牌内容 | 高 |
| **T3: 非法移动** | 玩家提交超出移动范围的坐标 | 高 |
| **T4: 非法卡牌** | 玩家使用不在合法卡池中的卡牌 | 高 |
| **T5: 拒绝揭示** | 被探测到的玩家拒绝 Reveal | 中 |
| **T6: 恶意掉线** | 对手故意断开连接以避免失败 | 中 |
| **T7: 重放攻击** | 重放旧的 Blob 数据冒充新操作 | 中 |
| **T8: 时序攻击** | 利用 Blob 提交时间差获取信息 | 低 |
| **T9: MEV 攻击** | Celestia 节点抢跑或审查 Blob | 低 |

### 11.2 安全保障措施

**针对 T1 & T2 (位置/手牌窥探)：**

- **防御机制：** AES-256-GCM 加密 + 每回合轮换 PEK
- **安全假设：** AES-256 的语义安全性。即使攻击者获得密文，无法推断出明文的任何信息。
- **额外保护：** 密文长度填充（Padding），防止通过密文长度推断手牌数量。

```
// 所有坐标密文统一填充到固定长度
padded_plaintext = pad_to_fixed_length(plaintext, BLOCK_SIZE)
ciphertext = AES-GCM-Encrypt(key, padded_plaintext)
```

**针对 T3 & T4 (非法移动/非法卡牌)：**

- **防御机制：** ZK 证明验证
- **安全假设：** ZK-SNARK 的 Soundness 性质——作弊者无法生成有效证明。
- **验证流程：** 对手客户端和/或结算合约验证每个 VE 证明。

**针对 T5 (拒绝揭示)：**

- **防御机制：** 超时强制揭示 + Absence Proof 义务
- **流程：**

```
IF 探测声明发布后 T 个区块内:
    B 未提交 REVEAL_RESPONSE 且 未提交 ABSENCE_PROOF:
    → A 提交 TIMEOUT_REVEAL_CLAIM
    → 结算合约强制解密 B 的位置数据
    → B 被判定为"被发现" + 扣除信誉分
```

**针对 T6 (恶意掉线)：**

- **防御机制：** Celestia 数据持久性 + 超时结算
- **关键优势：** 所有游戏状态已发布到 Celestia。即使对手完全离线，另一方仍可：
  1. 从 Celestia 获取完整游戏历史
  2. 向结算合约提交 TIMEOUT_CLAIM
  3. 合约验证后判定离线方负

```
// 结算合约伪代码
function claim_timeout(game_id, blob_proofs[]) {
    // 1. 验证最后一个 Blob 的时间戳
    last_blob = get_latest_blob(game_id);
    assert(current_block - last_blob.block > TIMEOUT_BLOCKS);
    
    // 2. 验证所有历史 Blob 的完整性
    for proof in blob_proofs {
        assert(verify_celestia_inclusion(proof));
    }
    
    // 3. 确认超时方
    timeout_player = get_expected_next_actor(last_blob);
    
    // 4. 判定胜负
    settle_game(game_id, winner = opponent_of(timeout_player));
}
```

**针对 T7 (重放攻击)：**

- **防御机制：** 回合编号 + 时间戳 + Nonce
- **每个 Blob 包含：** `turn_num`, `timestamp`, `prev_blob_hash`
- **验证：** 客户端检查 Blob 的时序合法性

**针对 T8 (时序攻击)：**

- **防御机制：** 固定时间窗口提交
- **设计：** 每个阶段有固定的时间窗口（如 30 秒），不论是否已完成操作都在窗口结束时统一发布

**针对 T9 (MEV / Celestia 节点攻击)：**

- **防御机制：** 数据已加密 + Celestia 的去中心化验证者集
- **分析：** 即使 Celestia 节点看到加密 Blob，也无法解密。节点审查 Blob 的成本极高（Celestia 的 DAS 机制确保数据可用性）。

### 11.3 安全性总结

| 安全性质 | 保障机制 | 密码学假设 |
|---|---|---|
| 隐私性 (Privacy) | AES-256-GCM + 密钥隔离 | AES 的 IND-CPA 安全性 |
| 完整性 (Integrity) | Poseidon 承诺 + ZKP | 哈希函数的碰撞抗性 + SNARK Soundness |
| 公平性 (Fairness) | VE 证明 + 超时机制 | SNARK Soundness + 活性假设 |
| 可用性 (Availability) | Celestia DAS + 超时结算 | Celestia 的 DA 保证 |
| 不可抵赖 (Non-repudiation) | Ed25519 签名 + Blob 历史 | 签名方案的不可伪造性 |

---

## 12. 为什么选择 Celestia Private Blockspace？

### 12.1 与替代方案的详细对比

```
┌─────────────────────────────────────────────────────────────────────┐
│                        方案对比矩阵                                 │
├──────────────┬───────────┬───────────┬───────────┬────────────────┤
│    维度       │ 中心化    │ 纯 ZKP    │ Commit-   │ Celestia PB    │
│              │ 服务器    │ (链上)    │ Reveal    │ + VE           │
├──────────────┼───────────┼───────────┼───────────┼────────────────┤
│ 去中心化     │ ✗         │ ✓         │ ✓         │ ✓              │
│ 隐私深度     │ 高*       │ 高        │ 低        │ 高             │
│ 验证成本     │ N/A       │ 极高      │ 低        │ 中等           │
│ 延迟         │ 低        │ 极高      │ 中等      │ 低-中等        │
│ 活性保证     │ ✗         │ ✓         │ 部分      │ ✓              │
│ 抗审查       │ ✗         │ ✓         │ ✓         │ ✓              │
│ 数据可用性   │ 依赖服务器│ 链上存储  │ 链上存储  │ Celestia DA    │
│ 开发复杂度   │ 低        │ 极高      │ 中等      │ 中等           │
│ 可组合性     │ ✗         │ 高        │ 高        │ 高             │
│ 扩展性       │ 有限      │ 差        │ 差        │ 极高           │
├──────────────┼───────────┼───────────┼───────────┼────────────────┤
│ 综合评分     │ ★★☆☆☆    │ ★★★☆☆    │ ★★☆☆☆    │ ★★★★★         │
└──────────────┴───────────┴───────────┴───────────┴────────────────┘

* 中心化服务器的"高隐私"建立在信任服务器的基础上，非密码学保证
```

### 12.2 Celestia 的核心优势

**1. 真正的去中心化迷雾**

传统链上游戏要实现迷雾，要么引入中心化服务器（破坏去中心化），要么使用纯链上 ZKP（延迟不可接受）。Celestia 的方案通过将加密数据存储在 DA 层 + VE 证明验证合法性，在去中心化和性能之间取得了最优平衡。

**2. 安全退出 (Safe Exit)**

这是 Celestia PB 最重要的特性之一。在传统 P2P 游戏中，如果对手关闭客户端，你可能丢失所有游戏进度。在 FogCard 中：

```
对手下线？没关系。

1. 所有历史操作都已以 Blob 形式发布到 Celestia
2. Celestia 的 DAS 确保这些数据永远可获取
3. 你可以随时从 Celestia 重建完整的游戏状态
4. 提交超时声明，强制结算
```

**3. 高性能数据吞吐**

Celestia 的路线图目标是达到 1 Tb/s 的数据吞吐量。这意味着即使是数据密集型的卡牌战略游戏（大量单位、复杂地形、长时间对局），也不会遇到数据瓶颈。

**当前性能参考：**

| 指标 | 当前值 | 未来预期 |
|---|---|---|
| 区块时间 | ~12 秒 | 持续优化 |
| Blob 大小限制 | ~2 MB | 持续扩展 |
| 数据吞吐量 | ~6.67 MB/s | 1 Tb/s (长期) |
| Blob 发布成本 | ~$0.001/KB | 持续降低 |

**4. 与 Rollup 生态的天然整合**

FogCard 的验证层可以部署在任何使用 Celestia 作为 DA 的 Rollup 上，如：

- Arbitrum Orbit + Celestia DA
- OP Stack + Celestia DA
- Eclipse (SVM on Celestia)
- 自建 Sovereign Rollup

通过 **Blobstream** 桥接，结算合约可以直接验证 Celestia 上的 Blob 存在性证明。

---

## 13. 性能与 Gas 估算

### 13.1 ZK 证明生成时间估算

| 证明类型 | 电路约束数 (估计) | RiscZero (CPU) | SP1 (CPU) | GPU 加速 |
|---|---|---|---|---|
| 手牌验证 | ~10K | ~2s | ~1.5s | ~0.3s |
| 移动合法性 | ~5K | ~1s | ~0.8s | ~0.2s |
| 探测范围验证 | ~3K | ~0.5s | ~0.4s | ~0.1s |
| 不在区域内证明 | ~5K | ~1s | ~0.8s | ~0.2s |
| 战斗属性验证 | ~8K | ~1.5s | ~1.2s | ~0.3s |

**用户体验优化策略：**

1. **预计算**：在玩家思考时，预计算可能的移动方向的证明
2. **增量证明**：利用上一回合的证明状态加速本回合证明生成
3. **WebGPU 加速**：在浏览器中利用 GPU 并行计算

### 13.2 Celestia Blob 成本估算

| Blob 类型 | 预估大小 | 单次成本 (当前) | 每局游戏 (30 回合) |
|---|---|---|---|
| INIT_HAND | ~2 KB | ~$0.002 | $0.002 × 2 = $0.004 |
| MOVE_ACTION | ~1 KB | ~$0.001 | $0.001 × 60 = $0.060 |
| PROBE_DECLARE | ~0.5 KB | ~$0.0005 | $0.0005 × 30 = $0.015 |
| REVEAL_RESPONSE | ~1 KB | ~$0.001 | $0.001 × 10 = $0.010 |
| STATE_CHECKPOINT | ~3 KB | ~$0.003 | $0.003 × 6 = $0.018 |
| **总计** | | | **~$0.11 / 局** |

### 13.3 结算合约 Gas 估算 (Arbitrum)

| 操作 | Gas 消耗 (估计) | 成本 (Arbitrum) |
|---|---|---|
| 创建游戏 | ~100K | ~$0.01 |
| VE 证明验证 (Groth16) | ~230K | ~$0.02 |
| 超时结算 | ~300K | ~$0.03 |
| 争议裁决 | ~500K | ~$0.05 |
| **正常对局总结算成本** | | **~$0.03** |

**玩家每局总成本估算：~$0.14 (DA + Settlement)**

---

## 14. 开发路线图

### Phase 0: 研究与验证 (Research & PoC) — 4-6 周

```
┌──────────────────────────────────────────────────────────────┐
│  Week 1-2: 密码学原语研究                                     │
│  ├── 调研 Celestia Private Blockspace 最新 API               │
│  ├── 调研 Hibachi 架构（首个 PB 应用）                        │
│  ├── 选定 VE 方案 (Hybrid AES + ZKP)                        │
│  └── 选定 ZK 框架 (RiscZero vs SP1 基准测试)                 │
│                                                              │
│  Week 3-4: 最小可行证明 (Minimum Viable Proof)               │
│  ├── 实现"猜数字"VE Demo:                                    │
│  │   "我加密了一个 1-100 的数字，证明它确实在范围内"          │
│  ├── 实现"坐标移动"VE Demo:                                  │
│  │   "我加密了新坐标，证明移动距离 ≤ 3"                       │
│  └── Celestia Testnet 上发布/读取加密 Blob                   │
│                                                              │
│  Week 5-6: 端到端 PoC                                        │
│  ├── 两个客户端通过 Celestia 交换加密移动                     │
│  ├── 验证对方移动合法性                                       │
│  └── 简单的探测 & Reveal 流程                                │
└──────────────────────────────────────────────────────────────┘
```

### Phase 1: 核心协议开发 (Core Protocol) — 8-10 周

```
┌──────────────────────────────────────────────────────────────┐
│  功能模块:                                                    │
│  ├── 完整的 ZK 电路库:                                        │
│  │   ├── HandValidationCircuit                               │
│  │   ├── MovementCircuit                                     │
│  │   ├── ProbeCircuit                                        │
│  │   ├── AbsenceProofCircuit                                 │
│  │   └── CombatCircuit                                       │
│  │                                                           │
│  ├── 客户端 SDK (TypeScript/Rust):                            │
│  │   ├── 密钥管理模块                                         │
│  │   ├── Blob 编解码模块                                      │
│  │   ├── VE 证明生成/验证模块                                 │
│  │   ├── Celestia 交互模块                                    │
│  │   └── 游戏状态管理模块                                     │
│  │                                                           │
│  ├── 结算合约 (Solidity):                                     │
│  │   ├── GameFactory.sol                                     │
│  │   ├── FogCardGame.sol                                     │
│  │   ├── VEVerifier.sol                                      │
│  │   ├── DisputeResolver.sol                                 │
│  │   └── AssetSettlement.sol                                 │
│  │                                                           │
│  └── 集成测试:                                                │
│      ├── 完整对局模拟 (Happy Path)                            │
│      ├── 超时场景测试                                         │
│      ├── 争议场景测试                                         │
│      └── 压力测试 (100+ 回合)                                 │
└──────────────────────────────────────────────────────────────┘
```

### Phase 2: 游戏客户端开发 (Game Client) — 8-12 周

```
┌──────────────────────────────────────────────────────────────┐
│  游戏设计:                                                    │
│  ├── 卡牌设计 (50+ 张初始卡牌, 5 种职业)                      │
│  ├── 地图设计 (8×8 / 10×10 网格, 多种地形)                    │
│  ├── 技能系统                                                 │
│  └── 平衡性调整                                               │
│                                                              │
│  前端开发:                                                    │
│  ├── 游戏界面 (Unity WebGL / React + Pixi.js)                │
│  ├── 迷雾渲染系统                                             │
│  ├── 战斗动画                                                 │
│  ├── 卡牌展示与交互                                           │
│  └── 集成 Web3 钱包 (MetaMask / WalletConnect)               │
│                                                              │
│  后端服务 (可选, 仅用于匹配):                                  │
│  ├── 匹配服务器 (WebSocket)                                   │
│  ├── 排行榜 (可链上或链下)                                    │
│  └── 游戏回放系统                                             │
└──────────────────────────────────────────────────────────────┘
```

### Phase 3: 测试网上线 (Testnet Launch) — 4-6 周

```
┌──────────────────────────────────────────────────────────────┐
│  ├── Celestia Mocha Testnet 部署                              │
│  ├── Arbitrum Sepolia 结算合约部署                             │
│  ├── 封闭 Alpha 测试 (50 名玩家)                              │
│  ├── Bug 修复与性能优化                                       │
│  ├── 安全审计 (ZK 电路 + 合约)                                │
│  └── 公开 Beta 测试                                           │
└──────────────────────────────────────────────────────────────┘
```

### Phase 4: 主网上线与迭代 (Mainnet & Beyond) — 持续

```
┌──────────────────────────────────────────────────────────────┐
│  ├── Celestia Mainnet + Arbitrum One 部署                     │
│  ├── NFT 卡牌系统上线                                         │
│  ├── 锦标赛系统                                               │
│  ├── 多人模式 (2v2, 自由对战)                                 │
│  ├── 自定义规则游戏                                           │
│  ├── SDK 开源，支持社区创建新游戏模式                          │
│  └── 跨链资产桥接                                             │
└──────────────────────────────────────────────────────────────┘
```

### 里程碑总览

```
2025 Q3        Q4         2026 Q1       Q2         Q3
  │            │            │            │          │
  ▼            ▼            ▼            ▼          ▼
  Phase 0      Phase 1      Phase 2      Phase 3    Phase 4
  PoC          Core Dev     Game Client  Testnet    Mainnet
  ████         ████████     ████████████ ██████     ████→→
```

---

## 15. 参考文献

1. **Celestia Documentation** — *Data Availability Layer*  
   https://docs.celestia.org/

2. **Celestia Private Blockspace** — *Verifiable Encryption on Celestia*  
   Celestia Labs Blog, 2025

3. **Hibachi** — *First Perpetual Trading Platform using Private Blockspace*  
   https://hibachi.xyz/

4. **Blobstream** — *Streaming Celestia data to Ethereum*  
   https://docs.celestia.org/developers/blobstream

5. **RiscZero** — *General-purpose zkVM*  
   https://www.risczero.com/

6. **SP1 (Succinct)** — *High-performance zkVM*  
   https://docs.succinct.xyz/

7. **Camenisch, J. & Shoup, V.** — *Practical Verifiable Encryption and Decryption of Discrete Logarithms*  
   CRYPTO 2003

8. **Groth, J.** — *On the Size of Pairing-Based Non-interactive Arguments*  
   EUROCRYPT 2016

9. **Grassi, L. et al.** — *Poseidon: A New Hash Function for Zero-Knowledge Proof Systems*  
   USENIX Security 2021

10. **Dark Forest** — *zkSNARK Space Warfare*  
    https://zkga.me/ — 首个使用 ZKP 实现迷雾的链上游戏

11. **MUD Framework** — *Fully On-chain Game Engine*  
    https://mud.dev/

12. **Dojo Engine** — *Provable Game Engine on Starknet*  
    https://www.dojoengine.org/

---

## 附录 A: 卡牌设计示例

| 卡牌名称 | 职业 | 攻击力 | 防御力 | HP | 移动范围 | 视野范围 | 费用 | 特殊技能 |
|---|---|---|---|---|---|---|---|---|
| 暗影刺客 | 刺客 | 8 | 2 | 4 | 4 | 3 | 5 | 暗杀：首次攻击伤害×2 |
| 铁壁守卫 | 坦克 | 3 | 8 | 10 | 1 | 2 | 4 | 嘲讽：强制相邻敌方攻击自己 |
| 鹰眼斥候 | 侦察 | 4 | 3 | 5 | 3 | 5 | 3 | 远视：视野范围+2 |
| 战争法师 | 法师 | 7 | 1 | 3 | 2 | 3 | 6 | AOE：对目标周围 1 格造成伤害 |
| 圣光牧师 | 辅助 | 2 | 4 | 6 | 2 | 2 | 4 | 治愈：恢复相邻友方 3 HP |
| 剧毒陷阱 | 陷阱 | 0 | 0 | 1 | 0 | 0 | 2 | 触发：敌方踩中时失去 4 HP，陷阱销毁 |
| 侦察守卫 | 陷阱 | 0 | 0 | 1 | 0 | 3 | 2 | 视野：提供固定 3 格视野，隐身状态不可见 |

## 附录 B: 地图示例 (8×8 网格)

```
    0   1   2   3   4   5   6   7
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
0 │ A │ A │ . │ . │ . │ . │ B │ B │  A = Player A 部署区
  ├───┼───┼───┼───┼───┼───┼───┼───┤
1 │ A │ A │ . │ ▓ │ ▓ │ . │ B │ B │  B = Player B 部署区
  ├───┼───┼───┼───┼───┼───┼───┼───┤
2 │ . │ . │ . │ . │ . │ . │ . │ . │  ▓ = 障碍物 (不可通行)
  ├───┼───┼───┼───┼───┼───┼───┼───┤
3 │ . │ ▓ │ . │ ♦ │ . │ . │ ▓ │ . │  ♦ = 资源点 (可选机制)
  ├───┼───┼───┼───┼───┼───┼───┼───┤
4 │ . │ ▓ │ . │ . │ ♦ │ . │ ▓ │ . │  . = 空地
  ├───┼───┼───┼───┼───┼───┼───┼───┤
5 │ . │ . │ . │ . │ . │ . │ . │ . │
  ├───┼───┼───┼───┼───┼───┼───┼───┤
6 │ B │ B │ . │ ▓ │ ▓ │ . │ A │ A │
  ├───┼───┼───┼───┼───┼───┼───┼───┤
7 │ B │ B │ . │ . │ . │ . │ A │ A │
  └───┴───┴───┴───┴───┴───┴───┴───┘
```

## 附录 C: 消息序列图 (完整对局)

```
Player A                 Celestia DA              Player B
   │                         │                        │
   │  ── GAME_CREATE ──────→ │                        │
   │                         │ ←── GAME_JOIN ──────── │
   │                         │                        │
   │  ── INIT_HAND_A ──────→ │                        │
   │                         │ ←── INIT_HAND_B ────── │
   │                         │                        │
   │  ←── verify π_B ─────── │                        │
   │                         │ ── verify π_A ────────→│
   │                         │                        │
   │  ══ GAME START ═══════════════════════════════   │
   │                         │                        │
   │  ── MOVE_A_T1 ────────→ │                        │
   │                         │ ── verify π_move ────→ │
   │                         │                        │
   │                         │ ←── MOVE_B_T1 ──────── │
   │  ←── verify π_move ──── │                        │
   │                         │                        │
   │  ── PROBE_A_T2 ───────→ │                        │
   │                         │ ── check presence ───→ │
   │                         │                        │
   │                         │ ←── ABSENCE_PROOF ──── │
   │  ←── verify π_abs ───── │                        │
   │                         │                        │
   │           ...           │         ...            │
   │                         │                        │
   │  ── PROBE_A_T5 ───────→ │                        │
   │                         │ ── check presence ───→ │
   │                         │                        │  ← B 的单位被发现！
   │                         │ ←── REVEAL_B ────────── │
   │  ←── decrypt & render ── │                        │
   │                         │                        │
   │  ── ATTACK_A ─────────→ │                        │
   │                         │ ←── COMBAT_COMMIT_B ── │
   │  ── COMBAT_COMMIT_A ──→ │                        │
   │                         │ ←── COMBAT_REVEAL_B ── │
   │  ── COMBAT_REVEAL_A ──→ │                        │
   │                         │                        │
   │  ←── COMBAT_RESULT ════════════════════════════→ │
   │                         │                        │
   │           ...           │         ...            │
   │                         │                        │
   │  ══ GAME END ═════════════════════════════════   │
   │                         │                        │
   │  ── VICTORY_CLAIM ────→ │                        │
   │                         │ ←── ACK ────────────── │
   │                         │                        │
   │  ════ SETTLEMENT (on-chain) ════════════════════ │
   │                         │                        │
```

---

**文档结束**

*FogCard Protocol — 让链上游戏第一次拥有真正的战争迷雾。*

*ccboomer. All rights reserved.*
*This document is released under CC BY-SA 4.0.*