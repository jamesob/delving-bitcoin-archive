# Op_inout_amount

Chris_Stewart_5 | 2024-02-12 15:05:39 UTC | #1

Hi all,

I've attempted to implement `OP_INOUT_AMOUNT`. The purpose of `OP_INOUT_AMOUNT` is to push the input value and a set of output values onto the Script interpreter stack.

Here is my draft BIP:

https://github.com/Christewart/bips/blob/92c108136a0400b3a2fd66ea6c291ec317ee4a01/bip-op-inout-amount.mediawiki

Here is the implementation. 

https://github.com/Christewart/bitcoin/tree/op-inout-amount

One purpose of implementing this is to see how this works in conjunction with my [64bit op code PR](https://github.com/bitcoin/bitcoin/pull/29221). For more information on this PR see the [delving bitcoin discussion here](https://delvingbitcoin.org/t/64-bit-arithmetic-soft-fork/397/25).

### Design questions

One limitation of the current implementation is that it only can push the input and output amounts at the _current_ index being verified inside of the Script interpreter. While easiest to implement, I wonder if this should be extended to verifying more than just the input and output at the current input index being verified. It seems like something similar to how `SIGHASH` flags work would be interesting. Interested in hearing others thoughts.

### Implementation questions

Currently I extend `BaseTransactionSignatureChecker` to have 2 new methods

- `GetNIn()` - the input index we are currently verifying
- `GetTransactionData()` - gives us access to `PreComputedTransactionData` so we have access to the output that is funding us, and the set of outputs we are spending to.

I don't think this is necessarily the best place to put these methods, but it seemed like the most convenient place for putting these to hack something together. Would be interested in hearing others thoughts of how to structure the implementation.

My end goal is to use all this stuff in conjunction for [OP_TLUV](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2021-September/019419.html).

-------------------------

