# Mutual exclusiveness of op_codes

PierreRochard | 2024-05-21 03:35:59 UTC | #1

Doing a very surface-level readthrough of current softfork proposals, a good amount of narrative seems to be "X is better than Y because..." implying a mutually exclusive predicament, that we have to choose one proposal over another. Without picking a side of whether that's true or not, I'd like to at least understand the philosophical arguments on both sides. 

Some initial thoughts:

* Needing to maintain more code. In some cases where the op_code is trivial like OP_CAT this seems inconsequential, in other cases like Simplicity it's a very substantive point. 
* Attack / bugs / vulnerability surface area, not always correlated with lines of code. 
* Limited number of op_code "slots" available to take.
* Activation politics, bundling could increase contention but separating might exhaust participants' energy too.
* Ecosystem complexity, multiple similar options to accomplish the same outcome.                                                                                                                                                                                                      

I just philosophically like the idea of letting developers and users opt in to the op_code they want even if there's a 90% overlap between two different op_codes. They might find one easier to grok or have a very particular use case. As long as node resource usage limits are respected and protocol maintainers are not overwhelmed, it seems fairly innocuous. Interested in hearing your thoughts!

-------------------------

garlonicon | 2024-05-21 12:06:58 UTC | #2

I think some opcodes could be simplified, if you do the same thing, which was already done in assembly: split each opcode into smaller parts, and calculate the real cost on a different layer. For example: `OP_CHECKSIG` is quite complex. It has at least some sub-opcodes inside, like calculating z-value (message hash), calculating R-value from r-value (lifting x-value public key into the real (x,y) point on secp256k1), and of course doing addition and multiplication, between given public keys and raw values, to get a matching equation.

Which means, that if you want to implement for example `OP_CHECKSIGFROMSTACK`, then by having `OP_CHECKSIG` alone, a lot of that is already done, and the only replacement is the message to be hashed. The same is true for many other contracts, where you want to validate a given signature, by using a different message. For example: each change into sighashes (like `SIGHASH_ANYPREVOUT`) fall into a similar category.

So, is it more code? Kinda, but not quite, because a lot of code can be reused (or even simplified, because if you have `OP_CHECKSIGFROMSTACK`, then you can implicitly convert each `OP_CHECKSIG` into `OP_CHECKSIGFROMSTACK`, and have a single place to handle all of that). So, instead of `<signature> <pubkey> OP_CHECKSIG`, you can convert it statically into `<message> <signature> <pubkey> OP_CHECKSIGFROMSTACK`, and then, in practice, `OP_CHECKSIG` could become just an alias into `OP_TUCK_MESSAGE OP_CHECKSIGFROMSTACK`, where `OP_TUCK_MESSAGE` don't have to be any real opcode, but just some internal function call in the source code (note that when it comes to existing code, there are lot of internal things, which already exist, and have no representation in the Script, so the whole diff is not that much bigger, as it seems to be; for example we already have functions for checking any signature, but they are just used in places like "Bitcoin Message" handling).

When it comes to available slots for new opcodes, then we still have a lot of them (but maybe we need some kind of page to trace them, like BIP numbers). Even Taproot can show us, that the whole Script can be completely changed into TapScript, without affecting the original one. Which means, that if there would be any need, to have some "assembly for Script" in the future, then a single opcode is all you need to introduce it. I can even think of cases, where you can use your own Script, for example with Homomorphic Encryption, and get it activated in Lightning Network, without touching any mainnet rules. Then, you would encrypt things, execute them in LN-only, and then decrypt and post on-chain. But in-between, you can have some execution steps, which are not allowed in the current Script, and could be handled only by some second layer.

Also, that last case partially answers your question about activation politics: it could be done locally first, to test things properly, and deploy later, if there would be enough users, willing to move those changes on-chain (but if that kind of activation would be successful, then some improvements could just stay in L2, because in case of Homomorphic Encryption, they could be trustlessly decrypted, and shared in on-chain compatible form, if needed).

-------------------------

ajtowns | 2024-05-21 17:29:13 UTC | #3

[quote="PierreRochard, post:1, topic:890"]
Limited number of op_code “slots” available to take.
[/quote]

That one mostly probably doesn't matter: for tapscript we've got 87 free opcodes via OP_SUCCESS, and any of those could be used to create multibyte opcodes if desired. The upgradable OP_NOPs form a much more limited set, restricting how we could upgrade p2sh or segwit v0 scripts, but that's probably not that interesting. Providing new bare outputs by using a new opcode directly in a scriptPubKey is similarly restricted, but probably even less interesting, since that would require creating a new address format and a bunch of upgrade hassles.

[quote="PierreRochard, post:1, topic:890"]
Ecosystem complexity, multiple similar options to accomplish the same outcome.
[/quote]

Script already has a variety of ways of doing the same thing: `EQUALVERIFY` is equivalent to `EQUAL VERIFY` (and similarly for all the other `FOOVERIFY` ops), `HASH256` is equivalent to `SHA256 SHA256`, and `HASH160` is the same as `SHA256 RIPEMD160`, `GREATERTHAN` is the same as `SWAP LESSTHAN`, `LESSTHANOREQUAL` is the same as `GREATERTHAN NOT`, before they were disabled `SUBSTR`, `LEFT` and `RIGHT` had moderately overlapping featuresets too. tapscript continues this trend, with `CHECKSIGADD` being the equivalent of `ROT SWAP CHECKSIG ADD`.

The thing we probably don't want is subtly different ways of doing the same thing, where choosing the wrong one is likely to result in bugs that lose people money. eg, satoshi removed `OP_NOTEQUAL`, [writing](https://github.com/bitcoin/bitcoin/commit/e071a3f6c06f41068ad17134189a4ac3073ef76b#diff-27496895958ca30c47bbb873299a2ad7a7ea1003a9faa96b317250e3b7aa1fefR494-R498):

```c++
    //case OP_NOTEQUAL: // use OP_NUMNOTEQUAL
    ...
    // OP_NOTEQUAL is disabled because it would be too easy to say
    // something like n != 1 and have some wiseguy pass in 1 with extra
    // zero bytes after it (numerically, 0x01 == 0x0001 == 0x000001)
    //if (opcode == OP_NOTEQUAL)
    //    fEqual = !fEqual;
```

Personally, I think a bigger issue is figuring out what we can do with the various features in practice, rather than just on a whiteboard. There are lots of essentially equivalent ways of doing things (eg compare CTV with similar approaches based on TXHASH, APO, or CAT/CHECKSIG, or with [CTV-v2](https://github.com/bitcoin/bips/pull/1587)) where the difference is pretty trivial when you're just talking about what you could do with it, but when you're actually trying to put things into production, you might end up with a strong preference to one or the other.

If we deployed all those different features in mainnet, and then discovered everyone ends up just using (eg) TXHASH, that's a lot of wasted code in consensus, review effort, and makes for a confusing setup when people want to just develop on bitcoin ("oh, you should just ignore these parts"). Likewise if we were to implement everything then decide we actually want some other way to do essentially the same thing that's just slightly different.

I think the best way of overcoming those problems is actually going all the way to having fairly realistic wallet/lightning/etc software demoing the new features we want, before trying to add them to mainnet, so that we can see how they work in practice, and perhaps even have an easy way to compare how well/badly real software would work if we had one primitive versus a different one. Would be a way of getting concrete/objective answers to questions like: do the scripts end up being expensive on-chain, do they create new pinning vectors, is it hard to use them without introducing bugs/exploits, is it hard to debug contracts that use these features, etc.

Another big question is "is this feature enough on its own" -- eg CTV can do limited "vault" constructs on its own, but if you add some other features, you can make much more useful constructs; likewise eltoo was originally written up assuming that just a `NOINPUT` signature type would be enough, but when trying to actually make that work in ln-symmetry, it becomes pretty apparent that some way of attaching fees is also needed (`ANYONECANPAY` introduces pinning attacks, so ephemeral anchors was invented, but that then requires more work on package relay, which then really needs cluster mempool to work well), and some way of attaching data at signing time is also desirable (in this case it removes a round trip when forwarding HTLCs, and can be achieved via using the taproot annex or perhaps `CHECKSIGFROMSTACK`; or could perhaps be avoided by clever use of adaptor signatures).

(Doing all the things via inquisition and signet, building demos on top of it, then choosing the features that turn out best and pulling them into mainnet is my best answer for how to turn the above into a practical todo list. YMMV obviously!)

-------------------------

