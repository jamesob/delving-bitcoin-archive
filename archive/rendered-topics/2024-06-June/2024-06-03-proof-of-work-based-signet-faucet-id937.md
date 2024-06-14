# Proof-of-work based signet faucet

ajtowns | 2024-06-03 08:56:16 UTC | #1

With the recent activation of [`OP_CAT`](https://github.com/bitcoin-inquisition/bitcoin/pull/39) on signet via the [bitcoin inquisition client](https://delvingbitcoin.org/t/bitcoin-inquisition-25-2/838), it's at least theoretically possible to some [weird scripting tricks](https://covenants.info/extra/cat/). One of the easier things to do with `OP_CAT` is to validate proof-of-work, since you can program that as more or less "take the non-zero bytes of a hash, cat that togethether with a bunch of zero bytes, then check that the original thing hashes to what you just catted together", ie a script something like `<"\x00\x00\x00" CAT SWAP SHA256 EQUALVERIFY>`.

The obvious thing to do with proof of work on the blockchain is to give people coins for it, and that's perhaps particularly interesting on signet, since as a permissioned blockchain, the only way you can normally get coins is to be given them by the signet block producers. That is, the idea is that every block, the signet block producers send some sBTC to an contract that allows funds to be claimed by providing proof of work.

If you're the sort of person that skips straight to the last page when reading a murder mystery, you can see a [demo transaction claiming some funds](https://mempool.space/signet/tx/25deff74b7775918aa6824cfeab9121a519fb419d31ea74f6cb6a53f98cc863a) or mess around with the [python script to claim funds](https://github.com/ajtowns/bitcoin/blob/634e72cbfc0aa3f657a35c7b597f688bb2bb29a6/contrib/signet/powcoins).

There's a few constraints here. For example, you don't want the proof of work to be reusable -- if I spend 5 minutes of CPU grinding a way to claim 1 sBTC, I don't want you to be able to just sit their watching the mempool, and replace my transaction with one sending the funds to you, without you having to do any work. But that turns out pretty straightforward: you can just make the preimage of your PoW-hash be `x + sig + y` and then check `p sig CHECKSIG`. If you try to reuse my proof of work, you have to keep the same signature, but that signature commits to where the funds go, which is to me, so you can't steal the funds. If we go a step further and check that `p` is 32 bytes, and `sig` is 64 bytes, then we know that it's a `SIGHASH_DEFAULT` signature, committing to everything about the tx, as well.

Just as you don't want other people to reuse your proof of work, you don't want the same person to be able to use a single proof of work twice -- eg, what if you started with `x + sig1 + sig2 + sig3 + y` and tried different values for `x` and `y` until you got a lot of zeroes? Then you'd be able to reuse that single value to claim three coins: one with `sig1` (`x'=x`, `y'=sig2+sig3+y`), one with `sig2` (`x'=x+sig1`, `y'=sig3+y`) and the third with `sig3` (`x'=x+sig1+sig2`, `y'=y`). So we avoid that by committing to the length of `x` and `y` before hashing.

Finally, there's a question of what sort of "work" do we want -- something where you can point bitcoin ASICs at it, or something where you can't? I picked something as compatible as possible with bitcoin ASICs, so the python script above constructs a 4-byte `x` and a 12-byte `y`, so that if you were to interpret `x+sig+y` as an 80-byte block header, `x` serves as `nVersion`, `sig` serves as the combination of the prev block hash and the merkle root, and `y` covers the timestamp, nBits and the nonce. The script goes a step further and sets the `nBits` value to the expected difficulty, and then passes the whole thing to a CPU miner in the form of `bitcoin-util grind`.

So how's it implemented? There's more or less six parts to the script, so we'll go through them one by one.

```txt
OP_PUSHNUM_16 OP_SUB
OP_DUP OP_0 OP_GREATERTHANOREQUAL OP_VERIFY
OP_DUP OP_PUSHBYTES_1 40 OP_LESSTHAN OP_VERIFY
OP_DUP OP_DUP OP_ADD OP_DUP OP_ADD
OP_PUSHBYTES_2 0001 OP_SWAP OP_SUB OP_CSV OP_DROP
```

This first part takes a difficulty specifier from the top of the stack, subtracts 16, checks the result is between 0 and 64 (so the original was between 16 and 80), multiplies the value by 4, subtracts that from 256, and invokes `CHECKSEQUENCEVERIFY` on the result -- so a difficulty specifier of 80 requires 0 blocks' delay, while a difficulty specifier of 79 requires 4 blocks delay, etc.

```txt
OP_PUSHBYTES_2 0000  OP_TOALTSTACK
OP_DUP OP_PUSHBYTES_1 20 OP_GREATERTHANOREQUAL 
OP_IF OP_FROMALTSTACK OP_PUSHBYTES_4 00000000 OP_CAT OP_TOALTSTACK
OP_PUSHBYTES_1 20 OP_SUB OP_ENDIF
OP_DUP OP_PUSHNUM_16 OP_GREATERTHANOREQUAL
OP_IF OP_FROMALTSTACK OP_PUSHBYTES_2 0000 OP_CAT OP_TOALTSTACK
OP_PUSHNUM_16 OP_SUB OP_ENDIF
OP_DUP OP_PUSHNUM_8 OP_GREATERTHANOREQUAL
OP_IF OP_FROMALTSTACK OP_PUSHBYTES_1 00 OP_CAT OP_TOALTSTACK
OP_PUSHNUM_8 OP_SUB OP_ENDIF
```

The difficulty specifier is the number of zero bits we want, so since we subtracted 16 already we start with 2 zero bytes, and if the remaining difficulty is more than 32 that's another 4 zero bytes, etc. We progressively CAT those together and leave them on the alt stack, with the remainder (a figure from 0 to 7) on the main stack.

```txt
OP_SWAP OP_SIZE OP_PUSHNUM_1 OP_EQUALVERIFY
OP_DUP OP_TOALTSTACK
OP_PUSHNUM_1 OP_CAT OP_PUSHBYTES_2 0001 OP_SUB
OP_SWAP
OP_DUP OP_0 OP_EQUAL OP_IF OP_PUSHBYTES_2 0001 OP_ELSE
OP_DUP OP_PUSHNUM_1 OP_EQUAL OP_IF OP_PUSHBYTES_2 8000 OP_ELSE
OP_DUP OP_PUSHNUM_2 OP_EQUAL OP_IF OP_PUSHBYTES_1 40 OP_ELSE
OP_DUP OP_PUSHNUM_3 OP_EQUAL OP_IF OP_PUSHBYTES_1 20 OP_ELSE
OP_DUP OP_PUSHNUM_4 OP_EQUAL OP_IF OP_PUSHNUM_16 OP_ELSE
OP_DUP OP_PUSHNUM_5 OP_EQUAL OP_IF OP_PUSHNUM_8 OP_ELSE
OP_DUP OP_PUSHNUM_6 OP_EQUAL OP_IF OP_PUSHNUM_4 OP_ELSE
OP_PUSHNUM_2 
OP_ENDIF OP_ENDIF OP_ENDIF OP_ENDIF OP_ENDIF OP_ENDIF OP_ENDIF
OP_NIP OP_LESSTHAN OP_VERIFY
```

The second item on the stack is the first byte of the hash after the leading zero bytes. If the remaining difficulty is 0, this can be anything, if it's 1, it should be less than 0x80, if it's 2, it should be less than 0x40, etc, and if it's 7 it should be less than 0x02 (ie, 0x01 or 0x00 exactly). Unfortunately you can't just use "less than" here, because 0x80 would be treated as negative zero, and 0x81 through 0xFF would be treated as -1 to -127 rather than 129 to 255. To work around this, we append 0x01 and subtract 128, which gives us the correct value (minimally encoded), after copying the actual byte that we want onto the alt stack. A bunch of IFs later and we can do the comparison we want.

```txt
OP_FROMALTSTACK OP_FROMALTSTACK OP_CAT
OP_CAT OP_TOALTSTACK
```

On the alt stack we have a string of zero bytes, and the first non-zero byte, and now the third witness item is at the top of the main stack, so catting all those together should give the full 32 byte hash that we want. Calculate that and put it on the alt stack.

```txt
OP_2OVER
OP_SIZE OP_PUSHBYTES_1 20 OP_EQUALVERIFY OP_DROP
OP_SIZE OP_PUSHBYTES_1 40 OP_EQUALVERIFY
OP_SWAP
OP_SIZE OP_SWAP OP_CAT
OP_CAT
OP_SWAP
OP_SIZE OP_SWAP OP_CAT
OP_SWAP
OP_CAT
OP_HASH256
OP_FROMALTSTACK OP_EQUALVERIFY`
```

The remaining items on the stack are, from top to bottom, the preimage suffix, the preimage prefix, the pubkey and the signature. Copy the pubkey and signature to the top of the stack, check their sizes, and throw away the pubkey. Swap the signature and the suffix, calculate the size of the suffix, and cat those three things together (`sig + suffix_size + suffix`). Swap that with the prefix, cat the size of the prefix at the beginning of the prefix, then cat that at the beginning of everything for the full preimage (`prefix_size + prefix + sig + suffix_size + suffix`). Hash the preimage, and check that the result matches what we pushed onto the altstack earlier.

```
OP_CHECKSIG
```

Finally, we just have the pubkey and signature left on the stack, so check the signature.

As far as the python script goes, it has two parts:

```none
$ ./powcoins setup-wallet
```

The first is a short command to setup a `varpow` wallet via bitcoind that tracks the faucet coins it can spend. You can use `bitcoin-cli -signet -rpcwallet=varpow listunspent` to see what they are.

The second is

```none
$ ./powcoins claim --max-diff=16 tb1qzv66dscl3uauh5jesfem33wws747naapcgt4c5
INFO:root:sendrawtransaction result 71fb0d9593dd7c697cf4bb595bb2f0b63a4f6cb3d675e81b6e7b6e5fd07e0027                                                                                                                                         
```

which will attempt to claim a coin that's currently less than the specified max difficulty, picking the coin that has the most value per unit work needed. Note that you'll need to have a bitcoin inquisition node to use this, as otherwise a script that uses `OP_CAT` will fail the `DISCOURAGE_OP_SUCCESS` checks, and you'll probably want to run that node with `-addnode=inquisition.bitcoin-signet.net` to make sure you have a peer that will actually accept transactions that use `OP_CAT`.

Thanks to @theStack for [collaborating](https://github.com/theStack/bitcoin-inquisition/issues/1) on this!

-------------------------

garlonicon | 2024-06-04 14:02:59 UTC | #2

[quote]Just as you don’t want other people to reuse your proof of work, you don’t want the same person to be able to use a single proof of work twice[/quote]
All of those problems could be solved, if instead of using Proof of Work from the Script, you check your Proof of Work from your [url=https://mempool.space/tx/000000000fdf0c619cd8e0d512c7e2c0da5a5808e60f12f1e0d01522d2986a51]transaction hash[/url]. Then:
1. It cannot be stolen, because modifying the transaction in any way (when it comes to legacy parts) will lead to a different transaction hash.
2. It can be used once, or thrown away. If you include transaction with one hash, you cannot include it again (there are some quirks, when you reuse the same coinbase transaction in some future block numbers, but most of the time, the system is resistant to duplicated future transaction hashes: if you can duplicate transactions, then there are more serious issues ongoing, like "history flood attack").
3. If the size of the whole transaction will take 80 bytes, it will be compatible with ASICs. That part can be achieved for example by using different sighashes, like `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY`, and then require for example z-value of the transaction (hash of the signed message) to contain some leading zeroes.
4. Also note, that it may be possible to claim N coins at once, by producing a sufficiently huge Proof of Work. Then, you share for example [url=https://mempool.space/tx/00000000c0747ece95852c2b7d37b09770873ed8b50fb7be2ba3d400defab06c]transaction like this one[/url], and then you claim 10 coins instantly, without providing separate proofs for each and every input. You just claim them all by providing sufficiently low hash in the whole transaction.

-------------------------

levantah | 2024-06-14 10:15:05 UTC | #3

Thanks! This still works for me:

```
$ ~/src/bitcoin/contrib/signet/powcoins setup-wallet
$ ~/src/bitcoin/contrib/signet/powcoins claim --max-diff=25 mfqXWKj6cmRq18yH9pyrLMGY7L5QxAMHWR
INFO:root:sendrawtransaction result ce82bca0bfd81cacac0f7b108b20de5b888957ba516745f6acdea26244138817
$ ~/src/bitcoin/contrib/signet/powcoins claim --max-diff=16 mfqXWKj6cmRq18yH9pyrLMGY7L5QxAMHWR
ERROR:root:Faucet is too difficult (min difficulty 24)
$ ~/src/bitcoin/contrib/signet/powcoins claim --max-diff=24 mfqXWKj6cmRq18yH9pyrLMGY7L5QxAMHWR
INFO:root:sendrawtransaction result 9d3a5fd2f885b490ce60447321ecd9a379f7ac980e5b9ef218ac3f16e7d5a766
```

May those txids serve as timestamps here.

-------------------------

