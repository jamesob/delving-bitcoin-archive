# A simple approach to allowing recursive covenants by enabling quines

bramcohen | 2025-05-02 23:23:13 UTC | #1

Here is an idea for how to add a few simple opcodes to Bitcoin Script which would allow for recursive covenants to implemented in a natural and straightforward way:

The opcodes needed are as follows:

To allow incremental sha256: OP_SHA256_START OP_SHA256_UPDATE, and OP_SHA256_DIGEST which do the same things as they do in all cryptography libraries, putting the intermediate state of sha256 on the stack.

To make the above easier to quine with: OP_IN_REVERSE, which is required to be the very first opcode in the whole script and makes it so the rest of the script is parsed from the end of the file backwards

To make it easier for scripts to do anything: OP_ASSERT_OUTPUT which takes a hash of a Bitcoin Script and an amount and requires that an output corresponding to that be in the transaction.

The ethos of this is that quining generally involves concatenation and then hashing, so it's more efficient to skip the concatenation part and jump directly to the end. All but the last opcode could be replaced with OP_CAT and the same thing could still be done but it would make scripts twice as large.

To write a quine-like thing which outputs its own sha256 hash you do the following: First write a portion which assumes the intermediate state of applying sha256 to some program encoded in reverse is already written onto its stack. It takes that program hash and prepends to it Bitcoin Script commands to push that same value onto its own stack. You then take that program, calculate the intermediate state of hashing it, and then prepend a command to push that value onto the stack. Presto, a hashquine.

Let's consider a simple recursive covenant which people might want to actually use: A vault which has two keys, one hot and one cold, both of which can be used to spend but the hot one is subject to a rate limit and the cold one is not. The core of the Bitcoin Script to enable this knows the identities of the hot and cold keys and assumes that already on its stack are a self hash and its own amount. The scriptpubkey specifies whether the hot or cold key is used. If it's the cold key then the script asserts the witness contains a signature to that key and that's it. If it's a hot key then the scriptpubkey further specifies the size of the new TXO. The script makes sure that the size of the new TXO doesn't drop by too much, enforces a relative timelock necessary to be allowed to spend that much, and requires a witness with a signature from the hot key. It further requires an output whose size is the new amount and whose script is the self script with pushing the self hash and new amount onto the stack prepended. (The size can also go up with no timelock, to allow the vault to take in more funds).

This example shows a practical and useful script can be enabled using no more complex trickery than needing to quine. It's best to simply admit that quining is something which recursive covenant authors are going to have to be aware of.

It would of course be useful when making more complex smart coins to have a loop construct available in Bitcoin Script but it isn't necessary for the above example and other simple ones. In some sense it does enable loops, but they require a UTXO spend for every jump back.

-------------------------

bramcohen | 2025-05-07 04:32:30 UTC | #2

Some more details: The enforced output should be specifically P2WSH. It adds more power but is not necessary to make an option for taproot as well. Given the vagaries of the parsing of Bitcoin Script the 'reverse order' opcode should probably cause the rest of the script to be in reverse order at the byte level. To make that work with quining there needs to also be either an opcode for reversing the byte order of a string or the incremental hashing opcodes need to have an option for implicitly reversing the strings.

The opcodes needed for this are all simple, in keeping with how Bitcoin Script already works, immediately useful, and overwhelmingly likely to be a good basis for any more powerful extra functionality added later. That most likely would involve capabilities which explicitly are not enabled by this. The scope of this proposal is strictly recursive covenants.

-------------------------

