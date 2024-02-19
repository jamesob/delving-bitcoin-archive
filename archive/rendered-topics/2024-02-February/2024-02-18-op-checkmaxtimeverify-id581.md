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

