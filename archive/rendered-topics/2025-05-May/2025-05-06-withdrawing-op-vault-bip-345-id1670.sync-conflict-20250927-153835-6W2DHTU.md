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

