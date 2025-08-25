# Taking a second look at OP_EXPIRE

cmp_ancp | 2025-08-25 01:47:39 UTC | #1

Hi all!

In recent weeks, I’ve been thinking about a feature bitcoin script doesn’t have yet: the possibility of expiration date for transactions.

I made a rapid research and found out this opcode have already been proposed before (OP_EXPIRE: Mitigating replacing cycling attacks - Protocol Design - Delving Bitcoin https://share.google/70ClGDzIW0NONuDgx ), but mainly with LN and replace-cycling attacks in mind. I thought about this feature for different reasons, and I think it hasn’t been given enough credit or sufficiently stressed out by the community.

# Context

I first had this idea after reading about the protocol level of a project I am interested in, the implementation of coinswap on Bitcoin ( https://github.com/citadel-tech/coinswap ). AFAIK, this is the first onchain BTC to BTC atomic swap implementation so far.

The protocol roughly goes:

* Participants create a closed loop of “channels” made of musigs
* HTLCs are used to swap coins in the loop
* After the pre image reveal, there is a handover of private keys to the new owners
* The new owners use the private key to spend the musig and successfully finish the session

If everything goes right, onchain footprints are only of musigs in the P2WSH version, and after implementing P2TR version, indistinguishable form every other P2TR.

After understanding the steps, it boggled my mind the necessity of the last tx by all participants. The last tx is necessary to be certain that the former partner doesn’t try to steal funds with the HTLC, so the only way a wallet wouldn’t need to constantly check for steal attempts is by spending the musig.

But this is suboptimal. Private keys have already been handovered, if the HTLCs didn’t exist, the new owner would already have sufficient info for spending the new UTXO, and we would have a single tx swap.

In order to BTC to have a safe future, we need high fees, this is crucial for hashrate maintanance. In the ideal world, txs on the blockchain need to be made minimal. In that way, I think OP_EXPIRE could reduce one tx in contracts involving swaps and privkey handover.

# Further use cases

This first use case made me come with this opcode (even though, I didn’t know it was already been proposed lol), but after further thinking, I found out it can be used in several other ways, and that it composes really well with OP_CSFS and OP_CTV.

How would the opcode work:

* It pops up the top value on the stack, representing the expiration block or time (TBD if it could have different encodings of height, relative time, etc, but the absolute height case is essential for some cases)
* It asserts if the UTXO have expired or not (TBD if it would have a verify functionality or if it could push a bool in the stack, which permits IF ELSE composability, but I wonder if multiple tapleaves aren’t useful enough)

We could build scripts where the expiration height/delta on the stack is signed, allowing for diverse use cases composing with OP_CSFS, such as:

* Time restrained delegations (used together with OP_CTV)
* Contracts operating on real time data confirmed by an oracle (such as prices, exchange rates, etc)
* Short time approval of owner or associated entity
* Voting, elections, with restrained time for consensus
* More options of contracts and L2 applications

Delegation can be implemented in a clever way, such as:

* Owner have a master key pair, and a first OP_CSFS tests agains the master pubkey
* Owner creates an ephemeral key pair, and signs the ephemeral pubkey, validated by the first OP_CSFS
* They use the ephemeral privkey to sign an expiration height and a OP_CTV hash, or any other value they wish to be used by the script. Thes values are further validated by other calls to OP_CSFS, now validating against the ephemeral pubkey
* After the expiration height, all approvals are revoked, and if it wasn’t confirmed onchain, the delegated operator may need a new approval. The ephemeral key pair grants that chosen values are tied to the expiration, and a new expiration height cannot be used to validate an old chosen value

# Trust assumptions and possible drawbacks

I understand that, depending on the way the opcode is used and the height is calculated, values may be burn forever. However, such assumptions already exists (in a certain degree) in any HTLC contract: if your revoke tx isn’t confirmed in time, the counterparty may get your funds.

Other possible attack vector I think may exist are miners holding to approve txs in order to make participants pay higher fees under pressure. However, this would only work in a centralized mining world, because otherwise miners may lose possible earnings to concurrent pools.

Aside from those considerstions, I see this opcode as a simple feature that doesn’t affect MEVil nor would have unpredicted outcomes. But I am interested in hearing other takes on ths subject.

-------------------------

garlonicon | 2025-08-25 07:59:36 UTC | #2

> I see this opcode as a simple feature that doesn’t affect MEVil nor would have unpredicted outcomes

There is some MEV, because if a given transaction can expire, and make some funds unspendable, then it means, that by not mining some transaction, you can force someone, to burn some coins.

Lost coins is like a donation to everyone. Lose half of the supply, and everything will be worth at least two times more, than it was. And when the block reward will be set to zero, and only existing coins will circulate, then whales will have more reasons, to encourage other people to burn more coins, because it will increase the value of their own coins even more.

> Other possible attack vector I think may exist are miners holding to approve txs in order to make participants pay higher fees under pressure. However, this would only work in a centralized mining world, because otherwise miners may lose possible earnings to concurrent pools.

We don’t have that many mining pools, so it is centralized to some extent. Now, if 1% hashrate is including your transactions, censored by everyone else, you have a chance to get a confirmation once per 100 blocks. When new rules will be activated, then it would mean, that if your timelock is less than 100 blocks, it can simply expire, and other pools will benefit from that, so there will be an incentive, to censor transactions, because later, nobody will be able to include them, after they will expire.

I think there is only one proper way, to really reach transaction expiration: just confirm a different version of the same transaction. Then, all versions, which use the same inputs, will expire automatically, without any consensus changes.

-------------------------

