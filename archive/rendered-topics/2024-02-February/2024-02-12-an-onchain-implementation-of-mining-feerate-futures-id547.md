# An Onchain Implementation Of Mining Feerate Futures

ZmnSCPxj | 2024-02-12 11:15:33 UTC | #1

Subject: An Onchain Implementation Of Mining Feerate Futures

Introduction
============

Future onchain fees to be paid to miners cannot, in
general, be predicted, as unpredictable novel uses of
the blockchain may increase use of block space, and
unpredictable novel innovations on block space use
reduction may decrease use of block space.
This uncertainty makes onchain use unpalatable, and
even non-custodial offchain uses like the Lightning
Network incur risk of onchain enforcement, and the
uncertain onchain fees could still affect offchain
behaviors.

On the other hand, as more halvenings occur, the
proportion of miner earnings that come from onchain
fees increases, such that low onchain fees may reduce
miner earnings, which discourages miners from mining
and thereby reduce the security of the blockchain
layer.

As such, mining fee futures are incentivized on both
sides:

* Blockchain users want to bet that future mining
  fees will be high, so that if mining fees *become*
  high the users will be compensated, offsetting
  their losses due to high onchain fees.
* Miners want to bet that future mining fees will be
  low, so that if mining fees *become* low the miners
  will be compensated, offsetting their reduced income
  due to low onchain fees.

In effect, a mining feerate futures scheme has
blockchain users pay miners a flat rate above the
median low-fee rate, with miners giving assured
confirmations if a high-fee spike occurs.
Miners get assured income, blockchain users get
assured confirmation even in a high-fee spike.

In this writeup I describe a method by which a miner
and a blockchain user may create a simple futures
contract in a trustless manner.

A Binary Mining Fee Futures Contract
====================================

First, the parameters:

* An amount `A[Alice]` from the blockchain user
  `Alice`, which will be given to the miner in case
  of low onchain fees.
* An amount `A[Bob]` from the miner `Bob`, which
  will be given to the blockchain user in case of
  high onchain fees.
* A feerate boundary `F`: "low onchain fees" means
  below this feerate, "high onchain fees" means
  above this feerate.
* A future blockheight `T`, and a miner execution time
  in blocks `N` after this blockheight (`T + N`).

Onchain, the SCRIPT involves the three branches below:

* `MuSig(Alice, Bob)` - Taproot keyspend branch
  - Cooperative resolution branch
* `OP_SIZE <520> OP_EQUALVERIFY <T> OP_CHECKLOCKTIMEVERIFY OP_DROP <Alice> OP_CHECKSIGVERIFY <Bob> OP_CHECKSIG`
  - Miner unilateral execution branch
* `<T + N> OP_CHECKLOCKTIMEVERIFY OP_DROP <Alice> OP_CHECKSIG`
  - User unilateral execution branch

The user and the miner both cooperatively make a
transaction spending from SegWit funds (`A[Alice]`
from the user, `A[Bob]` from the miner) to a
Taproot address with the above.
But before they sign, broadcast, and confirm the
funding transaction, the user first needs to
provide signatures to spend from the miner
unilateral branch.

Miner Unilateral (Low Fees) Branch
----------------------------------

This branch is implemented using a
blockspace-wasteful Taproot transaction, like all
proper modern constructions, such as ordinals.
(This is a joke.
The cooperative branch can be used by the miner so
it can get its funds immediately and with little
block space wasted, if the user agrees that fees
are low.
Block space is only vicariously consumed if the
user does not agree fees are low, or is not
responding when the miner requests for execution
of the low-fee case.)

(Being space-wasteful is also the reason why the
miner unilateral execution branch uses separate
`<Alice> OP_CHECKSIGVERIFY` and `<Bob> OP_CHECKSIG`,
instead of just `<1> OP_CHECKSIG`.)

Wasting block space in this branch is not a problem
as this branch only triggers when mining fees are
low (i.e. block space is cheap).

The transaction that spends from this branch is the
miner unilateral transaction, which has only an
`OP_RETURN` output.
It thus has one input with the wasteful witness
spending from the address defined above, and one
`OP_RETURN` output with some large data of 80 bytes
or 320 weight units (or whatever `datacarriersize`
setting is typically available on miners).

The total `A[Alice] + A[Bob]` should be divided by
the boundary feerate `F` to get a number of weight
units to target.
This means that the `OP_SIZE <520> OP_EQUALVERIFY`
part may vary the `<520>` so that the targeted
weight units is achieved.
The `OP_RETURN` output size can also be varied to
get the targeted weight units.

An alternative to `OP_SIZE <targetsize> OP_EQUALVERIFY`
would be to use `OP_SHA256 <hash> OP_EQUALVERIFY` with
the preimage of `<targetsize>` bytes generated
cooperatively (say by seeding a crypto PRNG from the
ECDH of the miner and user).
This would prevent witness malleation, and is in fact
what is recommended; we simply show the `OP_SIZE`
version to better explain how we create the contract,
but an `OP_SHA256` of a pre-arranged large data is
strictly better.

If the total amount `A[Alice] + A[Bob]` is large,
or the boundary feerate `F` is low, then the
number of weight units to target might be too
large for a single transaction, even with a large
witness item and a large `OP_RETURN` output.
In that case, it should have another output with
the SCRIPT branches below (internal pubkey can be
some standardized NUMS point, or just the
`MuSig(Alice[eph], Bob[eph])` with neither side
ever signing this particular aggregated keypair
--- these can be ephemeral pubkeys generated by
user and miner instead of their normal pubkeys,
with the private key forgetten immediately).

* `OP_SIZE <520> OP_EQUALVERIFY <Alice> OP_CHECKSIGVERIFY <Bob> OP_CHECKSIG`
  - Continuation branch
* `<1> OP_CHECKSEQUENCEVERIFY OP_DROP <Alice> OP_CHECKSIG`
  - Single-block assurance branch

The continuation branch should then be spent in
another transaction with expensive witness and
`OP_RETURN` outputs, and again optionally with
another output for continuation.
The size being compared to can be varied if
the particular continuation is the last one.

The second branch exists to force the miner to
put the entire sequence of transactions in a
single block.
If the miner puts only the first transaction in
the sequence in a block, then the user can steal
the remaining fund unilaterally on the next block
(in particular, since a rational miner wold only
use this in a low-fee condition, the user can
easily pay a different miner to confirm that
punishment transaction).
A rational non-majority miner would thus prefer
to just put all the miner-unilateral transactions
into a single block.

### Using The Miner Unilateral Branch

Prior to signing and broadcasting the funding
transaction, the miner unilateral transaction
(or transactions if the targeted weight is
particularly high) is signed by the user
using the miner unilateral execution branch
(and for continuation transactions, the
continuation branch).
The user sends those signatures to the miner,
but the miner does **NOT** send back signatures
to the user, as the transactions are intended
to be a miner-unilateral control.

If the mempool has only transactions with fees
below the boundary, then the miner would earn
more by actually taking the miner unilateral
transactions and mining them.
The `N` miner execution time is the grace period
to allow the miner to get *some* block into the
blockchain with the miner-unilateral transactions.

The miner effectively gets the `A[Alice]` amount,
and gets back its `A[Bob]` wager, via mining fees.

There is a risk of chain reorgs, with the
miner-unilateral transaction already seen by
other miners, who can then take the same
transction and acquire its mining fees.
On the other hand, chain reorgs are unlikely,
and deliberate chain forking in order to
acquire the miner-unilateral transaction of
another miner is expensive.
This risk can be considered by the miner when
proposing its `A[Bob]` wager.

If the mempool is dominated by transactions
with fees above the boundary, and this condition
persists up to blockheight `T + N`, then the
miner can earn more by putting the higher-fee
transactions into its blocks rather than this
unilateral transaction.
While it would "lose" its `A[Bob]` wager, it
would end up earning more than the combined
`A[Alice] + A[Bob]` amount anyway if there are
enough transactions with feerate higher than
`F` to fill a block (i.e. `A[Bob]` is a sunk
cost for the miner).

### Cooperative Low Fees Case

When fees are low, there is a mild incentive
for blockchain users to cooperate by instead
signing a transaction that simply transfers
the funds to the miner.

As the unilateral miner transaction is
wasteful of block space, if the miner is
forced to use it, this puts a mild pressure
on mempool space usage, which mildly
increases onchain fees.

The expectation is that the blockchain user
is engaging in this contract in order to
mitigate the effect of high fees.
By cooperating, the blockchain user is able
to provide a small help in keeping fees low.

Although the blockchain user would lose its
`A[Alice]` wager, it would lose it anyway
if it did not cooperate (i.e. sunk cost),
as the miner can always exercise its
unilateral transaction.
The blockchain user would still prefer to
*keep* onchain fees low by cooperating.

The miner also has a mild incentive to
cooperate in this branch: the resulting
transaction is smaller, it ends up paying
to the miner directly instead of via fees
(thus making it safe to broadcast to
competitor miners and increase the chance
that it can be enforced before `T + N`,
and also letting the miner access the funds
immediately instead of 100 blocks after it
wins a block).

Unilateral User (High Fees) Branch
----------------------------------

The miner wants the low fees branch to trigger,
as it gets `A[Alice] + A[Bob]` in that branch.
It gets first dibs by being given an earlier
timelock (`T`) compared to the timelock the
user has (`T + N`).

However, as noted in previous sections, this
branch has economic incentives to not be taken
in a high-onchain-fee environment.

Thus, we expect that if the high-fee condition
persists until `T + N`, the miner will not have
claimed the fund shared with the user.
At that point, the user will be able to claim
those funds via its unilateral branch.

The user unilateral branch can be used with any
transaction.
For example, if the user needs to add fees to
some high-priority transaction that needs to
confirm *right now*, the user can just use
this fund to pay for the fees by adding just
one more input (albeit with a witness that
includes a Tapscript revelation with its
33-byte pubkey, at least one 32-byte Merkle
Tree branch, a 32-byte internal pubkey,
and a 64-byte signature).

The miner can offer to also cooperatively
sign a transaction that spends the fund in a
transaction specified by the user.
This allows the user to reduce the witness
to just a 64-byte signature.

In order to pay fees effectively even at
ridiculously high feerates, it is likely
that the miner would have to offer an
`A[Bob]` wager that is at least one order
of magnitude larger than `A[Alice]`.
Nevertheless, if the probability of high
fees is low enough, the miner would be
willing to take on that risk in order to
get some assured income during low fee
periods.

-------------------------

harding | 2024-02-18 18:08:44 UTC | #2

Most onchain contracts are motivated by eliminating the counterparty risk that comes from relying on a trusted third party.  In this case, we get that benefit like usual, but I find it notable that we also (theoretically) get the benefit of not having to worry about principle actor corruption.  It's like being able to bet for your favorite sports team without creating a temptation for the athletes on that team to bet against themselves.

For example, a trusted third party arbitrating futures contracts for a centralized exchange or an oracle providing data for DLC users can only fully reproducibly use fee data from confirmed transactions---but miners can manipulate the fees in their blocks to a certain degree.  The trusted third party could use transactions from their local mempool to derive an estimated feerate range for a particular block, but miners could make that third party look unreliable by including a different set of transactions in their blocks (and this could happen accidentally if the miner received significant fees out-of-band).  With this onchain contract, miners are (in theory) incentivized to make decisions based on the candidate blocks they're actually producing, aligning their incentive to maximize fee revenue with their incentive to maximize future contracts revenue.

That said, I think a [contemporary post](https://delvingbitcoin.org/t/mempool-incentive-compatibility/553#a-simple-analysis-of-a-toy-problem-14) on this forum points to a weakness in this theoretical model: a miner with a large percentage of total network hashrate can forgo mining a transaction in a particular block and still have a realistic chance of being able to mine that transaction in one of their later blocks.  In the case of this contract, the large miner may be willing to settle this contract in their favor even if it means being unable to mine some higher feerate transactions in the current block because the large miner knows that they'll have a chance of mining some of the excluded transactions in the next block.  A miner with a small percentage of total network hashrate doesn't have that option: they'll have to let the contract settle against them because their probability of getting a second chance at mining any transactions is tiny.

Given the above, I suspect that widespread use of this contract might make large miners even more profitable than smaller miners, increasing centralization of mining.  I haven't analyzed how significant of an effect that would be and I don't have a good enough intuition to hazard a guess.

-------------------------

1440000bytes | 2024-02-18 19:42:28 UTC | #3

Interesting approach which could solve lot of problems.

[quote="harding, post:2, topic:547"]
That said, I think a [contemporary post](https://delvingbitcoin.org/t/mempool-incentive-compatibility/553#a-simple-analysis-of-a-toy-problem-14) on this forum points to a weakness in this theoretical model: a miner with a large percentage of total network hashrate can forgo mining a transaction in a particular block and still have a realistic chance of being able to mine that transaction in one of their later blocks. In the case of this contract, the large miner may be willing to settle this contract in their favor even if it means being unable to mine some higher feerate transactions in the current block because the large miner knows that they’ll have a chance of mining some of the excluded transactions in the next block. A miner with a small percentage of total network hashrate doesn’t have that option: they’ll have to let the contract settle against them because their probability of getting a second chance at mining any transactions is tiny.
[/quote]

This can be avoided by using average fee rates over N blocks and settle discreet log contracts using multi oracles.

-------------------------

ZmnSCPxj | 2024-02-19 00:34:09 UTC | #4

[quote="harding, post:2, topic:547"]
That said, I think a [contemporary post](https://delvingbitcoin.org/t/mempool-incentive-compatibility/553#a-simple-analysis-of-a-toy-problem-14) on this forum points to a weakness in this theoretical model: a miner with a large percentage of total network hashrate can forgo mining a transaction in a particular block and still have a realistic chance of being able to mine that transaction in one of their later blocks. In the case of this contract, the large miner may be willing to settle this contract in their favor even if it means being unable to mine some higher feerate transactions in the current block because the large miner knows that they’ll have a chance of mining some of the excluded transactions in the next block. A miner with a small percentage of total network hashrate doesn’t have that option: they’ll have to let the contract settle against them because their probability of getting a second chance at mining any transactions is tiny.
[/quote]

My intuition here is that if the congestion is large enough, then even a large miner is forced to leave off its execution branch of this contract.

Consider the case where each block can contain only one transaction each.  Suppose there are two other transactions (let us assume it is an ordinal transaction so they are as large as the miner-unilteral tx branch here) whose feerates are strictly higher than the feerate of the fixed miner-unilateral tx branch.  Now further suppose the miner is in possession of a perfect oracle that tells it that it WILL mine the next 2 blocks (i.e. it is so large it has 100% of the hashrate).

Since there is congestion (i.e. there are more high-fee-paying transactions than a single block can fit), the miner, of whatever size, is better off putting the two other transactions in both of the blocks it will get, than the miner-unilateral branch of this contract.

Thus, in terms of the `N` in this contract, if at time `T` the high-fee-paying transactions are many enough that they would fill more than `N` blocks, the miner is still incentivized to pack those transactions into its blocks.  This is because it has to decide between high-paying transaction for N blocks now, or a low-paying one now and high-paying transactions for N-(1transaction) blocks.  Note in particular that the post you linked has the miner deciding between a low-paying tx now or a high-paying tx replacement later.

This should be useful in the case where it is obvious that there is a situation where the blockchain is congested for `N` blocks at time `T`.  In boundary cases where the blockchain is congested for less than `N` blocks I think the user would be fine with the miner exercising its branch, as `N` is pre-agreed-to anyway.

-------------------------

ajtowns | 2024-02-19 09:40:13 UTC | #5

[quote="harding, post:2, topic:547"]
a miner with a large percentage of total network hashrate can forgo mining a transaction in a particular block and still have a realistic chance of being able to mine that transaction in one of their later blocks.
...
I suspect that widespread use of this contract might make large miners even more profitable than smaller miners, increasing centralization of mining
[/quote]

I don't think that's true? If a miner mines the low feerate "miner tx" before it expires, leaving some random higher feerate txs in the mempool, that raises the fees available in the next block for *everyone* including low hashrate miners. So I think that actually gives an advantage to low-hashrate miners: they get the benefit of an occassional bonus high feerate tx, without having to pay the cost of having mined a low feerate tx first.

That is: yes, there is an incentive for large miners to behave **differently** to small miners, but that difference benefits whoever finds future blocks equally, whether they participated in the strategy or not, whether they'rea small miner or a large one.

-------------------------

