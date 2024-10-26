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

1440000bytes | 2024-10-25 19:22:36 UTC | #6

[quote="moonsettler, post:5, topic:1216"]
But the question was really not about this, but rather if it makes any sense to do a mini-hashing of the sizes instead of static padding, for making the preimage more mutable when you redistribute bytes between the stack elements?
[/quote]

I do not like when someone disagrees, because we spent days building this and such comment.

But this is against LNHANCE. I am not a "core developer" so it might not feel the same.

-------------------------

moonsettler | 2024-10-25 21:50:45 UTC | #7

It's okay, but I'm looking for opinions on the way these pieces are hashed, specifically from a security and efficiency pov.

-------------------------

