# Winternitz One Time Signatures, contrasting between Lisp and Script

ajtowns | 2024-11-13 08:07:02 UTC | #1

Jonas Nick tweeted about an implementation of [WOTS+](https://eprint.iacr.org/2017/965.pdf) via the expanded script opcodes proposed for [the GSR project](https://github.com/rustyrussell/bips/pull/1):

https://x.com/n1ckler/status/1854552545084977320

It's very interesting and worth looking at. The basic idea is:

 * You generate a WOTS+ secret/public key pair (67 secrets, 1 seed, and 15 randomizers; each of which are 256 bits; the 67 secrets correspond to 67 hash-images that serve as the public key)
    ```txt
   $ .lake/build/bin/wots seckeygen mysecrets
   ```
 * Using these, you generate a very large script that encodes the pubkey, seed and randomizers, and can be used to verify a signature:
   ```txt
   $ .lake/build/bin/wots scriptgen mysecrets myscript
   $ .lake/build/bin/wots scriptparse myscript | cat | head -c60; echo
   (some [OP_DUP, OP_1ADD, OP_PUSH(79be667ef9dcbbac55a06295ce87
   $ du -h ./myscript
   24K	./myscript
   ```
 * You can then pretend that you've created a bitcoin transaction spending some funds, and calculated the appropriate message to sign, eg following the BIP 342 rules, for demo purposes, we imagine it is `0xabababab...ab`. This is then hashed following the BIP 340 rules, so that it can be validated via a CHECKSIG via the `s=m+1` [CAT trick](https://medium.com/blockstream/cat-and-schnorr-tricks-i-faf1b59bd298). Thus the tx signature is a witness stack consisting of the CAT-trick hash, and 67 256-bit signature hashes, one for each of the secrets/hash-images.
   ```txt
   $ SIGHASH=abababababababababababababababababababababababababababababababab
   $ .lake/build/bin/wots witnessgen mysecrets $SIGHASH mywitness
   ```
 * Finally you can run the script against your witness, and see if it worked:
   ```txt
   $ .lake/build/bin/wots verify myscript $SIGHASH mywitness
   valid
   ```

The tweet raises two concerns:

> * [...] Stack management remains very challenging, as GSR doesn't address this aspect.
> * A significant limitation is the size: the script is 22kB, with a 2kB witness. This represents a roughly 375-fold increase over a Schnorr signature witness and would substantially change the nature of Bitcoin. I believe there's limited room for optimization using the standard W-OTS scheme.

These are pretty parallel to the idea [I raised in March](https://delvingbitcoin.org/t/chia-lisp-for-bitcoiners/636): namely "there are two things that as a language [bitcoin script] can’t do well: looping and structured data." Not handling structured data is what makes stack management challenging, and in this case not being able to loop means having to repeat the same lines of hashing code 15 times for each of the 67 sigs/pubkeys you need to verify, giving you an immediate 1000 times increase in code size.

Redoing the same logic in [bllsh](https://delvingbitcoin.org/t/debuggable-lisp-scripts/1224/) gives a much smaller script, of about 3600 bytes. This could be further reduced by ~2200 bytes if the pubkeys were hashed together, rather than included literally, bringing the script down to about 1400 bytes (about 40 times larger than p2wpkh, which when combined with the 2200 bytes of witness data is about 35 times larger than a p2wpkh in total). If you also generated the randomization data from the seed, I suspect you could probably reduce the script by up to a further 480 bytes without much loss in security (down to about 30 times larger than p2wpkh in total). Improvements in the serialization format might also make a difference.

The source is [available as an example in the bllsh tree](https://github.com/ajtowns/bllsh/blob/3ed435a56ca1331da18567fbfee4d30e515bb947/examples/test-winternitz), but it's probably short enough to run through directly:

 * WOTS+ has a seeded/randomized hash function, so we make a function for that with our seed hardcoded. We also hardcode the randomization constants, and the pubkey data.
   ```txt
   def (HASH R X) (sha256 0xb10dce54185a124a89cf3432ea7eb0475d993cf767b3ee93da70d2a8408e7834 (^ X R))`
   def RANDOMIZATION (q 0x9bc6d76b5c9d462db42274e2d8218a8fe4437ee11f26a1073109896d373883b0 ...)
   def HASHES (q 0xe7a848b9a2176261b38d952ff6d9cc982e9330f903c4e49df0254ce99c5fe710 ...)
   ```
 * The main Winternitz function is chaining the hash function, which looks like:
   ```txt
   def (CHAIN N RRAND X) (if N (HASH (h RRAND) (CHAIN (- N 1) (t RRAND) X)) X)
   ```
   This isn't tail recursive (which sucks), but when verifying a signature, it means we can just set N to the correct number of hashes, and have the RRAND list be ordered so that the last randomizer comes first, and it all works out.

 * The parameters we choose for WOTS+ means it operates on hexidecimal digits or half-bytes, so we need to convert the strings we start with into lists of hexidecimal digits to operate on. We also need to calculate a checksum in order for the signature scheme to be remotely secure. That occurs like this:
   ```txt
   def (REVERSE L TAIL) (if L (REVERSE (t L) (rc TAIL (h L))) TAIL)
   def (HEXPAIR BYTE TAIL) (rc TAIL (shift BYTE -4) (& BYTE 0x0f))
   def (HEXDIGITS STR TAIL) (if STR (HEXDIGITS (substr STR 1) (HEXPAIR (substr STR 0 1) TAIL)) TAIL)
   def (HEXTRIP SHORT) (rc nil (+ (shift SHORT -8)) (+ (shift (& SHORT 0xf0) -4)) (+ (& SHORT 0x0f)))
   def (CHECKSUM W C REVDIG TAIL) (if REVDIG (CHECKSUM W (- (+ C W) 1 (h REVDIG)) (t REVDIG) (rc TAIL (h REVDIG))) (REVERSE (HEXTRIP C) TAIL))
   ```
 * Beyond that, there's the CAT trick calculation:
   ```txt
   def (SIGMSG_ M SECP_G) (sha256 SECP_G SECP_G M)
   def (SIGMSG M) (SIGMSG_ M 0x79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798)
   ```
 * And then it can all be put together:
   ```txt
   def (WOTSCHECK_ W PUBKEY SIG DIGITS RESULT) (if PUBKEY (WOTSCHECK_ W (t PUBKEY) (t SIG) (t DIGITS) (all RESULT (= (h PUBKEY) (CHAIN (- W (h DIGITS)) (RANDOMIZATION) (h SIG))))) RESULT)

   def (WOTSCHECK SIGHASH WITNESS) (WOTSCHECK_ 15 (HASHES) (t WITNESS) (CHECKSUM 16 0 (HEXDIGITS (h WITNESS) nil) nil) (= (h WITNESS) (SIGMSG SIGHASH)))
   ```

That is not necessarily all that simple to follow, but most of it is already fairly analogous to the functional lean4 implementation code in [Winternitz.lean](https://github.com/jonasnick/GreatRSI/blob/664be9d7f0de4df5c56e91c5a1f8c13ccde12eec/GreatRSI/Winternitz.lean) (see f_k/chain vs HASH/CHAIN, base16/checksum vs HEXDIGITS/CHECKSUM, winternitz vs WOTSCHECK, and challenge_cat_trick from Schnorr.lean vs SIGMSG). That seems substantially easier to deal with than the complexity of [ScriptBuilder.lean](https://github.com/jonasnick/GreatRSI/blob/664be9d7f0de4df5c56e91c5a1f8c13ccde12eec/GreatRSI/Winternitz.lean), and, as a consequence, perhaps also more amenable to formal verification?

-------------------------

