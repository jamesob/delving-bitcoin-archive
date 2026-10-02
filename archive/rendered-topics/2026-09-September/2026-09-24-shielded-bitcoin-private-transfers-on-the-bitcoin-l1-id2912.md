# Shielded Bitcoin: Private Transfers on the Bitcoin L1

nemothenoone | 2026-09-24 17:45:34 UTC | #1

## ***No soft forks. No operators. No bridges.***

*A plain-language companion to our paper "[Shielded Bitcoin: Private Transfers on the Bitcoin L1](https://x.com/allocinitxyz/status/2103124026214568209)" (Clara Shikhelman, Mikhail Komarov, Aleksei Moskvin).*

Bitcoin provides strong ownership guarantees without trusted intermediaries, but its transaction history is public by design: amounts, timing, and transaction links are visible on-chain, and known wallets can often be associated with real people or institutions. This is a problem. It’s a serious limitation for individuals and institutions alike, including treasuries, trading desks, and businesses using bitcoin for transfer.

Our paper asks a narrow question: can bitcoin move without publicly revealing transfer amounts and counterparties using Bitcoin as it is today, with no soft fork, no new blockchain, and without relying on trusted bridge operators?

Shielded Bitcoin is our proposed metaprotocol for doing so. It is our effort to introduce the first Bitcoin privacy architecture designed to eliminate trusted operators, interactivity, liveness dependencies, and liquidity/exit-collateral requirements.

Moving bitcoin into and out of the shielded metaprotocol using security vaults on the L1 via PIPEs will be detailed in a companion paper coming for release soon.

## The idea in one paragraph

Value inside the metaprotocol lives in **notes**: small encrypted records, each holding an amount and a way to reach its owner. Notes are never visible on Bitcoin in readable form. When someone initiates a transaction, they publish an encrypted transfer to Bitcoin, along with a zero-knowledge proof that the transfer is valid. Bitcoin just stores and orders these bytes. It doesn't understand them, and it doesn't need to. Separate software reads the transfers in Bitcoin's order, checks the proofs, and keeps the shared list of notes. Think of Bitcoin as a public bulletin board: Bitcoin publishes and orders the data, while anyone can independently apply the Shielded Bitcoin rules to determine the resulting state.

## How a shielded transfer works
![image|690x452](upload://dmnbpCxcvEmAfzvNGVcukG8n2FT.jpeg)

Say Alice wants to pay Bob.

**1. Alice seals a note for Bob.** She creates a new note with an amount and Bob's receiving details, then encrypts it so only Bob can open it.

**2. She publishes a transfer.** It goes out as ordinary Bitcoin transaction data and contains three things: the encrypted notes, a unique serial number (called a *nullifier*) for each note she is spending, and a compact proof. The proof says three things at once: the notes she's spending exist, she's authorized to spend them, and the amounts going out equal the amounts coming in. It says this without revealing which notes they are or how much they hold.

**3. Anyone can check it.** Programs called *indexers* watch Bitcoin for these transfers. For each one, they verify the proof and confirm that none of the serial numbers have been used before. If everything passes, they add the new notes to a list and record the serial numbers as used. Because a serial number can only appear once, a note can't be spent twice. No indexer has special authority here: anyone can rerun the same checks from the published history.

**4. Bob finds his money.** His wallet tries to open each new encrypted note with his private viewing key. Most attempts fail, since they belong to other people. When one succeeds and passes a few consistency checks, it's his. When he spends it later, the same cycle repeats.

## Bitcoin Self-Custody.

Only your spending key grants spend authority inside the transfer protocol. Every transfer carries a proof that the spender is authorized to spend the notes being consumed, and indexers reject anything without a valid proof. No indexer, miner, or outside observer can spend your notes, and the read-only keys described below cannot either. We never hold users’ funds. There is no operator holding bitcoin on users’ behalf.

A dishonest indexer can delay, omit, or serve stale data to a wallet that relies on it. That can disrupt the wallet, but it does not give the indexer the ability to spend the wallet’s funds. A user can switch indexers or independently replay the published transfer history by themselves.

## Getting Bitcoin in and out.

Moving bitcoin into the shielded metaprotocol (peg-in) and back out to ordinary bitcoin (peg-out) is covered in the upcoming paper. The solution we explore there uses PIPEs, our work on using witness encryption to condition access to Bitcoin signing keys. We never hold users’ funds, including during entry and exit. The companion paper specifies the mechanism and analyzes its trust, confidentiality, liveness, and failure properties.

## Keys and selective disclosure

A wallet starts from one seed and derives separate keys for separate jobs. One key spends. One is a read-only key that detects incoming transfers. Another is a read-only key that recovers your own outgoing history. You can hand out the read-only keys without ever handing over the ability to spend.

That makes selective disclosure possible. You could show an accountant your incoming transfers, selectively disclose the details of one transfer to a counterparty, or export a report for a given period. There is no global disclosure key, and none of these read-only capabilities grant spending power. There are two caveats. A raw viewing key shows everything it can see, so for narrow audits it's better to disclose a single transfer than a whole key. And auditors can check what you show them, but disclosures don't prove you've shown everything.

## What an outside observer can and can't see

| **Not revealed to public** | **Observable by public** |
|----|----|
| Amounts | That a shielded transfer happened, and when |
| Who sent it | How many notes were spent and created |
| Who received it | The fee, and the Bitcoin transaction carrying the data |
| Which earlier notes were spent | The size and timing of the data posted |

Here "sender" means the shielded sender; a recognizable Bitcoin wallet used to fund the carrier transaction may still identify who published it.

Shielded Bitcoin does not publicly reveal transfer amounts, which shielded notes were spent, or who received the new notes. It does reveal that a shielded transfer occurred, when it occurred, its shape, and the Bitcoin transaction that carried it. The upcoming paper separately analyzes what becomes observable when value enters or leaves the system.

## How this differs from other approaches

Shielded Bitcoin sits alongside a growing body of work aimed at reducing how much transaction information is publicly exposed on Bitcoin, with different approaches making different tradeoffs.

**CoinJoin, PayJoin, and Silent Payments** work within Bitcoin’s existing transaction model. They can make ownership relationships harder to infer, but amounts, transaction structure, and much of the public transaction graph remain observable.

**Zcash** is the closest precedent for the encrypted-note model itself: encrypted notes, nullifiers, and zero-knowledge proofs allow transfers without making their contents public. Shielded Bitcoin borrows from that architecture, but does not introduce its own blockchain or consensus rules. Its state is derived from Bitcoin history instead.

**Glass Coin (prev. Shielded CSV)** is the closest Bitcoin-native system. Like Shielded Bitcoin, it aims to keep transfer information from being publicly exposed without changing Bitcoin consensus. But it uses a client-side-validation architecture: participants hold and pass around their own private transaction data and proofs, and Bitcoin is used mainly for double-spend detection.

Shielded Bitcoin makes a different tradeoff: the data needed to reconstruct the shared state is published to Bitcoin itself. Once the system has been initialized, a wallet can reconstruct its later state from its own secrets and Bitcoin history, rather than depending on private proof data that a user or counterparty might lose.

**PIPEs** address the separate question of how this shielded state connects to actual bitcoin. The construction described in our companion work is designed so that we never hold users’ funds; access to bitcoin is controlled cryptographically rather than through custody by an operator.

None of these approaches are mutually exclusive. They're different explorations of the same question: how much privacy can be built around Bitcoin without changing Bitcoin itself?

## Limitations and Tradeoffs

Three things are worth flagging up front.

**Big deposits aren't automatically a big crowd.** Keeping transfer contents from being publicly revealed is not the same as preventing all inference. If a few actors created most notes, or wallets behave in distinctive ways, observers may still narrow down possible relationships between transfers. Publication fees can create another signal if the same recognizable Bitcoin wallet repeatedly pays them. We are exploring several approaches to reduce that linkage.

**Entry and exit are visible.** This paper specifies how value moves inside the shielded metaprotocol. Peg-in and peg-out have their own specification, coming soon, and this paper does not make claims about their confidentiality, censorship resistance, or other boundary properties. For example, amounts and timing visible during entry or exit may support linkage with later activity. The PIPE-based boundary is designed so that we never hold users’ funds.

**There's a trusted setup.** The proof system in our reference profile, Groth16, needs a one-time setup ceremony, and our security claims hold only if it was run with at least one honest party. Other proof systems trade proof size against setup assumptions, and that's still an open deployment decision.

A few smaller points. Most wallets will rely on an indexer they choose rather than replaying the chain themselves; as described above, a dishonest indexer can disrupt or mislead that wallet but cannot spend its funds. Shielded transfers have a larger on-chain footprint than ordinary Bitcoin transactions, but the difference is within the same order of magnitude. The exact footprint depends on the transfer shape and publication method, so fees scale accordingly.

## What's Next

Shielded Bitcoin specifies and analyzes the transfer system of a larger metaprotocol for Bitcoin transfers with less publicly exposed transaction information. The next paper addresses entry and exit. If you’d like to go deeper, the full Shielded Bitcoin paper contains the protocol specification, formal model, security proofs, and analysis of what remains observable.

Read [Shielded Bitcoin: Private Transfers on Bitcoin L1](https://www.allocinit.xyz/uploads/shielded-bitcoin.pdf).

-------------------------

Anzus_GemWallet | 2026-09-27 11:37:26 UTC | #2

From an ordinary mobile-wallet user’s perspective, what would recovery look like after losing a phone? Would the seed phrase alone recover all shielded funds and transaction history on a new device, or would users need to back up any additional changing data? It would also be helpful to know how long a full recovery might take.

-------------------------

nemothenoone | 2026-09-27 12:02:08 UTC | #3

Seed phrase is enough to recover the full wallet state and retain control over the funds 'cause all the encrypted notes are stored on the Bitcoin L1 (in the witness field, in OP_RETURN or Taproot annexes) so a user can always replicate the latest state of the wallet from Bitcoin itself.

There is a different question about PIPEs v2/WE ciphertexts for the unilateral exit. One'd need them to un-shield Bitcoin in the most paranoid mode (or if the whole world except Bitcoin suddenly disappeared). But those are public, there will be a ton of them to choose from and we expect the emergence of ciphertext storage providers offering those. So for the mobile wallet user there would be no need to store ciphertexts on their local device.

-------------------------

Anzus_GemWallet | 2026-09-28 06:52:31 UTC | #4

Thanks, knowing that the seed phrase is enough is reassuring. Do you have a rough idea how long recovery might take on a phone after several years of activity? Could a service speed this up without learning which transactions belong to the user?

-------------------------

ZmnSCPxj | 2026-09-30 12:09:51 UTC | #5

[quote="Anzus_GemWallet, post:4, topic:2912"]
Do you have a rough idea how long recovery might take on a phone after several years of activity? Could a service speed this up without learning which transactions belong to the user?
[/quote]

Not OP, but the fact that the history is stored on Bitcoin L1 blockchain suggests that it would be similar to recovering onchain wallets from seed, i.e. if it is not SPV, it would have to download all the blocks.

An indexer service that reads the Bitcoin L1 blockchain and only reads the Shielded Bitcoin transfers would reduce the amount of data, assuming of course that not everybody uses Shielded Bitcoin (if ~all use of the Bitcoin blockchain were Shielded Bitcoin, then such a service would not  significantly reduce the amount of data that needs to be downloaded). However, your resource-constrained device would trust that such a service does not censor and that the service follows the history with the rules you believe to be correct for Bitcoin (i.e. SPV security)

A service that can only provided the Shielded Bitcoin transfer subset of all Bitcoin transactions would not learn your transactions (but would learn you are using Shielded Bitcoin), though your resource-constrained device would still need to scan through the Shielded Bitcoin transfer subset to look for your coins (in addition to trusting that the service did honestly report all Shielded Bitcoin transfers).  An alternate kind of service that is given your viewing key would know all your transactions, but would be able to give an even smaller subset of the blockchain that actually contains your coins (i.e. SPV lack-of-privacy, like Electrum).

As I understand the math of it (I AM NOT A MATHEMATICIAN) there does not seem to be a way similar to BIP-157 to create filters for Shielded Bitcoin transfers (and in any case BIP-157 is still SPV security, it just happens to be widely deployed so you can switch service providers trivially or just use multiple service providers).

-------------------------

Anzus_GemWallet | 2026-09-30 14:35:46 UTC | #6

Thanks, that makes the tradeoff clearer. A more private recovery would require scanning more data, while a faster targeted recovery could reveal more information or require additional trust in a service. That seems like something wallets should explain clearly before users need to restore—not only during recovery.

-------------------------

ZmnSCPxj | 2026-10-01 12:37:31 UTC | #7

I skimmed both the Shielded Bitcoin and PIPEv2 paper, and I wonder about how peg-outs would work.

In PIPEv2, once the condition is met, the secret key is now known by the participant that is able to meet the condition (i.e. the "Recipient").  As I understand it, with a UTXO model like in Bitcoin, that implies the entire UTXO amount becomes semantically owned by that participant, as it now learns the secret key to that UTXO and in the "real Bitcoin" can spend that UTXO arbitrarily with no conditions.

This seems to imply to me that for every peg-out from the Shielded Bitcoin "domain", I would have to select some existing peg-in that exactly matches that amount I am pegging out, and then provide the proof that I have published an intent-to-peg-out on the Shielded Bitcoin domain.  Presumably the peg-in condition would also need to encode the amount (and therefore the Shielded Bitcoin domain would need to include a rule to validate that the peg-in condition and the peg-in amount match).  Is that correct?

If so, how is it resolved if different actors want to peg out the same amount and happen to pick the same peg-in UTXO to get?

Also, I would presume that the PIPEv2 condition would need to check that the Shielded Bitcoin intent-to-peg-out is validly confirmed onchain, and even deeply confirmed. I would presume that the condition would need to include some kind of (???) zkVM trace (???) of validating that the intent-to-peg-out is deeply confirmed, presumably requiring checking that a series of block headers meeting the correct difficulty target have buried a block containing the intent-to-peg-out publication in Shielded Bitcoin.  This seems to me effectively identical to a zk-proof version of the "SPV proof" concept brought up by the old sidechains paper, do you agree?

-------------------------

ClaraShk | 2026-10-01 15:33:13 UTC | #8

We'll detail the peg-out processes in the paper we are working on.

As a quick sneak peek: when you peg-out, you name the vault you want to use in the burn. A later burn naming the same vault will be ignored (that is, the notes are not burned, and the vault will not open).

The denominations are fixed, so if you want to use a vault to peg-out you need the exact amount. There are other options that one can use, like atomic swaps that can be done with any amount.

-------------------------

ZmnSCPxj | 2026-10-01 16:29:19 UTC | #9

Thank you, that makes sense. I suppose my only remaining question from the previous post is: it seems you would need to also do something very much like the "SPV Proof" described in the previous Sidechains paper, except expressed somehow (??) as a condition in the "WE" scheme used in the PIPEv2 scheme, is that correct?

I won't pretend I understand the maths and I assume that somehow the encrypted key in the PIPEv2 is.... somehow invisibly created by some kind of multiparty setup, maybe? :P

>  There are other options that one can use, like atomic swaps that can be done with any amount.

That is fine for a lot of day-to-day operations, but would be more akin to offchain-onchain swaps in Lightning than true unilateral exit in Lightning; without true exit, nobody would trust the swaps.

-------------------------

ClaraShk | 2026-10-01 17:44:47 UTC | #10

[quote="ZmnSCPxj, post:9, topic:2912"]
seems you would need to also do something very much like the “SPV Proof”

[/quote]

The amount of PoW is a parameter that we are using, but I think it's a bit different from "SPV proof". It will be an interesting point to come back to once we have our vault paper out.

-------------------------

AdamISZ | 2026-10-02 16:13:44 UTC | #11


[quote="ZmnSCPxj, post:9, topic:2912"]
Thank you, that makes sense. I suppose my only remaining question from the previous post is: it seems you would need to also do something very much like the “SPV Proof” described in the previous Sidechains paper, except expressed somehow (??) as a condition in the “WE” scheme used in the PIPEv2 scheme, is that correct?
[/quote]

It might help in this to take a look at Section 4 of the [BitVM2 paper](https://eprint.iacr.org/2025/1158.pdf); yes, SPV proofs, kinda, there. But there needs to be a way to challenge such a proof (with a heavier chain or similar).

I remember being rather flummoxed at first that anyone was proposing such a "crazy" idea: to prove that a transaction exists on the Bitcoin blockchain, inside Bitcoin Script (via other steps of course!). But it's not crazy, it's just ... rather difficult :) and rather limited.

In PIPESv2 (at least as it was written earlier?) I don't think they specified this "SPV" part and I'm not sure how it would work as this "one shot proof" thing: how you define a valid SPV proof might not cover the possibility of it not being the canonical chain. I think.

-------------------------

ZmnSCPxj | 2026-10-02 19:01:30 UTC | #12

[quote="AdamISZ, post:11, topic:2912"]
But there needs to be a way to challenge such a proof (with a heavier chain or similar).
[/quote]

Hmm, I suppose for a "true" sidechain where the sidechain has a different set of miners as the true Bitcoin blockchain, as in the original sidechains paper, this distinction makes sense.

However, I think for the purposes of what is effectively a sidechain published inside the true Bitcoin blockchain, the distinction does not matter.

With the original sidechains paper, at *some* point, on the true Bitcoin blockchain, the amount must be released, at which point challenging with a heavier chain is no longer possible.  If after that timeout, you come into possession of a heavier chain on the sidechain that contradicts the peg-out, the coins on the true Bitcoin blockchain have already been released, so it is too late.

In the case of a sidechain or sidechain-like construction whose publication is itself tied to the true Bitcoin blockchain (as in this proposal, or with qevirpunvaf), that timeout can be encoded as the requirement to present a Bitcoin header chain of that length, proving burial of a block that contains (a commitment to) the peg-out sidechain-side transaction.

Whether to make any scheme similar to this a "sidechain" or not is largely a detail of whether the transactions are encoded directly into some `OP_RETURN` or similar in the true Bitcoin blockchain, or only commitments to those transactions, with the transaction data being stored in a separate network.

-------------------------

