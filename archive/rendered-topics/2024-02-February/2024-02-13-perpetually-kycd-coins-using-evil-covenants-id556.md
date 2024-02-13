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

