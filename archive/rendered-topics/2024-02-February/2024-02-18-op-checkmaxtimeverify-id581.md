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

