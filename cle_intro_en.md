# What is FogCard? — A Project Introduction for Newcomers

Simply put, **FogCard is a "decentralized strategic card game" running on the blockchain**. Its biggest selling point, and the major technical challenge it solves, is realizing a true **"Fog of War"** for on-chain games.

If you haven't encountered similar concepts before, you can quickly understand its value and gameplay through the following three questions.

---

### Question 1: Why is "Fog of War" hard for blockchain games?

In traditional online games (like *StarCraft*, *League of Legends*, or regular card games), the "Fog of War" is a fundamental setting—you cannot see the enemy's cards or the uncharted territories where opposing units are located. This is typically maintained by the game company's **centralized server** (e.g., Tencent, Blizzard), which knows everyone's information but only sends you the parts you are allowed to see according to the rules.

However, **a blockchain has no centralized server; all data is open and transparent**. If unit positions or hand data are written on the chain, opponents can simply check the on-chain records to see all your hidden cards, making tactical deception or ambushes impossible. This is the fatal flaw limiting the depth of traditional fully on-chain games (FOCG).

### Question 2: How does FogCard break the stalemate?

FogCard utilizes two cutting-edge technologies to magically resolve the dilemma of "not handing data over to a company while preventing opponents from peeking":

1. **Celestia's Private Blockspace**: 
   Every action a player takes (playing a card, moving) is **first encrypted on their own computer** into a string of gibberish, and then broadcasted to Celestia (a highly affordable blockchain network with massive data availability throughput). In this way, the data is permanently stored, but no one except you can decrypt it.
2. **Verifiable Encryption and Zero-Knowledge Proofs (ZK Proofs)**:
   Since it's encrypted, how does the opponent know you didn't cheat (for example, moving 100 steps in one turn)? When broadcasting the encrypted gibberish, you must attach a mathematical proof, commonly known as a Zero-Knowledge Proof. This proof guarantees to the entire network: "The gibberish I broadcasted fully complies with the game rules, and it is mathematically impossible for me to have cheated."

**Through these two technologies, the game achieves: playing blind chess where both sides are strictly locked by the rules against cheating, all without needing a referee (centralized server) present.**

### Question 3: What does the gameplay feel like?

Despite the hardcore underlying technologies based entirely on mathematics and cryptography, the experience for players is like a standard advanced tactical card game:

* **Deck Building**: Choose 5 cards from a pool of different classes (Assassin, Knight, Mage, etc.) to form a lineup (typically comprising 3 hero units and 2 hidden trap/spell cards).
* **Secret Deployment**: Like playing blind chess, you deploy your units and traps on the board, and the opponent has no idea where you placed them.
* **Fog Exploration**: You command your soldiers to probe forward. When your unit gets close enough to an opponent's ambushing unit, the underlying cryptographic logic is triggered, forcing the opponent to reveal their coordinates and attributes to you. This moment is called **"Encounter Reveals Fog"**.
* **Face-to-Face Duel**: Once units from both sides encounter each other, they reveal their HP and attack power, battling to the death based on numerical rules.
* **Anti-Disconnect Mechanism**: What if you are about to win and the opponent suddenly unplugs their internet to stall? Don't worry, the game features a **forced settlement mechanism**. Because action records are permanently on the public chain, if you wait the required amount of time and the opponent hasn't made a move, you can unilaterally apply to the smart contract for a victory and take all the rewards.

---

### Summary

For **players**, FogCard is a "dark tactical" game that requires zero trust in any game company, relying entirely on mathematics and cryptography  to ensure fairness.

For the **industry**, it is a highly representative benchmark case of applying cutting-edge cryptographic technologies (Zero-Knowledge Proofs + Data Availability Layer) to a decentralized game system. It proves that even without a centralized server, we can create large-scale games with profound strategic depth (information asymmetry).
