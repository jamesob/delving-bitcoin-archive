# CTV++ OP_TEMPLATEHASH and OP_INPUTAMOUNTS

moonsettler | 2024-12-30 12:34:21 UTC | #1

There have been complaints voiced about being too paiful to work with `CTV`. Especially when it comes to Vaults. `OP_TX` and `OP_TXHASH/VERIFY` are examples how developers sought to overcome certain pain points. However they expand the scope considerably and also rely on 64 bit integer arithmetic to handle
amounts. Here we aim to show an alternative approach that covers a lot of
useful ground in the in-between without state-carrying covenants or general
detailed introspection.

Naturally some of this functionality could be coupled into `CTV` by adding new template types `TXHASH` style, there are advantages and disadvantages to that.

For now, let's put that aside and take a look at what a highly specialized hash function can do for amount flexibility!

Yes, this is yet an other "since we don't have `OP_CAT`..." thing.

https://gist.github.com/moonsettler/d2369e043473c42ff7fa52878dd064a5

## CTV++

Two additional opcodes to consider as an addition to `OP_CHECKTEMPLATEVERIFY`:

* OP_TEMPLATEHASH
* OP_INPUTAMOUNTS

Together they relax the strict limitations that `OP_CHECKTEMPLATEVERIFY`
imposes, because it explicitly commits to the output amounts and therefore
implicitly commits to the spendable input amounts in a lot of cases.

### OP_TEMPLATEHASH

Builds the `CTV` template hash from the stack.

```sh
<inputIndex>
# FOR i = inputCount-1; i >= 0; i--; LOOP
  <sequence[i]>
# END LOOP
<inputCount>
<nLockTime>
# FOR j = outputCount-1; j >= 0; j--; LOOP
  <out[j].amount>
  <out[j].script>
# END LOOP
<outputCount>
<nVersion>
OP_TEMPLATEHASH
OP_CHECKTEMPLATEVERIFY
```

### OP_INPUTAMOUNTS
Taproot only. Consumes a 32 bit signed integer `n` as parameter
* `n = 0` return the SUM of all input amounts with same script
* `n < 0` return the SUM of last abs(n) input amounts including current input
* `n > 0` return the SUM of first n input amounts of the transaction

fails if
* `n < 0` and `abs(n) > inputIndex + 1`
* `n > inputCount`

```sh
<n>
OP_INPUTAMOUNTS
```

#### Example use:

This contract below allows the combining of UTXOs locked by the same script for
something like a withdrawal from a Vault contract to a staging address where
the relative time lock can begin. It works with any amount UTXOs unlike basic
CTV Vaults. Also allows for paying fees endogenously and registering a change
address. The fee paying input would sign with SIGHASH_ALL.

```sh
### Witness stack BEGIN ###

<inputIndex>
# FOR i = inputCount-1; i >= 0; i--; LOOP
  <sequence[i]>
# END LOOP
<inputCount>
<nLockTime>
<changeAmount>        # out[1].amount
<changeScriptPub>     # out[1].script

### Witness stack END ###

<0>                   # sum of all inputs with same script
OP_INPUTAMOUNTS       # out[0].amount
<stagingScriptPub>    # out[0].script 33 bytes for P2TR
<2>                   # outputCount
<2>                   # nVersion
OP_TEMPLATEHASH
OP_CHECKTEMPLATEVERIFY
```

### Credits:
* Jeremy Rubin who have already came up with everything many years ago
* James O'Beirne for his awesome work on OP_VAULT
* Salvatore Ingala for his work on CCV/MATT and to generalize state carrying
covenants
* Many others who have explored the covenant design space and CTV in particular

-------------------------

ProofOfKeags | 2024-12-30 21:52:24 UTC | #2

I don't think it is a good idea to introduce new points on the continuum between CTV and TXHASH. By doing this it submits to the frame that we should be regulating how users want to specify spending conditions. CTV offers max specificity, TXHASH offers max generality. Any point on the interior is just going to make activation of *any* of these much harder.

-------------------------

moonsettler | 2024-12-30 22:16:52 UTC | #3

It all depends on if said generality is even wanted. There is a gigantic threshold between being able to introspect the inputs or parent transactions in a meaningful way or not. It's like two different universes.

The point of this exercise if you like, is to show what you can do by staying within the exact set of fields `CTV` commits to, just with more flexibility or programmability than you get with something like a singular use of `TXHASH`.

And you get a lot of utility out of it without really violating said threshold boundaries.

**edit:** See https://delvingbitcoin.org/t/optimistic-zk-verification-using-matt/1050/1!

-------------------------

harding | 2024-12-31 20:04:34 UTC | #4

[quote="moonsettler, post:1, topic:1344"]
* `n = 0` return the SUM of all input amounts with same script
* `n < 0` return the SUM of last abs(n) input amounts including current input
* `n > 0` return the SUM of first n input amounts of the transaction
[/quote]

Does `OP_INPUTAMOUNTS` return values over 31 bits (~21 BTC)?  Does `OP_TEMPLATEHASH` accept input amounts over 31 bits?

-------------------------

moonsettler | 2025-01-01 00:19:49 UTC | #5

`OP_INPUTAMOUNTS` would return up to total supply cap. Now if it does it in a compact form or always 64 bits padded with 0, is an open design decision. The former would allow for amount arithmetic in the sub 42 btc TX size. Which some might like, others might not like.

`OP_TEMPLATEHASH` would accept amounts up to 64 bits. And would generally accept all numeric inputs in compact int form.

-------------------------

harding | 2025-01-01 02:39:58 UTC | #6

Allowing amount arithmetic for a limited range of values seems unergonomic in the best case (wallet devs have to write code to prevent overflows) and a footgun in the worst case (wallet devs don't write correct code addressing overflows and money can be stolen by [fee siphoning](https://diyhpl.us/~bryan/irc/bitcoin/bitcoin-dev/linuxfoundation-pipermail/lightning-dev/2020-September/002796.txt) attacks or destroyed by accident).

Although I take no position on the desirability of these opcodes, if they are to be added, I would favor a design that either doesn't allow arithmetic or which is compatible with a previously or simultaneously activated soft fork adding 64-bit operators.

-------------------------

moonsettler | 2025-01-01 14:55:09 UTC | #7

I also don't think we have another instance in bitcoin where some feature works for small amounts but not for large amounts. It would be weird to say the least. But it could also be useful and we would need to put an explicit effort into preventing it.

I would be okay with either decision, personally I think if we want amount arithmetic we should add support for bigint math in script! It could also be achieved later with GSR.

For now in first evaluation I would recommend ignoring the possibility of using OP_ADD & OP_SUB and maybe revisit that later on.

-------------------------

JeremyRubin | 2025-01-03 15:39:46 UTC | #8

OP_TEMPLATEHASH actually seems pretty nice to work with on the whole. It "probably" gives you something like OP_PAIRCOMMIT if you were to abuse e.g. the outputs field... but I don't think that's the most interesting way to consider it.

I think it's more interesting to compare this to e.g. working with TXHASH or OP_CAT, and it seems like it's pretty ergonomic! I would maybe consider enabling a way to pass a sha256 midstate in for the sequence and output (maybe a negative index means n-more after midstate?), because then you can do more compact stuff...

I would also consider an OP_PUSHTEMPLATE and OP_TEMPLATEHASH which would decrease txn size, rather than needing to replicate the entire tx on the witness stack.

(less sure about the input amount opcode, holding comment for now)/

-------------------------

1440000bytes | 2025-01-03 16:08:13 UTC | #9

[quote="JeremyRubin, post:8, topic:1344"]
OP_TEMPLATEHASH actually seems pretty nice to work with on the whole. It “probably” gives you something like OP_PAIRCOMMIT if you were to abuse e.g. the outputs field… but I don’t think that’s the most interesting way to consider it.
[/quote]

It is interesting and could make CTV more complex, potentially turning it into a candidate for bikeshedding, which had been lacking until now because everyone considered it simple and well-reviewed.

-------------------------

moonsettler | 2025-01-04 23:15:55 UTC | #10

At this stage all the "bikeshedding" is about extensions to CTV. Which also means it's the lowest common denominator.

-------------------------

cguida | 2025-01-03 21:44:28 UTC | #11

>By doing this it submits to the frame that we should be regulating how users want to specify spending conditions

While I understand the impulse to want to give developers as much freedom as possible to make more expressive scripts, I don't think "make bitcoin maximally expressive" is the correct frame, either.

Ethereum has maximally expressive contracts, and what has it gotten them?

Clearly there are certain behaviors we want to avoid if we don't want bitcoin to end up like Ethereum. I'm sure you would agree that we should never make bitcoin script Turing complete, for instance.

One of the most important tasks ahead of us is to define where this line is and what constitutes stepping over it, so we can be confident that the new behavior we enable is net positive for bitcoin.

Then, I think the optimal strategy would be to get as close to this line as possible, without stepping over.

-------------------------

