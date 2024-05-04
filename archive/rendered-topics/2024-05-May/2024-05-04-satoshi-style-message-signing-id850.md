# Satoshi Style Message Signing

satsie | 2024-05-04 02:57:32 UTC | #1

Right now the most common way to sign general messages is still with a private key that corresponds to a legacy P2PKH address. Many hardware and software projects have implemented this including Ledger, Trezor, Coldcard, and Sparrow, as well as a few multisig companies. One reason why they are so useful is they help users test that the hardware controlling their keys has not succumbed to bit rot.

This technique for message signing comes from Bitcoin Core code that Satoshi wrote (see `src/util/message.cpp` [[link](https://github.com/bitcoin/bitcoin/blob/v27.0/src/util/message.cpp)]). This was created before the BIP process, so the best technical documentation I have found, aside from the actual implementation, is [this page](https://en.bitcoin.it/wiki/Message_signing) on the Bitcoin Wiki.

While comprehensive, the Bitcoin Wiki page doesn’t make some very important technical details as obvious as they should be. Examples include:

* The “Bitcoin Signed Message:\n” magic bytes. I think this is alluded to in the [Displaying signed messages](https://en.bitcoin.it/wiki/Message_signing#Displaying_signed_messages) section. But the actual magic bytes are never stated on the page.
* The double hashing that is performed on each message. See https://github.com/bitcoin/bitcoin/blob/v27.0/src/hash.h#L111 

![|624x148](upload://dnTMJ1B6hT9wQ0m41eU8ToHCyEi.png)

This wiki page also covers the rules for the signature header byte (see `src/key.cpp::SignCompact()`) but I found [this post](https://bitcoin.stackexchange.com/questions/83035/how-to-determine-first-byte-recovery-id-for-signatures-message-signing) on Bitcoin Stack Exchange did a much better job explaining this tricky part.

## Related work & message signing in the wild

There has been lots of good work to address message signing for other address types, including BIP-137, [BIP-notatether-messageverify](https://notatether.com/notabips/bip-notatether-messageverify/), and BIP-322, but I have yet to find a single reliable source of documentation on the “Satoshi format” of message signing described above.

This baffles me because there are plenty of projects that have implemented this. Apart from the examples listed earlier, we also have things like:

* [Bitcoin Message Tool](https://github.com/shadowy-pycoder/bitcoin_message_tool) [January 2023, likely built on [this project](https://github.com/stequald/bitcoin-sign-message/blob/b5e8b478713fdbb8d65e6cd1aeff8b2d5545ff91/signmessage.py#L271) and [this one](https://github.com/nanotube/supybot-bitcoin-marketmonitor/blob/master/GPG/local/bitcoinsig.py)]
* https://checkmsg.org/ [another verification tool, this time from CoinKite]

It seems like plenty of people know how to do this, so either I’m bad at reading the material we have today, or those that have come before me have spent a substantial amount of time and effort figuring it out. Can someone help me tease apart the information I have been able to gather here? Is the "Satoshi format" of message signing fully documented anywhere other than the code?

-------------------------

sipa | 2024-05-04 04:50:53 UTC | #2

The message signing feature and encoding wasn't designed by Satoshi, but by me, so you can direct all blame for quirks and lack of documentation here. You're right that it predates the BIP process, so the code is really the specification I'm afraid, though with many reimplementations there are probably several languages to choose from.

It is roughly:
* Serialize the string "Bitcoin Signed Message:\n" plus the message being signed in the P2P protocol serialization format (which means: prepending each with a [CompactSize](https://en.bitcoin.it/wiki/Protocol_documentation#Variable_length_integer)-encoding of the number of bytes that follow)
* Double-SHA256 hash that serialization.
* Construct an ECDSA signature with that double-SHA256 as hashed message $z$, and the address' key as private key $d$ (with corresponding public key $Q$), resulting in two integers $(r, s)$.
* That signature is encoded as a "recoverable signature", which consists of a header byte plus 32-byte big-endian encodings of $r$ and $s$. This header bytes is there to assist the verifier in recovering the public key from the signature. It can be computed by running the verification algorithm:
  * Let $u_1 = s^{-1}z\, (\operatorname{mod}\, n)$
  * Let $u_2 = r^{-1}z\, (\operatorname{mod}\, n)$
  * Let $R = u_1G + u_2Q$
  * From the result the "recid" can be derived:
    * The X coordinate of $R$ must now either be $r$ (recid=0 or 1) or $r + n$ (recid=2 or 3).
    * The Y coordinate of $R$ must be even (recid=0 or 2) or odd (recid=1 or 3)
  * The header byte equals the recid + 27 for uncompressed $Q$ and recid + 31 for compressed $Q$.

-------------------------

ajtowns | 2024-05-04 07:45:10 UTC | #3

I've also been confused at the lack of docs. Might be good to at least update the wiki with the extra detail? sipa's PR is:

https://github.com/bitcoin/bitcoin/pull/524

Forum discussion linked from there at https://bitcointalk.org/index.php?topic=6428.0;all is probably also interesting.

-------------------------

