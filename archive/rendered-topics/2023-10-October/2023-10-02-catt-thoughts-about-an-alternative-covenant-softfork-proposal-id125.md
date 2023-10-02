# CATT: thoughts about an alternative covenant softfork proposal

stevenroose | 2023-10-02 00:40:20 UTC | #1

Since I just discovered this forum and some interesting conversations are going on here. I definitely saw James' proposal [here](https://delvingbitcoin.org/t/covenant-tools-softfork/98/5), not trying to undermine that, just trying to give an alternative perspective.

I've been thinking about this idea, which I dub "covenant all the things", which I guess could go a few ways. The idea is to not focus on specific use-cases (APO, VAULT), but provide more general tools that 

A very minimal version would be a combination of 

- OP_TXHASH
  - and OP_CHECKTXHASHVERIFY for templating
- OP_CHECKSIGFROMSTACK
  - together with TXHASH this is very powerful
  - easily emulates APO to build eltoo-style protocols

These two are already very powerful together. It basically introduces a "sighash on steroids" that allows signing off on anything you want. Plus a very flexible templating mechanism that supports bring-your-own-fee constructions.

So, on top of that, to allow real introspection, there are some possibilities:

- the full version would be
  - OP_CAT
    - who doesn't like CATs?
  - OP_TX
    - for full introspection, this works quite well with OP_TXHASH's TxFieldSelector
  - 64-bit arithmetic
    - for anything with amounts
  - OP_TWEAKADD
    - for doing taproot stuff

- a lesser version
  - OP_HASHMULTI (aka OP_CATSHA256) or something
  - OP_TWEAKADD (this seems kinda needed in any case)

One of the nice properties of all of this is that almost all of these opcodes already have production implementations in Elements (Liquid), except for the OP_TXHASH and OP_TX which are closely related. I formalized a potential OP_TXHASH BIP [here](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-September/021975.html) (discussion here on Delving [here](https://delvingbitcoin.org/t/draft-bip-for-op-txhash-and-op-checktxhashverify/121/1)), and OP_TX could be implemented by re-using the TxFieldSelector specified in that BIP.

-------------------------

