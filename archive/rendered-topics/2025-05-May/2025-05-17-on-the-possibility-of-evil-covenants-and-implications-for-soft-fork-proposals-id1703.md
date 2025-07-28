# On the possibility of evil covenants and implications for soft fork proposals

light | 2025-05-17 21:36:35 UTC | #1

In his post "[Perpetually KYC’d Coins Using Evil Covenants](https://delvingbitcoin.org/t/perpetually-kycd-coins-using-evil-covenants/556)" @RobinLinus sketches out a high level design for a "perpetual KYC contract" using `OP_CTV`, `OP_CSFS`, `OP_CAT`, and `OP_EXPIRE`. While the technical discussion around that particular design is interesting, I see it as downstream of a more philosophical question, which is: **should we rule out any soft fork proposal (or combination of proposals) that enables "evil" covenants?[^1]**

If the answer to this question is yes, then I believe research such as that in the thread linked above is worth pursuing, because we would of course want to know ahead of time whether or not (and perhaps how, exactly) a given proposal / combination of proposals enables evil covenants if we are explicitly trying to avoid enabling them.

If, otoh, the answer is no, then I think this research is maybe academically interesting but, for the most part, only really helpful to tyrants by doing some of their dirty work for them. (I say this with all due respect to Robin and other researchers who have spent time digging into the implementation details of evil covenants.)

Net of all considerations, my answer to whether we should rule out soft fork proposals (or combinations of proposals) that enable evil covenants is a categorical "no". That is, I don't find the possibility of evil covenants _of any conceivable design_ to be a compelling reason to oppose a soft fork that enables them.

I arrive at this conclusion for two reasons, one more technical and one more philosophical:

## 1. The existence (or possibility) of workable alternatives

I strongly doubt that absent KYC covenant abilities, tyrants who want to strictly control how their citizens use bitcoin will simply throw up their hands and say "oh well, bitcoin doesn't have the ability to create these KYC covenants, I guess we'll have to let our citizens use it freely". No, they will come up with some workable alternative that more or less achieves the desired outcome. A few options that are close at hand are:

- A [P1WP](https://medium.com/@RubenSomsen/21-million-bitcoins-to-rule-all-sidechains-the-perpetual-one-way-peg-96cb2f8ac302) scheme, where users must permanently transfer their coins into a sidesystem that runs according to rules set by the tyrant (with harsh penalties for holding/transferring coins outside of this scheme)

- A cosigner scheme, such as [AMP](https://docs.liquid.net/docs/blockstream-amp-overview), where the tyrant mandates that all coins must held in a 2-of-2 address with the tyrant as cosigner and the tyrant only cosigns transfers to approved addresses (with harsh penalties for holding/transferring coins outside of this scheme)
  
- Requiring use of a custodian, and requiring the custodians to only send to approved addresses (with harsh penalties for doing otherwise)
  
- Allowing self-custody but requiring users to only send to approved addresses (with harsh penalties for doing otherwise)

## 2. Taking the bad with the good, respecting freedom and encouraging responsibility

On the other side of the coin (no pun intended) recursive covenants can give us strong tools for making self-custodial bitcoin more private, fungible, and scalable; in short: better freedom money. I believe the potential for good here far outweighs the potential for bad. As much as I would not like to see evil covenants forced upon people, I think it's even more important that people have access to the good capabilities of covenants.

The same way firearms have empowered both tyrant forces and freedom fighters, and the internet has enabled both mass surveillance and mass free speech, bitcoin (with or without covenants) can be used for both bad and good; to control people or to liberate them.

How the tools we build actually end up being used is in some cases within our control and in some cases outside of our control. It is up to each of us as individuals to use the tools at our disposal responsibly, at the very least, to not use them to harm others. To the extent that we have a say in the organizations in society, such as govts and corporations, we should endeavor that they use available tools with the same level of care we expect of individuals. And when individuals and organizations stray from the path of righteousness, we should endeavor to have strong institutions of justice to hold those who harm others accountable, provide justice to victims, and prevent further harm. (And I would certainly consider the coercive imposition of KYC covenants to be a harmful act deserving of opposition and justice.)

[^1]: To define terms here, I consider "evil covenants" to be covenants that inflict some kind of unwanted restriction or outcome upon individual users of the covenant. I put changes that result in mevil or other _systemic_ issues in a separate category worthy of their own philosophical and technical discussions.

-------------------------

gmaxwell | 2025-05-21 00:06:44 UTC | #2

I agree. Today I greatly regret ever introducing the term covenant and starting the discussion with examples of misuses-- I thought keeping them silly would help keep the context of "just don't do this", but sadly the idea was stripped from the original context and used abusively.  I think the concept is now far more often abused to deceptively fearmonger than it is used to improve our understanding.

The fact that abuse can be done is a reason to reject abuse, not reject freedom.  There are a half dozen ways in which the consensus rules almost allow arbitrary programmatic covenants 'by accident' and almost any meaningful flexibility inherently does so-- to avoid it requires an absolute straight jacket.

And there is no reason to do so because the 'evil' usage can just be accomplished through plain multisig (or an undetectable threshold signature).   The evil actor is not meaningfully emboldened by the modest uptime improvements that a more autonomous system would provide, particularly since the covenity way of doing things comes with a *massive* increase in transaction costs.  While in non-coercive usage transaction cost inflation can usually be avoided by consensual key path joint signatures.

There is a flip side to this point however:  The productive use of covenants can also almost always be accomplished with some compromise via multisig.  There are reasons that it's not quite symmetrical (evil overlord loses only some uptime guarantees with multisig, while honest covenant user takes an addition security risks)  but I do think it's fair to say that any proposal for increased functionality which could be emulated with some kind of multisig oracle but isn't should explain why the public should believe that their application is so important and yet no one is doing it via emulation.

It's important because the utility and efficiency of the constructs can only be optimized if they match usage, which would be so much easier if the usage were real and concrete rather than speculative.  Particularly in that some of the speculative usage just amounts to "I want to mint and trade shitcoins on Bitcoin which will compete in the market with the value of Bitcoin".  ... Not exactly a great selling point to Bitcoin users.  Absence of reasonable sounding concrete usage encourages people to fixate on the speculative uses they find less reasonable.

-------------------------

jamesob | 2025-05-23 15:18:47 UTC | #3

[quote="gmaxwell, post:2, topic:1703"]
I do think it’s fair to say that any proposal for increased functionality which could be emulated with some kind of multisig oracle but isn’t should explain why the public should believe that their application is so important and yet no one is doing it via emulation.
[/quote]

I think this aphorism has done much more harm than good in the last 5 years. 

Imagine if this same bar had been applied to the deployment of `CLTV`/`CSV` in the context of Lightning. We may still be waiting on a "proof" version of Lightning that delegates locktimes to a multisig oracle. Who would invest the time to build such a thing?

Could e.g. Lightning Labs have raised capital absent those consensus changes?

Some have used this kind of rationale as a cudgel to claim that if there were "real demand" for, say, vaults that we'd see some similar multisig-crutched prototype. While Revault was essentially this -- but likely failed _because_ vaults without consensus support are too complex. Meanwhile many companies are asking for consensus-driven vaults (AnchorWatch, Anchorage, NYDIG, River).

---

If we have soft fork proposals that are simple to verify for safety, have mature implementations and demand, and potentially wide use -- as in the case of CLTV -- we should deploy them. I don't think we should handwring about a lack of multisig-enabled Rube Goldberg machines.

-------------------------

gmaxwell | 2025-05-23 15:45:07 UTC | #4

CLTV wasn't created for lightning, and it was in fact "pre-emulated" with nLocktime and had in fact shown its worth in that form.

It's fine that it isn't appropriate for some things, as I said "should explain".

As an aside, turning every discussion into another opportunity to rail about vaults has been destroying your reputation and creating a hostile environment around you.  Please reconsider your approach.

-------------------------

jamesob | 2025-05-23 16:20:44 UTC | #5

[quote="gmaxwell, post:4, topic:1703"]
CLTV wasn’t created for lightning
[/quote]

That might be strictly true, but payment channels were [named as a motivation in the BIP](https://github.com/bitcoin/bips/blob/5c7120f418d19e01f38827493085108997f01d27/bip-0065.mediawiki#payment-channels).

Bitcoin's most popular technical journalist at the time, Aaron Van Wirdum, wrote [in 2015](https://bitcoinmagazine.com/technical/checklocktimeverify-or-how-a-time-lock-patch-will-boost-bitcoin-s-potential-1446658530):

> As perhaps its most important function, CLTV is necessary for properly functional payment channels.

Prior to CLTV's deployment.

---

> As an aside, turning every discussion into another opportunity to rail about vaults has been destroying your reputation and creating a hostile environment around you. Please reconsider your approach.

Touchy! Though you forgot about the "scaling UTXO ownership" rant too ;).

-------------------------

1440000bytes | 2025-05-23 17:25:01 UTC | #6

[quote="gmaxwell, post:2, topic:1703"]
The productive use of covenants can also almost always be accomplished with some compromise via multisig. There are reasons that it’s not quite symmetrical (evil overlord loses only some uptime guarantees with multisig, while honest covenant user takes an addition security risks) but I do think it’s fair to say that any proposal for increased functionality which could be emulated with some kind of multisig oracle but isn’t should explain why the public should believe that their application is so important and yet no one is doing it via emulation.
[/quote]

Multisig and pre-signed transactions are used (mainnet) for several things that would benefit from covenants. Adding an oracle only makes things worse.

In fact, we have more PoCs to test covenants (pool, vault, ark etc.) than any other soft fork in the last 10 years.

-------------------------

CubicEarth | 2025-07-28 10:43:29 UTC | #7

[quote="gmaxwell, post:2, topic:1703"]
There is a flip side to this point however: The productive use of covenants can also almost always be accomplished with some compromise via multisig. There are reasons that it’s not quite symmetrical (evil overlord loses only some uptime guarantees with multisig, while honest covenant user takes an addition security risks) but I do think it’s fair to say that any proposal for increased functionality which could be emulated with some kind of multisig oracle but isn’t should explain why the public should believe that their application is so important and yet no one is doing it via emulation.
[/quote]

Vaults aren't just any application, they are an attempt to *improve* the security situation for users protecting what are possibly their lifetimes savings, or perhaps billions of dollars worth of assets on behalf of others.

If people have bitcoin, they are implicitly trusting the core network itself to validate things according to the rules. There is absolutely demand for *trustless vaults*, which would be native to the chain. 

If one relies on an oracle or other external source, that would require a different line of trust to said external system for the protection of coins. In a sense that is already available with BitGo or Coinbase or Casa, using various implementation of shared multisig schemes. 

So we have evidence of implementation and robust demand for trusted, external vaults! But until they are implemented and available we of course cannot directly observe usage for trustless ones, we can only infer.

If we can reasonably replace a widely used trusted option with a crustless one, I think we should. Usage might well end up being higher for the trustless one owing to the superior security guarantees .

-------------------------

gmaxwell | 2025-07-28 10:56:09 UTC | #8

[quote="CubicEarth, post:7, topic:1703"]
If we can reasonably replace a widely used trusted option with
[/quote]

Examples of wide usage? 

*characters
characters
characters
characters
characters
characters
characters
characters
characters
characters*

-------------------------

CubicEarth | 2025-07-28 12:27:54 UTC | #9

Plenty of companies have built products, such as: BitGo Wallet, Coinbase Vault, Unchained Vaults, Casa Keymaster, Green Wallet, Nunchuk Wallet, Bitkey Wallet, Swan Vault, Theya Multi  Vault, Trident Vault.

Do you think these products aren't used?

-------------------------

gmaxwell | 2025-07-28 15:55:10 UTC | #10

Same name doesn't mean similar functionality.  I think it would be interesting to see an example of how any mapped, to be sure consensus functionality captured the behavior.   For example, which consensus proposal allows the network to perform SMS validation before sending funds?

-------------------------

