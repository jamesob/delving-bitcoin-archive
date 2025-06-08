# Perpetually KYC'd Coins Using Evil Covenants

RobinLinus | 2024-02-13 17:10:40 UTC | #1

# Perpetually KYC'd Coins Using Evil Covenants

Some governments, such as the EU, are working hard on crippling Bitcoin with [excessive KYC laws](https://www.coindesk.com/learn/mica-eus-comprehensive-new-crypto-regulation-explained/). This also means that protocol updates could be abused to introduce a 'perpetual KYC' contract. Financial institutions would likely tend to welcome such a mechanism as it simplifies regulatory clarity while preserving the advantages they care about, e.g., quick international settlement and a limited supply.

Adversarial thinking is what keeps bitcoin secure. So we should explore and become aware of the different ways to implement "evil covenants". For example, combining the opcodes `OP_CTV`, `OP_CSFS`, `OP_CAT`, and `OP_EXPIRE` enables such a perpetual KYC contract:

1. Every two weeks the government signs the Merkle root of their whitelist. Additionally, that signature signs the current date
2. The contract checks the government's signature using `OP_CSFS`
3. The contract verifies the inclusion proof for the recipient's address using `OP_CAT`
4. The contract enforces the covenant using `OP_CTV`
5. The contract uses `OP_EXPIRE` to ensure that the government's signature is at most 2 weeks old

## Features
- The whitelist can be updated without having to change the contracts of existing UTXOs
- The government does not have to run a cosigning server
- The government does not have to use a hot key. It can sign offline using air-gapped devices
- The government can add addresses to the whitelist at any time
- The government can remove addresses from the whitelist every two weeks
- The government has to publish only the updates to the list and their new signature on static file servers
- The contract can tighten (or relax) spending limits. E.g., send at most $1000 to non-KYC'd addresses. Or receiving more than $50000 could require more strict KYC processes.
- Self custody becomes much safer as attackers cannot steal KYC'd coins 
- The government can force users to update their contracts
- The government can revoke its control of BTC held under this policy by whitelisting some non-covenant address

-------------------------

light | 2024-02-13 17:04:01 UTC | #2

Does the use of "perpetual" here mean that the BTC can never leave this covenant, it is forever under the control of the government's policy? Related, is there a way for the government to revoke its control of BTC held under this policy? (effectively making the policy "users can spend to any address")

-------------------------

RobinLinus | 2024-02-13 17:09:04 UTC | #3

[quote="light, post:2, topic:556"]
revoke its control of BTC held under this policy?
[/quote]

Yes, the government can simply whitelist an address that is not a covenant. Will add that to the features.

-------------------------

recent798 | 2024-02-13 17:47:20 UTC | #4

For the government, it seems strictly worse than using multisig.

[quote="RobinLinus, post:1, topic:556"]
The whitelist can be updated without having to change the contracts of existing UTXOs
[/quote]

The same is true for multisig.

[quote="RobinLinus, post:1, topic:556"]
Or receiving more than $50000 could require more strict KYC processes.
[/quote]

How can KYC be enforced on the receiving end? Anyone with non-covenant bitcoin can freely decide how to spend them, including to a KYC address.

[quote="RobinLinus, post:1, topic:556"]
The government does not have to use a hot key. It can sign offline using air-gapped devices
[/quote]

So to register a new address with the government, a government employee would have to carry the address(es) over to the air-gapped device and back? Whatever the process is, it seems no different than an air-gapped process to sign a KYC-multisig.

[quote="RobinLinus, post:1, topic:556"]
The government does not have to run a cosigning server
[/quote]

Instead of running an (airgapped) cosigining sever, they'd have to run an (airgapped) whitelist server.

-------------------------

RobinLinus | 2024-02-13 18:27:42 UTC | #5

Thanks for your questions, @recent798. 

[quote="recent798, post:4, topic:556"]
How can KYC be enforced on the receiving end?
[/quote]

In this covenant model, once you're KYC'd, you can send bitcoins only to other KYC'd addresses. This is enforced by `OP_CTV`

[quote="recent798, post:4, topic:556"]
carry the address(es) over to the air-gapped device
[/quote]

No, only the Merkle root of the list, which is very compact.

[quote="recent798, post:4, topic:556"]
no different than an air-gapped process to sign a KYC-multisig
[/quote]


There are significant differences. 

First, the covenant model requires only a single signature every two weeks. That's 26 signatures per year. In contrast, the cosigner model requires to co-sign every single transaction, which could be millions per year if there are thousands of institutions.

Second, the covenant model tolerates a slow signing process (e.g., it can take a week and use some threshold scheme) because that doesn't affect transaction speeds. In contrast, in the cosigner model without a hot key you have to wait for the government to complete their cold signing process before you can transact. The longer the waiting time the higher is your opportunity cost. The shorter the waiting time the more complex and fragile is the air-gapped signing.


[quote="recent798, post:4, topic:556"]
they’d have to run an (airgapped) whitelist server
[/quote]

No, not airgapped. The server hosting the whitelist is just a simple, untrusted, static file server. It's very easy to run multiple such file servers (particularly, in comparison to the complexity of a cosigning server).

-------------------------

recent798 | 2024-02-13 18:59:47 UTC | #6

[quote="RobinLinus, post:5, topic:556"]
No, only the Merkle root of the list, which is very compact.
[/quote]

Sure, but the merkle root changes every time an item is added to the list. Assuming that users don't want to re-use addresses, and that users (at least some of them) create a new address for each payment, implies that the merkle root changes with every such payment. So signing the merkle root is more hassle than just signing the payment itself.

[quote="RobinLinus, post:5, topic:556"]
First, the covenant model requires only a single signature every two weeks.
[/quote]

This also implies that a new address not on the list will have to wait for up to two weeks, which seems unacceptable from a UX standpoint.

-------------------------

RobinLinus | 2024-02-13 20:05:17 UTC | #7

[quote="recent798, post:6, topic:556"]
create a new address for each payment
[/quote]

During registration you can give a xpub to the government, which allows you to register thousands of addresses at once.

[quote="recent798, post:6, topic:556"]
new address not on the list will have to wait for up to two weeks
[/quote]

Probably all existing financial institutions register at once during the initial setup. If you create a new financial institution it's not an issue to wait for two weeks to get registered -- probably they're used to much more lengthy compliance processes. 

Also note that in this model the government can add new addresses instantly if they want.

-------------------------

juangalt | 2024-02-13 20:15:48 UTC | #8

Seems like such a covenant simply makes a KYC'd coins more efficient for such a government institution, but this evil construct is already possible anyway with standard enterprise Bitcoin software, hardware and best practices. 

On the other hand, can the terms of this covenant be publicly verifiable by the public? If so, such a covenant would be restrained by it's own structure and rules. In other words, changes to it's terms would be public when implemented, where as the terms and rules of a multisig KYC'd coin pool are entirely arbitrary to the key holders. 

If those assumptions are correct, I think I prefer covenant KYC pools than multisig ones.

-------------------------

recent798 | 2024-02-13 20:35:41 UTC | #9

[quote="RobinLinus, post:7, topic:556"]
If you create a new financial institution it’s not an issue to wait for two weeks to get registered
[/quote]

Why would financial institutions be required to use covenants (or multisig) on-chain? This seems easier to achieve by having them do the checks in their backed server code that creates transactions. The government could release the (signed) backend server code every two weeks, without having to go through the hassle of implementing it as a convenant (or even multisig).

-------------------------

QcMrHyde | 2024-02-13 22:17:35 UTC | #10

Could you do the same type of recursive covenants without OP_CAT?

-------------------------

roasbeef | 2024-02-13 23:13:17 UTC | #11

Yep, there're around half a dozen different ways we know how to introduce maximally expressive transaction introspection.

-------------------------

sanket1729 | 2024-02-13 23:50:36 UTC | #12

Before delving further into this topic, I'd like to address the concern about the potential risks associated with evil covenants. Here are a few points to consider:

1. Evil covenant functionality already exists in altcoins with significant market capitalization. Despite this, we haven't observed any notable issues arising from such features in these coins.
2. The capability for evil covenants exists within current Bitcoin scripting. Therefore, implementing them doesn't introduce new risks per se. Concerns regarding high interactivity requirements can be mitigated by incentivizing compliance with the covenants. For instance, implementing a one-time transition into a script controlled by an internal key held by the government, coupled with a script path featuring a CSV of `n` blocks. Users than must send to addresses that recursively enforce this covenant or risk losing funds. In case of policy violations, the government can freeze the funds associated with the address.

Now, returning to the topic at hand, I think we can improve it:

Given the government's ability to freeze addresses, a simpler approach might involve incorporating a designated freeze internal key. This approach would separate the freezing policy from the Bitcoin script. Utilizing `CAT` and `CSFS`), it's possible (albeit challenging) to restrict the merkle root in such a way that all outputs spending from the covenant input require the same freeze key for spending. 

Here's how this could work:

* Each coin would have a freeze key that directs funds to a predetermined government-controlled address, enforced through a recursive covenant by leveraging `CAT` and `CSFS`.
* The government can maintain and update a list of permitted addresses dynamically as needed.
* Before transacting users check if the transaction satisfies the requirement and goes on with the transaction.

One approach to implementing this involves restricting the internal key to function as the freeze key. Alternatively, in the context of SegWit scripting, it might be simpler to ensure that every script begins with an `IF <freeze_key> CHECKSIG ELSE <unchecked input provided by user>` structure. I've successfully employed a similar approach utilizing `CAT` and `CSFS` for a different script, which I can delve into in a subsequent post if there's interest.

On a practical note, it's important to acknowledge that the applications of `CAT` often encounter limitations, particularly with the 520-byte constraint. While workarounds like `CODESEPARATOR` exist, they can become cumbersome over time.

-------------------------

1440000bytes | 2024-02-14 07:28:22 UTC | #13

Most of the things described in https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32023R1114 will affect custodians.

 > "Hardware or software providers of non-custodial wallets should not fall within the scope of 
this Regulation."

I like the attempt to think about different ways in which covenants can be used for evil things. However, do not agree with this specific example. Governments do not need covenants to enforce KYC and could already use multisig.

In fact, with no covenants or scaling improvements, use of custodians will increase which makes it easier to KYC everyone.

-------------------------

harding | 2024-02-16 16:27:39 UTC | #14

Some previous discussion about this issue:

- https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2021-July/019200.html (covered in [Optech Newsletter #157](https://bitcoinops.org/en/newsletters/2021/07/14/#request-for-op-checksigfromstack-design-suggestions))

- https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-February/019891.html (covered in [Optech Newsletter #190](/en/newsletters/2022/03/09/#gradual-loss-of-censorship-resistance) along with a list of other arguments against recursive covenants)

-------------------------

light | 2025-05-17 20:50:31 UTC | #15

I have been thinking about this topic and have a random collection of thoughts to share here.

---

Regarding the features of the KYC covenant described in OP:

[quote]
* The government does not have to use a hot key. It can sign offline using air-gapped devices
[/quote]

This is an advantage; at the same time, a signing server can create multiple spending paths such that if the key on one path is compromised, they can switch to another key. That doesn't help in cases where the attacker has also compromised the user key, but helps recover in cases where the user key is not compromised. This might be "good enough", since the case of the covenant, if the user key is compromised, this can also lead to theft (see point below about stealing KYC'd coins).

[quote]
* The government can remove addresses from the whitelist every two weeks
[/quote]

I don't think this will be acceptable in practice. If an address is found to be compromised authorities will want to be able to immediately remove it from the KYC list. Two weeks is a long time to allow an address to be abused.

[quote]
* Self custody becomes much safer as attackers cannot steal KYC’d coins
[/quote]

Stealing coins is likely more difficult in the KYC covenant paradigm but we only have to look at how prevalent theft and fraud are in the actually-existing KYC banking system to see that "cannot steal" is far too strong a claim. Attackers have many tools at their disposal to steal KYC funds, such as money mules, account takeover, and identity theft, to name just a few. Translating these methods into the KYC covenant context, this is how an attacker could circumvent the KYC restrictions to steal coins (assuming they have already compromised the private key, of course):

- Money mules: the attacker would hire willing agents to go through the KYC process and receive funds stolen from another account, and quickly cash them out before the theft victim can report the theft and freeze the funds.

- Account takeover: the attacker would compromise an existing KYC'd account and use it to cash out stolen funds before either the theft victim or the takeover victim realizes what is happening.

- Identity theft: the attacker would create a KYC account using a stolen identity and use it to cash out stolen funds before either the theft victim or the identity theft victim realizes what is happening.

These methods can also be combined to create "chains" or "fan-out" structures of transfers that make the theft harder to (fully) recover from. Again this is not theoretical, this is already how attackers steal money in the KYC banking system.

---

Regarding alternatives:

Today we have a few comparable alternatives to point to, one that uses a signing server ([AMP](https://docs.liquid.net/docs/blockstream-amp-overview)) and two that use smart contracts ([Railgun](https://docs.railgun.org/wiki/assurance/private-proofs-of-innocence) and [Privacy Pools](https://docs.privacypools.com/layers/asp)). AFAIK neither model has been adopted or mandated by any govt, so TBD which model tyrants will end up preferring.

(Notably, trustless implementations of Railgun and Privacy Pools could be built on bitcoin using only `OP_CAT`, by using `OP_CAT` to create a trustless bridge to an EVM-compatible rollup and then deploying the open-source Railgun/Privacy Pools smart contracts to that rollup.)

-------------------------

RobinLinus | 2025-05-17 21:49:58 UTC | #16

[quote="light, post:15, topic:556"]
I don’t think this will be acceptable in practice. If an address is found to be compromised authorities will want to be able to immediately remove it from the KYC list. Two weeks is a long time to allow an address to be abused.
[/quote]

A compromised account can still transact only within the KYC'd subset of accounts. So in fact it doesn't matter when you blacklist them (or the addresses to which they sent the compromised coins).


[quote="light, post:15, topic:556"]
quickly cash them out before the theft victim can report the theft and freeze the funds
[/quote]

KYC would be probably pointless without some sort of rate limit which prohibits draining funds like that. Particularly for large amounts, for which this "theft protection" is the most relevant.  

[quote="light, post:15, topic:556"]
cash out stolen funds
[/quote]

Though where would you cash out those stolen funds if your trace is fully KYC'd as can only send them to KYC'd accounts and exchanges?

[quote="light, post:15, topic:556"]
fan-out” structures of transfers
[/quote]

Again, this assumes there's no rate limiting for large amounts.
[quote="light, post:15, topic:556"]
signing server
[/quote]

Key management is hard. Hot-key management is harder. Even more so if you care about uptime and throughput.

[quote="light, post:15, topic:556"]
neither model has been adopted or mandated by any govt
[/quote]

But they also haven't rolled out CBDCs yet...

-------------------------

light | 2025-05-25 00:16:13 UTC | #17

[quote="RobinLinus, post:16, topic:556"]
A compromised account can still transact only within the KYC’d subset of accounts. So in fact it doesn’t matter when you blacklist them (or the addresses to which they sent the compromised coins).
[/quote]

It does matter, for the reasons I explained in this part of my response:

> an attacker could circumvent the KYC restrictions to steal coins...

Merely requiring KYC on an account does not stop abuse of the account or any downstream accounts where the stolen funds could be moved to. But if you can quickly "blacklist" the accounts then the funds can be frozen in their tracks (same way the stablecoin issuers freeze coins when a hack happens -- this generally becomes less effective the longer it takes for them to act).

[quote="RobinLinus, post:16, topic:556"]
KYC would be probably pointless without some sort of rate limit which prohibits draining funds like that. Particularly for large amounts, for which this “theft protection” is the most relevant.
[/quote]

But we see in practice with tradfi KYC accounts that such rate limits are either untenable from a UX perspective (leading to there being no effective limit) or the limits are circumvented when they are put in place anyways. Many horror stories of scam victims wiring their life savings to scammers (using fully KYC'd bank accounts on both ends of the transaction). The banks will put up all kinds of roadblocks to try to stop them but the victims jump through all the hoops and get past the limits and still lose all their money. This stuff is not hypothetical, I am talking about how the attackers in the KYC tradfi system actually operate today. Adding smart contracts into the mix is unlikely to meaningfully change any of this.

[quote="RobinLinus, post:16, topic:556"]
Key management is hard. Hot-key management is harder. Even more so if you care about uptime and throughput.
[/quote]

Yeah but governments already run many different kinds of PKI systems, including for mass scale, high volume signing e.g. TLS on govt websites. It's not uncharted territory for them.

[quote="RobinLinus, post:16, topic:556"]
But they also haven’t rolled out CBDCs yet…
[/quote]

Sure I was just pointing out that we don't yet have any empirical data about which model tyrants will prefer (if any) for controlling cryptocurrency usage.

-------------------------

RobinLinus | 2025-05-26 05:12:52 UTC | #18

[quote="light, post:17, topic:556"]
> an attacker could circumvent the KYC restrictions to steal coins…
[/quote]

As I said, this is hard for an attacker trying to steal large sums, which is what institutions care about.

[quote="light, post:17, topic:556"]
TLS on govt websites
[/quote]

The difference is that TLS certificates can be easily revoked if a  key gets lost or compromised. 

That's not possible with a co-signing key once it is enshrined into millions of UTXOs. In contrast, with CAT, EXPIRE, and CSFS you can build an infrastructure similar to TLS, including certificate authorities and such.

-------------------------

light | 2025-05-30 15:44:35 UTC | #19

[quote="RobinLinus, post:18, topic:556"]
this is hard for an attacker trying to steal large sums
[/quote]

It's probably not as hard as you think:

> The federal government cannot say for sure how much of the more than $900 billion in pandemic-related unemployment relief has been stolen, but credible estimates range from $87 billion to $400 billion — at least half of which went to foreign criminals, law enforcement officials say...
>
> "Organized crime has never had an opportunity where any American's identity could be converted into $20,000, and it became their Super Bowl."
> 
> [Source](https://www.nbcnews.com/news/us-news/easy-money-how-international-scam-artists-pulled-epic-theft-covid-n1276789)


[quote="RobinLinus, post:18, topic:556"]
The difference is that TLS certificates can be easily revoked if a key gets lost or compromised.

That’s not possible with a co-signing key once it is enshrined into millions of UTXOs. In contrast, with CAT, EXPIRE, and CSFS you can build an infrastructure similar to TLS, including certificate authorities and such.
[/quote]

I have only napkin sketched this so not 100% sure, but I think the govt could use a connector output to prevent spending on the path with the compromised key, forcing transactions to go through a secondary path with a backup key that the govt keeps offline until it is needed for a situation like this. The govt can prepare an arbitrarily large number of such paths in the emulated covenant tapscript. In the mean time, the attacker still needs to compromise the end user key in addition to the cosigner key, so as soon as compromise is detected the govt can alert users to roll their funds over into a new covenant with new cosigner keys.

In any case I do agree that with new opcodes the system becomes easier to manage. My argument is, despite that, the limitations that exist today seem unlikely to be _critical impediments_ for tyrants (and there are [credible alternatives](https://delvingbitcoin.org/t/on-the-possibility-of-evil-covenants-and-implications-for-soft-fork-proposals/1703) if the cosigning method does prove to be too cumbersome).

-------------------------

