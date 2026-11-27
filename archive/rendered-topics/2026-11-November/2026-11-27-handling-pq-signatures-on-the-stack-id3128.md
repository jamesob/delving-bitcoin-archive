# Handling PQ signatures on the stack

conduition | 2026-11-27 00:08:48 UTC | #1

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

GSR's varops budget ([BIP-440](https://github.com/bitcoin/bips/blob/9f3d3bc6a60db16efbdc71ac8f7ea30217fc0c39/bip-0440.mediawiki)) would nullify this problem, by introducing a proper accounting system for compute and memory costs. If a transaction fails to pay for its compute cost, the script will fail without wasting additional cycles. 

### Benefits

- Resource limits are enforced by consensus.
- Doesn't restrict scripting features (if coupled with [BIP-441](https://github.com/bitcoin/bips/blob/9f3d3bc6a60db16efbdc71ac8f7ea30217fc0c39/bip-0441.mediawiki), it expands them)

### Drawbacks

- Complex protocol implementation
- Slew of additional vetting and testing needed

## Isolationist Option: Sig stack

Introduce a new stack specifically for signatures, and have CHECKSIG opcodes pop from that stack and only that stack. Keep signatures completely separate from the regular main/alt stacks. No opcodes can act on signatures except those which explicitly pull from the sigstack. 

### Benefits

- Conceptually clean.
- Demonstrably nullifies interpreter DoS risks of large stack elements, outside the CHECKSIG opcodes and any others that touch the sigstack.

### Drawbacks

- Removes features: we can't work with signatures in script at all.
- Departure from traditional bitcoin script semantics


## Middle option: Mix & Match

We could combine implementation-level fixes with new consensus rules. For instance: implement a pointer representation of the stack so that opcodes like and `OP_DUP` and `OP_ROLL` become constant cost, but blacklist other opcodes like hashing or scanning.

-------------------------

ajtowns | 2026-11-27 04:35:39 UTC | #2

My view, fwiw:

 * :-1: Chunking -- dumb; the tapscript limits are upgradable precisely so that new consensus changes don't have to work around arbitrary restrictions.
 * :person_gesturing_no:Opcode blacklisting -- defeats the purpose: if we're trying to have big signature data on the regular stack to be able to manipulate it in interesting ways, then this approach would blacklist some of those interesting ways
 * :smiling_face_with_horns: Optimized implementations -- this seems the opposite of "thorough" to me: you have to guess at how things might be used and work out clever optimisations that cover those use cases, and if you forget some things you've opened up a potential vulnerability that real spends/users might already rely on that you can't close. Worse, optimising is good on its own merits, and full of fun puzzles, so this approach can be dangerously seductive.
 *  :1st_place_medal: GSR's varops budget -- this seems obviously the right approach to me if you want scripts to be able to do remotely clever things with non-trivial amounts of data
 *  :1st_place_medal: Isolated sig stack -- this seems the simplest approach if you're willing to throw away the idea of scripts doing clever things with signature data (or defer it until GSR/similar is implemented on some separate track). It has the additional potential benefit that an isolated stack could get a greater discount (as [discussed here](https://delvingbitcoin.org/t/segwit-commitment-to-post-quantum-witness-data/2702)).

(Someone should tell the admins not to let me use emojis)

-------------------------

conduition | 2026-11-27 15:19:04 UTC | #3

[quote="ajtowns, post:2, topic:3128"]
* :person_gesturing_no:Opcode blacklisting – defeats the purpose: if we’re trying to have big signature data on the regular stack to be able to manipulate it in interesting ways, then this approach would blacklist some of those interesting ways
[/quote]

I mean, you can still do interesting things, e.g. OP_SIZE on SHRINCS sigs would let you restrict signatures to stateful or stateless paths. Just not duplication.

[quote="ajtowns, post:2, topic:3128"]
* :1st_place_medal: Isolated sig stack – this seems the simplest approach if you’re willing to throw away the idea of scripts doing clever things with signature data (or defer it until GSR/similar is implemented on some separate track). It has the additional potential benefit that an isolated stack could get a greater discount (as [discussed here](https://delvingbitcoin.org/t/segwit-commitment-to-post-quantum-witness-data/2702)).
[/quote]

I had assumed that new witness discounts would probably not be deployed alongside SHRINCS, but if you're considering that, then we should probably do something a little more thorough. One possible idea here, similar to @sipa's witness styles idea

https://gist.github.com/conduition/eafd067cd1df553fc6c5da8a05290dc9

-------------------------

