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

