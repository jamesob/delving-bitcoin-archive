# Economic-Majority Signaling for OP_CTV Activation

ZmnSCPxj | 2024-03-03 11:36:44 UTC | #1

Introduction
=====

`OP_CTV` and other covenant opcodes have somehow managed to trigger a countervailing "ossify-now" movement, which seeks to keep the Bitcoin consensus unchanged indefinitely, citing that changes introduce risk of exploitation.

Ultimately, what matters is how much actual users care to activate, or block, a particular consensus change. As the writeup [Money: The Unit of Caring (LessWrong)](https://www.lesswrong.com/posts/ZpDnRCeef2CLEFeKM/money-the-unit-of-caring) suggests, however, how much you care about something can be signalled by "how much are you willing to put on the line to ensure it happens / ensure it does not happen?"  Such signalling is, literally, expensive, and thus difficult to fake.

Indeed, the concept of an "economic majority" that controls miners via the invisible hand of the market is not new. The "user-activated soft fork" insisted that users could coerce miners to follow their desired chain, by rejecting blocks of miners that do not follow the desired consensus rules.

Economics of Consensus Change
-------------------------

In order to cause consensus change, miners must be convinced to run the modified rules.  Can ordinary users coerce miners?

An existing HODLer is in possession of some number of UTXOs whose keys are under their control. Suppose a HODLer strongly desires to have some rule set to be used (whether the existing rules, or a new set of rules).  Now suppose further that there is a chainsplit, where one chain uses rules wanted by the HODLer, and another uses rules not wanted by the HODLer.  In that case:

* On both sides of the chainsplit, the HODLer has possession of coins.
* A miner can only point its purchased hashrate at one chainsplit or the other, and can only earn coins on one side of the chainsplit.

The HODLer can then sell coins on the chainsplit of the undesired ruleset. If so, the value of coins in that chainsplit lowers, unless other users arise which prefer that ruleset (who then buy out the value of the sold coins).  If there are more sellers of one side of the chainsplit than the other, then the price of the coins on that chainsplit drops.

Miners are incentivized to maximize their earnings in terms of real-world purchasing power.  This is because hashrate is costly: real-world energy must be utilized in order to actually generate hashes and therefore blocks, which must be purchased, and furthermore, actual real-world utilization of energy increases universal entropy, including the entropy of actual hashing hardware (i.e. hardware breaks down, especially as it is more utilized, and must eventually be replaced).  This forces miners to be both earnings-maximizing and short-sighted relative to HODLers of coins, who can generally have a time preference proportional to human lifespans rather than proportional to hashing hardware lifespans.

Thus, miners will be coerced into mining the chainsplit of the desired ruleset, by simple economic theory.

Betting On `OP_CTV` Activation / Blockage
=======================

Any signal for "caring to activate / block" can be formed as a bet.  For instance, I can say "I will sell coins on a chainsplit that does not activate `OP_FOO`" by betting that it will activate.  If it activates, then I get back my funds and HODL it. If it does not activate, I have effectively sold the equivalent amount and have divested myself of the coin.

Now, for softforks that replace `OP_NOP` opcodes, such as the BIP-119 rendition of `OP_CHECKTEMPLATEVERIFY`, we can actually use a simple technique of violating the rules of the opcode to determine if the chainsplit does not activate the opcode, and serves as "the opcode was not activated".  Then we can have a second time-delayed branch, which can be used if the first branch is not entered, which serves as the branch for "the opcode was activated".

As a concrete example for `OP_CTV`, suppose we have the actors below:

* Alice: I bet that `OP_CTV` DOES NOT activate ("ossify-now").
* Bob: I bet that `OP_CTV` DOES activate ("`OP_CTV` now").

Suppose that we have a defined activation blockheight for `OP_CTV`, called `N`.  Then we can use the following P2WSH script:

    <hash(pay to Bob)> OP_CTV OP_DROP
    <Alice> OP_CHECKSIGVERIFY
    <Bob> OP_CHECKSIG

The `hash(pay to Bob)` is the `OP_CTV` hash where the output goes to an address Bob controls, at time `N+1month`.

Then Alice and Bob exchange signatures:

* Alice gives a signature to Bob:
  * Matches `hash(pay to Bob)`, spending the above script and giving the output to Bob.
  * `nLockTime` is `N+1month`.
* Bob gives a signature to Alice:
  * Spends the above script and gives the output to Alice
  * `nLockTime` is `N`.

Then, depending on the chainsplit:

* Chainsplit where `OP_CTV` activated:
  * At time `N` Alice cannot use the Bob-side signature, because `OP_CTV` is violated --- the signature is for a transaction that pays to Alice, but the `OP_CTV` requires the output to pay to Bob.
  * At time `N+1month`, Bob can use the Alice-side signature to claim the funds, because it wins its bet in this chainsplit where `OP_CTV` activated.
* Chainsplit where `OP_CTV` was NOT activated:
  * At time `N` Alice can use the Bob-side signature, because `OP_CTV` is just `OP_NOP4` here and the `<hash(pay to Bob)>` has no meaning, and it won the bet in this chainsplit where `OP_CTV` was not activated.
  * At time `N+1month`, hopefully Alice has already claimed the funds before then, and Bob is unable to claim the funds now.

After exchanging signatures, Alice and Bob can now cooperatively sign the funding transaction whose output is the above script, and whose inputs are the ante of Alice and Bob.

A market maker can then provide a venue for Alices and Bobs to meet and make such bets.  If there is more economic power in Alice than in Bob, then Alices will become willing to take smaller antes from Bobs (and vice versa).  The market maker can then monitor which side has more money willing to bet on activation/blocking of `OP_CTV` and report it.

Miners can then use the market venue to check which chainsplit is more economically-rational to mine in the future.  We thus hope that this economic control is sufficient to force one chainsplit or the other to never be mined in the first place.

Generalization
===============

I believe that this technique generalizes to any opcode that replaces an `OP_NOP` in P2WSH.

Unfortunately, the same technique cannot be used for any opcode that replaces an `OP_SUCCESS` in Tapscript.  This is because the Alice-wins remains anyone-can-spend in the chainsplit where the opcode does not activate.  Thus, it cannot be used for `OP_INTERNALKEY` or `OP_CHECKSIGNATUREFROMSTACK`.  However, in practice if both are packaged with the same softfork that activates the BIP-119 `OP_CHECKTEMPLATEVERIFY` that replaces `OP_NOP4`, then the entire package can be bet on via this mechanism.

-------------------------

moonsettler | 2024-03-03 11:52:58 UTC | #2

if CTV does not activate your node will reject upgradeable NOP scripts from it's own mempool and so will every other node. i think the time locked spend should be separated via tapscript leaves?

-------------------------

ZmnSCPxj | 2024-03-03 11:55:20 UTC | #3

Unfortunately the chainsplit where `OP_CTV` does not activate requires that the script interpreter actually go through the `OP_NOP4`. The inverse is not possible --- you can only detect whether `OP_NOP4` is still an no-op or not, thus it requires an earlier timelock.

-------------------------

moonsettler | 2024-03-03 12:01:01 UTC | #4

right, NOP4 upgrading to CTV has to block the against bet.

-------------------------

martinschwarz | 2024-03-09 16:27:28 UTC | #5

Bets are not equivalent to selling BTC. The price is not necessarily impacted by either outcome of the bet, thus the miner's reward doesn't necessarily change in fiat value either way. 

During the fork wars, it was said that the introduction of BTC futures at Bitfinex at that time decisively predicted the economic future of each fork and decisively influenced the miner's choice. 

I think directly bribing the miners with batches of juicy fees in signed transactions only valid on either side of the fork would be a more direct on-chain signal.

-------------------------

ZmnSCPxj | 2024-03-10 05:26:28 UTC | #6

The same scheme can be adapted such that the bet offered by your counterparty is paid as fee to the miner while you retain your own funds.

You forgo the value of what your counterparty sells if you win, and instead just retain your own funds in the chainsplit you desire, but also directly encourage miners to mine transactions for your desired chainsplit.

-------------------------

