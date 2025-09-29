# Flattening Nested 2-of-2 Of a 1-of-1 And a k-of-n

ZmnSCPxj | 2025-09-28 15:59:18 UTC | #1

Title: Flattening Nested 2-of-2 Of a 1-of-1 And a k-of-n

# Note

It is possible to flatten the below into a single-layer
quorum signing group:

* a 2-of-2 composed of:
  * a normal single signer
  * a k-of-n quorum signer

This is done by simply requiring that the “single signer”
participant holds multiple shares in a larger non-nested
k-of-n group.

To determine the flattened k-of-n and the number of shares
the single signer has:

```
flattened_k = n + 1
flattened_n = 2 * n - k + 1
single_signer_shares = n - k + 1
```

Here are a few concrete examples:

* Example 1
  * Suppose we have:
    * Single Signer Ursula
    * 2-of-3 Signers Alice, Bob, Carol
  * This flattens to 4 of 5 with Ursula holding 2 shares:
    * `flattened_k = 3 + 1 = 4`
    * `flattened_n = 2 * 3 - 2 + 1 = 5`
    * `single_signer_shares = 3 - 2 + 1 = 2`
  * So we have Ursula1, Ursula2, Alice, Bob, and Carol.
    * Ursula1 and Ursula2, plus any 2 of the 3 Alice, Bob,
      and Carol, can form a quorum in the 4-of-5.
    * The group Alice, Bob, and Carol is just 3, and cannot
      form a quorum of 4 to overpower Ursula.
* Example 2
  * Suppose we have:
    * Single Signer Ursula
    * 2-of-5 Signers Alice, Bob, Carol, Dave, Evans
  * This flattens to 6 of 9 with Ursula holding 4 shares:
    * `flattened_k = 5 + 1 = 6`
    * `flattened_n = 2 * 5 - 2 + 1 = 9`
    * `single_signer_shares = 5 - 2 + 1 = 4`
  * So we have Ursula1, Ursula2, Ursula3, Ursula4, Alice,
    Bob, Carol, Dave, Evans.
    * Ursula1, Ursula2, Ursula3, Ursula4, plus any 2 of
      Alice, Bob, Carol, Dave, and Evans, can form a
      quorum in the 6-of-9.
    * The group Alice, Bob, Carol, Dave, and Evans is just
      5, and cannot form a quorum of 6 to overpower Ursula.
* Example 3
  * Suppose we have:
    * Single Signer Ursula
    * 6-of-7 Signers Alice, Bob, Carol, Dave, Evans, Fergus,
      Greg
  * This flattens to 8 of 9 with Ursula holding 2 shares:
    * `flattened_k = 7 + 1 = 8`
    * `flattened_n = 2 * 7 - 6 + 1 = 9`
    * `single_signer_shares = 7 - 6 + 1 = 2`
  * So we have Ursula1, Ursula2, Alice, Bob, Carol, Dave,
    Evans, Fergus, Greg.
    * Ursula1 and Ursula2, plus any 6 of Alice, Bob, Carol,
      Evans, Fergus, and Greg, can form a quorum in the
      8-of-9.
    * The group Alice, Bob, Carol, Dave, Evans, Fergus, and
      Greg is just 7, and cannot form a quorum of 8 to
      overpower Ursula.

## Derivation

The key here is `flattened_k = n + 1`.
This assures us that the group of `n` participants cannot
overpower the priveleged single signer, thus always
requiring the participation of the priveleged single
signer, as in the original 2-of-2 of the single signer
plus the k-of-n quorum signers.

From there, we need to ensure that the group of `n`
participants all just retain having one share in the
group, but the priveleged single signer needs to fill in
more than one share.
As the original is `k`, then the priveleged single
signer logically has to get the difference between
the `flattened_k` and `k`, or in other words,
`single_signer_shares = flattened_k - k = n + 1 - k = n - k + 1`.

`single_signer_shares` cannot be less than that as
then even `k` of the quorum signers plus the single
signer would not even achieve `flattened_k`.
If it were more than that, then the priveleged single
signer could overpower the quorum signers by choosing
less than `k` of the quorum signers to achieve
`flattened_k`.

There are still `n` participants in the quorum signing
group, and the priveleged single signer has
`single_signer_shares`, so adding them together gives us
the total new number of shares for the flattened group:
`flattened_n = n + single_signer_shares = n + n - k + 1 = 2 * n - k + 1`.

## Applications

* Currently we have no proof that FROST-in-MuSig is safe.
  However, in many actual applications where such nesting
  may be desirable, one side is often a single signer.
  * For example, a paranoid user might want to have k-of-n
    of their signing devices, and otherwise connect to the
    Lightning Network, which uses 2-of-2 channels, with
    some LSP as the other signer in the channel.
  * For [MultiChannel](https://delvingbitcoin.org/t/multichannel-and-multiptlc-towards-a-global-high-availability-consistent-partition-tolerant-database-for-bitcoin-payments/1983), Ursula is a priveleged single
    signer that wants to use the same liquidity with the
    flexibility to offer outgoing HTLCs/PTLCs/MultiPTLCs
    amongst N LSPs, with improved availability by allowing
    one or more of the LSPs to go down when Ursula wants
    to make a payment.
* Statechain BS can have its security “improved” by having
  the statechain operator be a k-of-n with the current user
  being a priveleged single signer.
  * Resharing inside a 2-of-2 of a k-of-n is actually
    cryptographically dubious as novel cryptography.
    However, resharing a flat k-of-n is not as novel.
    * There *are* a lot of broken VSS schemes still though.
      But at least you are not adding even more brokenness
      by putting a 2-of-2 on top of a k-of-n and *then*
      resharing.
  * Against this, no amount of invoking the ghost of TEE
    can protect end users against recovery of old backups
    of supposedly-deleted keys.
    Literally my first job was an underpaid third-world
    engineer at just-above-minimum-wage (minumum wage for
    third-worlders, not first-worlders) reverse-engineering
    ICs: opening them up, taking pictures from microscopes,
    carefully removing a metal layer, taking pictures again,
    etc. until you reach the polysilicon and doped silicon
    layers, and then figuring out the overall circuit so that
    the parent company could check if patents were violated
    and sue.
    Yes, you can see burnt fuses that way, it is just
    another metal layer.
    For reference, Intel SGX stores the per-chip hardware
    attestation key as burnt fuses on the integrated circuit,
    and Intel hands over possession of the CPU integrated
    circuit to buyers, who can now open up the IC and
    extract the hardware attestation key to attest to
    *anything they want* and Intel will give its CA to
    support that “remote” hardware attestation key.
    Possession is eleven points of the law, and they say
    there are but twelve.
    Just use a hardware wallet, because in that case, you
    can hire the same team of underpaid third-world
    reverse-engineers to see if the wallet has any hidden
    private keys that the manufacturer embedded.
    Use real-world cut-and-choose (buy 2 HW wallets, mark
    one “heads” the other “tails”, flip a coin, have the
    reverse-engineers work on the losing wallet, if they
    find nothing wrong, that is good evidence you can
    rely on the winning wallet).

-------------------------

AdamISZ | 2025-09-29 13:00:20 UTC | #2

Thanks for the writeup.

A refreshing piece of pure logic instead of that dirty cryptography and coding stuff :slight_smile: 

I agree with your formulas. It’s fairly simple in hindsight, but I’d guess a lot of people might never think of looking for it.

I guess it’s worth mentioning that we are not so fortunate for any other structure, like “A of B and C of D” generalized (the thresholds overlap so you can’t get it to cover all possibilities), or “1 of 1 or A of B” (unless I missed something these are impossible to flatten). As you point out, this particular structure is actually practically useful.

[quote="ZmnSCPxj, post:1, topic:2018"]
Against this, no amount of invoking the ghost of TEE can protect end users against recovery of old backups of supposedly-deleted keys. Literally my first job was an underpaid third-world engineer at just-above-minimum-wage …

[/quote]

Interesting anecdotes. I am against TEEs in principle, you apparently have literally dirtied your hands on the topic, so I guess I’m glad to see you agree. In the absence of such (dirty or otherwise) hands-on, I’ll just keep going with my airy fairy abstract principles and say I don’t really trust hardware wallets not because they’re constructed with physical circuits but because they’re bitcoin specific devices, basically … with or without 1-round cut and choose :laughing:

-------------------------

