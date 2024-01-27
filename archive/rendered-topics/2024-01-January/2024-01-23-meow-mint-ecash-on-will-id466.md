# MEOW: Mint eCash On Will

1440000bytes | 2024-01-23 04:21:49 UTC | #1

<h2>Problem:</h2>

Chaumian eCash used in cashu and fedimint is custodial.

<h2>Research:</h2>
 
I researched [HTLC](https://en.bitcoin.it/wiki/Hash_Time_Locked_Contracts), [Chaumian eCash](https://www.youtube.com/watch?v=VwMzNE1D3so) and [Hawala](https://en.wikipedia.org/wiki/Hawala) to use certain procedures for solving this problem.

[url=https://imgbb.com/]![](upload://lWZD1wBsT0F8KVitJWDRS1qeyI2.png)[/url]

@moonsettler has documented a process to use [eCash without custodial risk](https://gist.github.com/moonsettler/42b588fa97a1da3ac0adea0dd16dadf2) which helped me understand things better and come up with a solution. @ZmnSCPxj suggested a few ideas in a [tweet thread](https://x.com/jxpcsnmz/status/1748156002619588791?s=20). They could also be used to research further and implement better solutions.

<h2>Solution:</h2>

1. Bob mints eCash tokens for Alice and adds (Bob, Carol, Dave) as redeemers in it. 

> A fork of [moksha](https://github.com/ngutech21/moksha) could be used for mint because it supports on-chain minting and melting

2. Alice creates HTLC with a long preimage secret, her pubkey and redeemers pubkey. She shares `BOB123` part of preimage secret with Bob, `CAROL456` with Dave and `DAVE789` with Eve.

```
import hashlib

from bitcoin import SelectParams
from bitcoin.core import Hash160
from bitcoin.core.script import CScript, OP_HASH160, OP_SHA256, OP_EQUALVERIFY, OP_0, OP_DUP, OP_IF, OP_ELSE, OP_ENDIF, OP_CHECKSEQUENCEVERIFY, OP_DROP, OP_EQUALVERIFY, OP_CHECKSIG
from bitcoin.wallet import P2WSHBitcoinAddress

SelectParams('signet')

preimage_secret = b"BOB123CAROL456DAVE789EVE012"
preimage = hashlib.sha256(preimage_secret).digest()

pubkey_alice = bytes.fromhex("03b8821767bb83f8928a25735d94ddd2f1f481ed82d3a05e8decdc926bac2e185a")
pubkey_redeemer = bytes.fromhex("03ee8232b06b14d1f996350345a51b715e3df265fec3209b73930dbe3f890f5403")

redeem_script = CScript([
    OP_IF,
        OP_SHA256, preimage, OP_EQUALVERIFY, OP_DUP, OP_HASH160, Hash160(pubkey_redeemer),
    OP_ELSE,
        2016, OP_CHECKSEQUENCEVERIFY, OP_DROP, OP_DUP, OP_HASH160, Hash160(pubkey_alice),
    OP_ENDIF,
    OP_EQUALVERIFY,
    OP_CHECKSIG,
])

redeem_script_hash = hashlib.sha256(redeem_script).digest()
script_pubkey = CScript([OP_0, redeem_script_hash])
address = P2WSHBitcoinAddress.from_scriptPubKey(script_pubkey)
print(f"HTLC Address: {address}")
```

3. Alice gets the HTLC address. She funds it with some bitcoin based on eCash minted, transaction fees, redeem fees etc.

4. Alice uses eCash to pay Eve, shares 3 sha256 hashes and `EVE012` secret along with eCash:

  ```
  Bob: c6f3e97b0314172f37f84b4383aed77532aa0ed46fe2a59479b81d24148dfe2e
  Carol: a76a9c3dd985816d90f66aa679f165c53cbfdc7b26ee9395894a9c31797c3e53
  Dave: b734eb0b425d9bb099506e18ebcb50f904d07de94f9d9b9ba0b56d54cc33a3a0
  ```
5. Eve verifies that eCash has not expired (2016 blocks). She accepts eCash payment and tries to re-issue eCash using it. Frank mints eCash for her and adds (Frank, Gracy, Henry) as redeemers in it.

6. Eve creates HTLC with a new `preimage_secret`: `FRANK345GRACY678HENRY901IAN123`, her pubkey and redeemers pubkey. She shares the HTLC address with Bob.

7. Bob creates a transaction to spend the HTLC that was created by Alice earlier and adds 4 outputs in it (1 HTLC created by Eve and others to pay redeem fees to self, Carol and Dave). He shares redeem script and signature with Eve after coordinating with other redeemers to get partial preimage.

*If Bob, Carol or Dave are unable to coordinate and complete the preimage required for next steps then eCash would be invalidated and Alice gets back locked bitcoin after 2016 blocks.*

```
txid = "1111111111111111111111111111111111111111111111111111111111111111"
vout = 0

amount_locked = int(1 * COIN)
redeem_fee_bob = int(0.001 * COIN)
redeem_fee_carol = int(0.0005 * COIN)
redeem_fee_dave = int(0.0005 * COIN)
total_redeem_fee = redeem_fee_bob + redeem_fee_carol + redeem_fee_dave
amount_eve_htlc = int(amount_locked - (total_redeem_fee + (0.0001 * COIN)))

txin = CMutableTxIn(COutPoint(lx(txid), vout))
txout_bob = CMutableTxOut(redeem_fee_bob, P2WPKHBitcoinAddress("tb1qq4kmxcjgfmeg2psvth2c905np4pctdeh8rd34x").to_scriptPubKey())
txout_carol = CMutableTxOut(redeem_fee_carol, P2WPKHBitcoinAddress("tb1qnef4evey7592xh56sdsenjr00c7vh2afh2947k").to_scriptPubKey())
txout_dave = CMutableTxOut(redeem_fee_dave, P2WPKHBitcoinAddress("tb1qe5hm92mmgfgflj8l3hrwe03zfme79pmkqr4x53").to_scriptPubKey())
txout_eve_htlc = CMutableTxOut(amount_eve_htlc, P2WPKHBitcoinAddress("tb1qcgrvanjt2y9tckvermnanwsex062vuhq3yqxss").to_scriptPubKey())
tx = CMutableTransaction([txin], [txout_bob, txout_carol, txout_dave, txout_eve_htlc])

sighash = SignatureHash(
    script=redeem_script,
    txTo=tx,
    inIdx=0,
    hashtype=SIGHASH_ALL,
    amount=amount_locked,
    sigversion=SIGVERSION_WITNESS_V0,
)

sig = CBitcoinSecret("cSCdFuVUYAEDhakyRJr4PXwhfwysnfF6khRkdyLnZPRyKFMjiVLq").sign(sighash) + bytes([SIGHASH_ALL])

```

8. Eve adds `preimage_secret` required for witness and broadcasts the transaction

```
witness = CScriptWitness([sig, pubkey_redeemer, preimage_secret, b'\x01', redeem_script])
tx.wit = CTxWitness([CTxInWitness(witness)])

print("Serialized transaction: \n{}".format(b2x(tx.serialize())))
```

*If Frank, Gracy or Henry are unable to coordinate and complete the preimage required for next steps then eCash would be invalidated and Eve gets back locked bitcoin after 2016 blocks.*

<h2>Conclusion:</h2> This protocol can be used for decentralized eCash and isn't custodial. There are some tradeoffs and it could be improved further. If this protocol works, I would recommend everyone involved in minting and redeeming to stay anonymous. Feel free to comment with improvements or corrections in the suggested solution.

-------------------------

ursuscamp | 2024-01-26 21:25:44 UTC | #2

Thanks for this proposal. I've read it several times and I have some questions (maybe more to come):

1. What problem is this proposal intending to solve? Scaling, privacy, or something else?
2. Who are all of these parties in the exchange? Alice seems to be paying Eve, so that makes some sense, but are the other the equivalent of money brokers/Hawaladars? Is their reputation backing up the ecash in some way for this protocol?
3. In step 1 and 2 you mention the "redeemer's" pubkey, but there are four redeemers: Bob, Carol, Dave and Eve. Is this some kind of MPC-type pubkey construction, or is it a pubkey for which they all know the private key?

-------------------------

1440000bytes | 2024-01-26 22:27:20 UTC | #3

1. At no point in this protocol issuers or redeemers have custody of users funds. This is the main problem with other eCash implementations which its trying to solve and still provide privacy.

[quote="1440000bytes, post:1, topic:466"]
Chaumian eCash used in cashu and fedimint is custodial.
[/quote]

2. Yes, others involved in this process could be compared to hawaldars but they are just coordinators that do not have the custody of users funds. No, their reputation or identity is not required in the protocol.

3. There are 3 redeemers: Bob, Carol and Dave. Public key for any of the redeemer could be used because you need more than just signature to spend i.e. `preimage_secret`. In this example it was Bob's key.

---

HTLCs are moving from a group of coordinators to another. On-chain transactions cannot be used to trace sender and recipient of eCash payment. Alice could also pay Eve using multiple eCash tokens minted from different issuers, it will just require parallel coordination with others which can be automated and abstracted away from users.

-------------------------

