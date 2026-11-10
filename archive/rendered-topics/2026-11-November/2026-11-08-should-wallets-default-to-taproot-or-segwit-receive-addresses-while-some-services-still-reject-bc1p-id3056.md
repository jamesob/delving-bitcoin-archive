# Should wallets default to Taproot or SegWit receive addresses while some services still reject bc1p?

Anzus_GemWallet | 2026-11-08 13:33:13 UTC | #1

I work in BD and support at Gem Wallet rather than as a developer, so this comes from the support side.

A recurring case: a user copies their receive address, tries to withdraw from an exchange or another service, and gets "invalid address". The address is a valid Taproot (bc1p…) address; the sending service just doesn't accept it. From the user's point of view, the wallet gave them a broken address.

For wallet teams here:

1. Do you default to Taproot or SegWit (bc1q) for receiving, and what decided it?
2. If you offer both, where do you put the choice so people find it when they need it, without confusing everyone else?
3. Is there a reliable way to know which services accept bc1p, or is it still trial and error?

-------------------------

edil.medeiros | 2026-11-10 14:00:47 UTC | #2

[quote="Anzus_GemWallet, post:1, topic:3056"]
2. If you offer both, where do you put the choice so people find it when they need it, without confusing everyone else?

[/quote]

Not a wallet dev here, but for as much as I appreciate the benefits of taproot, I don't see it as advantageous for the average Joe compared to p2wpkh since I expect the later to be quite the standard by now (https://mainnet.observer/charts/output-type-distribution-count/).

By average Joe, I mean someone who's just willing to send and receive some bitcoin from an exchange or a p2p exchange. If I were implementing a wallet today, I would default to that if my product is not something that supports more interesting scripts (which seems to be the case of Gem Wallet).

Curiously, your own support page that explains Bitcoin ( https://gemwallet.com/bitcoin-wallet/ ) does not mention \`bc1p\` address types, no wonder they think taproot addresses are broken.

-------------------------

sipa | 2026-11-10 14:40:30 UTC | #3

I think it's a fallacy to think that Taproot has no advantages for average users who don't use "more interesting scripts". The intention was absolutely for it to be usable and beneficial to everyone. Extra script functionality and higher resource limits are one aspect, but only secondary to the idea behind the taproot construction itself: a singular output type that covers both the "to pubkey" and "to script" use cases, is typically cheaper than earlier constructions for both, and most importantly: incentivizes building solutions that hide their policy on chain (don't reveal what scripts existed, as long as the key path spend was used).

Practically, the benefits for simple wallets are lower transaction fees and better policy privacy by sharing an anonymity set that includes non-simple wallets (i.e., not revealing you were just a simple wallet).

I do recognize that there are limitations to this. The privacy benefit is small as long as Taproot adoption isn't great, which is a bit of a chicken-and-egg problem for now. The fact that 6 years later not all entities support even sending to P2TR is a very valid reason to hesitate to make receiving to it the default. The fee reductions are minor, especially while we're in historically low feerate territory, and do depend on transaction composition (outputs are more expensive while inputs are cheaper; for transfers the output cost is paid by the sender, but for change it is paid by yourself, so change-heavy workloads benefit less).

More recently, I've also seen arguments against it based on PQ threats. I think that is a fallacy too; Bitcoin may well need to migrate to PQ cryptography in the short to medium future, but sticking with P2WPKH/P2WSH isn't going to save anyone over P2TR. Public keys, xpubs, and descriptors are being exposed all over the place, and for good reason: the cryptography is designed to treat them as public, and software all over the place is designed to make use of that property. 

So to summarize, I find it discouraging to see people recommend not defaulting to it because they fail to see the benefits it brings. It's fine to acknowledge the trade-offs, but I think we should absolutely tell people to use P2TR by default at this point.

Disclaimer: I'm a co-author of the Taproot BIPs.

-------------------------

edil.medeiros | 2026-11-10 17:51:22 UTC | #4

[quote="sipa, post:3, topic:3056"]
I find it discouraging to see people recommend not defaulting to it because they fail to see the benefits it brings

[/quote]

It was not my intention to sound that way, I would love to see taproot as the major output type after so much time.

I think the main issue here is that bitcoin addresses don't work as proper abstractions for spending conditions, not if this or that address type should be the default. Having incompatible address encodings should be fine to implementors, but seems not to be the case for the closed source side of the ecosystem that's making decisions in regard of their users (which is a legit part of packaging any technology as a product). As we all, they are sensitive to incentives and, for as much unfortunate it is:

1. I don't feel privacy is a serious (or even concrete) concern for the average person and can even be a (regulatory) threat for business;
2. transaction fee differences between segwit v0 and v1 are quite irrelevant in practice with such low feerates. 

We could propose some form of universal bitcoin address that simply communicates "here's a script, just make an output with it", but passing arbitrary scripts around in addresses is probably a bigger footgun then the current state of things.

I also don't see how the wallet implementor can steer users by answering "this is a legit address, please ask your exchange to implement support for it"; they'll simply download another app and be gone. If you have any ideas to act on the human side of things, I'm all ears and willing to help as much as I can.

-------------------------

