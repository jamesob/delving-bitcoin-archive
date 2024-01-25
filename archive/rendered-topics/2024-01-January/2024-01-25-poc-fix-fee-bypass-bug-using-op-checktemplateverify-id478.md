# PoC: Fix fee bypass bug using OP_CHECKTEMPLATEVERIFY

1440000bytes | 2024-01-25 17:32:41 UTC | #1

<h2>Problem</h2>

HodlHodl uses 2-of-3 multisig for P2P trades and a new multisig address is created for each trade using public keys for buyer, seller and HodlHodl. Buyer and Seller can coordinate with each other during a trade and spend bitcoin locked by seller without paying fees to HodlHodl. There is an open source tool to help users achieve this: https://gitlab.com/hodlhodl-public/escrow_extractor/

<h2>Research</h2>

- HodlHodl Multisig [contract specification](https://gitlab.com/hodlhodl-public/hodl-client-js/-/blob/master/multisig-spec.md)
- OP_[CHECKTEMPLATEVERIFY](https://github.com/bitcoin/bips/blob/deae64bfd31f6938253c05392aa355bf6d7e7605/bip-0119.mediawiki)

> HodlHodl is aware of this bug and acknowledged it in this [tweet](https://x.com/hodlhodl/status/1739363250515406912)

[url=https://ibb.co/2YcQXst]![](upload://5XZqeekOXta3JNKq0tSVBjGW9kr.png)[/url]

<h2>Solution</h2>

1. Seller funds a CTV address with bitcoin that can only be spent to two addresses (2-of-3 multisig and hodlhodl) using `lock_tx`.

   Example: [76b79ff326522dccbe46befe40d7f4e9b66e63695707ae0e11cc4f65f0d1db9d](https://mempool.space/signet/tx/76b79ff326522dccbe46befe40d7f4e9b66e63695707ae0e11cc4f65f0d1db9d)

2. Seller shares `unlock_tx` hex with buyer and HodlHodl.

   Example:
   ```
   02000000019ddbd1f0654fcc110eae075769636eb6e9f4d740febe46becc2d5226f39fb7760000000000ffffffff0268f40c000000000069522103a9a8b3ee7b0fb6d097ca1f878b103c6ebdfdd735b56a7730b5f6f6ffeda5646a2102afabe1ff44b40e775a76bc2e5c11217fa6f47d03eea3d1743894bd15722210bb2102ecfbf9e8cf29422dd809d316f8a21e937c2eaf50864bbc91599734cbd8de080c53ae50c30000000000001600148eeb90fd37f496fd40fadc135939609acc13c90600000000
   ```
3. Buyer sends money to seller's bank account and broadcasts `unlock_tx`. This transaction pays trading fee to HodlHodl in second output and locks left amount in a 2-of-3 multisig.

   Example: [85e1db10c47d222b83ed0b540acbe2568e65ad34f25968725d06d7e7a8c02b1b](https://mempool.space/signet/tx/85e1db10c47d222b83ed0b540acbe2568e65ad34f25968725d06d7e7a8c02b1b)

4. 2-of-3 multisig is spent using 2 keys to send bitcoin to buyer. In case of dispute, HodlHodl decides if it goes back to seller or not.

```
import struct
import hashlib
import sys
import pprint
import typing as t
from dataclasses import dataclass
import hashlib
from bitcoin import SelectParams
from bitcoin.core import (
    CTransaction,
    CMutableTransaction,
    CMutableTxIn,
    CTxIn,
    CTxOut,
    CScript,
    COutPoint,
    CTxWitness,
    CTxInWitness,
    CScriptWitness,
    COIN,
    lx,
)
from bitcoin.core import script
from bitcoin.wallet import CBech32BitcoinAddress, P2WPKHBitcoinAddress, CBitcoinSecret
from buidl.hd import HDPrivateKey, PrivateKey
from buidl.ecc import S256Point

SelectParams('signet')

OP_CHECKTEMPLATEVERIFY = script.OP_NOP4
TX_FEE = 1000
BUY_AMOUNT = int(0.01 * COIN)
HHFEES_AMOUNT = BUY_AMOUNT * 0.05

funding_prvkey = CBitcoinSecret("cTQmtSFJpEYxMPui7LnF6m3gM8DeimmvUbjpGYY47NMK5HZPbAJv")
funding_pubkey = funding_prvkey.pub
funding_address = P2WPKHBitcoinAddress.from_scriptPubKey(CScript([script.OP_0, script.Hash160(funding_pubkey)]))
print("funding address:", funding_address)


def sha256(input):
    return hashlib.sha256(input).digest()


def get_txid(tx):
    return tx.GetTxid()[::-1]


def create_template_hash(tx: CTransaction, nIn: int) -> bytes:
    r = b""
    r += struct.pack("<i", tx.nVersion)
    r += struct.pack("<I", tx.nLockTime)
    vin = tx.vin or []
    vout = tx.vout or []

    if any(inp.scriptSig for inp in vin):
        r += sha256(b"".join(ser_string(inp.scriptSig) for inp in vin))
    r += struct.pack("<I", len(tx.vin))
    r += sha256(b"".join(struct.pack("<I", inp.nSequence) for inp in vin))
    
    r += struct.pack("<I", len(tx.vout))

    r += sha256(b"".join(out.serialize() for out in vout))
    r += struct.pack("<I", nIn)
    return hashlib.sha256(r).digest()


def hodlhodl_template(buy_amount: int = None, hhfees_amount: int = None):
    tx = CMutableTransaction()
    tx.nVersion = 2

    tx.vin = [CMutableTxIn()]

    buyer_pubkey = bytes.fromhex("03a9a8b3ee7b0fb6d097ca1f878b103c6ebdfdd735b56a7730b5f6f6ffeda5646a")
    seller_pubkey = bytes.fromhex("02afabe1ff44b40e775a76bc2e5c11217fa6f47d03eea3d1743894bd15722210bb")
    hh_pubkey = bytes.fromhex("02ecfbf9e8cf29422dd809d316f8a21e937c2eaf50864bbc91599734cbd8de080c")
    multisig_script = CScript([script.OP_2, buyer_pubkey, seller_pubkey, hh_pubkey, script.OP_3, script.OP_CHECKMULTISIG])

    tx.vout.append(CTxOut(buy_amount - 100000, multisig_script))

    hh_witness_program = bytes.fromhex("8eeb90fd37f496fd40fadc135939609acc13c906")
    hh_script = CScript([script.OP_0, hh_witness_program])

    tx.vout.append(CTxOut(hhfees_amount, hh_script))

    return tx


def hodlhodl_tx(amount: int=None, hhfees_amount: int=None, vin_txid=None, vin_index=None):
    tx = hodlhodl_template(amount, hhfees_amount)
    tx.vin = [CTxIn(COutPoint(lx(vin_txid), vin_index), nSequence=0xffffffff)]
    return tx


def ctv_tx(amount: int = None, vin_txid: str = None, vin_index: int = None):
    template = hodlhodl_template(buy_amount=amount - TX_FEE - HHFEES_AMOUNT, hhfees_amount=HHFEES_AMOUNT)
    hodlhodl_ctv_hash = create_template_hash(template, 0)
    tx = CMutableTransaction()

    tx.vin = [CTxIn(COutPoint(lx(vin_txid), vin_index))]
    tx.vout = [CTxOut(amount - TX_FEE, CScript([hodlhodl_ctv_hash, OP_CHECKTEMPLATEVERIFY]))]

    redeem_script = funding_address.to_redeemScript()
    sighash = script.SignatureHash(
        script=redeem_script,
        txTo=tx,
        inIdx=0,
        hashtype=script.SIGHASH_ALL,
        amount=amount,
        sigversion=script.SIGVERSION_WITNESS_V0,
    )
    signature = funding_prvkey.sign(sighash) + bytes([script.SIGHASH_ALL])

    tx.wit = CTxWitness([CTxInWitness(CScriptWitness([signature, funding_pubkey]))])
    return tx


if __name__ == "__main__":
    lock_tx = ctv_tx(
        amount=BUY_AMOUNT,
        vin_txid="6c5cda350a5fb367b90ea18bf442ff73d619acbe3b898e8c11607dd368ac42d2",
        vin_index=1,
    )
    print("\nLOCK TX:", lock_tx.serialize().hex()) 
    
    unlock_tx = hodlhodl_tx(
        amount=BUY_AMOUNT - TX_FEE - HHFEES_AMOUNT,
        hhfees_amount=HHFEES_AMOUNT,
        vin_txid=get_txid(lock_tx).hex(),
        vin_index=0,
    )
    print("\nUNLOCK TX:", unlock_tx.serialize().hex())
```

<h2>Alternatives</h2>

HodlHodl could use 3-of-3 multisig however that would make it custodial and users will not be able to release bitcoin from escrow in case HodlHodl goes down.

<h2>Conclusion</h2>

Using OP_CHECKTEMPLATEVERIFY ensures HodlHodl gets the fee in every trade. This is a proof of concept and could be improved further.

<h2>Acknowledgement</h2>

- [Jeremy Rubin](https://twitter.com/JeremyRubin)
- [HodlHodl](https://hodlhodl.com/)
- [katsu](https://twitter.com/0x0ff_)

-------------------------

