# Betting on Bitcoin Upgrades: A Smart Contract Wager on OP_CAT Activation

sCrypt-ts | 2025-04-22 09:42:38 UTC | #1

# Betting on Bitcoin Upgrades: A Smart Contract Wager on OP_CAT Activation

## ***No oracles. No middlemen. Pure on-chain logic that works today.***


OP_CAT, proposed in BIP 0347, aims to enhance Bitcoin’s scripting capabilities by allowing concatenation of two stack elements, enabling complex covenants and smart contracts. In the ever-evolving world of Bitcoin, soft forks like OP_CAT spark heated debates about functionality, security, and adoption. But talk is cheap — what if you could put your money where your mouth is? Enter Alice and Bob, two Bitcoin enthusiasts betting on whether OP_CAT will activate by a specific time or block height. Using a clever Bitcoin smart contract, they create a trustless, oracle-free wager that ensures one walks away with the prize, with real skin in the game. We can implement a trustless bet on `OP_CAT`’s activation **today**, using only **existing Bitcoin opcodes**. It’s Bitcoin-native, self-enforcing, and doesn’t rely on third-party arbitration like [Polymarket](https://polymarket.com/event/will-bitcoin-activate-op-ctv-or-op-cat-in-2025?tid=1745283822768). No soft fork is needed.

![|700x700](upload://nXKci52FlavfyBy4cbqqRWtLIYC.png)

# The Bet: OP_CAT Activation by Block Height N

Alice believes OP_CAT, which redefines OP_SUCCESS126 to enable string concatenation (BIP-347), will activate by block height 900,000 (roughly November 2025, assuming 2-week difficulty adjustments). Bob, skeptical of miner consensus, bets it won’t. They agree to lock 1 BTC each into a smart contract with clear rules:

* If OP_CAT is active by block height 900,000, Alice wins and claims 2 BTC.
* If OP_CAT is not active by block height 900,000, Bob wins and claims 2 BTC.
* No oracles or third parties are allowed — the contract must resolve using Bitcoin’s blockchain alone.

This isn’t just a friendly wager; it’s a financial commitment that forces both to back their convictions with real capital, embodying the “skin in the game” ethos.

# Why It’s Tricky: The OP_SUCCESS126 Challenge

Designing this contract is no small feat. OP_CAT’s activation hinges on redefining OP_SUCCESS126, a placeholder opcode that, pre-activation, causes any script executing it to **succeed** immediately. This means a naive script expecting OP_CAT to fail pre-activation (e.g., to block Alice’s spending) would instead allow Alice to claim funds prematurely, breaking the bet’s logic. To overcome this, Alice and Bob need a contract that distinguishes OP_CAT’s post-activation behavior (concatenation) from OP_SUCCESS126’s pre-activation behavior (immediate success) without external data.

# The Smart Contract Design

The contract uses a Taproot output with a single script path/tapleaf with two conditional branches¹. The script is compiled from the following [sCrypt](https://docs.scrypt.io/) smart contract.

```ts
01 export class OPCAT_Wager extends SmartContract {
02     @prop()
03     pubA: PubKey;
04
05     @prop()
06     pubB: PubKey;
07
08     @prop()
09     deadline: int32;
10
11     @prop()
12     delta: int32;
13
14     @method()
15     public unlock(a: ByteString, b: ByteString, sigA: Sig, sigB: Sig) {
16         if (this.checkSig(sigB, this.pubB)) {
17             this.absTimelock(this.deadline);
18             assert(sha256(a + b) == toByteString('00..00'));
19         } else {
20             this.absTimelock(this.deadline + this.delta);
21             this.checkSig(sigA, this.pubA);
22         }
23     }
24 }

```

## Why This Works

* `sha256(a + b) == toByteString('00..00')` is impossible: no inputs from concatenating two strings hash to all zeros.` +`is the concatenation operator and compiles to OP_CAT. This assertion at Line 18 will always fail if OP_CAT is re-enabled. It can only succeed if OP_CAT is not re-enabled, when Bob wins.
* Absolute timelock`absTimelock()` (compiled to OP_CHECKLOCKTIMEVERIFY/CLTV, instead of relative timelock OP_CSV) ensures neither can spend before the agreed activation deadline.
* We add more time delay `delta` for Alice at Line 20, because otherwise **she can also spend** even if OP_CAT is not re-enabled at the block height and may frontrun Bob. The time delay can be set conservatively long to ensure Bob’s spending transaction can be mined before Alice’s attempt, say, 48 hours. Note that once OP_CAT is reactivated, it will remained reactivated after `delta`.

![|700x560](upload://p788hM0GGaJW6A9F7dFbP7FCRgL.png)

Execution Flow: How the Bet Resolves

## Miner Extractable Value (MEV) Risks

In the context of this bet, MEV could arise if miners influence OP_CAT’s activation to manipulate the outcome. For instance, miners could accelerate signaling to favor Alice. However, as long as the total bet amount remains relatively small, the financial incentive for miners to engage in such manipulation for a softfork is limited, and we do not consider MEV a significant concern here and deem it out of scope for this analysis.

# 🏁 Final Thoughts

This isn’t theoretical — you can deploy this right now. It leverages Bitcoin’s existing rules to create a self-executing, trustless bet on `OP_CAT`’s future activation. It also sets a precedent for other prediction markets (e.g., betting on other opcodes, protocol changes).

Want to try it? Find a counterparty, agree on activation block height and lock the funds. The blockchain enforces the rest.

No soft forks. No oracles. **Just Bitcoin.** 🚀

[1] Alternatively, we can uses a Taproot address with two script paths: one for Alice’s win, the other for Bob’s. We choose one script path for ease of exposition here.

-------------------------

JeremyRubin | 2025-04-22 15:49:36 UTC | #2

doesn't seem to work because of how OP_SUCCESSx works -- it makes immediately trivially satisfiable any script containing an op_sucess, with no logic executed around it.

-------------------------

garlonicon | 2025-04-22 18:18:47 UTC | #3

[quote]doesn’t seem to work because of how OP_SUCCESSx works[/quote]
True. If you want to enforce any time-based restrictions, then all you can do, is just using the nLockTime field of the whole transaction.

-------------------------

sCrypt | 2025-04-22 18:50:56 UTC | #4

Note OP_SUCCESSx (such as OP_SUCCESS126, i.e., OP_CAT after activation) only makes a script immediately succeeds if **EXECUTED**. 

For example, the script will **not automatically succeed** just because `OP_SUCCESS3` is in the code — it only triggers success if actually run. If this script is evaluated with `OP_FALSE` on the stack, the `OP_IF` branch with `OP_SUCCESS3` is **not executed**, so the `ELSE` branch runs instead.
```
OP_FALSE
OP_IF
  OP_SUCCESS3
OP_ELSE
  OP_CHECKSIG
OP_ENDIF
```

Similarly, in our case, branching condition at Line 16 ensures only Bob can get OP_SUCCESS126 executed (i.e., when OP_CAT is still disabled), thus succeed. Alice cannot trigger OP_SUCCESS126, which is what we want.

-------------------------

sipa | 2025-04-22 19:39:02 UTC | #5

[quote="sCrypt, post:4, topic:1632"]
Note OP_SUCCESSx (such as OP_SUCCESS126, i.e., OP_CAT after activation) only makes a script immediately succeeds if **EXECUTED**.
[/quote]

This is incorrect. The mere *presence* of an `OP_SUCCESSx` opcode in a BIP 342 tapscript makes it automatically evaluate to true. Quoting [BIP 342](https://github.com/bitcoin/bips/blob/757e15e56883b754963dff732dde01a425f9cec6/bip-0342.mediawiki), emphasis mine:

> The script as defined in BIP341 (i.e., the penultimate witness stack element after removing the optional annex) is called the tapscript and **is decoded into opcodes**, one by one:
> 1. ...
> 2. If any opcode numbered *80, 98, 126-129, 131-134, 137-138, 141-142, 149-153, 187-254* **is encountered, validation succeeds (none of the rules below apply)**. This is true even if later bytes in the tapscript would fail to decode otherwise. These opcodes are renamed to `OP_SUCCESS80`, ..., `OP_SUCCESS254`, and collectively known as `OP_SUCCESSx`.
> 3. ...
> 4. **The tapscript is executed** according to the rules in the following section, with the initial stack as input.

I think the scheme works, but only if you split the branches into separate script leaves.

-------------------------

Chris_Stewart_5 | 2025-04-22 19:38:02 UTC | #6

> The inclusion of `OP_SUCCESSx` in a script will pass it unconditionally. It precedes any script execution rules to avoid the difficulties in specifying various edge cases, for example: `OP_SUCCESSx` in a script with an input stack larger than 1000 elements, `OP_SUCCESSx` after too many signature opcodes, or even scripts with conditionals lacking `OP_ENDIF`.

https://github.com/bitcoin/bips/blob/757e15e56883b754963dff732dde01a425f9cec6/bip-0342.mediawiki#cite_note-1

Link to code: https://github.com/bitcoin/bitcoin/blob/e5a00b24972461f7a181bc184dd461cedcce6161/src/script/interpreter.cpp#L1795

-------------------------

sCrypt | 2025-04-22 21:35:03 UTC | #7

Correct, *OP_SUCCESSx* will make a script succeed, even if **NOT** executed. "split the branches into separate script leaves" would not help, since timelock check at Line 17 will be bypassed, even if it precedes OP_SUCCESS126. Bob can take the bet fund **before** the deadline. Additional measures, like [Taplock](https://github.com/taproot-wizards/taplocks/blob/16a07c446977337da2dd278dae2ed9f32151eade/README.md), must be taken to hide the script, so Bob is unable to spend it before the deadline.

-------------------------

