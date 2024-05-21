# Mutual exclusiveness of op_codes

PierreRochard | 2024-05-21 03:35:59 UTC | #1

Doing a very surface-level readthrough of current softfork proposals, a good amount of narrative seems to be "X is better than Y because..." implying a mutually exclusive predicament, that we have to choose one proposal over another. Without picking a side of whether that's true or not, I'd like to at least understand the philosophical arguments on both sides. 

Some initial thoughts:

* Needing to maintain more code. In some cases where the op_code is trivial like OP_CAT this seems inconsequential, in other cases like Simplicity it's a very substantive point. 
* Attack / bugs / vulnerability surface area, not always correlated with lines of code. 
* Limited number of op_code "slots" available to take.
* Activation politics, bundling could increase contention but separating might exhaust participants' energy too.
* Ecosystem complexity, multiple similar options to accomplish the same outcome.                                                                                                                                                                                                      

I just philosophically like the idea of letting developers and users opt in to the op_code they want even if there's a 90% overlap between two different op_codes. They might find one easier to grok or have a very particular use case. As long as node resource usage limits are respected and protocol maintainers are not overwhelmed, it seems fairly innocuous. Interested in hearing your thoughts!

-------------------------

garlonicon | 2024-05-21 12:06:58 UTC | #2

I think some opcodes could be simplified, if you do the same thing, which was already done in assembly: split each opcode into smaller parts, and calculate the real cost on a different layer. For example: `OP_CHECKSIG` is quite complex. It has at least some sub-opcodes inside, like calculating z-value (message hash), calculating R-value from r-value (lifting x-value public key into the real (x,y) point on secp256k1), and of course doing addition and multiplication, between given public keys and raw values, to get a matching equation.

Which means, that if you want to implement for example `OP_CHECKSIGFROMSTACK`, then by having `OP_CHECKSIG` alone, a lot of that is already done, and the only replacement is the message to be hashed. The same is true for many other contracts, where you want to validate a given signature, by using a different message. For example: each change into sighashes (like `SIGHASH_ANYPREVOUT`) fall into a similar category.

So, is it more code? Kinda, but not quite, because a lot of code can be reused (or even simplified, because if you have `OP_CHECKSIGFROMSTACK`, then you can implicitly convert each `OP_CHECKSIG` into `OP_CHECKSIGFROMSTACK`, and have a single place to handle all of that). So, instead of `<signature> <pubkey> OP_CHECKSIG`, you can convert it statically into `<message> <signature> <pubkey> OP_CHECKSIGFROMSTACK`, and then, in practice, `OP_CHECKSIG` could become just an alias into `OP_TUCK_MESSAGE OP_CHECKSIGFROMSTACK`, where `OP_TUCK_MESSAGE` don't have to be any real opcode, but just some internal function call in the source code (note that when it comes to existing code, there are lot of internal things, which already exist, and have no representation in the Script, so the whole diff is not that much bigger, as it seems to be; for example we already have functions for checking any signature, but they are just used in places like "Bitcoin Message" handling).

When it comes to available slots for new opcodes, then we still have a lot of them (but maybe we need some kind of page to trace them, like BIP numbers). Even Taproot can show us, that the whole Script can be completely changed into TapScript, without affecting the original one. Which means, that if there would be any need, to have some "assembly for Script" in the future, then a single opcode is all you need to introduce it. I can even think of cases, where you can use your own Script, for example with Homomorphic Encryption, and get it activated in Lightning Network, without touching any mainnet rules. Then, you would encrypt things, execute them in LN-only, and then decrypt and post on-chain. But in-between, you can have some execution steps, which are not allowed in the current Script, and could be handled only by some second layer.

Also, that last case partially answers your question about activation politics: it could be done locally first, to test things properly, and deploy later, if there would be enough users, willing to move those changes on-chain (but if that kind of activation would be successful, then some improvements could just stay in L2, because in case of Homomorphic Encryption, they could be trustlessly decrypted, and shared in on-chain compatible form, if needed).

-------------------------

