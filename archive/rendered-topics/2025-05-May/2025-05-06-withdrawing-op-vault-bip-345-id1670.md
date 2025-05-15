# Withdrawing OP_VAULT (BIP-345)

jamesob | 2025-05-06 15:51:58 UTC | #1

*(crossposted from birdland)*

Maybe it's worth being explicit: in my eyes, OP_VAULT (BIP-345) has been essentially replaced by @salvatoshi's OP_CHECKCONTRACTVERIFY (CCV) ([https://github.com/bitcoin/bips/pull/1793](https://t.co/yjvcmOw9TA)).

CCV is basically a more general version of the VAULT design and inherits some of its features, like the amount modes and some form of deferred (cross-input) checks. But CCV allows you to replace multiple tapleaves instead of just one, and it has a smaller interface and implementation in the script interpreter.

You can still create all the VAULT-style vaults that you know and love using CCV. One good outcome of VAULT is that it set a concrete "use benchmark" for competing proposals to evolve towards.

The one deficit of CCV that has kept me from making this proclamation to date is that some of the supporting documentation and tooling hasn't emerged yet. Salvatoshi's been working on the BIP, but it isn't fully complete yet. However the implementation and vault testcases have recently been well fleshed out, so CCV is definitely progressing.

The other minor hold-out I had was the prospect of possible VAULT-decorator opcodes, adding the ability to for example necessitate collateral lockup for unvaulting, or adding some rate-limiting behavior (see https://delvingbitcoin.org/t/op-vault-fanfiction-for-rate-limited-and-collateralized-unvaulting/55), which is impossible to do without either 64 bit math happening in the script interpreter (read: probably not anytime soon), or shelling out to the C++ implementation with some kind of `OP_VAULT_RATELIMIT` decorator opcode.

But I think CCV is still the superior base to build on, and if we need vaulting-specific "decorator" opcodes, they aren't out of the question. Congrats to @salvatoshi for finding a better design, and please consider VAULT more or less archived. It's still a workable change that would be a nice addition to bitcoin script, but having CCV would probably be better.

-------------------------

sjors | 2025-05-14 09:44:53 UTC | #2

What's the latest thinking with regards to the MEvil potential of `OP_VAULT`? Not having such potential could be another selling point for an application specific opcode.

Other than that, I also think that the vaults created with CCV are good enough, and the missing features don't justify a dedicated op code (yet).

-------------------------

instagibbs | 2025-05-14 18:11:48 UTC | #3

You can do some pretty weird games with it, so I am not particularly persuaded it's substantially less MEVil-y.

-------------------------

salvatoshi | 2025-05-15 09:14:49 UTC | #4

[quote="sjors, post:2, topic:1670, full:true"]
What’s the latest thinking with regards to the MEvil potential of `OP_VAULT`? Not having such potential could be another selling point for an application specific opcode.
[/quote]

I think you can make a strong case that the MEVil concerns are identical, in that there is virtually nothing you can build with one opcode, and not with the other. Constructions might vary in ergonomics/costs, but not enough to justify different MEVil categories.

A feature that VAULT has and CCV leaves to a separate opcode for vector commitments (CAT/PAIRCOMMIT/VECTORCOMMIT) is to propagate *multiple pieces of data* to the output - OP_VAULT prepends them as push opcodes for the next script. However, there are tricks to partially (= non-economically and non-ergonomically) simulate this with CCV.  Further exploring this is however not very interesting to me because I would not advocate activating CCV alone.

[quote="sjors, post:2, topic:1670, full:true"]
and the missing features don’t justify a dedicated op code (yet).
[/quote]
I don't think OP_CCV is missing any OP_VAULT feature, except (*only if merged on its own*) the fact that you can pass multiple pieces of data instead of a single one as mentioned above. However, in the context of natural vaults and vault-like scripts, this doesn't really seem to be a necessary feature, to the best of my knowledge.

In practice, CCV is more generic for vaults thanks to the complete freedom of choosing the next taptree (instead of replacing a single leaf). One notable place where this is relevant is when using [vault-like spending conditions for recovery](https://delvingbitcoin.org/t/using-op-vault-for-recovery/150): if you have multiple vault-like leaves, you probably want to drop *all of them* for the next output if *any of them* is triggered; with OP_VAULT, the only way to obtain this effect is to put all the vault-like spending paths in a single leaf with several IF/ELSE branches).

-------------------------

