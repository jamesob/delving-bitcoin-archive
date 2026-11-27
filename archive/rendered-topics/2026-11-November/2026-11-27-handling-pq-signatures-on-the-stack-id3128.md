# Handling PQ signatures on the stack

conduition | 2026-11-27 00:01:03 UTC | #1

A few weeks ago i opened this PR, which would be the first step towards optimizing core to handle large stack elements: https://github.com/bitcoin/bitcoin/pull/36721

The primary motivation for me is for PQ signatures like SHRINCS. There is no PQ signature scheme available today (except isogenies) whose signatures would fit inside the current 520 byte stack element size limit. 

This thread is meant to broaden the discussion. The general question is: 

# How do we safely handle large stack elements?

## Background

The SHRINCS working group's secondary project, in collaboration with P2MR devs, is [a BIP that adds a new tapscript version that redefines the CHECKSIG opcodes, and raises the stack element size limit to 6kb](https://github.com/SHRINCS/shrincs-bip/pull/76) to accomodate PQ signatures. This BIP contains rationale for why we feel raising the stack element size limit is the right way to go. (Warning, the BIP is currently being rewritten.)

Further to that, we also have an open github issue for discussion of the subject. https://github.com/SHRINCS/shrincs-bip/issues/3 - I would ask that readers avoid chiming in on that issue unless they would specifically like to help us draft the CHECKSIGMUX BIP linked above. Let's keep general design discussion here.

Increasing the stack element size limit presents DoS risks to bitcoin nodes. Scripts can duplicate and hash large elements, causing much worse CPU cycle consumption than is possible today with the 520 byte limit. `OP_RIPEMD160` is especially bad for this, as it lacks hardware acceleration. Other opcodes like `OP_VERIFY` and `OP_IFDUP` and `OP_EQUAL` scan stack elements byte-by-byte, which can be surprisingly slow if run repeatedly (not as bad as hashing though).

Now I'll survey the ways that I know of to handle large stack elements. I'm going to assume we are deploying a new tapscript version that supports PQ sigs, but most of the ideas apply equally well if we deployed PQ sigs using `OP_SUCCESS`.

## Lazy Option: Chunking

Keep the existing 520 byte limit in consensus, and expect any new opcode that requires consuming more than 520 bytes to pop multiple stack elements and concatenate them. 

### Benefits 
- Would require no work on the part of devs to improve implementation performance or assess changes to existing tapscript rules.
- Nothing new to reason about in terms of consensus, or code DoS surface.

### Drawbacks

This raises some unintuitive protocol-level edgecases:

- The same object could now be encoded on the stack in myriad ways, by translating bytes between leading and trailing edges of different elements prior to concatenation.
- The order of concatenation must be standardized.
- Opcodes should not determine how many elements to pop based on untrusted input.

Most edgecases are easy to handle at the protocol-level, if a bit awkward.

The main drawbacks of chunking are at user and wallet level. 

- Parsing signatures from a transaction (e.g. a pre-signed one), or pushing signatures onto a transaction witness, requires carefully replicating consensus-level chunking logic.
- More complex scripting may need extensive stack element juggling.
- Chunking incurs a small amount of size overhead due to the additional length encoding prefixes for each chunk. Negligible compared to the size of PQ signatures.

## Strict Option: Opcode blacklisting

In the new tapscript version, we can define whatever rules we want for opcodes, and this means we can define some opcodes so that they work on "large" (>520b) stack elements, while others only accept small (<=520b) elements, and fail the script otherwise. 

This begs the question: Which opcodes are blacklisted? How do we decide?

Obviously signature-checking opcodes would need to be whitelisted to work on large elements. 

Duplication opcodes like `OP_DUP` should probably reject large inputs, or at the very least have some sort of budget which prevents repeated duplication. This seems like an easy win: It removes an entire class of attack vector (DUP-and-hash, DUP-and-scan, etc). 

Some opcodes (hashing, scanning, etc) may still be expensive when run on large elements, but notice that there is no longer any way for a script to duplicate a large element. It is impossible to run these opcodes on the same element more than once, unless the transaction explicitly pushes the element in the witness again (expensive bandwidth for the attacker). So i would reason that we could probably leave these opcodes as-is, provided that duplication (or mutation) is tightly guarded.

We need to be careful about opcodes which don't discard the inputs, but whose runtime cost can still be linear in the byte-size of their inputs. E.g. `OP_ROLL`, or `OP_TOALTSTACK`, due to `memcpy` calls. These can be easily mitigated by representing stack elements as pointers (shameless plug for [my PR](https://github.com/bitcoin/bitcoin/pull/36721)). 

### Benefits

- Clean implementation profile, on the protocol and wallet side.
- Resource limits are enforced by consensus.

### Drawbacks

- Removes features, by disabling DUP for large elements. It's not clear if duplicating signatures is useful, but it could potentially hinder some forms of simple covenants if it is not possible to both hash and check a signature in the same script. 
  - Whatever utility there may be in duplicating large non-signature data is unexplored territory. 
- Requires careful reasoning about the remaining non-blacklisted opcodes to ensure no other DoS surface exists.


## Thorough Option: Optimized Implementations

Enumerate all the opcodes whose runtime cost depends on input size. Profile and sort them by most-vs-least expensive per byte, and start chipping away at the worst offenders with in-code optimizations. 

For instance:

- deploy a hash-cache to handle expensive hash opcodes.
- represent stack elements with pointers reduce the cost of duplication.
- cache an element's truthiness to handle `OP_VERIFY`
- etc etc.

### Benefits

- Exact same utility profile as today.
- Improves today's TX/block validation performance as a side-effect.
- Non-consensus. Not coupled to any BIP. Work can be started today.

### Drawbacks

- Increases node implementation burden.
- Requires careful reasoning about opcodes to survey DoS surface.


## Adventurous Option: GSR

GSR's varops budget ([BIP-440](https://github.com/bitcoin/bips/blob/master/bip-0440.mediawiki)) would nullify this problem, by introducing a proper accounting system for compute and memory costs. If a transaction fails to pay for its compute cost, the script will fail without wasting additional cycles. 

### Benefits

- Resource limits are enforced by consensus.
- Doesn't restrict scripting features (if coupled with [BIP-441](https://github.com/bitcoin/bips/blob/master/bip-0441.mediawiki), it expands them)

### Drawbacks

- Complex protocol implementation
- Slew of additional vetting and testing needed


## Middle option: Mix & Match

We could combine implementation-level fixes with new consensus rules. For instance: implement a pointer representation of the stack so that opcodes like and `OP_DUP` and `OP_ROLL` become constant cost, but blacklist other opcodes like hashing or scanning.

-------------------------

