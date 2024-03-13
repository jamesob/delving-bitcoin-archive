# Op_checkmaxtimeverify

EvanWinget | 2024-02-18 02:35:08 UTC | #1

Hi all, I've been a mailing list lurker for quite a while and have recently been enjoying the conversations here on Delving Bitcoin. 

Over the last few weeks I've been thinking about the value of adding a new opcode which would allow for unconfirmed transactions to be made invalid beyond a user-specified blockheight or blocktime. My initial thoughts are in [this early draft BIP](https://github.com/EvanWinget/bips/blob/7d70761da8fe43ae5d4fdd0c18b14765652ccfec/bip-checkmaxtimeverify.mediawiki)

The primary use case would be to improve the economic efficiency of peer-to-peer asset swaps and to avoid pushing users towards centralized marketplaces which can theoretically operate more efficiently. I don't have a strong view on the value or lack of value of fungible and non-fungible asset protocols on Bitcoin, but I do recognize the current popularity and a possible future state where asset transactions compete with bitcoin transactions for Bitcoin block space. If there is going to be persistent demand for asset swaps in the future (which remains to be seen), I would like to make these markets as efficient as possible in order to encourage decentralization and to minimize their use of onchain footprint (i.e. remove the need to use a transaction to cancel an asset swap offer).

Before I invest time in a draft implementation, I wanted to first wanted to turn to the Delving Bitcoin community (who are much more experienced than I am) in order to learn whether there are any obvious issues with this concept. Thanks!

-------------------------

ProofOfKeags | 2024-02-19 19:39:00 UTC | #2

You may be interested in looking at the [OP_EXPIRE](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-October/022042.html) post as it outlines something like this. I haven't read the draft bip you linked yet but if you're interested in prior art and discussion on whether or not this is a good idea and why, I'd start there.

-------------------------

EvanWinget | 2024-02-20 02:12:32 UTC | #3

Thanks for the suggestion! rearden_code pointed me there from Twitter as well, I had missed this mailing list post last fall and it's a good read.  Peter's proposal for how this could be implemented is preferable to the proposal that I had in mind. Very different motivation behind his proposal, and ultimately I think this strengthens the case for adding an op code that enables transaction expiration.

One note on the OP_EXPIRE conversation is that Peter shared: "Time-based anything is sketchy, as it could give miners incentives to lie about
the current time in the nTime field. If anything, the fact that nLockTime can
in fact be time-based was a design mistake."

I agree with him that block height is strongly preferable to block time for the reason that miners can't misrepresent the block height.

Interestingly though, if we *are* going to keep around timestamp based time locks (and I've not seen any indication anyone is going to really push to remove them), then I think that having timestamp based transaction expiration helps to balance economic incentives for miners.

In the current world we only have timestamp-based time locks and a miner wants to include the highest fee rate tx in their block. They may be incentivized to put a lower-than-accurate nTime in the block header such that they can include transactions that should only become valid at a future time. If we had an op code that could declare a max block time in which the transaction is valid and the mempool had a mixture of high fee rate transactions with expiration times and lock times, the miner would want to pull nTime forward for the time locks but push nTime backwards to include soon-to-expire transactions.

-------------------------

orkunkilic | 2024-02-20 12:12:48 UTC | #4

I don't have a concrete idea, but I wonder about the MEV implications of this potential change. Right now, PSBTs are off-chain, so MEV is solved off-chain, but moving this on-chain may introduce it.

I need to think about the implications of CSV versus CMTV. If scripts unlock at some point, you can spend whenever you want. However, if scripts become unspendable at some point, there might be incentives for miners to censor.

-------------------------

EvanWinget | 2024-02-21 04:47:51 UTC | #5

I have similar concerns about incentives to censor a UTXO such that it becomes unspendable due to expiration (“Lost coins only make everyone else’s coins worth slightly more. Think of it as a donation to everyone.”), and I have concerns that users would accidentally end up constraining confirmed outputs with an expiration and they would become unspendable due to lack of movement prior to expiration based on user error.

Luckily in the context of creating a PSBT for an asset swap you don't need to worry about either of these scenarios because even if the transaction expires prior to confirmation, the PSBT creator's output can be spent in a new transaction.

-------------------------

murch | 2024-02-29 19:29:30 UTC | #6

The biggest issue that comes to mind is that such an expiration mechanism would introduce another source of confusion about the finality of a transaction. Especially if a transaction were included in the ultimate permitted block, this could incentivize reorging out that block instead of working on the next block.

Another issue would be that it would enable attacks to waste bandwidth, e.g. you could publish a transaction that may only be included in the next block, but give it a feerate that is short of the transaction being attractive for the next block.

One way of expiring a transaction in an incentive-compatible manner would be to create a conflicting transaction that is timelocked to the expiration time but pays a higher fee and feerate. When the conflicting transaction becomes valid for block inclusion, it would be more attractive for miners to include the conflict than the original transaction which then makes the original invalid.

-------------------------

EvanWinget | 2024-03-03 02:22:15 UTC | #7

Thanks for the thoughtful response and all of the work you have been doing for so many years to help educate Bitcoiners and improve Bitcoin.

> The biggest issue that comes to mind is that such an expiration mechanism would introduce another source of confusion about the finality of a transaction. Especially if a transaction were included in the ultimate permitted block, this could incentivize reorging out that block instead of working on the next block.

I considered this and don't find it to be super concerning, but I probably need to think about it more. 0x10BC has provided historical reorg data [in the stale-blocks repo](https://github.com/bitcoin-data/stale-blocks) and I converted block heights to block times and [plotted the data](https://github.com/EvanWinget/bips/blob/7d70761da8fe43ae5d4fdd0c18b14765652ccfec/bip-checkmaxtimeverify/Monthly_stale_block_chart.png). I know this is just one node's view of the network, but based on this data we have only a few stale blocks per month and it would take extremely high fees for expiring transactions to create incentives to reorg compared to working on the next block. If it is possible that fees for expiring transactions would be high enough to create this incentive, it would demonstrate that there may be strong demand for transaction expiration and ironically increases my interest in considering such a feature.

> Another issue would be that it would enable attacks to waste bandwidth, e.g. you could publish a transaction that may only be included in the next block, but give it a feerate that is short of the transaction being attractive for the next block.

In the OP_EXPIRE mailing list thread that Peter Todd started, he suggested a solution to this: "One notable consideration is that
nodes should require higher minimum relay fees for transactions close to their
expiration height to ensure we don't waste bandwidth on transactions that have
no potential to be mined. Considering the primary use-case, it is probably
acceptable to always require a fee rate high enough to be mined in the next
block." [[source](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-October/022042.html)]

> One way of expiring a transaction in an incentive-compatible manner would be to create a conflicting transaction that is timelocked to the expiration time but pays a higher fee and feerate. When the conflicting transaction becomes valid for block inclusion, it would be more attractive for miners to include the conflict than the original transaction which then makes the original invalid.

While I see no flaw with this logic, it doesn't enable to the outcome that I find compelling about an op code that allows for transaction expiration (which is reducing on-chain footprint and optimizing economic efficiency of atomic asset swaps).

-------------------------

murch | 2024-03-12 20:09:36 UTC | #8

[quote="EvanWinget, post:7, topic:581"]
Thanks for the thoughtful response and all of the work you have been doing for so many years to help educate Bitcoiners and improve Bitcoin.
[/quote]
Cheers, thanks!

[quote="EvanWinget, post:7, topic:581"]
it would take extremely high fees for expiring transactions to create incentives to reorg compared to working on the next block
[/quote]

Fees of expiring transactions creating an incentive for reorging the previous block to include them sounds indeed not particularly problematic to me: if their fees had been high enough to warrant that, they should have been included in the block in the first place. Rather, I meant the opposite: a transaction for a substantial amount is confirmed in the ultimate block in which it is permitted, and a miner collaborating with the sender reorganizes that block out to cancel the payment. In that case, rather than just the transaction fees providing an incentive, the attack’s reward could be up to the payment amount.

[quote="EvanWinget, post:7, topic:581"]
nodes should require higher minimum relay fees for transactions close to their expiration height to ensure we don’t waste bandwidth on transactions that have no potential to be mined
[/quote]

This seems insufficient to solve the problem, unless the premium is so high that it virtually guarantees that the transaction will be mined before it expires. However, if the feerate were that high, wouldn’t OP_EXPIRE simply waste blockspace? If however the feerate of the transaction is merely competitive, the presence of OP_EXPIRE creates a bandwidth-wasting vector: an attacker would submit e.g. OP_EXPIRE transactions at the bottom of the top block and push them out of the top block with further OP_EXPIRE transactions. This way the attacker could issue a constant stream of transactions, but never pay for more than a couple barely sliding in at the bottom of the block.

-------------------------

ajtowns | 2024-03-12 23:45:54 UTC | #9

[quote="murch, post:8, topic:581"]
However, if the feerate were that high, wouldn’t OP_EXPIRE simply waste blockspace?
[/quote]

I think the ideal use case is more so that if you have a conditional path and a timeout path, if you encumber the conditional path with an `OP_EXPIRE`, then you can defer collecting via the timeout occurs.

eg, if you have a closed lightning channel with 50 HTLCs with varying timeouts over the next 7 days, currently you should probably be claiming each of them as soon as they timeout. But if the "reveal preimage" path were encumbered by an `OP_EXPIRE`, you could wait until they had all timed out, and claim them all in a single tx, paying to a single new output, and avoiding ending up with various dust-y utxos in your wallet.

-------------------------

murch | 2024-03-13 13:55:05 UTC | #10

[quote="ajtowns, post:9, topic:581"]
But if the “reveal preimage” path were encumbered by an `OP_EXPIRE`, you could wait until they had all timed out
[/quote]

Thanks, that’s an interesting application to consider. I would agree that it undermines my point about OP_EXPIRE being a waste of blockspace in conjunction with requiring a sufficiently high feerate

-------------------------

