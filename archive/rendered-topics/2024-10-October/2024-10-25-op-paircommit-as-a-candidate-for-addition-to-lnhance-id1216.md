# OP_PAIRCOMMIT as a candidate for addition to LNhance

moonsettler | 2024-10-25 14:34:33 UTC | #1

Hi All,

Looking for some feedback on what I ended up with for PC.

The idea is that the number of SHA256 iterations should be minimized in the most likely use case we can optimize for, which is LN-Symmetry. Since the Tag can be pre-computed as a mid-state it would only take 1 or 2 hash cycles in validation for the LN-Symmetry use case.

In case of a 7 byte balance commitment + 32 byte CTV hash (no HTLCs in-flight), the total preimage size is 55 bytes. Which should make it fit into a single block with the SHA256 length commitment.

In case there are 2x 32 byte CTV hash commitments, the first 64 byte block is comprised of those hashes, and the second block is the vectors' size and total length commitment, which would be a largely 0 filled block with a very few bits set to 1.

It's a particular concern for LN-Symmetry with CTV that the concatenation of the two preimages allows for length redistribution attacks, because CTV is only defined for 32 byte templates and will act as NOP for different template sizes for upgradeability.

The question is, is it a good idea to increase the number of bits expected to changed in the preimage in case of stack element resizing attacks?

```c++
const HashWriter HASHER_PAIRCOMMIT{TaggedHash("PairCommit")};

namespace {
/* uint32_t hash function using primes 0x3B9ACA07 multiplier and 0x7FFFFFFF modulo
 * expected to change on average ~16 bits in output for a single bit change in input */
__inline uint32_t uint32_t_hash_x3B9ACA07(const uint32_t& i)
{
    static const uint64_t p = 0x3B9ACA07;
    static const uint32_t m = 0x7FFFFFFF;

    return (p * i) % m;
}
}

/* PairCommitHash preimage is expected to change over 32 bits on average in case of
 * length redistribution between the two input vectors */
uint256 PairCommitHash(const std::vector<unsigned char>& x1, const std::vector<unsigned char>& x2)
{
    const uint32_t x1_size = x1.size();
    const uint32_t x2_size = x2.size();
    const uint32_t x1_sh = uint32_t_hash_x3B9ACA07(x1_size);
    const uint32_t x2_sh = uint32_t_hash_x3B9ACA07(x2_size);

    HashWriter ss{HASHER_PAIRCOMMIT};
    ss << x1
       << x2
       << x1_size
       << x1_sh
       << x2_size
       << x2_sh;

    return ss.GetSHA256();
}
```
Link: https://github.com/lnhance/bitcoin/pull/7/files

-------------------------

moonsettler | 2024-10-25 14:38:37 UTC | #2

From the prelimiary spec, where PC fits in LN-Symmetry:

### Vector Commitments

`OP_PAIRCOMMIT` can be used to commit to a vector of stack elements in a way that
is not vulnerable to various forms of witness malleability especially when used
in conjunction with `OP_CHECKSIGFROMSTACK` and `OP_INTERNALKEY`,
since SHA256 implicitly commits to size of the stack elements, making the script
cleaner, and simpler. If `OP_CAT` was used naively, the contract could be easily
broken since `OP_CHECKTEMPLATEVERIFY`is only defined for 32 byte parameters.

```text
# S = 500000000
# IK -> A+B
<sig> <state-n-recovery-data> <state-n-hash> | CTV PC IK CSFSV <S+1> CLTV
```
before funding sign first state template:
```text
# state-n-hash { nLockTime(S+n), out(contract, amount(A)+amount(B)) }
# settlement-n-hash { nSequence(2w), out(A, amount(A)), out(B, amount(B)) }
# state-n-recovery-data { settlement-n-hash or state-n-balance }

# contract for state n < m
IF
  <sig> <state-m-recovery-data> <state-m-hash> | CTV PC IK CSFSV <S+n+1> CLTV
ELSE
  <settlement-n-hash> CTV
ENDIF
```

-------------------------

1440000bytes | 2024-10-25 17:57:14 UTC | #3

I am sorry but these new opcodes added to proposals will only delay or cancel them forever. I understand you like 2 things combined on the stack, OP_CAT etc. but this brings a lot of complexity in something that needs to be kept simple.

If we ever get OP_CAT, maybe this could be possible with OP_SHA256 and OP_CAT at that moment.

Its better if we get OP_CTV activated first instead of trying to build the next Great Covenant Proposal.

-------------------------

moonsettler | 2024-10-25 19:06:11 UTC | #4

It would be possible with OP_CAT to closely emulate OP_PAIRCOMMIT, they are more complicated and or more expensive. We have redundancies in bitcoin script for much smaller optimizations.

```text
<x1> <x2>
OP_SWAP
OP_SIZE
<0x00000001>
OP_ADD
OP_TOALTSTACK
OP_SWAP
OP_SIZE
<0x00000001>
OP_ADD
OP_TOALTSTACK
OP_CAT
OP_FROMALTSTACK
OP_CAT
OP_FROMALTSTACK
OP_CAT
OP_HASH256
```
or simply
```text
<x1> <x2>
OP_HASH256
OP_SWAP
OP_HASH256
OP_SWAP
OP_HASH256
```

-------------------------

moonsettler | 2024-10-25 19:11:15 UTC | #5

But the question was really not about this, but rather if it makes any sense to do a mini-hashing of the sizes instead of static padding, for making the preimage more mutable when you redistribute bytes between the stack elements?

The "original" variant:
```c++
const HashWriter HASHER_PAIRCOMMIT{TaggedHash("PairCommit")};

uint256 PairCommitHash(const std::vector<unsigned char>& x1, const std::vector<unsigned char>& x2)
{
    // PAD is 0x00000001 in little endian serializaton
    const uint32_t PAD = 0x01000000;

    HashWriter ss{HASHER_PAIRCOMMIT};
    ss << x1
       << x2
       << uint32_t(x1.size()) << PAD
       << uint32_t(x2.size()) << PAD;

    return ss.GetSHA256();
}
```

-------------------------

moonsettler | 2024-10-25 21:50:45 UTC | #7

It's okay, but I'm looking for opinions on the way these pieces are hashed, specifically from a security and efficiency pov.

-------------------------

ajtowns | 2024-10-28 11:16:51 UTC | #8

[quote="moonsettler, post:1, topic:1216"]
The idea is that the number of SHA256 iterations should be minimized in the most likely use case we can optimize for, which is LN-Symmetry.
[/quote]

Not really seeing that as particularly important to minimise, personally; the difference between hashing 64 bytes and 256 bytes is pretty minor, compared to the checksig operation(s) you also have. I'd just write `[BAL] [CTVHASH] SHA256 SWAP SHA256 CAT SHA256`.

[quote="moonsettler, post:1, topic:1216"]
In case of a 7 byte balance commitment + 32 byte CTV hash (no HTLCs in-flight), the total preimage size is 55 bytes. Which should make it fit into a single block with the SHA256 length commitment.
[/quote]

[quote="moonsettler, post:1, topic:1216"]
It’s a particular concern for LN-Symmetry with CTV that the concatenation of the two preimages allows for length redistribution attacks, because CTV is only defined for 32 byte templates and will act as NOP for different template sizes for upgradeability.
[/quote]

This seems as much an argument against doing upgradeability that way as anything else; but if you do want that, and want to minimise hashing as well for whatever reason, then writing `[BAL] [CTVHASH] SIZE 32 EQUALVERIFY CAT SHA256` seems like it would solve the problem.

If you want to support CTV upgradeability prior to knowing what that upgradeability will do, you could do `[BAL] [CTVHASH] SIZE DUP VERIFY DUP 127 LESSTHANOREQUAL VERIFY SWAP CAT SWAP CAT SHA256` to construct `sha256(1B: <size(CTVHASH)>; 0B-127B: <CTVHASH>; <BAL>)`. (Having CTV error if the hash is 0 bytes would let you avoid the `DUP VERIFY` step and might be a reasonable update to the BIP)

-------------------------

moonsettler | 2024-10-28 12:05:19 UTC | #9

[quote="ajtowns, post:8, topic:1216"]
Having CTV error if the hash is 0 bytes would let you avoid the `DUP VERIFY` step and might be a reasonable update to the BIP
[/quote]

Yes, don't think there is a legitimate reason to have a 0 size argument for CTV. Thank you for the suggestion!

Btw the PC code on that branch is broken, this is in good shape now:
https://github.com/lnhance/bitcoin/pull/6

PS: not gonna lie, the way `valtype` is serialized to `HashWriter` in bitcoin caught me off-guard.

-------------------------

moonsettler | 2024-10-29 09:38:38 UTC | #10

@ajtowns could you please delete or archive this topic? I want a redo with less confusing information for a PAIRCOMMIT thread.

The code is out of draft and it's not this variant, also updated the spec, which I should start the topic with.

-------------------------

ajtowns | 2024-10-29 10:52:30 UTC | #11

Don't want to delete threads/replies unless they're spam/non-constructive; probably not going to archive things in general. If you want to open a new topic, that's fine. I think it's best to approach this as a conversation/presentation rather than a new edition of a book -- ie, "am I saying something new that's interesting and worthwhile", rather than "whoops, I made a few errors, and would like to update a section, better republish the whole thing". If you want somewhere so you can say "here's the current comprehensive doc" better to publish things as a BIP/BINANA, a gist, or in the docs/ directory on your repo, I think?

-------------------------

moonsettler | 2024-10-29 11:36:58 UTC | #12

Alright, here is the current state of OP_PAIRCOMMIT spec, making a PR against BINANA was actually the original plan once I'm done.
https://gist.github.com/moonsettler/d7f1fb88e3e54ee7ecb6d69ff126433b

-------------------------

moonsettler | 2024-11-23 15:47:36 UTC | #13

To try to keep the [BIP](https://gist.github.com/moonsettler/d7f1fb88e3e54ee7ecb6d69ff126433b) as short and to the point as possible, an in depth discussion of the design rationale especially regarding alternatives was omitted. I'm posting these here, so that they are available for those interested.

## Background

LNhance at it's core is `CTV` + `CSFS` with the primary intent of not only enabling more scalable less interactive timeout tree and covenant pool constructions, but also to enable LN-Symmetry (formerly known as eltoo). `IKEY` was added to access the internal public key from the control block (which would be a 2-of-2 MuSig key in case of a lightning channel allowing for cooperative closes on the taproot keypath) making the contract more efficient.

As I recall, later on @instagibbs discovered in an attempt to implement Symmetry, that one of the main benefits of not having to keep all backups is lost when the channel peer can not reconstruct the script that spends an intermediate state pushed on chain. While most pieces are deterministic, the specific distribution of funds for that particular state is not. We call this the "data availability problem" of LN-Symmetry. For APO the alternative solution discussed was to use the taproot annex.

## Rationale

If `OP_CAT` was available, it could be used to combine multiple stack elements,
that get verified with `OP_CHECKSIGFROMSTACK` as a valid state update.

`OP_PAIRCOMMIT` solves this specific problem without introducing a wide range
of potentially controversial new behaviors, such as novel 2-way peg mechanisms.

### Alternatives discussed

#### OP_CAT

`OP_CAT` allows for fine grained introspection possibly bigint operations and
extending the arithmetic capabilities of bitcoin script using lookup tables.

#### SHA256 streaming opcodes

These would predictably allow for the same functionality as `OP_CAT` for
introspection purposes, since verification of a computation is largely
equivalent with carrying it out. Bigint and new arithmetic operations would
be hard or even impossible.

Naively implemented they relax the script limitations on what is possible both the limitation of stack element size that can get hashed with CAT only and without CAT it allows for custom construction of 'sighashes' like CTV templates or with CSFS pretty much everything CAT enables in terms of introspection.

#### Merkle operation opcodes

These would be of very limited general use and hard to rationalize without OP_CAT. Their complexity and resource cost is hard to justified for vector
commitments only.

#### 'Kitty' CAT (result or inputs limited in size)

The original idea would have limited the maximum size of `OP_CAT` output to a
size that is smaller than the smallest sighash preimage, thus disabling the
introspection capabilities and trivial ways to extend the arithmetic repertoire
of bitcoin script. This turned out to be an awkward, arbitrary and offering
weak .

#### OP_CHECKTEMPLATEVERIFY committing to the taproot annex in tapscript

A CTV template can be considered a sighash, however relaxing the relay policy
to take advantage of this change would make various endogenous asset protocols
more efficient, and therefore be controversial. There is also no consensus on
how to use or how to structure the annex.

#### OP_CHECKSIGFROMSTACK on n elements as message

This was previously discussed and also implemented, it complicates the code
and is a pretty arbitrary coupling of behaviors.

#### OP_VECTORCOMMIT

The obvious generalized solution for committing to n > 2 stack elements, however
it involves looping and hard to argue about setting the proper limits to it. It could be forked in later with for example `OP_CHECKCONTRACTVERIFY`.

It's impossible to predict what would be more optimal for the user a) leave the items on the stack and only consume the number of items, or b) consume the items from the stack.

We could probably do `<vch1> .. <vchn> <n> VECTORCOMMIT` where n as a signed char can be `2..127` or `-2..-127`, and if it's positive the stack elements are left on stack, if negative they are consumed.

### Possible future improvements

LNhance + `OP_CHECKCONTRACTVERIFY` (aka `CCV` the centerpiece of MATT by @salvatoshi) or `OP_VAULT/RECOVER` (aka BIP-345 by @jamesob) would enable good vaults with flexible amount withdrawal and immediate re-vault of change. The both assume `OP_CHECKTEMPLATEVERIFY` as an available building block, and `OP_CHECKCONTRACTVERIFY` especially benefits from `OP_PAIRCOMMIT` as a means to carry multiple stack elements.

-------------------------

