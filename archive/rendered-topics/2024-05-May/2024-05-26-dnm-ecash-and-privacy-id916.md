# DNM, eCash and privacy

1440000bytes | 2024-05-26 13:22:19 UTC | #1

<h1>Problem</h1>

Bitcoin usage has declined on darknet markets and replaced by monero. 

<h1>Research</h1>

Main reason for low usage is privacy and not on-chain fees. A user recently [posted on reddit](https://www.reddit.com/r/darknet/comments/1czo5eh/why_is_btc_not_considered_safe_anymore/) asking if bitcoin is not considered safe anymore and most comments suggest using monero.

Some DNMs are custodial and the biggest market recently did an [exit scam](https://x.com/DarkDotFail/status/1765104459913330820). Owner was [arrested](https://www.justice.gov/opa/pr/incognito-market-owner-arrested-operating-one-largest-illegal-narcotics-marketplaces) by LEA on 20 May 2024. A few of them even use multisig. I have avoided sharing names or links for any DNM however, things can be verified, researched etc. on forums like [dread](http://dreadytofatroptsdj6io7l3xptbet6onoyno2yv7jicoxknyazubrad.onion).

<h1>Solution</h1>

![dnme|690x323](upload://tytj0ii4JZPzwlZiauwOeU37HMo.png)


In last few months I have observed that some developers are trying to find a use case for eCash and experimenting with different things: dollar, dlc, mining payouts, tips/zaps etc. Some even consider it as 'the' scaling solution and I don't agree with it because self custody cannot be scaled by giving up self custody. I think there are some risks involved in running such mints and the **only use case for eCash** that makes sense is DNM.

<h3>Why is eCash perfect for DNMs?</h3>

- Some DNMs are custodial and users are okay with the risks involved
- They do not care about laws and already violating multiple laws when using the markets
- Follow better opsec
- Looking to improve their privacy

<h3>How would this work?</h3>

Mints could be run by anonymous users that already have some repuation in these markets and also active in bitcoin communities. Since most DNM users prefer on-chain we could use [moksha](https://github.com/ngutech21/moksha) for minting and melting. DNM owners can also run mints.

DNM and its vendors will accept eCash as payment for different products and services. This improves privacy and UX for the users with the similar rugpull risks involved. It will benefit bitcoin usage which has declined over the years, replaced by monero and also help other bitcoin privacy projects.

Coinjoin can be used before minting eCash by users and during redeem process by mints. Users can also use payjoin while spending the redeemed UTXO.

---

**Note:** Please stay anonymous if you work on any project related to DNM usage for eCash. Feel free to suggest any improvements in the post or the use case suggested for eCash.

-------------------------

ajtowns | 2024-05-27 03:16:05 UTC | #2

[quote="1440000bytes, post:1, topic:916"]
I think there are some risks involved in running such mints and the **only use case for eCash** that makes sense is [Dark Net Markets].
[/quote]

This seems backwards to me: ecash (in the digicash style) generally requires a centralised mint that's a single point-of-failure for everyone using ecash. If most uses of ecash are dark (illegal/black market), then once law enforcement decides to crack down on them, the mint becomes an easy and attractive target. That's very high risk for the operators of the mint, even if they attempt to protect themselves via a federation or anonymity tools, and it's also high risk for users of the system: if the mint is forced to shut down by a hostile actor, none of the funds stored as ecash are recoverable. If the mint's operators are anonymous to law enforcement, then they're also anonymous to the majority of its users, and that probably elevates the risk of the operators doing a rug pull as well: a rug pull has the benefit of both getting a chunk of funds immediately, and also bounds the risk that law enforcement are going to come after you because you've been enabling crime or terrorism or whatever.

-------------------------

1440000bytes | 2024-05-27 13:03:36 UTC | #3

[quote="ajtowns, post:2, topic:916"]
That’s very high risk for the operators of the mint, even if they attempt to protect themselves via a federation or anonymity tools, and it’s also high risk for users of the system: if the mint is forced to shut down by a hostile actor, none of the funds stored as ecash are recoverable.
[/quote]

These mints will be operated by users who already take the risk of running or using darknet markets. As mentioned in the post, users of custodial darknet markets assume the market could shutdown one day and funds won't be recoverable.

[quote="ajtowns, post:2, topic:916"]
If the mint’s operators are anonymous to law enforcement, then they’re also anonymous to the majority of its users, and that probably elevates the risk of the operators doing a rug pull as well: a rug pull has the benefit of both getting a chunk of funds immediately, and also bounds the risk that law enforcement are going to come after you because you’ve been enabling crime or terrorism or whatever.
[/quote]

All DNM users, vendors etc. are anonymous but everything works based on some reputation, due diligence and calculated risks.

---
[Mint operators](https://bitcoinmints.com/) at present have the same risk of getting shutdown if one of them becomes too big. Doxxed operators and clearnet used for them makes it easier to target. Custodying funds and providing anonymity is illegal in most of the countries irrespective of use case.

This post just tries to find the right use case for eCash mints in which users have the same threat model.

-------------------------

