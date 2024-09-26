# Efficient Multi-Input Transaction Grinding for OP_CAT-based Bitcoin Covenants

sCrypt-ts | 2024-08-20 09:32:12 UTC | #1

In a [previous post](https://scryptplatform.medium.com/trustless-ordinal-sales-using-op-cat-enabled-covenants-on-bitcoin-0318052f02b2), we have implemented Bitcoin covenants using OP_CAT, based on [this Schnorr trick](https://www.wpsoftware.net/andrew/blog/cat-and-schnorr-tricks-i.html). We can greatly simplify signature calculation (**s**) by choosing specific signing key and ephemeral key.

> ***s = 1 + e***

Bitcoin Script, more specifically **OP_ADD**, only works on 32-bit signed integer and thus cannot directly add 1 to **e** by using the following script, because **e** is a 256-bit integer. It is the SHA256 hash of the transaction data among other things.

`<1> OP_ADD`

The proposed workaround is to break **e** into two parts: the Least Significant Byte (e[-1])¹ and the rest (e_).

> ***e = e_ || e[-1]***

**||** denotes concatenation, i.e., OP_CAT.

![|700x700](upload://jB4aTVqr4ngZwGbV4INlUsMaxwD.jpeg)

We keep changing the transaction till its hash ends in 1. This is akin to proof of work in Bitcoin mining, but with a negligible difficulty. Typically, we can malleate the *nLockTime* or *nSeq* field of a transaction, since slightly changing them generally do not affect the validity of the transaction. Then **s** is simply

> ***s = e_ || 2***

# The Problem

This transaction grinding is blazingly fast, since each hashing try has *1/256* chance of success. It only takes *256* tries on average and completes in milliseconds on a personal computer. Problem arises when the transaction involves multiple inputs (**N**) using covenants.

To see why, note **e** value differs from input to input when computing signatures. That is why each input in a transaction has to be individually signed. An *nLockTime* value makes **e** for one input end with 1 may make it end with 5 for another input. The probability of finding a common *nLockTime* value that makes **e**’s in all inputs end with 1 is now

> ***(1/256)^N***

In many applications that require contract interaction and complex business logic, **N** could easily be 4 or 5. Now it could take minutes to finish the grinding. If **N** is even larger like 10, it is computationally infeasible.

Besides inefficiency, another issue is the limiting range of mutable fields. For example, ***nLocktime*** is only 32 bit long and can only support **N** up to 4. This issue cannot be circumvented if ASICs are used for the hashing.

# The Solution

To address this, we leverage Script’s ability to perform arithmetic on signed 32-bit integers. Instead of limiting **e[-1]** to a specific value like 1, we allow it to be in a much wider range.

If the range is extended to non-negative integers, **e[-1]** can be any integer other than 127, which causes overflows. Now we can use `OP_ADD` to calculate **(e[-1]** + 1)

`<1> OP_ADD`

**s** becomes

> ***s = e_ || (e[-1] + 1)***

The following sCrypt code demonstrates this process.

```ts
@method()
static checkSHPreimage(shPreimage: SHPreimage): Sig {
    // Check e[-1] is in range.
    assert(shPreimage.eSuffix < 127n, 'e suffix not in range [0, 127)')

    const e = sha256(SigHashUtils.ePreimagePrefix + shPreimage.sigHash)
    assert(e == shPreimage._e + int2ByteString(shPreimage.eSuffix), 'invalid value of _e')

    const s = SigHashUtils.Gx + shPreimage._e + int2ByteString(shPreimage.eSuffix + 1n)
    
    ...
}

```

With this change, the probability of finding a valid *nLockTime increases* to

> ***(127/256)^N ~= (1/2)^N***

Even for **N** of 10, the grinding only takes less than a second.

## Alternative

We can accelerate the grinding by further expanding **e[-1]**’s range to [-126, 126]. That is, we only exclude -127 and 127 to avoid underflows and overflows. Note integer is encoded using [signed magnitude](https://en.wikipedia.org/wiki/Signed_number_representations) in Script. When we increment a negative integer by one, we actually have to do the following.

`<-1> OP_ADD`

To see why, let us look at **e[-1]** of *0x83*, which is treated as *-3* in Script. If we want to it to be 0x84 (interpreted as *-4*) after increment, we subtract 1 from it.

```ts
@method()
static checkSHPreimage(shPreimage: SHPreimage): Sig {
    // Check e[-1] is in range.
    assert(
          shPreimage.eSuffix > -127n && shPreimage.eSuffix < 127n, 
          'e suffix not in range [-126, 127)'
    )

    const e = sha256(SigHashUtils.ePreimagePrefix + shPreimage.sigHash)
    assert(e == shPreimage._e + int2ByteString(shPreimage.eSuffix), 'invalid value of _e')

    // If e[-1] is negative, we have to substract. (e.g. 0x83 - 0x01 = 0x84)
    const sDelta: bigint = shPreimage.eSuffix < 0n ? -1n : 1n;
    const s = SigHashUtils.Gx + shPreimage._e + int2ByteString(shPreimage.eSuffix + sDelta)

    ...
}
```

Now the success probability increases to

> ***(254/256)^N***

Full change can be found in [this Github commit](https://github.com/sCrypt-Inc/cat-contracts/commit/3f48ae33da08046a3c2121083031ef523dd7aef9).

[1] We use python notation here: `my_array[-1]` returns the last element of the array `my_array`.

-------------------------

benthecarman | 2024-09-26 07:49:09 UTC | #2

Instead of using <1/-1> OP_ADD, it'd be more efficient to just push the last byte and last byte +1 in the witness, saves a byte and removes the need to handle if its 1 or -1

-------------------------

