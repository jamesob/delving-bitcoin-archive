# Comparing the performance of ECDSA signature validation in OpenSSL vs. libsecp256k1 over the last decade

theStack | 2025-11-01 09:18:07 UTC | #1

## Introduction

With the upcoming release of v31.0 in spring next year, Bitcoin Core will celebrate its ten-year anniversary of replacing [OpenSSL](https://github.com/openssl/openssl) with [libsecp256k1](https://github.com/bitcoin-core/secp256k1) for ECDSA signature validation in consensus code. Having researched a bit on early history in this topic recently, I thought this might be a good moment to take a closer look on how the performance has evolved over time for these two libraries. At the time the corresponding patch ([#6954](https://github.com/bitcoin/bitcoin/pull/6954)) was merged in November 2015 (released in v0.12 a few months later), signature validation using libsecp256k1 was *“anywhere between 2.5 and 5.5 times faster”* \[than OpenSSL\] according to its PR author and libsecp256k1 creator Pieter Wuille. That’s very impressive already, but as we will see throughout the post, the libsecp256k1 wizards didn’t stop there and continuously shipped incremental performance improvements over the years. In some sense one could claim that a comparison between the two projects is not a completely fair one to begin with, since the scope of OpenSSL is arguably quite different; with the library describing itself as being a “full-featured toolkit for general-purpose cryptography”, it includes a large suite of cryptographic primitives, with support for a very wide range of applications. Still, it’s not unreasonable to assume that OpenSSL might have caught up a bit and implemented some performance improvements for elliptic curve cryptography, and it’s nevertheless interesting to know how the performance gap evolved over time. To the best of my knowledge, since the switchover to libsecp256k1, no concrete investigations have been done how fast ECDSA signature verification would be today, imagining a hypothetical scenario that we would have sticked to OpenSSL.

## Methodology

In order to quantify speed improvements between versions and across different libraries, we need a proper method for benchmarking, ideally one that is as generic and extensible as possible. One idea I found compelling was to do this involving dynamic loading. In a first preparatory step that involves some light shell scripting, all the relevant versions of both OpenSSL and libsecp256k1 are fetched and built to emit a shared library (.so file) each. The actual benchmark is then done with a single binary, which would load in one shared library after another and resolve the corresponding library function addresses at run-time, using the interface functions of the dynamic loading linker (see [dlopen(3)](https://man7.org/linux/man-pages/man3/dlopen.3.html), [dlsym(3)](https://man7.org/linux/man-pages/man3/dlsym.3.html)). Each loop iteration involves the following three steps for each (signature, 32-bytes msghash, 33-bytes compressed pubkey) input triplet:

| step description | function in OpenSSL | function in libsecp256k1 |  |
|----|----|----|----|
| parse compressed public key | `o2i_ECPublicKey` | `secp256k1_ec_pubkey_parse` |
| parse DER-encoded signature | (included in call below) | `secp256k1_ecdsa_signature_parse_der` |  |
| verify ECDSA signature | `ECDSA_verify` | `secp256k1_ecdsa_verify` |  |

The benchmark input data, i.e. for this scenario a list of pseudo-random key pairs, messages, and signatures, is notably created with a _statically_ linked version of libsecp256k1 with the same binary before the actual benchmarks are executed for each version.

Implementing this was actually quite straight-forward and surprisingly painless, especially making things work on the OpenSSL side which
I was less familiar with. Throughout all the OpenSSL versions since 0.9.8h [1] up to 3.5.0, doing a default build of a shared library involved the same commands (`./config shared && make`), and the used
API remained stable since the beginning (though being deprecated now for while), so no individual treatment for different versions was needed. Same for libsecp256k1. The most tedious part was probably finding out which concrete library commit was used in which Bitcoin Core release, considering that there were no tagged releases available [until December 2022](https://gnusha.org/pi/bitcoindev/v-5cMoyHzxi0ZUUECBV_l_87TwzxBXr7-aIMrjC5taolnlKi256ZFnoH6EGw4MpvwVwAJkBwhPToRfSp1DFK314O7edTwjKndQ0azBRGfgI=@wuille.net/t/).

## Demo

The source code is available on GitHub: https://github.com/theStack/secp256k1-plugbench, you can see a quick demo of it being in action here:

[![asciicast](upload://3Ga3nQyzyiTPaDEJ504LDV6PlXy.svg)](https://asciinema.org/a/752929)

(Note that building all the library versions took significantly longer than shown; that preparation part in the recording has been speed up by ~30x using [https://github.com/suzuki-shunsuke/asciinema-trim](https://asciinema-trim)).

I'd encourage everyone to give it a try on their machine (given they run Linux or a similar UNIX-like OS that works with .so files)

```
$ git clone https://github.com/theStack/secp256k1-plugbench
$ cd secp256k1-plugbench
$ ./build_libs.sh && make && ./secp-plugbench results.csv
```

and report the results, especially if they are somehow unexpected or much different and what my sample run yielded. Also, feel free to open an issue or pull request if something doesn't work as expected or could be improved.

## Results and analysis

The following bar plot shows the benchmark results on my arm64 machine [2], using gcc 14.2.0  (some versions were skipped for better visibility):

![openssl_libsecp256k1_bench_results|690x230](upload://iBrdGZ99e05OLPltOlfYeirKcP7.png)


It’s clearly visible that in OpenSSL, the runtime for ECDSA signature verification on the
curve secp256k1 hasn’t changed, while libsecp256k1 improved steadily,
leading to an increasing performance gap over time between the two libraries.

The two largest speedups between libsecp256k1 versions can be primarily attributed to the following changes in the implementation:
 *  bc-0.20 (~28% speedup to bc-0.19): enabling of the GLV endomorphism optimization (implemented very early in libsecp256k1 [3] since it was one of the motivations to start the project in the fist place, but disabled for many years due to potential patent violation issues, see
[US7110538B2](https://patents.google.com/patent/US7110538B2/en)), [PR #830](https://github.com/bitcoin-core/secp256k1/pull/830)
 * bc-22.0 (~30% speedup to bc-0.20): introduction of safegcd-based
modular inverses (PRs [#831](https://github.com/bitcoin-core/secp256k1/pull/831), [#906](https://github.com/bitcoin-core/secp256k1/pull/906)); this speedup is significantly higher than the stated 15-17% in the Core PR [#21573](https://github.com/bitcoin/bitcoin/pull/21573), so there might have been other significant improvements as well?

Note that the benchmarks only measure libsecp256k1 built with defaults,
not taking into account any potential special configuration settings or compile
flags that Bitcoin Core might have set, so the numbers
might not fully reflect what went into a certain release.

Having run this on three different machines with very similar results, I think it's safe to state that libsecp256k1 is more than 8x faster than OpenSSL for verifying ECDSA signatures on the
secp256k1 curve using the respective latest versions. Note that it would be misleading to conclude from these results that OpenSSL is slow in general and/or not open for performance improvements.
For example, the implementation of the curve secp256**r1** [4] seems to be heavily heavily optimized, according to [an issue mentioning the speed differences between secp256k1 and secp256r1](https://github.com/openssl/openssl/issues/23524).  So a more proper conclusion might be here that
outside of the Bitcoin ecosystem, the secp256k1 is just not that relevant and doesn’t count as first-class citizen that would justify spending
a lot of labor hours in a general-purpose cryptographic library. That said, last year
there was actually a PR opened in the OpenSSL repository that aims to improve the speed
of operations on the secp256k1 curve: https://github.com/openssl/openssl/pull/26097. The name mentioned in the PR title that the code is based on might sound familiar to you :) 

## Outlook

Doing this was a very interesting and fun mini project. I currently don’t plan to put too much more effort into it, but I guess it's a good idea to
run the benchmark before each release of libsecp256k1 / Bitcoin Core (it’s as simple as adding two lines), in order to quantify improvements and ensure
we don’t have performance regressions. In that sense, it could even make sense to add Schnorr signature verification and functionalities from other
modules that are performance-critical as potential benchmarking scenarios, to comfortably track progress over the next years. Maybe someone is even motivated to add a benchmark scenario for signing.

Cheers,
Sebastian

[1] 0.9.8h is the first OpenSSL version explicitly mentioned in the Bitcoin Core [repository's first commit (having that exact subject) by sirius-m](https://github.com/bitcoin/bitcoin/commit/4405b78d6059e536c36974088a8ed4d9f0f29898#diff-9e6e4772050998a5c0dc3c61acf3dab0a7e594566171fa5746d6b62f9598efb6R38)

[2] note that calling into OpenSSL 0.9.8 and 1.0.0 led to a crash on my arm64 machine, so I skipped those; I don't know what the concrete issue is, I'm assuming it could have to do with memory alignment. On x86-64, all the built versions could be benchmarked successfully.

[3] see https://github.com/bitcoin-core/secp256k1/commit/949bea92624fbd65bfb21d773f1df6a115af71ff (that's only the 13th commit in the repository!)

[4] the secp256r1 curve was recently discussed on the Bitcoin mailing list, see https://groups.google.com/g/bitcoindev/c/XSYL0gx0cDM

-------------------------

sipa | 2025-11-02 15:49:38 UTC | #2

I ran your script on a Ryzen 5950X CPU, with 500000 iterations (as I saw a decent amount of variance between runs with lower numbers), showing a speed ratio of 8.5 currently.

![secp256k1_5950x|690x230](upload://ckNAeCSoatUJzmwLvF0FxtI3tLE.png)

Interestingly, in these numbers there does not appear to be any speed grain in bc-22.0 (in fact, a 3.6% slowdown!) from the safegcd code, but there is an 8% speed up in bc-27.0, presumably due to the removal(!) of x86_64 assembly in https://github.com/bitcoin-core/secp256k1/pull/1446 (libsecp256k1 v0.4.1).

I think it's also important to be aware of some biases exhibited by these numbers:
* We're benchmarking them all (necessarily for fairness) on the same identical hardware, but the efforts that went into creating this code were informed by benchmarks on hardware available at the time. This holds both for different systems within the same (e.g. x86_64) architecture, and popularity of architecture over time (e.g. aarch64 is much more popular and relevant now than 10 years ago).
* We're benchmarking them all with the same compiler, but historical efforts aimed at improving performance for compilers that existed at the time. For example, the x86_64 field assembly code was better than machine code generated from pure C code 10 years ago, but there is barely a difference with modern compilers (which is why 0.4.1 removed it).

-------------------------

sipa | 2025-11-02 15:59:40 UTC | #3

@theStack What do you think about making a graph which puts date/time (release/commit date?) on the X axis, and perhaps a log scale on the Y axis (so that the same % improvement shows as the same distance on the graph)? That would make it less bad to add more versions to the graph as well.

-------------------------

theStack | 2025-11-03 12:50:14 UTC | #4

That sounds like a great idea! To prepare for that, I've [pushed a commit](https://github.com/theStack/secp256k1-plugbench/commit/202d6358e48c31afcbee094992b617088be3d75d) to include a "date" column with the release date (in format yyyy-mm-dd, i.e. ISO 8601) in the generated .csv file for each benchmarked version; having that it should be relatively easy to make such a graph. I haven't adapted plot.py yet to make use of that new column, will look into that later that week. If you or anyone else is eager to give it a shot already (my plotting skills are close to zero, the code is largely derived from ChatGPT interaction), feel free :)

-------------------------

