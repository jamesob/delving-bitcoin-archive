# Segwit Ephemeral Anchors

instagibbs | 2023-11-02 17:50:55 UTC | #1

Ephemeral Anchors are the concept where we can have a key-less output type that is allowed to have dusty values, provided they are atomically spent in the mempool.

Implementation and BIP text here for more background: https://github.com/bitcoin/bitcoin/pull/26403

One large drawback of this approach is that it relies on Bitcoin's legacy script. This means that the transaction *spending* the anchor is txid-malleable by miners. This is due to legacy script's lack of CLEANSTACK consensus rules. This means composition with other protocols like splicing can be problematic, where the pre-signed transaction chains can be broken unless a new signature type like ANYPREVOUT or similar is used.

At first blush, something like wsh(OP_TRUE) neatly solves this, at the expense of 33 more output vbytes, and a witness script of "OP_TRUE" being revealed. This seems pretty costly, and theoretically could also already be used in mainnet, both for output creation and redemption.

Instead, why not just utilize bip141's softfork for witness programs, and use an otherwise undefined witness program? For example, replace the script "OP_TRUE" with "OP_TRUE 0xffff" which triggers the scriptSig rule in bip141:

> The scriptSig must be exactly empty or validation fails.

But is otherwise undefined, allowing an empty witness to spend. This is 3 additional vbytes over the original proposal total, and is implemented here:

https://github.com/instagibbs/bitcoin/commits/ephemeral-anchors-segwit

Note that these (non-dust)outputs are already standard to *create*, but not to spend, which is the additional relay relaxation in the commit.

The branch will actually get smaller since I'm not messing with OP_TRUE outputs anymore which tests in master love to use.

-------------------------

ajtowns | 2023-11-02 21:44:47 UTC | #2

If you made the two digit code be `0x4e73` instead of `0xffff` that would give it the bech32m address `bc1pfeessrawgf` (ie "bc1p", "fees", checksum).

You can choose any three letters in the bech32 charset followed by either `q` or `s`, followed by the 6 char checksum. (0xffff is `bc1plllsqd8ech`; 0x0100 (1) gives `bc1pqyqqgs7ult`; 0x0001 (256) gives `bc1pqqqs4em24r` ~~and 0x0000 gives `bc1pqqqqqcnjqs`~~)

EDIT: testnet/signet has 4e73 look like `tb1pfees9rn5nz` or with regtest `bcrt1pfeesnyr2tx`. `bitcoin-cli decodescript 51024e73` if you want to try other combinations for something fun.

-------------------------

instagibbs | 2023-11-02 19:51:15 UTC | #3

I mostly did ff's because I realized I couldn't do 0's....

bc10feespdsrh8 (0fees)
bc1z9gfqhanchr (anchr)
bc1zfeesuqhy82 (zfees)

I'm glad we're discussing the real questions left.

good point we'll probably want it obvious in with any checksum due to network/address differences.

-------------------------

ajtowns | 2023-11-02 21:45:36 UTC | #4

Sticking with witness version 1 (taproot-adjacent) and `bc1p` seems like it would be wise, at least.

-------------------------

instagibbs | 2023-11-08 17:25:23 UTC | #5

I'm considering supporting both `OP_TRUE` and `OP_TRUE 0x4e73` at this point. It's a tiny difference, and users can opt into non-segwit versions if they simply don't care about the child transactions' txid stability.

-------------------------

stevenroose | 2023-11-13 05:25:40 UTC | #6

[quote="instagibbs, post:1, topic:160"]
Instead, why not just utilize bip141’s softfork for witness programs, and use an otherwise undefined witness program? For example, replace the script “OP_TRUE” with “OP_TRUE 0xffff” which triggers the scriptSig rule in bip141:
[/quote]

I don't understand what you're doing here. The BIP you link seems to only talk about plain OP_TRUE, right? So the malleable version? What part of the BIP141 are you using to do this?

I don't understand "otherwise undefined witness program". You mean v1 programs that are not 32 bytes and redefine those? But then why would you need OP_TRUE? Can't you simply standardize a v1 program that is 1-byte in size and make it standard if thar 1 byte has a fixed value? Which is anyone can spend anyway, so it's the same as OP_TRUE?

-------------------------

instagibbs | 2023-11-13 14:25:54 UTC | #7

[quote="stevenroose, post:6, topic:160"]
I don’t understand what you’re doing here. The BIP you link seems to only talk about plain OP_TRUE, right? So the malleable version?
[/quote]

That's the malleable version yes.

[quote="stevenroose, post:6, topic:160"]
Can’t you simply standardize a v1 program that is 1-byte in size and make it standard if thar 1 byte has a fixed value?
[/quote]
Because that's not a witness program, meaning it's allowed to have non-empty scriptSig data, making it malleable.

-------------------------

stevenroose | 2023-11-13 15:38:31 UTC | #8

[quote="instagibbs, post:7, topic:160"]
Because that’s not a witness program, meaning it’s allowed to have non-empty scriptSig data, making it malleable.
[/quote]

Heh? Can you re-read my original message? And explain what you are proposing/describing? What I said was to standardize a v1 (taproot-version) witness program of a single byte. So like scriptPubkey of  `OP_1 OP_PUSHBYTES_1 0x42`. I think for consensus any non-32-byte v1 program is anyone-can-spend, no?

-------------------------

