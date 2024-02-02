# Taxonomy of Transaction Fees in Smart Contracts

instagibbs | 2024-02-01 16:52:08 UTC | #1

We're going to take a look through many of the common
fee tropes that wallets and smart contracts use and hopefully
figure out a more common language in how we discuss these
rather than just "CPFP vs RBF". For mempool design, this
is important as we should be looking to support what users
are attempting to make, where reasonable, to avoid out-of-band fee solutions.
This becomes even
more important as people discuss the design characteristics
of covenants proposals, and their potential Bitcoin-scaling
benefits.

For this post, "smart contracts" can also simply
be understood as logical transactions: people
want some state change effectuated, and they
may make otherwise-unrelated logical transactions
to pay fees for them.

In the below diagrams, green
arrows indicate where fees are "coming from".

# Endogenous fees, single transaction RBF

These bring fees from "inside" the logical transaction
which also happens to be a single Bitcoin transaction.

```mermaid height=450,auto
flowchart TD
    Pi4([state_i]) -.->|conflicted| P4[Endo fees, Single RBF]
    P4 --> Po4([state_i+1])
    
    linkStyle 0 stroke-width:4px,stroke:green
    linkStyle 1 stroke-width:4px,stroke:green
```

This is the most basic setup, and includes things like
simple wallet sends, up to non-anchor ln-penalty channels.
They are "maximally" compact in that fee-sizing aside,
the transaction itself is quite compact.

# Exogenous fees, single transaction RBF

These bring fees from outside the smart contract, but contain
it within a single Bitcoin transaction.

```mermaid height=450,auto
flowchart TD
    Pi5([state_i]) -.->|conflicted| P5[Exo fees, Single RBF]
    Pi5_2([fee input])--> P5
    P5 --> Po5([state_i+1])
    P5 --> Co5([change])
    
    linkStyle 1 stroke-width:4px,stroke:green
    linkStyle 3 stroke-width:4px,stroke:green
```

Typically seen with `SIGHASH_SINGLE | ANYONCANPAY` arrangements.

Examples in LN spec HTLC presigned transactions,
ln-symmetry "update" transactions, Murcery Wallet's
statechains, degen NFT trades, and likely more.

Keeping it within a single transaction saves a bit
of overhead vs other strategies, but requires additional
inputs and possibly an output to be added.

Without additional mitigations it can 

Both single-transaction RBF types benefit
from simplicity of the RBF case in today's
mempool and relay policies.

# Endogenous fees, CPFP

Fees are brought from within the parent
transaction, with no requirements to
resolve a conflict from the parent's
input set.

```mermaid height=450,auto
flowchart TD
    Pi3([state_i]) --> P3[parent]
    P3 --> Po3([state_i+1])
    Po3 --> C3[Endo fees, CPFP]
    C3 --> Co3([change])
    
    linkStyle 2 stroke-width:4px,stroke:green
    linkStyle 3 stroke-width:4px,stroke:green
```

This is most applicable to when the parent
transaction is not under the control of the
user making the CPFP, or when replacing
it would be too expensive, disallowing RBF.

A common use-case would be users receiving
a transaction from someone and the sender
unwilling or unable to increase the fee
themselves. ln-symmetry settlement
transactions have all outputs freely spendable,
also allowing endogenous spends. 
Another would be fee bumping
an LN remote transaction using your own
balance output in pre-anchor channels.

# Exogenous fees, CPFP

Same use-cases as above, but perhaps
some outputs in the smart contract
are unable to be spent due to locktimes
or other factors.

```mermaid height=450,auto
flowchart TD
    Pi([state_i]) --> P[parent]
    P --> Po([state_i+1])
    Po --> C[Exo fees, CPFP]
    fee([fee input]) --> C
    C --> Co([change])
    
    linkStyle 3 stroke-width:4px,stroke:green
    linkStyle 4 stroke-width:4px,stroke:green
```

We see this pattern in today's LN anchor channels
which requires all fees to be exogenous to avoid
pinning scenarios.

# Endogenous fees, Package RBF

"Package RBF" is a term to describe the combination
of CPFP and package RBF, where the child is paying
for the parent's conflict.

If the smart contract's outputs are otherwise
unencumbered, fees can be paid for endogenously.

```mermaid height=450,auto
flowchart TD
    Pi6([state_i]) -.->|conflicted| P6[parent]
    P6 --> Po6([state_i+1])
    Po6 --> C6[Endo fees, package RBF]
    C6 --> Co6([change])
    
    linkStyle 2 stroke-width:4px,stroke:green
    linkStyle 3 stroke-width:4px,stroke:green
```  

Currently this is unavailable to Bitcoin's
various mempool implementations as this requires
evaluation of entire packages, but users of this
would be LN for commitment transactions plus
fee paying child if outputs became otherwise
unencumbered by a one block relative timelock.


# Exogenous fees, Package RBF

```mermaid height=432,auto
flowchart TD
    Pi2([state_i]) -.->|conflicted| P2[parent]
    P2 --> Po2([state_i+1])
    Po2 --> C2[Exo fees, package RBF]
    fee2([fee input]) --> C2
    C2 --> Co2([change])
    
    linkStyle 3 stroke-width:4px,stroke:green
    linkStyle 4 stroke-width:4px,stroke:green
```

Today's LN anchor channels is the primary example.

# Composeable Transaction Structures

Not all smart contract paradigms use the parent and child scheme.

For example, there are a number of schemes involving either pre-signed or
CTV-encumbered transaction trees. 

These transaction trees will end up fitting in these above
buckets in different ways, depending on tradeoffs like usual:

1. Are we able to relay and get package evaluation for the final package?
1. Are the ultimate ("virtual")UTXOs immediately spendable, allowing endogenous fees?
1. Do we need [sibling eviction](https://delvingbitcoin.org/t/sibling-eviction-for-v3-transactions/472/9) to avoid cluster limits? How would it need to differ from v3-style sibling eviction? 

Composing these ideas, you could imagine a Timeout Tree where the leaf nodes use
Endogenous, single transaction RBF-compatible channels, and if users want to go to
chain, it composes to endogenous fees paying for the CPFP, and exercising package
RBF if sibling eviction becomes required or any ancestor input in the chain was conflicted. This unilateral exit could also be used to pay for a separate exogenously-fee-settled smart contract that has failed.

-------------------------

rustyrussell | 2024-02-01 21:06:42 UTC | #2

 As fees rise, compactness will override all other concerns. (Including security!)

This implies that we'll will use endogenous fees where possible, and single transaction which stacks as many otherwise-unrelated operations, with a single exogenous fee in/out.

We should be designing for this reality, when considering covenant constructions.

-------------------------

