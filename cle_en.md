# Technical Design Document: Decentralized Fog of War Card Game Based on Celestia

**FogCard Protocol — Technical Design Document**

**Version:** v1.0  
**Date:** July 2025  
**Status:** Draft / RFC  

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Background & Motivation](#2-background--motivation)
3. [The Vision](#3-the-vision)
4. [Glossary](#4-glossary)
5. [Architecture](#5-architecture)
6. [Cryptographic Primitives](#6-cryptographic-primitives)
7. [Core Mechanisms](#7-core-mechanisms)
8. [User Flow](#8-user-flow)
9. [Protocol State Machine](#9-protocol-state-machine)
10. [Data Structures](#10-data-structures)
11. [Security Analysis](#11-security-analysis)
12. [Why Celestia Private Blockspace?](#12-why-celestia-private-blockspace)
13. [Performance & Cost](#13-performance--cost)
14. [Roadmap](#14-roadmap)
15. [References](#15-references)

---

## 1. Abstract

This document describes the complete technical design of the **FogCard Protocol**—a decentralized card battling game running on the Celestia Data Availability Layer. By leveraging **Private Blockspace** and **Verifiable Encryption (VE)** technologies, the game achieves the following without any centralized server:

- **Fog of War**: Players' card positions and hand contents remain hidden from the opponent;
- **Verifiable Fairness**: Every action is accompanied by cryptographic proofs, ensuring no one can cheat;
- **Asset Safety**: Players can unilaterally settle the game even if the opponent goes offline or maliciously disconnects.

This document is intended for engineers, researchers, and potential collaborators, providing a panoramic description from high-level architecture to low-level data structures.

---

## 2. Background & Motivation

### 2.1 The Dilemma of On-chain Games

Fully On-chain Games (FOCG) experienced significant growth between 2023 and 2025. With frameworks like MUD and Dojo, developers can now build complex game logic on EVM or StarkNet. However, these games naturally face a fatal flaw:

> **The blockchain is transparent.**

All on-chain states are visible to everyone, which means:

| Traditional Game Mechanic | On-chain Difficulty | Reason |
|---|---|---|
| Fog of War | ★★★★★ | All position coordinates are publicly readable |
| Hidden Cards / Hand | ★★★★★ | Contract storage is globally queryable |
| Ambush | ★★★★★ | Anyone can pre-read the next move |
| Random Card Draw | ★★★★ | On-chain randomness can be manipulated by miners |

To solve the transparency issue, the industry has attempted the following solutions:

1. **Commit-Reveal Mechanism**: Players first submit a hash of their action (Commit) and later reveal it (Reveal). The problem is that the Reveal phase can be maliciously skipped, and the legitimacy of the Commit content cannot be verified upfront.
2. **Off-chain Server + On-chain Settlement**: Introducing centralized servers to manage private states. The problem is that servers can cheat, crash, or censor.
3. **Pure ZK Solutions**: Using Zero-Knowledge Proofs to hide states. The problem is that generating ZKPs is extremely expensive, resulting in unacceptable delays, especially for complex game logic.

### 2.2 Re-inventing Paradigms with Celestia

The **Private Blockspace** concept proposed by Celestia in 2025 offers a brand new solution to this dilemma. Its core idea is:

> Publish encrypted data to Celestia's Data Availability (DA) Layer, paired with Verifiable Encryption (VE) proofs, achieving "data remains unreadable by others, but its legality is provable."

The uniqueness of this paradigm lies in:

- **Data is always online**: You don't rely on the opponent or a server to fetch data;
- **Verifiability**: VE proofs ensure that the encrypted data complies with predefined rules;
- **Selective Disclosure**: Decrypting specific fields only when certain conditions are met.

The FogCard Protocol is precisely the first Fog of War card game built on this paradigm.

---

## 3. The Vision

**Summary:**
> Breaking the "absolute transparency" limitations of traditional on-chain games using Celestia's Private Blockspace and Verifiable Encryption technologies. Ultimately realizing a card battling game with information asymmetry (Fog of War), provable fairness, and high asset safety—all without needing centralized servers.

### 3.1 Design Principles

| Principle | Description |
|---|---|
| **No Trusted Third Party** | The game runs entirely without servers or middlemen |
| **Verifiable Fairness** | Every action can be validated by opponents and arbitrary third parties |
| **Information Asymmetry** | Players only receive "need-to-know" information, realizing true Fog of War |
| **Liveness Guarantee** | The game can proceed and settle even if opponents go offline |
| **Asset Sovereignty** | In-game assets (Card NFTs, tokens) are 100% self-custodial |

---

## 4. Glossary

| Term | Abbr. | Definition |
|---|---|---|
| Verifiable Encryption | VE | A cryptographic scheme enabling a prover to prove the plaintext inside a ciphertext satisfies a given property, without revealing the plaintext itself. |
| Private Blockspace | PB | A concept by Celestia, referring to blockspace in the DA layer that stores encrypted data. |
| Zero-Knowledge Proof | ZKP | Proof allowing one party to prove a statement to another without revealing any additional information. |
| Data Availability | DA | Guarantees that block data has been published and is available for anyone to retrieve. |
| Selective Disclosure | SD | Decrypting only specific fields when certain conditions are met. |
| zkVM | — | Zero-Knowledge Virtual Machine, executing programs inside a VM and generating ZK proofs of correct execution. |
| TEE | — | Trusted Execution Environment. |
| Namespace | NS | Logical partitions in Celestia used to isolate data for different applications. |
| Blob | — | A unit of data published on Celestia. |
| Anchor | — | VE anchor, used to prove encrypted data has been published and is available. |
| Commit-Reveal | CR | A traditional two-phase commit and reveal scheme. |

---

## 5. Architecture

FogCard Protocol adopts a decoupled three-layer architecture. Each layer's responsibilities are precise and independently upgradeable.

```
┌──────────────────────────────────────────────────────────────┐
│                     Execution Layer                          │
│                                                              │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐    │
│   │ Game Client │   │ zkVM Engine │   │ Local Keystore  │    │
│   │ (Unity/Web) │   │ (RiscZero/  │   │                 │    │
│   │             │   │  SP1)       │   │                 │    │
│   └──────┬──────┘   └──────┬──────┘   └────────┬────────┘    │
│          │                 │                    │            │
│          └─────────────────┼────────────────────┘            │
│                            │                                 │
│              Encrypted Data + VE Proofs                      │
└────────────────────────────┼─────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────┐
│                  Data Availability Layer (Celestia)          │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐    │
│   │              Private Blockspace (Namespace)         │    │
│   │                                                     │    │
│   │   Blob_1: [encrypted_hand_A, VE_proof_A]            │    │
│   │   Blob_2: [encrypted_move_A_turn1, VE_proof_move]   │    │
│   │   Blob_3: [encrypted_hand_B, VE_proof_B]            │    │
│   │   ...                                               │    │
│   └─────────────────────────────────────────────────────┘    │
│                                                              │
│  Features: Immutable · Globally Available · NS Isolation     │
└────────────────────────────┬─────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────┐
│                    Verification Layer                          │
│                                                              │
│   ┌──────────────────┐   ┌──────────────────────────────┐    │
│   │ Settlement Smart │   │ Light Nodes / Offchain     │    │
│   │ Contract (EVM /  │   │ Verifiers (Verify VE &     │    │
│   │ SVM / Move)      │   │ Arbitrate Disputes)          │    │
│   └──────────────────┘   └──────────────────────────────┘    │
│                                                              │
│  Duties: VE Proof Verification · Dispute Arbitration         │
│          Asset Settlement · Timeout Penalties                │
└──────────────────────────────────────────────────────────────┘
```

### 5.1 Execution Layer

The execution layer runs locally on players' devices, serving as the sole entry point for game interaction.

**Component Description:**

| Component | Technology | Responsibility |
|---|---|---|
| Game Client | Unity (WebGL) / React + Canvas | Render game UI, process player inputs, display explored fog areas |
| zkVM Engine | RiscZero (RISC-V) / SP1 (Succinct) | Execute game logic, generate ZK proofs for legal moves |
| VE Prover | Custom circuits (Groth16 / Halo2) | Generate Verifiable Encryption proofs |
| Keystore | BIP-39 Derivation + AES-256-GCM | Manage player's encryption/decryption key pairs |
| P2P Comms | libp2p / WebRTC | Direct communication between players (optional, for low latency) |

**Key Design Decisions:**

- **Proof Generation is Done Locally**: Doesn't rely on remote provers. To ensure user experience, we employ Incremental Proving—pre-computing parts of the proof while players are "thinking."
- **Hierarchical Key Derivation**: Deriving multiple sub-keys from a master key, each corresponding to a single game dimension (hand key, position key, attribute key) to support granular selective disclosure.

```
Master Key (BIP-39 Seed)
├── Game Session Key
│   ├── Hand Encryption Key (HEK)
│   ├── Position Encryption Key (PEK)
│   ├── Attribute Encryption Key (AEK)
│   └── Reveal Key (RK) — used for selective disclosure
└── Asset Signing Key (for on-chain asset ops)
```

### 5.2 Data Availability Layer (Celestia)

Celestia serves as the DA layer to handle the persistent storage of all game states.

**Why Not Use On-chain Storage?**

| Comparison Dimension | On-chain (e.g., EVM storage) | Celestia DA |
|---|---|---|
| Cost | Extremely High (SSTORE ~20k gas) | Extremely Low (priced by Blob size) |
| Throughput | Bottlenecked by block gas limit | Theoretically up to 1 Tb/s (future roadmap) |
| Privacy | Storage is globally readable | Supports encrypted Blobs |
| Data Availability | Requires full nodes | DAS (Data Availability Sampling) |

**Namespace Design:**

Each session creates an independent Celestia Namespace formatted as follows:

```
Namespace = SHA256("fogcard" || game_id || version)[0:29]
```

Under this namespace, all Blobs are ordered chronologically, forming the entire game state history.

**Blob Types:**

| Blob Type | Identifier | Content | Timing |
|---|---|---|---|
| `INIT_HAND` | `0x01` | Encrypted initial hand + VE proof | Game start |
| `MOVE_ACTION` | `0x02` | Encrypted target coordinates + VE proof | During each movement phase |
| `ATTACK_DECLARE` | `0x03` | Attack intention + Target area | Combat phase |
| `REVEAL_RESPONSE` | `0x04` | Selective decryption key + Plaintext attributes | When fog is breached |
| `STATE_CHECKPOINT` | `0x05` | Mutual state snapshot | Every N turns |
| `DISPUTE` | `0x06` | Dispute data + Evidence | Upon dispute |
| `FORFEIT / TIMEOUT` | `0x07` | Timeout declaration | When the opponent times out |

### 5.3 Verification Layer

The verification layer handles arbitration and settlement, acting as the ultimate guarantor of fairness.

**Dual Verification Mode:**

1. **Optimistic Mode**: By default, clients mutually verify each other's VE proofs. If both agree, the game proceeds fluidly. Smart contracts only intervene when disputes arise.
2. **Forced Mode (Dispute)**: When one party detects cheating or timeouts, they can submit a Dispute to the settlement contract. The contract will:
   - Fetch the full Blob history from the Celestia Namespace;
   - Verify all VE proofs;
   - Output the ruling according to the rules;
   - Execute asset transfers.

**Settlement Contract Options:**

| Deployment Chain | Advantages | Disadvantages |
|---|---|---|
| Ethereum L1 | Highest security | High Gas, slow speeds |
| Arbitrum / OP Stack | Low cost, EVM compatible | Requires trusting the Sequencer |
| Celestia Rollup (Custom) | Best integration | High development overhead |
| Eclipse (SVM on Celestia) | High performance + Native Celestia DA | Newer ecosystem |

**Recommended Initial Approach:** Deploy on Arbitrum to leverage low costs and mature toolchains, bridging Celestia data proofs via Blobstream.

---

## 6. Cryptographic Primitives

### 6.1 Verifiable Encryption (VE)

**Definition:**

Verifiable Encryption allows an encryptor to generate a ciphertext along with a proof $\pi$ so that a verifier can confirm:

$$
\text{Verify}(\pi, C, \text{stmt}) = \text{true} \iff \exists\, m \text{ s.t. } C = \text{Enc}(pk, m) \land P(m) = \text{true}
$$

Where:
- $C$ is the ciphertext
- $m$ is the plaintext
- $pk$ is the encryption public key
- $P(m)$ is the property the plaintext must satisfy (e.g., "The coordinate is within range")

**Application in FogCard:**

| Scenario | Plaintext $m$ | Predicate $P(m)$ |
|---|---|---|
| Initial Hand | List of 5 Card IDs | All IDs ∈ Legal Pool ∧ No duplicates ∧ Cost ≤ Budget |
| Movement Action | Target coordinates $(x', y')$ | $\text{dist}((x,y), (x',y')) \le \text{MoveRange}$ |
| Attribute Declaration | Attack / Defense | Values ∈ Legal attribute range for the unit |

**Proposed Implementation: Hybrid (AES-GCM + ZKP)**

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

### 6.2 Key Management and Selective Disclosure

**Selective Disclosure Protocol:**

When fog is breached (e.g., two units closely encounter):

```
Step 1: Player A declares search area R
        → Publish(ATTACK_DECLARE, R)

Step 2: Player B's client detects its unit U_b ∈ R
        → Needs to reveal U_b's position and base stats

Step 3: Player B computes the Reveal Key:
        RK_specific = KDF(PEK_b, turn_number || unit_id)

Step 4: Player B publishes:
        Publish(REVEAL_RESPONSE, {
            RK_specific,
            unit_id,
            zk_proof: "This RK successfully decrypts the prior position ciphertext"
        })

Step 5: Player A decrypts B's position with RK_specific
        → Fog lifts
```

### 6.3 Commitment Scheme

We utilize the **Poseidon Hash** for commitments because it is ZK-friendly, collision-resistant (128-bit security), and widely supported (e.g., by RiscZero, SP1, Circom).

---

## 7. Core Mechanisms

### 7.1 Hidden Hand Initialization

**Goal:** Ensure each player's initial hand is legal but invisible to the opponent.

**Procedure:**

```
┌─────────────┐                          ┌─────────────┐
│  Player A   │                          │  Player B   │
└──────┬──────┘                          └──────┬──────┘
       │                                        │
       │  1. Select/Draw 5 cards locally        │
       │     (3 units + 2 traps)                │
       │     hand_A = [c1, c2, ..., c5]         │
       │                                        │
       │  2. Encrypt hand                       │
       │     ct_A = AES-GCM(HEK_A, hand_A)      │
       │                                        │
       │  3. Calculate Commitment               │
       │     com_A = Poseidon(hand_A || salt_A) │
       │                                        │
       │  4. Generate VE Proof π_A              │
       │     (Validating legality, uniqueness)  │
       │                                        │
       │  5. Publish to Celestia                │
       │     Blob(INIT_HAND, {ct_A, com_A, π_A})│
       │────────────────────────────────────────→│
       │                                        │
       │      6. B pulls Blob from Celestia     │
       │      7. B verifies π_A                 │
       │      8. B cannot decrypt ct_A, but     │
       │         confirms A's hand is legal     │
       │                                        │
       │←───────────────────────────────────────│
       │    B replicates Steps 1-5              │
       │    A verifies B's proof                │
       │                                        │
       │  ═══════════════════════════════════   │
       │        GAME OFFICIALLY STARTS          │
       │  ═══════════════════════════════════   │
```

**ZK Circuit Pseudo-code:**

```rust
fn verify_hand(
    commitment: Field,
    rules: GameRules,
    hand: [CardId; 5],
    salt: Field,
) -> bool {
    assert!(poseidon_hash(hand, salt) == commitment);
    
    for card in hand.iter() {
        assert!(rules.legal_card_pool.contains(card));
    }
    
    for i in 0..5 {
        for j in (i+1)..5 {
            assert!(hand[i] != hand[j]);
        }
    }
    
    let total_cost: u32 = hand.iter()
        .map(|c| rules.card_cost(c))
        .sum();
    assert!(total_cost <= rules.max_budget);
    
    true
}
```

### 7.2 Fog-of-War Movement

When Player A moves, they encrypt the new position and calculate a new position commitment. They then output a VE proof (`π_move`) verifying that the distance moved is legally within the bounds, doesn't cross obstacles, and matches the commitments. Player B confirms the proof is valid but continues to see A disappear into the fog.

### 7.3 Encounter & Selective Disclosure (Area Probe Protocol)

To trigger fog breaches, units use an "Area Probe":
- **Probe Declaration:** Player A selects an area and proves one of A's units can "see" this area, publishing it to Celestia.
- **Presence Check:** Player B inspects if any of their units are inside the probed box.
- **Forced Reveal/Absence Proof:** B must either publish a `REVEAL_RESPONSE` (if found) or an `ABSENCE_PROOF` (proving their units are mathematically outside the box). Failing both implies B is cheating or disconnected, allowing A to apply for a timeout resolution via smart contracts.

### 7.4 Combat Resolution

When both parties are actively clashing:
1. Both submit combat commitments: `Poseidon(attack, defense, skill, salt)`.
2. Both reveal combat attributes in the subsequent phase.
3. Damage calculations execute deterministically based on the attributes.
4. Encrypted variables (HP) are updated on Celestia. HP reaching 0 is public.

### 7.5 Fair Randomness

Achieved using a standard Commit-Reveal process. Both users commit to salts, reveal them, and calculate `R = Hash(r_A || r_B)`. A Verifiable Delay Function (VDF) can be added to close the slight time-gap advantage during the reveal phase.

---

## 8. User Flow

### 8.1 Global Flow

```
1. MATCHING -> 2. SETUP -> 3. TURNS -> 4. SETTLE
```

### 8.2 Matching Stage

Players create/join rooms via a settlement contract (with optional stakes) and exchange VE public keys.

### 8.3 Setup Stage

- Maps are generated deterministically based on the Hash of the `game_id`.
- Players deploy their 5 units (3 heroes + 2 traps) secretly in their zone.
- `encrypted_formation_A` is generated and published along with its VE proof. Both verify anchors up to readiness.

### 8.4 Turns Stage

Every turn loop proceeds as follows:

```
╔═══════════════════════════════════════════════════╗
║                  TURN N (A's Turn)                ║
╠═══════════════════════════════════════════════════╣
║  1. MOVE PHASE                                    ║
║  2. PROBE PHASE (A probes, B responds with        ║
║     Reveal or Absence Proof)                      ║
║  3. ACTION PHASE (Attack, Skill, Wait)            ║
║  4. END PHASE (State update, Switch to B)         ║
╚═══════════════════════════════════════════════════╝
```

### 8.5 Settlement Stage

- **Normal Settlement:** On victory conditions (e.g., enemy general dead, HQ broken), the winner submits a `VICTORY_CLAIM`.
- **Abnormal Settlement (Timeout/Disconnect):** Wait for specific block threshold on Celestia. Output `TIMEOUT_CLAIM`. If grace period concludes without data, submit blob history to the contract to fetch the opponent's deposit forcefully.

---

## 9. Protocol State Machine

The Global State Machine cycles safely from `IDLE -> WAITING -> SETUP -> IN_PROGRESS -> (DISPUTE handling if needed) -> SETTLING -> COMPLETED`.
Turn phases rotate precisely among `TURN_START, MOVE_PHASE, PROBE_PHASE, REVEAL_PHASE, ACTION_PHASE, COMBAT_RESOLVE, TURN_END`.

---

## 10. Data Structures

All data stored as Blobs on Celestia adheres to a unified structured payload containing headers like `version, blob_type, game_id, sender, turn_num, timestamp, payload, signature`. 

Key payloads include:
- `InitHandPayload`
- `MoveActionPayload`
- `ProbeDeclarePayload`
- `RevealResponsePayload`
- `StateCheckpointPayload`

Full game states maintain private states separated mathematically into `my_units` and selectively revealed `opponent_units`.

---

## 11. Security Analysis

### 11.1 Threat Model

| Threat | Description | Impact |
|---|---|---|
| **T1: Probe Positions** | Attacker tries to decrypt opponent's position ciphertexts | High |
| **T2: Probe Hands** | Attacker tries to identify opponent's hand | High |
| **T3: Illegal Move** | Player submits cross-map jumping | High |
| **T4: Illegal Deck** | Payer equips un-allowed cards | High |
| **T5: Reveal Refusal** | Spied player refuses to output Reveal data | Medium |
| **T6: Rage Quit** | Deliberately going offline upon looming defeat | Medium |
| **T7: Replay Attack** | Replaying old blobs for new turns | Medium |
| **T8: Tuning Attacks** | Utilizing blob confirmation gaps | Low |
| **T9: MEV**| Celestia node censorship / frontrunning | Low |

### 11.2 Safety Measures

- **Against T1/T2:** Handled with semantic security of AES-256-GCM. Ciphertext padding mitigates packet length identification.
- **Against T3/T4:** Evaluated reliably against ZK-SNARK Soundness verification.
- **Against T5/T6 (Timeouts & Quits):** Forced decryption triggers and absolute Data-Availability guarantees on Celestia. Game histories reconstruct fully from DA layer.

---

## 12. Why Celestia Private Blockspace?

### 12.1 Alternative Solutions Matrix

| Dimension | Centralized Server | Pure ZKP On-chain | Commit-Reveal | Celestia PB + VE |
|---|---|---|---|---|
| Decentralized | ✗ | ✓ | ✓ | ✓ |
| Privacy Depth | High* | High | Low | High |
| Verification Cost| N/A | Extreme | Low | Moderate |
| Wait Latency | Low | Extreme | Moderate | Low-Moderate |

Celestia's layout guarantees optimal decentralized latency, high performance data throughput (aimed precisely at Tb/s bandwidths avoiding chain bottlenecks), and total safe exit functionality without trust in off-chain servers or roll-up intermediaries.

---

## 13. Performance & Cost

### 13.1 ZK Generation Costs

Typical generation ranges up to ~2s per circuit on pure CPU (RiscZero/SP1). The goal is optimization through incremental proving and background generation relying on WebGPU acceleration to drop it to `~0.3s`.

### 13.2 Celestia Cost Estimation

Under expected load limits, calculating standard 30-turn encounters sizes up neatly:
**Total Blob costs are estimated `< $0.11` per Game.** Combined with settlement arbitrations via Arbitrum orbit configurations, total network gas costs hover safely below `$0.15` globally per full run. 

---

## 14. Roadmap

- **Phase 0:** Research & PoC (4 - 6 Weeks)
- **Phase 1:** Core Protocol Development (8 - 10 Weeks)
- **Phase 2:** Game Client Development (8 - 12 Weeks)
- **Phase 3:** Testnet Launch (4 - 6 Weeks)
- **Phase 4:** Mainnet & Beyond (Ongoing)

---

## 15. References

1. Celestia Documentation — Data Availability Layer
2. Celestia Private Blockspace — Verifiable Encryption on Celestia
3. Hibachi — First Perpetual Trading Platform using Private Blockspace
4. Blobstream — Streaming Celestia data to Ethereum
5. RiscZero \& SP1 — High-performance zkVM frameworks
6. Dark Forest — First zkSNARK Space Warfare implementation

---

## Appendix A: Card Design Examples

| Card Name | Class | Attack | Defense | HP | Move | Vision | Cost | Secret Skill |
|---|---|---|---|---|---|---|---|---|
| Shadow Assassin | Assassin | 8 | 2 | 4 | 4 | 3 | 5 | Assassinate: First attack 2x dmg |
| Iron Guard | Tank | 3 | 8 | 10 | 1 | 2 | 4 | Taunt: Forces adjacent hits on self |
| Holy Priest | Support | 2 | 4 | 6 | 2 | 2 | 4 | Heal: Restore adjacent unit 3 HP |
| Toxic Trap | Trap | 0 | 0 | 1 | 0 | 0 | 2 | Trigger: Losing 4 HP when stepped |
| Scout Ward | Trap | 0 | 0 | 1 | 0 | 3 | 2 | Vision: Permanent 3-tile grid vision |

## Appendix B: Map Flow Example

```
     0   1   2   3   4   5   6   7
   ┌───┬───┬───┬───┬───┬───┬───┬───┐
 0 │ A │ A │ . │ . │ . │ . │ B │ B │  A = Player A Zone
   ├───┼───┼───┼───┼───┼───┼───┼───┤
 1 │ A │ A │ . │ ▓ │ ▓ │ . │ B │ B │  B = Player B Zone
   ├───┼───┼───┼───┼───┼───┼───┼───┤
 2 │ . │ . │ . │ . │ . │ . │ . │ . │  ▓ = Obstacle
   ├───┼───┼───┼───┼───┼───┼───┼───┤
 3 │ . │ ▓ │ . │ ♦ │ . │ . │ ▓ │ . │  ♦ = Resource
   ...
```

## Appendix C: Message Sequence Chart

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
   ...
```

---

**End of Document**

*FogCard Protocol — First-generation real Fog of War for on-chain games.*

*ccboomer. All rights reserved.*
*This document is released under CC BY-SA 4.0.*
