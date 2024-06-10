# Drivechain with and without BIP 300/301

1440000bytes | 2024-06-10 00:27:35 UTC | #1

## What is Drivechain?

Drivechain allows you to create different types of sidechains in which withdrawals are approved by miners. Users have the freedom to experiment with different features by sending bitcoin from main chain to sidechain and withdraw from it later.

Since changing consensus rules on bitcoin will become difficult with time, sidechains seems to be the best way possible to scale, improve privacy, user experience etc.

![8taiv5|658x379](upload://y0wnU8hKdB8PcpwbFjzDxkrKyXz.jpeg)

## Is Drivechain possible without BIP 300 and 301?

Yes, it is possible. A [proof of concept](https://sha-gate-demo.netlify.app/) that uses OP_CAT and some introspection opcodes to achieve this: https://github.com/mr-zwets/upgraded-SHA-gate

Bytecode generated by the compiler has these opcodes:

```plaintext
OP_4 OP_PICK 
OP_0 OP_NUMEQUAL 
OP_IF 
    OP_INPUTINDEX OP_0 OP_NUMEQUALVERIFY 
    OP_0 OP_OUTPUTVALUE 
    OP_0 OP_UTXOVALUE OP_SUB 1027 OP_GREATERTHAN OP_VERIFY 
    OP_0 OP_OUTPUTBYTECODE OP_0 OP_UTXOBYTECODE OP_EQUALVERIFY 
    OP_2DROP OP_2DROP OP_DROP 
    OP_1 
OP_ELSE 
    OP_4 OP_PICK 
    OP_1 OP_NUMEQUAL 
    OP_IF 
        OP_8 OP_PICK 
        OP_10 OP_PICK OP_CAT 
        OP_11 OP_PICK OP_CAT 
        OP_12 OP_PICK OP_CAT 
        OP_13 OP_PICK OP_CAT 
        OP_HASH160 OP_EQUALVERIFY 
        OP_0 OP_2ROT OP_SWAP OP_6 OP_ROLL 
        OP_3 OP_8 OP_ROLL 
        OP_9 OP_ROLL 
        OP_10 OP_ROLL 
        OP_11 OP_ROLL 
        OP_12 OP_ROLL 
        OP_5 OP_CHECKMULTISIGVERIFY 
        OP_BIN2NUM 2808 OP_ADD 
        OP_CHECKLOCKTIMEVERIFY OP_DROP 
        OP_3 OP_PICK OP_SIZE OP_NIP 0x1f OP_NUMEQUALVERIFY 0x0200001f 
        OP_4 OP_ROLL OP_CAT OP_8 OP_CAT 
        OP_TXLOCKTIME OP_8 OP_NUM2BIN OP_CAT 
        OP_ACTIVEBYTECODE 0x2c OP_SPLIT OP_NIP OP_CAT 
        OP_INPUTINDEX OP_0 OP_NUMEQUALVERIFY 
        OP_0 OP_UTXOVALUE 0xe803 OP_SUB 
        OP_0 OP_OUTPUTVALUE OP_NUMEQUALVERIFY 
        0xa914 OP_SWAP OP_HASH160 OP_CAT 
        0x87 OP_CAT 
        OP_0 OP_OUTPUTBYTECODE OP_EQUAL 
        OP_NIP OP_NIP OP_NIP 
```
```plaintext
    OP_ELSE 
        OP_4 OP_PICK 
        OP_2 OP_NUMEQUAL 
        OP_IF 
            OP_1 OP_OUTPOINTTXHASH 
            OP_6 OP_PICK OP_HASH256 OP_EQUALVERIFY 
            OP_0 0x20 OP_NUM2BIN OP_6 OP_PICK 
            0x29 OP_SPLIT OP_DROP 0x0100000001 
            OP_ROT OP_CAT 0xffffffff OP_CAT OP_EQUALVERIFY 
            OP_1 OP_OUTPOINTINDEX OP_0 OP_NUMEQUALVERIFY 
            OP_5 OP_ROLL 0x2b OP_SPLIT OP_NIP 
            OP_3 OP_SPLIT OP_DROP OP_BIN2NUM 
            OP_ROT OP_BIN2NUM OP_GREATERTHAN OP_VERIFY 
            OP_ROT OP_BIN2NUM 
            OP_4 OP_ROLL 
            OP_IF 
                OP_DUP OP_1ADD OP_NIP 
            OP_ELSE 
                OP_DUP OP_2 OP_SUB OP_NIP 
            OP_ENDIF 
            OP_2 OP_NUM2BIN 
            OP_2 OP_SWAP OP_CAT 
            OP_ACTIVEBYTECODE OP_3 OP_SPLIT OP_NIP OP_CAT 
            OP_INPUTINDEX OP_0 OP_NUMEQUALVERIFY 
            OP_0 OP_UTXOVALUE 0xe803 OP_SUB 
            OP_0 OP_OUTPUTVALUE OP_NUMEQUALVERIFY 
            0xa914 OP_SWAP OP_HASH160 OP_CAT 
            0x87 OP_CAT 
            OP_0 OP_OUTPUTBYTECODE OP_EQUAL 
            OP_NIP OP_NIP OP_NIP 
        OP_ELSE 
            OP_4 OP_ROLL 
            OP_3 OP_NUMEQUALVERIFY 
            OP_ROT 0x17 OP_SPLIT OP_BIN2NUM 
            OP_1 OP_OUTPUTVALUE OP_OVER OP_NUMEQUALVERIFY 
            OP_1 OP_OUTPUTBYTECODE 
            OP_ROT OP_EQUALVERIFY 
            OP_ROT OP_BIN2NUM 0xe007 OP_ADD 
            OP_CHECKLOCKTIMEVERIFY OP_DROP 
            OP_ROT OP_BIN2NUM 
            OP_0 OP_GREATERTHANOREQUAL OP_VERIFY 
            OP_0 0x1f OP_NUM2BIN 0x0200001f OP_SWAP OP_CAT 
            OP_ACTIVEBYTECODE 0x23 OP_SPLIT OP_NIP OP_CAT 
            OP_INPUTINDEX OP_0 OP_NUMEQUALVERIFY 
            OP_0 OP_UTXOVALUE OP_ROT OP_SUB 0xe803 OP_SUB 
            OP_0 OP_OUTPUTVALUE OP_NUMEQUALVERIFY 
            0xa914 OP_SWAP OP_HASH160 OP_CAT 
            0x87 OP_CAT 
            OP_0 OP_OUTPUTBYTECODE OP_EQUAL 
            OP_NIP 
        OP_ENDIF 
    OP_ENDIF 
OP_ENDIF
```

In this script some introspection opcodes are used:

```
OP_INPUTINDEX
OP_OUTPUTVALUE
OP_UTXOVALUE
OP_OUTPUTBYTECODE
OP_UTXOBYTECODE

OP_OUTPOINTTXHASH
```

These can be emulated with `OP_CAT`, `OP_CHECKSIGFROMSTACK` and a few other basic opcdoes.

## Drivechain using BIP 300 and 301

[BIP 300](https://github.com/bitcoin/bips/blob/bc520fade5cc838d5c6e9b72d2fc12b691c80125/bip-0300.mediawiki) implmentation introduces an opcode `OP_DRIVECHAIN` and six new blockchain messages:

* M1. Propose New Sidechain
* M2. ACK Proposal
* M3. Propose Bundle
* M4. ACK Bundle
* M5. Deposit -- a transfer of BTC from-main-to-side
* M6. Withdrawal -- a transfer of BTC from-side-to-main

Sidechains are first proposed (with M1), and later acked (with M2) for creation. This process resembles Bip9 soft fork activation.

[BIP 301](https://github.com/bitcoin/bips/blob/bc520fade5cc838d5c6e9b72d2fc12b691c80125/bip-0301.mediawiki) describes bind merged mining and introduces 2 messages:

1. BMM Accept -- How h* enters a main:coinbase. When Mary "accepts" a BMM Request, Mary is *endorsing a side:block*.
2. BMM Request -- Simon offering money to Mary, if (and only if) she will Endorse a specific h*. When Simon broadcasts a BMM Request, Simon is *attempting a side:block*.

h* : sidechain's merkle root hash 

Blind Merged Mining (BMM) allows miners to mine a sidechain, without running its node software (ie, without "looking" at it, hence "blind"). Instead, a separate sidechain user runs their node and constructs the block, paying himself the transaction fees. He then uses an equivalent amount of money to "buy" the right to find this block, from the conventional layer1 Sha256d miners.

## Criticism

Some developers have written about their opinions on Drivechain. Its not possible to cover all of them. I will focus on the [blog post](https://petertodd.org/2023/drivechains) written by Peter Todd in 2023 and Paul Sztorc's [response](https://www.drivechain.info/peer-review/peter-todd-2023.png) to it.

I agree with Paul's response and want to add a few points about mining centralization issues shared in the blog post:

- Stratum v2 does not decentralize anything. It could be considered an improvement over v1 but mining pools can still reject block templates.
- Mining is inherently centralized in bitcoin for different reasons and will remain the same until more pools in different countries, more hardware manufacturers and braidpool gains some traction.
- If covenant proposals are activated instead of BIP 300/301, we would end up with MEV issues that affect mining decentralization more than sidechains.

## Sidechain templates

There are some [sidechains](https://www.drivechain.info/projects/index.html) availalble for testing and you can try them using [launcher](https://www.drivechain.info/releases/index.html). I tried zcash sidechain and like the privacy, UX etc. 

![melt-cast|690x474](upload://1wI34laTUL2UNpe596mY2rFEMbB.jpeg)

I got some coins for the main chain using the [faucet](https://drivechain.live). Did fake coinjoin spending equal amount inputs to my own addresses. Used one of the UTXO to deposit funds from main chain to sidechain. This could be followed by other users as well to avoid privacy issues with the deposit transaction however it can also be deposited directly. My goal was to try [melt-cast](https://www.truthcoin.info/blog/zside-meltcast/) feature and compare it with payment pools that could be enabled with covenants.

![GPPpUcwaEAAkURp|690x389](upload://lPZOxa0d8PRReVE412LcwyPR9rN.jpeg)

![GPPpYHXacAEYRlG|690x390](upload://7We7WLHi6enAn9wkiCtIdWy0ZMp.jpeg)

![GPPplWFawAY_bpG|689x395](upload://40Fw8bDUQ14Lkk0hiGxCuXX9eaB.jpeg)

## Conclusion

Drivechain will be possible on bitcoin with or without BIP 300/301 in a few years. We need to do more research and avoid trusting other opinions to decide if they should be activated with BIP 300/301 for evaluating their scaling, privacy etc. benefits.

-------------------------
