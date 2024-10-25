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

