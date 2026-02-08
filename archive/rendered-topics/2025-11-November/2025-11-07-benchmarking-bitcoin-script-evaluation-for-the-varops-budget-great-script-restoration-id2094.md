# Benchmarking Bitcoin Script Evaluation for the Varops Budget (Great Script Restoration)

Julian | 2025-11-10 20:59:13 UTC | #1

Hello everyone interested in Great Script Restoration and the Varops Budget,

The main concerns that led to the disabling of many opcodes in v0.3.1 were denial-of-service attacks through excessive computational time and memory usage in Bitcoin script execution. To mitigate these risks, we propose to generalize the sigops budget in a new Tapscript leaf version and apply it to all operations before attempting to restore any computationally expensive operations or lifting any other script limits.

Similar to the sigops budget (which is applied to each input individually), the varops budget is based on transaction weight, a larger transaction has proportionally more compute units available. Currently, the budget is set to 5,200 units per weight unit of the transaction.

To validate that this approach is working and that the free parameters are reasonable, we need to understand how it constrains script execution and what the worst-case scripts are.

=== Benchmark Methodology ===

For simplicity, we benchmark the script evaluation of block sized scripts with the goal of finding the slowest possible script to validate. This block sized script is limited by:

\- Size: 4M weight units

\- Varops budget: 20.8B compute units (4M × 5,200)

To construct and execute such a large script, it must be looped until one of the two limits is exhausted. For example, a loop of OP_DUP OP_DROP would take an initial stack element and benchmark the copying and dropping repeatedly until either the maximum size or the varops budget is reached. Computationally intensive operations like arithmetic or hashing on large numbers are generally bound by the varops budget, while faster operations like stack manipulation or arithmetic on small numbers are bound by the block size limit.

For simple operations like hashing (1 in → 1 out), we create a loop like:

```
OP_SHA256 OP_DROP OP_DUP (repeated)
```

Other operations have different restoration patterns. For bit operations (2 in → 1 out):

```
OP_DUP OP_AND OP_DROP OP_DUP (repeated)
```

These scripts act on initial stack elements of various sizes. The initial elements are placed onto the stack “for free” for simplicity and to make the budget more conservative. In reality, these elements would need to be pushed onto the stack first, consuming additional space and varops budget.

=== Baseline: Signature Validation ===

Currently, the theoretical limit for sigops in one block is:

```
4M weight units / 50 weight units per sig = 80,000 signature checks per block
```

Using nanobench, we measure how long it takes to execute pubkey.VerifySchnorr(sighash, sig) 80,000 times. On a modern CPU, this takes between one and two seconds.

If we want the varops budget to limit script execution time to be no slower than the worst case signature validation time, we need to collect benchmarks from various machines and architectures. This is especially important for hashing operations, where computational time does not scale linearly and depends on the implementation, which varies between chips and architectures.

The varops cost of each opcode depends on the length of its arguments and how it acts on the data; whether it copies, compares, moves, performs hashing, or does arithmetic. More details can be found in the BIP: https://github.com/rustyrussell/bips/blob/guilt/varops/bip-unknown-varops-budget.mediawiki

=== How to Help ===

To collect more data, we would like to run benchmarks on various machines. You can run the benchmark by:

1\. Checking out the GSR prototype implementation branch:

https://github.com/jmoik/bitcoin/tree/gsr

2\. Compiling with benchmarks enabled (-DBUILD_BENCH=ON)

3\. Running the benchmark:

./build/bin/bench_varops --file bench_varops_data.csv

This will store the results in a csv and predict a maximum value for the varops budget specifically for your machine depending on your Schnorr checksig times and the slowest varops limited script. It would be very helpful if you shared your results by opening a PR [here](https://github.com/jmoik/varopsData) so we can analyze the data across different systems and verify if the budget is working well or has to be adjusted!

Cheers

Julian

-------------------------

ajtowns | 2025-11-10 09:02:45 UTC | #2

[quote="Julian, post:1, topic:2094"]
2. Compiling with benchmarks enabled (-DBUILD_BENCH=ON)
[/quote]

Probably want to specify the build options more completely if you want benchmark results to be comparable? (`CMAKE_BUILD_TYPE` as something other than `Debug` in particular)

(Also `#ifdef DEBUG` and `#ifdef USE_GMP` instead of `#if` in various val64 code)

[quote="Julian, post:1, topic:2094"]
It would be very helpful if you shared your results so we can analyze the data across different systems
[/quote]

Probably should provide a git repo to add the CSV's to via PR?

I'm seeing errors, which look like they lead to useless data, fwiw:

```txt
Script error: OP_DIV or OP_MOD by zero
653/875: MOD_DUP_100Bx2                 0.000 seconds (     0 Schnorrs,    0.0% varops used)
```

May want to fix those first?

[quote="Julian, post:1, topic:2094"]
If we want the varops budget to limit script execution time to be ...
[/quote]

Capping at 100% varops budget seems to make the data less useful in evaluating whether the varops ratios are accurate, than if no capping was in place? eg these XOR tests:

```txt
511/875: DUP_XOR_DROP_DUP_10KBx2        0.514 seconds ( 12714 Schnorrs,  100.0% varops used)
509/875: DUP_XOR_DROP_DUP_100KBx2       0.446 seconds ( 11051 Schnorrs,  100.0% varops used)
514/875: DUP_XOR_DROP_DUP_1MBx2         0.477 seconds ( 11807 Schnorrs,  100.0% varops used)
516/875: DUP_XOR_DROP_DUP_2MBx2         0.475 seconds ( 11769 Schnorrs,  100.0% varops used) 
```

don't seem to give any useful information about relative performance at different input sizes, because in each case the varops limit is hit?

I would have expected to be able to use the data generated either to produce per-hardware calculations of the constant-and-per-byte factors for each opcode (and thus review the hardcoded varops factors) or alternatively to be able to judge what would cause the worst case blocks for my hardware (particularly for old hardware) -- eg, "a block full of schnorr checks will take 5s to validate, but a block of X hash256 ops with Y bytes of data each will take 10s to validate".

Looking at <100% varops hashing results, I think I can calculate:

 * ripemd: constant = 0.322, per byte = 0.00251
 * sha256: constant = 0.362, per byte = 0.00306
 * hash160: constant = 0.549, per byte = 0.00299

Or normalized from "seconds per block over-filled with repeating script" to "schnorr ops per op", perhaps:

 * ripemd: constant = 0.006 schnorr ops per op, plus 0.000047 schnorr ops per byte
 * sha256: constant = 0.007 plus 0.000057 per byte
 * hash160: constant = 0.010 plus 0.000056 per byte

Or normalized from 80k schnorr ops per block to 20.8B compute units per block:

 * ripemd: constant = 1554, per byte = 12.1
 * sha256: constant = 1747, per byte= 14.8
 * hash160: constant = 2650, per byte= 14.4

Those compare to current varops figures of constant=0, per byte=10, I believe. I'm surprised that ripemd seems to be faster than sha256 for me. Presumably either taking the DUP costs into account or setting up an absurdly large stack so that DUP is unnecessary would generate more accurate figures than I've done above though. FWIW DUP/DROP benchmarks seems to imply figures of 276 compute units per operation plus 0.05 compute units per byte for me (as opposed to 0 + 1 per byte).

-------------------------

Julian | 2025-11-10 20:57:20 UTC | #3

Hi aj, thanks for taking the time,

[quote="ajtowns, post:2, topic:2094"]
Probably want to specify the build options more completely if you want benchmark results to be comparable? (`CMAKE_BUILD_TYPE` as something other than `Debug` in particular)

[/quote]

Yes I have been using the default `CMAKE_BUILD_TYPE=RelWithDebInfo`, since this is probably what most use. I am not sure if the binaries on bitcoincore.org are built in release instead, anyways the performance difference between the two seems to be negligible.

[quote="ajtowns, post:2, topic:2094"]
Probably should provide a git repo to add the CSV’s to via PR?

[/quote]

That’s a great idea, I have opened a repo here and will update the post: https://github.com/jmoik/varopsData

[quote="ajtowns, post:2, topic:2094"]
I’m seeing errors, which look like they lead to useless data, fwiw:

```

```

[/quote]

The script errors are expected for some benchmarks and can be ignored since they produce a time of 0 seconds and we are interested only in the worst times, but you are correct, I will add more specific stacks/scripts to benchmark OP_DIV / OP_MOD properly.

[quote="ajtowns, post:2, topic:2094"]
Capping at 100% varops budget seems to make the data less useful in evaluating whether the varops ratios are accurate, than if no capping was in place? eg these XOR tests:

```

```

[/quote]

With 20.8 B compute units, the scripts are already fairly long, so you don’t gain much from not capping here, of course you lose the reference on how many ops were executed and larger stack element sizes might be cut of early and therefore the time / input bytes might be shifted downwards slightly.

If no capping was in place, some scripts would run extremely long.

If we want to measure performance relative to input bytes, we don’t need to call EvalScript() at all and remove all the noise from restoring the stack and the rest of the script interpreter (duplicating a large stack element is quite slow).

This benchmark is not designed to measure time / input size, I wanted to ensure that the budget is sufficient for any input size and having similar values for those XOR script tells me that the XOR cost scales appropriately.

[quote="ajtowns, post:2, topic:2094"]
I would have expected to be able to use the data generated either to produce per-hardware calculations of the constant-and-per-byte factors for each opcode (and thus review the hardcoded varops factors)

[/quote]

If we take the 5,200 compute units per weight as constant or renormalize this is interesting indeed, but there are too many free variables, the 5,200 was initially derived from the 10x hashing factor (520 bytes for a single script element x 10, such that one hashing opcode could always pay for a current max size element).

Therefore, I assumed the 10x for hashing (as well as the other costs) as constant and kept the 5,200 as the free parameter.

[quote="ajtowns, post:2, topic:2094"]
Those compare to current varops figures of constant=0, per byte=10, I believe. I’m surprised that ripemd seems to be faster than sha256 for me.

[/quote]

Yes that seems reasonable, although a bit slow, did you get those values through a linear fit of the different input sizes? I am also surprised, on my machine RIPEMD is the slowest hashing algorithm by far and the slowest script overall, the only one going above 80,000 Schnorrs. But unlike SHA, RIPEMD is still limited to 520 bytes in tapscript v2, maybe your machine is very slow with SHA on large inputs?

I don’t think we want to assign different costs to different hashing algorithms, since this is highly dependent on the implementation.

[quote="ajtowns, post:2, topic:2094"]
FWIW DUP/DROP benchmarks seems to imply figures of 276 compute units per operation plus 0.05 compute units per byte for me (as opposed to 0 + 1 per byte).

[/quote]

I have also experimented with a flat cost for each operation, since we do have interpreter overhead and acting on an empty input is obviously more expensive than zero, but it does not seem to be that important since we do have the block size limit.

[quote="ajtowns, post:2, topic:2094"]
Or normalized from “seconds per block over-filled with repeating script” to “schnorr ops per op”, perhaps:

[/quote]

We have actually worked with this normalization initially and it is helpful when there is no varops limit, but felt like it is simpler to discuss these benchmarks by using the block sized script and comparing it to the known time of 80,000 Schnorrs.

-------------------------

instagibbs | 2025-11-13 16:25:14 UTC | #4

[quote="Julian, post:1, topic:2094"]
Using nanobench, we measure how long it takes to execute pubkey.VerifySchnorr(sighash, sig) 80,000 times. On a modern CPU, this takes between one and two seconds.

[/quote]

TIL it was that slow. I worry that benchmarking against the *worst case* as the new “average” is the wrong goal as this may potentially become the average way of interacting with Bitcoin (unlike using lots of signatures, which seems rare). Basically we should expect \~every block to approach that level of verification cost, so I’d expect we would target something we would be happy with for IBD or at tip with relatively turbulent mempools. *waving wildly* On the order of 100ms?

edit: Or maybe I’m wrong and the average GSR block would be significantly faster, just like now with sigops. :person_shrugging: Might be harder to predict what will actually be deemed useful vs “more sigops” which seem to have marginal utility?

-------------------------

Julian | 2025-11-13 16:33:51 UTC | #5

Since the limit of the computational budget is fairly high I don’t expect the average block to come close to this budget at all, so fully validating the block should be way faster than the worst case, probably not quite on the order of 100ms, doing some benchmarks on real blocks would be interesting.

But also I am not sure how much we should worry about IBD times, to my understanding the default IBD in core uses assumevalid with very recent checkpoint blocks, so the average user does not even fully validate \~99% of scripts.

-------------------------

ajtowns | 2025-11-14 17:00:33 UTC | #6

[quote="instagibbs, post:4, topic:2094"]
TIL it was that slow. I worry that benchmarking against the *worst case* as the new “average” is the wrong goal
[/quote]

Signatures are also cached, meaning that if we have seen the tx recently, re-verifying the signature for the block is much faster; ~~whereas all the other logic isn't cached~~ (there is a script cache, but that is only reused mempool-to-mempool and block-to-block, eg in a reorg, ~~not mempool-to-block). So a high budget for non-signature operations likely has a worst impact on average validation time than a nominally-equivalent budget for signature operations.~~ EDIT: wrong, just wrong

Also, you can only have ~62k distinct signatures in 4MB of witness data, so using the full budget would presumably mean ~22% of your operations should/could be cache hits even if none of the data had been seen before.

(80k sigops is, of course, the figure from segwit v0/BIP141 which is 4x the original number of sigops per block. Taproot/BIP340-342 preserves that figure pretty closely, so that I think you could get about 79,988 BIP340 verify ops in a block, though it also introduces the potential for combining sig operations into a single validation to reduce computation time)

(Also, there's probably a difference between single-core cpu time and wall-clock time when multiple validatation threads are in operation, at least for average blocks (versus worst-case blocks where all the work is in a single 4MB witness script). 1000ms of cpu time, divided between 4 or 8 threads is already perhaps 150ms-300ms of wall clock time, after all)

[quote="Julian, post:5, topic:2094"]
doing some benchmarks on real blocks would be interesting
[/quote]

Doing benchmarks on the more powerful logic new opcodes enable might be more interesting; eg the [OP_CAT based zkp verifier](https://bitcoinmagazine.com/technical/a-zero-knowledge-proof-is-verified-on-bitcoin-for-the-first-time-in-history)? How many zkps (implemented that way) could we verify in a block, if 100ms (or whatever) of cpu time were the only constraint? Seems to have some 223k operations per script, 30k ADD/DUP, 20k IF/ENDIF/SUB, 10k SWAP/LESSTHAN/TOALT/FROMALT, 400 CAT/1SUB/DEPTH/SHA256, 1 CHECKSIG, etc.

-------------------------

instagibbs | 2025-11-14 15:27:00 UTC | #7

[quote="Julian, post:5, topic:2094"]
Since the limit of the computational budget is fairly high I don’t expect the average block to come close to this budget at all, so fully validating the block should be way faster than the worst case,

[/quote]

I think in that case, the budget should be lowered to whatever worst case target we would have in mind.

[quote="Julian, post:5, topic:2094"]
But also I am not sure how much we should worry about IBD times, to my understanding the default IBD in core uses assumevalid with very recent checkpoint blocks, so the average user does not even fully validate \~99% of scripts.

[/quote]

We should not justify new resource requirements based on practical hacks to get around the historical issues, imo.

[quote="ajtowns, post:6, topic:2094"]
Doing benchmarks on the more powerful logic new opcodes enable might be more interesting

[/quote]

Agreed!

-------------------------

Julian | 2025-12-05 14:43:09 UTC | #8

[quote="ajtowns, post:6, topic:2094"]
Doing benchmarks on the more powerful logic new opcodes enable might be more interesting; eg the [OP_CAT based zkp verifier](https://bitcoinmagazine.com/technical/a-zero-knowledge-proof-is-verified-on-bitcoin-for-the-first-time-in-history)? How many zkps (implemented that way) could we verify in a block, if 100ms (or whatever) of cpu time were the only constraint? Seems to have some 223k operations per script, 30k ADD/DUP, 20k IF/ENDIF/SUB, 10k SWAP/LESSTHAN/TOALT/FROMALT, 400 CAT/1SUB/DEPTH/SHA256, 1 CHECKSIG, etc.

[/quote]

Unfortunately it seems that this implementation won’t run properly due to the new number type used in GSR, we use an unsigned arbitrary length integer instead of the current signed 32bit int. The new stack limits and OP_MUL should also drastically reduce the number of transactions needed and probably also reduce their validation time.

[quote="instagibbs, post:4, topic:2094"]
TIL it was that slow. I worry that benchmarking against the *worst case* as the new “average” is the wrong goal as this may potentially become the average way of interacting with Bitcoin (unlike using lots of signatures, which seems rare). Basically we should expect \~every block to approach that level of verification cost, so I’d expect we would target something we would be happy with for IBD or at tip with relatively turbulent mempools. *waving wildly* On the order of 100ms?

[/quote]

We benchmark a large collection of possible scripts and extract the worst case as the slowest execution time, this does not mean that it will be the new average, the 15 opcodes were disabled due to paranoia of nodes being DDoSed, as in an attacker trying to construct a block that causes memory overflows or extremely slow execution times.

As seen in the benchmark, almost every script is much faster and only uses a small fraction of the varops budget. But we are interested in the worst cases and those worst cases are only reached when hashing (which you can do right now without any restrictions and produce blocks much slower than 100ms) and arithmetic on very large numbers.

I don’t think we can or should predict what the average block with GSR will look like, we only want to make sure there is a reasonable upper bound for script validation times, the purpose of the benchmark is to establish this upper bound.

If 80,000 sig ops is too slow, why was it preserved in Taproot/BIP340-342? Varops really just wants to generalize this limit to all operations, without introducing a new worst case script.

-------------------------

instagibbs | 2025-12-05 15:26:54 UTC | #9

[quote="Julian, post:8, topic:2094"]
If 80,000 sig ops is too slow, why was it preserved in Taproot/BIP340-342?

[/quote]

All else aside I don’t think we’re bound to make the same decisions going forward.

-------------------------

AntoineP | 2025-12-05 16:46:28 UTC | #10

I agree with that, and for the record i don't think maxing out the number of signatures is the worst case in Taproot. Last time i checked `3DUP RIPEMD160 DROP RIPEMD160 DROP RIPEMD160 DROP` a maximum size stack element over and over was the highest validation time per witness byte i could get. ~~Now with the GSR there is certainly a pretty efficient way of evading the signature cache (using `CAT`), so the worst case may be to max out the 80k sigops with different signatures, and then padding the witness size using repeating hashing of a maximum size element.~~ To @instagibbs' point, this is only possible because the hash operations in Tapscript do not count toward the sigop budget, i think they should have.

EDIT: actually you can't just create signatures with `CAT` like this since in Tapscript by consensus they must either be valid or the empty vector. This greatly constrains the cost an attacker can impose.

-------------------------

Julian | 2025-12-05 17:09:23 UTC | #11

[quote="AntoineP, post:10, topic:2094"]
Last time i checked `3DUP RIPEMD160 DROP RIPEMD160 DROP RIPEMD160 DROP` a maximum size stack element over and over was the highest validation time per witness byte i could get.

[/quote]

Yes RIPEMD160 of 520 byte elements is often the worst case (depends on the machine), this is why it also stays limited to 520 byte arguments in the new tapleaf version.

-------------------------

ajtowns | 2025-12-09 18:23:55 UTC | #12

[quote="instagibbs, post:9, topic:2094, full:true"]
[quote="Julian, post:8, topic:2094"]
If 80,000 sig ops is too slow, why was it preserved in Taproot/BIP340-342?
[/quote]

All else aside I don’t think we’re bound to make the same decisions going forward.
[/quote]

As I understand it, the [logic for tapscript](https://github.com/bitcoin/bips/blob/6ce21f4eaeb4a73373688e99d7ea74a6c0413f22/bip-0342.mediawiki#cite_note-11) was just "keep it roughly the same as segwit", [the logic for segwit](https://github.com/bitcoin/bips/blob/6ce21f4eaeb4a73373688e99d7ea74a6c0413f22/bip-0141.mediawiki#user-content-Sigops) was "we're expanding the blocksize by a factor of up to 4, and that should apply to both the bytes and sigop limits", and that brings us back to the [original block size limit introduction](https://github.com/bitcoin/bitcoin/commit/f1e1fb4bdef878c8fc1564fa418d44e7541a7e83#diff-608d8de3fba954c50110b6d7386988f27295de845e9d7174e40095ba5efcf1bbR1419-R1427):

```
+    // Check size
+    if (nHeight > 79400 && ::GetSerializeSize(*this, SER_NETWORK) > MAX_BLOCK_SIZE)
+        return error("AcceptBlock() : over size limit");
+
+    // Check that it's not full of nonstandard transactions
+    if (nHeight > 79400 && GetSigOpCount() > MAX_BLOCK_SIGOPS)
+        return error("AcceptBlock() : too many nonstandard transactions");
```

which seems to have just been limiting the DoS surface.

So I think there's plenty of potential for making a more informed decision now, rather than just assuming what we've already got is sensible.

-------------------------

Julian | 2025-12-10 11:02:20 UTC | #13

> So I think there’s plenty of potential for making a more informed decision now, rather than just assuming what we’ve already got is sensible.

Since the BIPs propose to leave the existing sigversions unchanged and instead aim to add a new sigversion (through a new tapscript leaf), which adheres to the varops budget and potentially implements GSR, it will not have any impact on those existing sigversions including the current base tapscript leaf.

Therefore, it is reasonable to construct a computational budget with parameters, such that the worst case script with GSR activated, is no slower than the current worst case for most or all machines.

From the DoS viewpoint, an attacker can choose any sigversion, so constraining the computational budget in a new sigversion further than the worst case in other sigversions, would not have any benefits, it would constrain the new sigversion unnecessarily.

Maybe you are arguing that the varops budget should also be added to the existing sigversions? That would be a completely different discussion imo.

-------------------------

ajtowns | 2025-12-18 00:56:31 UTC | #14

[quote="Julian, post:13, topic:2094"]
Therefore, it is reasonable to construct a computational budget with parameters, such that the worst case script with GSR activated, is no slower than the current worst case for most or all machines.
[/quote]

That might be reasonable, but I don't think it's good enough: there's a difference between "you can waste X resources doing this silly pattern that serves no purpose except as an attack" and "you can now waste X resources with arbitrary logic". In the former case, you're only vulnerable to deliberate attackers; the latter case you're vulnerable to everyone who comes up with new logic they want to deploy. It was easy to make 4MB blocks as soon as segwit was active; but until inscriptions came along, there was no real reason to do that. We don't want to have GSR result in a similar state change from "it's possible to make slow blocks" to "it's profitable to make slow blocks", if you see what I mean.

Limiting the worst case GSR execution costs to the equivalent of a block full of fairly normal transactions would be very safe, by contrast. I think that would be something on the order of ~13k signature operations per block rather than 80k (based on filling a block with ~3300 2-in, 2-out 2-of-3 multisig p2wsh transactions).

Targeting a validation time of perhaps one or two microseconds per vbyte (so one or two seconds per block) or so would be a better target, I think, though obviously that would require picking some baseline hardware as a minimum standard.

[quote="Julian, post:13, topic:2094"]
Maybe you are arguing that the varops budget should also be added to the existing sigversions? That would be a completely different discussion imo.
[/quote]

Restricting existing script functionality is a different discussion, definitely, because it potentially confiscates coins. It can still be worth having that discussion, either because we might be able to design restrictions that are only likely to restrict attacks (cf BIP 54), or because we might be able to figure out other improvements to avoid the need to restrict things at all (eg [PR#16902](https://github.com/bitcoin/bitcoin/pull/16902) or [CVS-2025-46598](https://bitcoincore.org/en/2025/10/24/disclose-cve-2025-46598/)).

But the easiest chance we have to do better is when introducing new functionality, so if we've got a better means of analysing the potential costs (which the varops budgets is!) then also thinking about the limits from a fundamental perspective rather than just cargo culting what we've had in the past.

-------------------------

rustyrussell | 2026-01-26 03:52:44 UTC | #15

[quote="ajtowns, post:14, topic:2094"]
That might be reasonable, but I don’t think it’s good enough: there’s a difference between “you can waste X resources doing this silly pattern that serves no purpose except as an attack” and “you can now waste X resources with arbitrary logic”. In the former case, you’re only vulnerable to deliberate attackers; the latter case you’re vulnerable to everyone who comes up with new logic they want to deploy. It was easy to make 4MB blocks as soon as segwit was active; but until inscriptions came along, there was no real reason to do that. We don’t want to have GSR result in a similar state change from “it’s possible to make slow blocks” to “it’s profitable to make slow blocks”, if you see what I mean.

[/quote]

This is an important subtlety that I obviously didn’t convey clearly in the BIP.  

There’s a large gap between the worst case, and the average case, and we spec for the worst case.  In particular:

Timings are done the largest possible objects (eg. 2MB objects for OP_ADD, since to do it repeatedly we need to DUP).  This doesn’t seem like it will be typical usage for anything, and my initial benchmarking showed how absurdly fast the cases of 10k object was: significantly faster.  Hard to tell in practice with the current benchmarks, since we start getting dominated by script interpretation time, not the actual ops.

Addition also assumes worst case: that we overflow and have to reallocate and thus copy the entire array (so we add 50% to the add cost).  This is rarely true: even if we do overflow, the allocator usually doesn’t have to copy.

Multiplication, division and modulus build on this add assumption, and create more: in particular, we assume the miss case in the divide guess, which is really hard to hit except by deliberate attempt.  Again, this means gross overcharging in the divide: I can’t come close to the worst case varops cost with any numbers I tried.

The result is a much more conservative approach for real usage.   But there’s more, for complex operations which may conceivably reach these limits:

The implementation is fairly optimal (treating things as 64-bit numbers, in particular), but it’s also naive, to simplify evaluation and finding the worst case.  It uses a simple schoolbook multiply (and thus, divide), which can be fairly easily optimized using Karatsuba or Toom Cook, let alone the zoo of hyperspeed functions in libGMP.  Going further, optimizing the script interpreter is also possible.

[quote="ajtowns, post:14, topic:2094"]
Limiting the worst case GSR execution costs to the equivalent of a block full of fairly normal transactions would be very safe, by contrast. I think that would be something on the order of \~13k signature operations per block rather than 80k (based on filling a block with \~3300 2-in, 2-out 2-of-3 multisig p2wsh transactions).

Targeting a validation time of perhaps one or two microseconds per vbyte (so one or two seconds per block) or so would be a better target, I think, though obviously that would require picking some baseline hardware as a minimum standard.

[/quote]

To be concrete: 80,000 signature checks takes 1.9 seconds on my laptop (i7-1280P, 2023). I know this has sped up since Schnorr was added: I tried to benchmark 0.13.1, to measure the original implementation, but I backed away after realizing that would need an old boost library, etc…

This is an important sanity check!  Even if the worst case becomes common, we’re still under 2 seconds for a modern machine, which seems reasonable.

-------------------------

ajtowns | 2026-01-26 05:31:22 UTC | #16

[quote="rustyrussell, post:15, topic:2094"]
Timings are done the largest possible objects (eg. 2MB objects for OP_ADD,
[/quote]

[quote="rustyrussell, post:15, topic:2094"]
The implementation is fairly optimal (treating things as 64-bit numbers, in particular),
[/quote]

Argh, for some reason I had in my head that GSR was targeting 64-bit maths (like elements), not bignum maths. (Calling it "val64" wasn't super helpful there...)

[quote="rustyrussell, post:15, topic:2094"]
1.9 seconds on my laptop (i7-1280P, 2023)
[/quote]

That seems too slow to me, fwiw (in an ideal world, anyway); taking the 13k vs 80k ratio would bring that down to ~300ms which seems more appropriate for ~current generation, high-end, consumer hardware. I think you'd want to get 1s-2s even from a cheap VPS or a refurbished celeron/i3/i5 minipc -- ie the sort of thing a random datum/stratumv2 miner might decide to use for block construction.

That presumably also means an single standard tx could take 190ms to verify (assuming the 400k weight standardness vs 4M weight consensus ratio dominates), which also feels high, and potentially usable as DoS vector to delay block propagation (by crafting invalid txs that take roughly as long to (in)validate as the worst case valid, standard tx takes to validate; though validating blocks/txs in parallel, or having a way for blocks to bypass the tx validation queue would mostly mitigate that).

[quote="rustyrussell, post:15, topic:2094"]
I tried to benchmark 0.13.1, to measure the original implementation
[/quote]

I guess making a regtest chain with a block with max sig checks, and feeding it into the [release binary](https://bitcoincore.org/bin/bitcoin-core-0.13.1/) might work without requiring dev environment archeology. I guess a deboostrap'ed chroot for xenial ought to allow building it though?

-------------------------

Julian | 2026-02-08 15:07:23 UTC | #17

[quote="ajtowns, post:6, topic:2094"]
Doing benchmarks on the more powerful logic new opcodes enable might be more interesting; eg the [OP_CAT based zkp verifier](https://bitcoinmagazine.com/technical/a-zero-knowledge-proof-is-verified-on-bitcoin-for-the-first-time-in-history)? How many zkps (implemented that way) could we verify in a block, if 100ms (or whatever) of cpu time were the only constraint? Seems to have some 223k operations per script, 30k ADD/DUP, 20k IF/ENDIF/SUB, 10k SWAP/LESSTHAN/TOALT/FROMALT, 400 CAT/1SUB/DEPTH/SHA256, 1 CHECKSIG, etc.

[/quote]

Coming back to this: The OP_CAT based zkp verifier implemented here is extremely lightweight, it has around 3m ops in total but it mostly uses cheap instructions like OP_ADD, OP_DUP, OP_PICK, OP_SWAP, it seems that the only expensive op OP_SHA256 accounts for only 0.4% of all operations. Furthermore, the QM31 elements (and their hashes) are at most 36 bytes so hashing them is also very fast.

All 72 transactions together use about 88% of block space while using only 6.6 million varops (0.03% of the block limit) and all scripts are evaluated in only \~10ms on my machine.

So while this is a super interesting application of OP_CAT, it does not really matter in terms of compute, it seems that Starknet is focusing on OP_CAT only, but we could have much more efficient and potentially compute heavy implementations using OP_MUL and larger stack element limits. I would be happy to benchmark these too but they don’t seem to be available.

-------------------------

