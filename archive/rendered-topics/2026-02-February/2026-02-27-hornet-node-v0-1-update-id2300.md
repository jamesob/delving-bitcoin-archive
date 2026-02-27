# Hornet Node v0.1 Update

tobysharp | 2026-02-27 06:53:09 UTC | #1

# Hornet Node

Bitcoin's consensus rules have never existed as a standalone specification -- they are defined implicitly by the reference client. Hornet Node is an attempt to change that: a minimal, ground-up Bitcoin client built around a declarative spec. As a side effect of the novel architecture, it currently validates mainnet 10x faster than Core v30 (`-assumevalid` on 32-core workstation).

I published a paper last September [1] introducing Hornet and posted it to the bitcoin-dev mailing list. In case you missed it, I'm recapping here with code snippets and the latest developments.

## Part I: Hornet Node

Hornet Node is a **specification-first** minimal alternative Bitcoin client built from scratch for clarity, correctness, and performance. 

### **What problem is Hornet aiming to solve?**

Bitcoin is defined by its consensus rules, yet those rules have never existed as a clear, independent specification. Hornet begins from the premise that the protocol deserves a precise, executable foundation separate from any particular codebase, so that Bitcoin's software can evolve and diversify safely even as its rules ossify.

### **The goals of Hornet Node are:**
* Show that Bitcoin consensus can be expressed as a clear, concise, executable specification without side effects.
* Provide a correct, auditable, and compact implementation with a path toward formal verification.
* Offer a working alternative client for experimentation, education, and client diversification.
* Demonstrate how clean architecture can also enable significant performance improvements.

### **Current status**

Currently Hornet Node can validate mainnet in 15 minutes using `-assumevalid` on a 32-core workstation.

![sync|690x458, 100%](upload://mbwtfTFCNqxH1ADeO2KgRrAgyIS.jpeg)
> *Hornet Node performing IBD and displaying status info in a browser window.*

## Part II: Hornet's Declarative C++ Consensus Spec

Validation rulesets are expressed as a hierarchy of composable, declarative, pure, free functions over immutable inputs, returning a single error code. The declarative focus means that we emphasize what must be true rather than how to compute it. The pure functions mean that there are no side effects or other mutated state.

At the highest level, the consensus library API exposes the `ValidateBlock` function to determine whether an incoming block is valid to be appended to a specific parent block in the timechain.

```cpp
// Each block MUST obey all consensus rules.
[[nodiscard]] inline Result ValidateBlock(const protocol::Block& block,       // The block to validate.
                                        const protocol::BlockHeader& parent,  // The preceding block's header.
                                        const HeaderAncestryView& view,       // A view onto the chain of previous headers.
                                        const int64_t current_time,           // The current time in milliseconds since 1970-01-01T00:00:00Z.
                                        const UnspentOutputsView& unspent) {  // A view onto the set of all unspent transaction outputs.
  // clang-format off
  static const auto ruleset = std::make_tuple(
    Rule{ValidateHeader,          MakeHeaderContext},         // Each block MUST begin with a valid header.
    Rule{ValidateNonSpending,     MakeEnvironmentContext},    // Each block MUST obey structural and contextual consensus rules.
    Rule{ValidateSpending,        MakeBlockSpendingContext}   // Each transaction input MUST be spendable in the current block.
  );
  // clang-format on                                            
  const BlockValidationContext context{block, parent, view, current_time, unspent};
  return ValidateRules(ruleset, view.Length(), context);
}
```

The lower-level validation functions like `ValidateHeader` are implemented themselves as a set of specification rules, each accompanied by a clear English statement of the rule being enforced.

```cpp
// Performs header validation, aligned with Core's CheckBlockHeader and ContextualCheckBlockHeader.
[[nodiscard]] inline Result ValidateHeader(const HeaderValidationContext& context) {
  // clang-format off
  static const auto ruleset = std::make_tuple(
    Rule{ValidatePreviousHash},         // A header MUST reference the hash of its valid parent.
    Rule{ValidateProofOfWork},          // A header MUST satisfy the chain's target proof-of-work.
    Rule{ValidateDifficultyAdjustment}, // A header's proof-of-work target MUST satisfy the difficulty adjustment formula.
    Rule{ValidateMedianTimePast},       // A header timestamp MUST be strictly greater than the median of its 11 ancestors' timestamps.
    Rule{ValidateTimestampCurrent},     // A header timestamp MUST be less than or equal to network-adjusted time plus 2 hours.
    Rule{ValidateVersion}               // A header version number MUST meet deployment requirements depending on activated BIPs.
  );
  // clang-format on
  return ValidateRules(ruleset, 0, context);
}
```

At the lowest level, validation rules each return either success, or a specific error code.

```cpp
// A header MUST reference the hash of its valid parent.
[[nodiscard]] inline Result ValidatePreviousHash(
    const HeaderValidationContext& context) {
  if (context.parent.ComputeHash() != context.header.GetPreviousBlockHash())
    return Error::Header_ParentNotFound;
  return {};
}

// A header's 256-bit hash value MUST NOT exceed the header's proof-of-work target.
[[nodiscard]] inline Result ValidateProofOfWork(
    const HeaderValidationContext& context) {
  const auto hash = context.header.ComputeHash();
  const auto target = context.header.GetCompactTarget().Expand();
  if (!(hash <= target)) return Error::Header_InvalidProofOfWork;
  return {};
}
```

The entire executable consensus spec occupies ~800 lines of code.

## Part III: The Hornet Validation Pipeline and UTXO Database

In a previous post [3] I wrote about the Hornet UTXO(1) lock-free database that I wrote to enable a highly parallel validation engine that can process many blocks concurrently during IBD. (Also discussed on Optech Podcast #391 [4].)

This is how I was able to get the IBD validation time down to 15 minutes on my local workstation, compared to 167 minutes with Core v30. The stateless nature of consensus validation is also an important aspect here. Some more detailed but rough notes on my custom database can be found here pending a full write-up [5].

Although consensus logic requires the concept of querying a set of unspent transaction outputs, the database mechanics are not exposed to the consensus layer, so that concerns remain separated. Instead, the consensus layer defines a pure interface that allows either to query for existence of unspent outputs (e.g. `-assumevalid`), or to actually retrieve the stored state for those outputs.

```cpp
// This class represents an abstract view onto the whole set of unspent outputs.
class UnspentOutputsView {
 public:
  virtual ~UnspentOutputsView() = default;

  // Returns success if all the block's transaction inputs correspond to pre-existing unspent transaction outputs.
  // Otherwise, returns Error::Transaction_NotUnspent.
  virtual Result QueryPrevoutsUnspent(const protocol::Block& block) const = 0;

  // Calls fn for each transaction input, passing in the stored state from the corresponding previous output.
  // Or, if any transaction input does not correspond to an unspent output, returns Error::Transaction_NotUnspent.
  template <typename Fn> Result ForEachSpend(const protocol::Block& block, Fn&& fn) const;
};
```

The rule to validate the spendability of a block's transactions calls this interface to validate scripts for each input:

```cpp
// Each transaction input MUST be spendable in the current block.
[[nodiscard]] inline Result ValidateSpending(const BlockSpendingContext& context) {
  return context.unspent.ForEachSpend(context.block,
    [&](const SpendRecord& spend) { 
      return ValidateInputSpend(spend, context.height);
    });
}
```

where the `SpendRecord` includes the pubkey script and other details from the funding transaction.

```cpp
struct SpendRecord {
  int funding_height;
  uint32_t funding_flags;
  int64_t amount;
  std::span<const uint8_t> pubkey_script;
  protocol::TransactionConstView tx;
  int spend_input_index;

  bool IsCoinbase() const { return funding_flags & 1; }
};
```

This keeps the consensus layer nicely compact and separate from all of the implementation choices of the UTXO database.

## Part IV: The Hornet DSL

Beyond the C++ declarative specification, the goal of Hornet is to go further and express the consensus rules in a pure functional language. Such a restricted language can be designed to facilitate formal verification of a client to the spec. An interpreter or compiler for such a spec could in theory be formally proven to implement the specification exactly. This would be the holy grail of consensus that allows for client diversity without risk of consensus split due to coding bugs.

This part of Hornet is future work. However, my early drafts for the specification DSL are experimenting with a precise, mathematical style. The language has the following properties:

- Pure functional style. Every rule is a function from inputs to a result with no hidden dependencies or side effects. Rules compose the way mathematical propositions do. All variables are immutable.
- Mathematical notation. Quantifiers (∀, ∃), set membership (∈), concatenation (⧺), and summation (Σ) express functional operators succinctly. The non-ASCII characters are a deliberate choice for a spec that will be read and audited far more than it is edited.
- Native serialization. Protocol types like `Block`, `Transaction`, and `Header` are defined as plain structs with built-in serialization semantics, so the spec defines the wire format and the validation logic in the same language.
- Built-in cryptographic primitives. Functions like SHA256 and secp256k1 verification are treated as built-in axioms, defined in other specification documents.

```f#
// Returns the Merkle root of a sequence of hashes, defined recursively.
Let MerkleRoot : (xs ∈ Hash⟨⟩, unique ∈ bool := true) -> (root ∈ Hash, unique ∈ bool)
    // Make a tuple from each consecutive pair in the sequence, ignoring any odd tail.
    Let pairs :=
        ⟨ (xs[2i], xs[2i + 1]) : i ∈ ⟦0, |xs| / 2⟯ ⟩
    // Evaluate whether uniqueness is preserved among all such pairs.
    Let unique_at_level := ∀ (a, b) ∈ pairs, a ≠ b
    // Duplicate an odd tail for completeness.
    Let extended :=
    ⎧   pairs                         if |xs| even,
    ⎩   pairs ⧺ (xs.last, xs.last)    otherwise

|-> ⎧   (xs.first, unique)  if |xs| = 1,  // Returns the root hash and the accumulated uniqueness.
    ⎨   MerkleRoot(⟨SHA256²(a ⧺ b) : (a, b) ∈ extended⟩, unique ∧ unique_at_level)
    ⎩                       otherwise     // Recurses into the next level of the tree.
```

In the `MerkleRoot` function above, we define the Bitcoin-specific logic for grouping pairs of leaves and repeating the last leaf in an odd cardinality. Notice how the functional style specifies the required logic and constructions, rather than what machine instructions should be used to compute the result.

The resulting code (9 lines plus comments) is remarkably concise without losing any precision. The style will be most legible to readers with familiarity in mathematics.

Below, the `Rule` keyword denotes a function with one or more `Require` statements: these evaluate a Boolean expression, and if false, a named error bubbles up the stack to fail validation.

```f#
// The total number of signature operations in a block MUST NOT exceed the consensus maximum.
Rule SigOpLimit(block ∈ Block)
    Let SigOpCost : (op ∈ OpCode) -> int32 |-> 
      ⎧  1  if op ∈ {Op_CheckSig,      Op_CheckSigVerify     },
      ⎨ 20  if op ∈ {Op_CheckMultiSig, Op_CheckMultiSigVerify},
      ⎩  0  otherwise
    Require Σ SigOpCost(inst.opcode)
            ∀ inst ∈ script.instructions
            ∀ script ∈ tx.inputs.scriptSig ⧺ tx.outputs.scriptPubKey
            ∀ tx ∈ block.transactions
        ≤ 20,000
```

In `SigOpLimit` here, we use projections, concatenation, and a local function to express the legacy sigop cost extremely compactly.

Below, these functions and some others not shown here are combined for the structural block validation:

```f#
Rule ValidateBlockStructure(block ∈ Block)
    // A block MUST contain at least one transaction.
    Require NonEmptyBlock:      block.transactions ≠ ∅

    // A block’s Merkle root field MUST equal the Merkle root of its transaction list.
    Require ValidateMerkleRoot(block)
    
    // A block’s serialized size (before SegWit) MUST NOT exceed 1,000,000 bytes.
    Require OriginalSizeLimit:  |SerializeNoWitness(block)| <= 1,000,000
    
    // A block MUST contain exactly one coinbase transaction, and it MUST be the first transaction.
    Require UniqueCoinbase:     ∃! tx ∈ block.transactions : IsCoinbase(tx)
    Require CoinbaseFirst:      IsCoinbase(block.transactions.first)

    // Every transaction in a block MUST be valid according to transaction-level consensus rules.
    Require ∀ tx ∈ block.transactions, ValidateTransaction(tx)

    // The total number of signature operations in a block MUST NOT exceed the consensus maximum.
    Require SigOpLimit(block)
```

The Hornet DSL specification will also be declarative and executable. But unlike the C++ specification, it will be a pure, formal specification with a path to formal reasoning and verification. It will also be legible to engineers, mathematicians, and all programmers, rather than only to C++ developers.


## Summary

Hornet Node is a new, compact, alternative Bitcoin client built around a pure, declarative specification of consensus rules in modern C++. This stage of development is almost complete, with IBD running 10x the speed of Core v30. A future phase will be to migrate the specification to a pure, functional domain-specific language custom-designed to formally express the consensus rules in the clearest and most precise form.

If you have questions or feedback, feel free to drop me a reply.

I also wrote an overview with more details and some FAQ [2].

Hornet is a self-funded project. If you'd like to support my work on a formal specification for Bitcoin consensus, you can donate to `33Y6TCLKvgjgk69CEAmDPijgXeTaXp8hYd`, or get in touch.

Thanks in advance for any interest, encouragement, and support.

Best wishes,\
T#

[1] Hornet Node and the Hornet DSL: A minimal, executable specification for Bitcoin consensus by Toby Sharp, September 2025. [arXiv PDF](https://arxiv.org/abs/2509.15754)

[2] hornetnode dot org / overview.html

[3] https://delvingbitcoin.org/t/hornet-utxo-1-a-custom-constant-time-highly-parallel-utxo-database/2201

[4] https://bitcoinops.org/en/podcast/2026/02/10/

[5] hornetnode dot org / utxo.html

-------------------------

