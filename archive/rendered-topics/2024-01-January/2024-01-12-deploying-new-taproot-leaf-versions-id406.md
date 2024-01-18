# Deploying new taproot leaf versions

Chris_Stewart_5 | 2024-01-12 16:14:14 UTC | #1

Hi all,

[I currently have another consensus proposal to add 64bit arithmetic op codes to the Script Interpreter](https://delvingbitcoin.org/t/64-bit-arithmetic-soft-fork/397/18). It has been suggested that an optimal way to deploy this feature is with a new taproot leaf version. 

I have been unable to find any examples in draft consensus PRs on github of deploying _new_ leaf versions (please link me if they exist!). This post is intended to talk about what different leaf versions implemented in the interpreter would look like. 

[From BIP341](https://github.com/bitcoin/bips/blob/deae64bfd31f6938253c05392aa355bf6d7e7605/bip-0341.mediawiki#cite_note-7)

> **[^](https://github.com/bitcoin/bips/blob/deae64bfd31f6938253c05392aa355bf6d7e7605/bip-0341.mediawiki#cite_ref-7-0)** **What constraints are there on the leaf version?** First, the leaf version cannot be odd as *c[0] & 0xfe* will always be even, and cannot be *0x50* as that would result in ambiguity with the annex. In addition, in order to support some forms of static analysis that rely on being able to identify script spends without access to the output being spent, it is recommended to avoid using any leaf versions that would conflict with a valid first byte of either a valid P2WPKH pubkey or a valid P2WSH script (that is, both *v* and *v | 1* should be an undefined, invalid or disabled opcode or an opcode that is not valid as the first opcode). The values that comply to this rule are the 32 even values between *0xc0* and *0xfe* and also *0x66*, *0x7e*, *0x80*, *0x84*, *0x96*, *0x98*, *0xba*, *0xbc*, *0xbe*. Note also that this constraint implies that leaf versions should be shared amongst different witness versions, as knowing the witness version requires access to the output being spent.

Lets arbitrarily choose `0x66` as our leaf version.

My understanding it is capable to alter the _interpretation of op codes_ in `interpreter.cpp` based on what the leaf version is. 

Here is my toy example. I decided to disable OP_CHECKSIGADD for leaf version `0x66` in my code. Setting aside the functionality of this new leaf version (i'm not advocating for disabling OP_CSA), is this how new leaf versions were intended to be used?

https://github.com/Christewart/bitcoin/tree/2024-01-12-new-tapleaf-ver-example

-------------------------

halseth | 2024-01-18 07:56:00 UTC | #2

Take a look at the commit I linked from my reply in the 64-bit discussion: https://delvingbitcoin.org/t/64-bit-arithmetic-soft-fork/397/18?u=halseth

I basically comes down do defining a new `SigVersion`. This value the interpreter already has access to, hence can choose to behave differently for certain opcodes based on the version.

-------------------------

