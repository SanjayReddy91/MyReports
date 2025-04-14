#   CoinFlip: Technicals

##   1   Introduction

###   1.1   Background

Simple yet Interesting with its implementation is a game CoinFlip. The basic working is the same as a coin toss where two players decide which side of the coin they want and throw the coin to see which way it lands to determine the winner. CoinFlip added some technicality to make this game fair. The fairness lies in the way the game is implemented.

###   1.2   Motivation

CoinFlip is created for a luck based game to be fair. Such that a honest player has an chance of winning. The technical implementation is based on bitcoin capabilities such as **Taproot** and **MultiSignature**.

###   1.3   Overview

This report dives into the technical aspects of coinFlip game and various bitcoin capabilities it relies on and what makes it fair for players participating.

##   2.   Protocol Description

###   2.1   Game Rules

The game emulates a simple coin toss where the actual choice of each player is stored as a secret and the choice is the length of the secret. Heads when length is 15 and tails when length in 16. Both players choose their secrets at the start and as the play goes on the winner is determined based on length of secrets matching. If the length of secrets match then player B wins else A wins. Simple as that!

###   2.2   Technical Details

There are three transactions involved in this game: Setup, Final and Cashout transactions.

**Setup transaction:** takes both players commitments in form of unspent VTXOs and commits them also forces player A to reveal their secret.

* **Inputs:**
    * VTXO 1 (Signed by A)
    * VTXO 2 (Signed by B)
* **Output:**
    * (A + B + secret A) OR (B after timeout) - This output enforces the reveal of secret A.
* **Purpose:**
    * This transaction locks the funds provided by both players.
    * It's designed to *force* Player A to reveal their secret. However, as noted later, this design is flawed.

**Final transaction:** is signed BEFORE the setup transaction, forces the second player to reveal his secret. Thus, once the setup transaction is submitted, the funds can only be spent through the final transaction.

* **Input:**
    * Output of the Setup transaction.
* **Output:**
    * If `len(secret A) == len(secret B)`: B + secret B (Forces B to reveal secret B, B wins)
    * Else if `len(secret A) != len(secret B)`: A + secret B (Forces B to reveal secret B, A wins)
    * Else: A after timeout
* **Signing:**
    * Presigned by A & B.
* **Purpose:**
    * This transaction determines the winner based on the length of the secrets.
    * Crucially, it *forces* Player B to reveal their secret, regardless of who wins.
    * The pre-signing is essential for the trustless nature of the protocol.

**Cashout transaction:** is made by the winner of the game to get his funds.

* **Input:**
    * Output of the Final transaction.
* **Output:**
    * Winner's address (Both player's funds)
* **Signing:**
    * Signed by the winner.
* **Purpose:**
    * This transaction allows the winner to claim the locked funds.

##   3.   Analysis and Issues

###   3.1   Vulnerability: Premature Secret Reveal

The most significant flaw in the current protocol is in the **Setup transaction**. It forces Player A to reveal their secret *before* Player B has committed to theirs. This allows Player B to cheat.

* **Scenario:** Player B observes `secret A` (or its length) in the Setup transaction. Player B can then choose `secret B` to be either 15 or 16 bytes long to guarantee a win.

###   3.2   Lack of Hashing/Commitment

The protocol lacks a proper commitment scheme (like hashing) to hide the secrets initially. This is the root cause of the vulnerability described above.

###   3.3   Trust Assumptions

While the protocol aims to be trustless, it still relies on the following assumptions:

* **Bitcoin's Security:** It assumes the security and immutability of the Bitcoin blockchain.
* **Correct Implementation:** It assumes that the players and the software implementing the protocol follow the rules correctly.

##   4.   Proposed Solutions

To fix the vulnerability, we need to introduce a commitment scheme:

###   4.1   Using Hashing

1.  **Modified Setup Transaction:**
    * Inputs: VTXO 1 (Signed by A), VTXO 2 (Signed by B)
    * Output: A + B + H(secret A)
    * Here, H() is a cryptographic hash function (e.g., SHA256). Player A commits to the *hash* of their secret, not the secret itself.

2.  **Modified Final Transaction:**
    * Input: Output of the Setup transaction.
    * Output:
        * If `len(secret A) == len(secret B)` AND `H(secret A) == hash_from_setup`: B + secret B
        * Else if `len(secret A) != len(secret B)` AND `H(secret A) == hash_from_setup`: A + secret B
        * Else: A after timeout
    * The `Final transaction` now *verifies* that the revealed `secret A` matches the hash committed in the `Setup transaction`. This prevents Player A from changing their secret after seeing B's commitment.

###   4.2   Alternative Commitment Schemes

More advanced commitment schemes could be used, but hashing is a simple and effective solution.

##   5.   Conclusion

The provided CoinFlip protocol has a critical vulnerability that allows Player B to cheat. This can be fixed by using a cryptographic hash function to commit to secrets before revealing them. This ensures fairness and trustlessness.

##   Questions

1.  Why is pre-signing the `Final transaction` important?
2.  How does the timeout mechanism in the transactions affect the protocol's security and usability?
3.  What are the limitations of using secret lengths (15/16 bytes) to represent coin sides? Could there be collisions or other issues?
4.  Are there any privacy implications in revealing the secrets on the blockchain, even with hashing?
5.  How would you extend this protocol to support more than two players?

Reference: https://coinflip.casino/how-it-works