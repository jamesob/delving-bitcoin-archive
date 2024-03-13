# Type Erasure & Script

JeremyRubin | 2024-02-27 18:44:54 UTC | #1

Sharing some (old) thoughts as I see there a lot of discussion around covenants and particularly around handling amounts, integers, etc.

Currently, Bitcoin operates as a low-level VM without type information. However, frequently, types are implicit (e.g., arith operations fail on too-long bitstrings). 

As an alternative (e.g., new tapscript version), suppose that tapscript fragments start with an ABI that specifies at each field position what the types of the arguments should be (N.B., this assumes that witnesses in Tapscript should be function-like, and not variadic or generic). Let's assume the basic types as follows (feel free to propose alternatives):

- empty
- bool
- u8 / i8
- u16 / i16
- u20/i20
- u32 / i32
- u64 / i64
- u128 / i128
- u256 / i256
- Scalar
- Pubkey
- Signature
- HeightSequence
- TimeSequence
- HeightNLockTime
- TimeNLockTime
- ByteVec
- Input
- Output
- Amount
- Transaction
- InputIndexInThisTxn
- OutputIndexInThisTxn

Any "unknown" type in the ABI would operate like an op_success, and make validation ignore the script. (This is sort of equivalent to having in script an <n> OP_PICK  OP_IS_<TYPE> sort of script preamble, but more efficient & pedantic.)

These types should be persisted _during_ execution. This has many benefits:

1) You get more opcodes because you can define OP_ADD on Byte Vectors to be different than OP_ADD on u64 etc. Otherwise you need to 
2) You can more easily define operations that expect arguments of a certain kind and put them into a certain type (e.g., instead of OP_CAT, you can have <amount> <script> OP_OUTPUT_FROM_PARTS which handles the length fields)
3) Validation can be -- especially for covenants -- more efficient as you don't need to re-parse stuff or do complicated field length logic
4) when you add a new external ABI type (op_success) you can define how all the existing opcodes should handle it
5) You can reject "nonsense" operations, even when the underlying byte types might work by chance
6) Tooling is greatly simplified since posession of the fragment hints at how to satisfy and at how much cost it might have

Sometimes types are not good for validation -- really, what do we care if we know a type is an 8 byte integer if validation will fail at some other step? Or if it will succeed no matter what type it is? However, having in-band primitive types, for all but the simplest scripts, can actually reduce complexity and lead to simpler scripts. Consider the bolt-3 offered HTLC script:

```
# To remote node with revocation key
OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
OP_IF
    OP_CHECKSIG
OP_ELSE
    <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
    OP_NOTIF
        # To local node via HTLC-timeout transaction (timelocked).
        OP_DROP 2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
    OP_ELSE
        # To remote node with preimage.
        OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
        OP_CHECKSIG
    OP_ENDIF
OP_ENDIF
```


We can decompose this to:

Branch 1:
```
ABI<SIGNATURE, PUBKEY>
OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUALVERIFY OP_CHECKSIG
```
Branch 2
```
        ABI<SIGNATURE, SIGNATURE>
        # To local node via HTLC-timeout transaction (timelocked).
        <remote htlcpubkey> CHECKSIGVERIFY <local_htlcpubkey> CHECKSIGVERIFY
```
Branch 3
```
    ABI<SIGNATURE, u128>
   OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
    <remote_htlcpubkey> OP_CHECKSIG
```

One way of more generically implementing these ABIs would be as an OP_STACKTYPECHECKS{,VERIFY} which takes a byte vector and asserts all the types on the stack & altstack either currently match, or could be "cast" to the desired type (perhaps as OP_TRYCASTSTACK). This would aid for circumstances where one tap-path must support >1 ABI, or could be helpful in joining two scripts together that part A can attest to the output of part B being properly checked. But I think I prefer the "one ABI" mode for simplicity.


We can view miniscript as a best effort attempt at being able to run type inference over well formed scripts, as an alternative to tracking actual types. More explicit type checks would be fully compatible with miniscript, and would (potentially) expand the depth of scripts that can fit in to a miniscript-like context, enabling more script programmability (espeically with covenants).

-------------------------

Chris_Stewart_5 | 2024-03-12 19:51:17 UTC | #2

So to be a bit pedantic - to have type erasure you have to have types in the first place. Script is _interpreted_ not _compiled_. It seems like this work would be similar to what miniscript is trying to achieve (IIUC miniscript correctly).

Compiled languages translate a higher level language (Java, C++, rust etc) to machine code. Script is similar to "machine code" IMO, which don't have a type system. I don't think the machine code level is the best place to be introducing a type system. x86 and arm64 don't have them - why should Script?

-------------------------

ProofOfKeags | 2024-03-12 22:42:56 UTC | #3

[quote="Chris_Stewart_5, post:2, topic:615"]
I don’t think the machine code level is the best place to be introducing a type system. x86 and arm64 don’t have them - why should Script?
[/quote]

While I haven't yet formed an opinion on whether it would be good for type information to be preserved all the way down to consensus, I would strongly argue that having a Script interpretation operational semantics that is 100% well defined/understood is a Good Thing. Preserving type information all the way to consensus can make script vm implementations more consistent, but it is not the only way.

For instance, are all opcodes defined for all possible arguments? If they are not defined, are failure semantics well-understood? Of all the places to have a high degree of specificity, the consensus-enforced Script VM *is the place*.

I would generally caution against appealing by analogy to silicon ISAs since we are solving a different class of problem here.

-------------------------

