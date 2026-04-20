# Eltoo State Chain on Signet Again: Three Rounds, Two Transactions (CSFS + CTV, with rekey and ladder, no CAT)

AaronZhang | 2026-04-20 15:41:49 UTC | #1

Last week I posted [Eltoo State Chain on Signet: Three Rounds, Six Transactions (APO+CTV)](https://delvingbitcoin.org/t/eltoo-state-chain-on-signet-three-rounds-six-transactions-apo-ctv/2413). This post is a counterpart to that experiment – same 3-round Alice/Bob/Carol scenario, switching the primitive to CSFS+CTV. The result is two on-chain transactions instead of six. No new opcode, no OP_CAT – only CSFS and CTV, both already activated on Inquisition signet.

## What I did

Encoded 3 rounds of state progression into a single CSFS delegation chain (a ladder), with CTV enforcing the final 3-output settlement:

```
State 0: Alice 50,000  Bob 30,000  Carol 20,000  (initial)
State 1: Alice 45,000  Bob 35,000  Carol 20,000  (Alice pays Bob)
State 2: Alice 40,000  Bob 35,000  Carol 25,000  (Alice pays Carol)
```

### Tapscript (42 bytes)

```
<K_pub_32> OP_CHECKSIGFROMSTACK OP_VERIFY
           OP_CHECKSIGFROMSTACK OP_VERIFY
           OP_CHECKSIGFROMSTACK OP_VERIFY
           OP_CHECKSIGFROMSTACK OP_VERIFY
           OP_CHECKTEMPLATEVERIFY
```

4x CSFS (3 delegation hops + 1 settlement signing) + 1x CTV (output enforcement).

Flat. No stack manipulation opcodes.

### Mechanics: rekey + ladder

```
K (channel root, hardcoded in script)
  |  K signs R_0's pubkey            <- rekey: K hands authority to R_0
  v
R_0   R_i = SHA256(K_secret || "state_i")
  |  R_0 signs R_1's pubkey          <- ladder hop 1
  v
R_1
  |  R_1 signs R_2's pubkey          <- ladder hop 2
  v
R_2
  |  R_2 signs CTV template hash      <- terminal: state <-> outputs
  v
CTV enforces 3-output payout          <- consensus-level
```

All delegation happens **off-chain**. Only the final spend goes on chain, carrying the entire ladder proof in the witness.

## On-chain results

| Phase | TxID | Notes |
|----|----|----|
| Fund | [`92efc475...531fda34`](https://mempool.space/signet/tx/92efc47554d25d74d9f567594d37375d8c8a1a2ea6bd1864600d5a49531fda34?showDetails=true) | 101,000 sats locked to ladder+CTV address |
| Spend | [`b96324da...238382de`](https://mempool.space/signet/tx/b96324da612950339f564fbc88fe1d5c5751070e57bf796d32ee7171238382de?showDetails=true) | 3-output settlement, CTV-enforced |

The spend transaction has **3 outputs** (click the TxID to verify):

```
vout 0: Alice  40,000 sats
vout 1: Bob    35,000 sats
vout 2: Carol  25,000 sats
Fee:            1,000 sats
```

### Witness layout (12 data elements, 512 bytes)

```
Idx   Size  Purpose
[0]   32B   CTV template hash (survives all CSFS, consumed by final CTV)
[1]   64B   R_2 signature over CTV hash (CSFS #4)
[2]   32B   CTV template hash (CSFS #4 message)
[3]   32B   R_2 pubkey
[4]   64B   R_1 -> R_2 delegation signature (CSFS #3)
[5]   32B   R_2 pubkey (CSFS #3 message)
[6]   32B   R_1 pubkey
[7]   64B   R_0 -> R_1 delegation signature (CSFS #2)
[8]   32B   R_1 pubkey (CSFS #2 message)
[9]   32B   R_0 pubkey
[10]  64B   K -> R_0 delegation signature (CSFS #1)
[11]  32B   R_0 pubkey (CSFS #1 message)
```

Each intermediate key R_i appears **twice**: once as the message verified by the previous CSFS, once as the pubkey for the next CSFS. With no OP_DUP in the script, the duplication lives in the witness layout – the script stays flat at 42 bytes.

The CTV hash also appears twice: \[2\] is the message R_2 signs (consumed by CSFS #4), \[0\] survives at the bottom for the final CTV opcode to check against actual transaction outputs.

### Script bytes on mempool.space

```
20 ff1f9fa3...9986b8    OP_PUSHBYTES_32 (K's pubkey)
cc                    OP_RETURN_204   (= CSFS on Inquisition)
69                    OP_VERIFY
cc 69                 CSFS + VERIFY   (x3 more)
cc 69
cc 69
b3                    OP_NOP4         (= CTV on Inquisition)
```

## APO+CTV vs CSFS+CTV side-by-side

|  | APO+CTV ([prior post](https://delvingbitcoin.org/t/eltoo-state-chain-on-signet-three-rounds-six-transactions-apo-ctv/2413)) | CSFS+CTV (this post) |
|----|----|----|
| States | 3 | 3 |
| **On-chain txs** | **6** | **2** |
| State mechanism | APO signature rebinding (1 tx per state) | CSFS rekey + ladder (all states in 1 witness) |
| Settlement | CTV (3 outputs) | CTV (3 outputs) |
| Payout | 40k / 35k / 25k | 40k / 35k / 25k |
| Script | \~35B per leaf | 42B single leaf |
| Witness per spend | \~150B | 512B (carries full state history) |
| Opcodes | APO + CTV | CSFS + CTV |

## Negative tests

Run via `testmempoolaccept` (not broadcast):

| Test | Intent | Result |
|----|----|----|
| stale_state | R_0 signs State 2’s CTV hash | **Rejected**: invalid Schnorr signature |
| broken_chain | Skip R_1 (K → R_0 → R_2) | **Rejected**: invalid Schnorr signature |

The CSFS ladder enforces the complete delegation chain. Stale keys can’t authorize newer settlements; broken chains fail signature verification.

## Notes on prior work

This construction follows two key lines of prior work:

* The *eltoo* protocol framing (Decker et al., [2018](https://blockstream.com/eltoo.pdf)), which separates the update mechanism from the underlying primitive.
* The CSFS+CTV rekey/ladder pattern described by Rubin & Rearden ([2024](https://rubin.io/bitcoin/2024/12/02/csfs-ctv-rekey-symmetry/)).

This post focuses on an implementation of that construction on Inquisition signet, with some engineering refinements, and on-chain validation.

## Code

Reproduction code: [`btcaaron/examples/eltoo`](https://github.com/aaron-recompile/btcaaron/tree/main/examples/eltoo) – the full flow that produced the TxIDs above. The companion APO+CTV variant from the previous post is at [`btcaaron/examples/braidpool/rca_eltoo_chain.py`](https://github.com/aaron-recompile/btcaaron/tree/main/examples/braidpool).

Reproducing end-to-end is non-trivial – if you want to run through it and hit something, please reach out. I’m happy to walk through it with anyone who actually wants to verify the construction independently.

---

*Previously: [Eltoo State Chain on Signet: Three Rounds, Six Transactions (APO+CTV)](https://delvingbitcoin.org/t/eltoo-state-chain-on-signet-three-rounds-six-transactions-apo-ctv/2413)*

-------------------------

