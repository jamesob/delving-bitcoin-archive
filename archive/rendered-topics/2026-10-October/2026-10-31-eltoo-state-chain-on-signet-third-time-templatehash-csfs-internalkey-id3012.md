# Eltoo state chain on signet, third time: TEMPLATEHASH + CSFS + INTERNALKEY

AaronZhang | 2026-10-31 23:05:32 UTC | #1

Hi All, I've now built the same little Alice/Bob/Carol channel three times. First with [APO+CTV](https://delvingbitcoin.org/t/eltoo-state-chain-on-signet-three-rounds-six-transactions-apo-ctv/2413), then with a [CSFS+CTV rekey ladder](https://delvingbitcoin.org/t/eltoo-state-chain-on-signet-again-three-rounds-two-transactions-csfs-ctv-with-rekey-and-ladder-no-cat/2430). Inquisition 29.4 turned on [`OP_TEMPLATEHASH`](https://delvingbitcoin.org/t/bitcoin-inquisition-29-4/2739), so I ran it again using the scripts from @instagibbs' [`2026-01-eltoo_th`](https://github.com/instagibbs/bolts/tree/2026-01-eltoo_th) BOLT draft.

```
update(n) = TEMPLATEHASH INTERNALKEY CSFS VERIFY <500000000+n> CLTV   # 10 B
settle(n) = TEMPLATEHASH <hash> EQUAL                                 # 35 B
```

This time I published an old state on purpose, then let a newer update replace it. I signed U3 before any state output existed.

| Step | TxID |
|----|----|
| Fund | [`1163d3d5…9c259124`](https://mempool.space/signet/tx/1163d3d5ae78069dd41ad951de47460358db689b3b5d6b951dbb57399c259124) |
| U1, old state published | [`22c0c67b…02188692`](https://mempool.space/signet/tx/22c0c67b33f6620fe9165dca835408de91f4dd873018ab1c06a26a2f02188692) |
| U3 replaces it, skipping state 2 | [`4a11cfca…dea15007`](https://mempool.space/signet/tx/4a11cfca7e35564f932ee30a66dec9f9bbd95cdbc8ad8b34190b4c8fdea15007) |
| Settle state 3 (40k / 35k / 25k) | [`e7476180…e8001568`](https://mempool.space/signet/tx/e7476180b2f784ce81c859c6a9d48a0f72c0e72937ac1cd37e183ec6e8001568) |

## Three versions side by side

|  | APO + CTV | CSFS + CTV ladder | TH + CSFS + IK |
|----|----|----|----|
| Update leaf | \~35 B | 42 B | 10 B |
| Update witness | \~150 B | 512 B, grows | 143 B, constant |
| Txs in the run | 6 | 2 | 4 (includes a deliberate stale state) |
| What the script gets back | yes/no | yes/no | the hash itself |

## Where the difference comes from

I didn't really get this until I read `interpreter.cpp`. It comes down to what the node hands back to the script.

* **APO:** the rebinding happens inside signature checking. The node just skips the prevout when it builds the sighash. The script only hears "valid" or "not valid".
* **CTV:** it checks the hash and says yes or no. It never gives the hash back. So in the ladder, to get CSFS to sign that hash, I had to put a second copy of it in the witness myself.
* **TEMPLATEHASH:** it puts the hash on the stack. CSFS signs whatever is on the stack, so the update script is just three opcodes.

So moving from CTV to TEMPLATEHASH didn't change much about what gets hashed. What changed is that the script gets the hash back instead of just a yes or no. Once the hash is on the stack, other opcodes can work with it.

It's one key and no HTLCs, so this is a toy channel, not a real one.

Code: [`templatehash.py`](https://github.com/aaron-recompile/btcaaron/blob/5ef4cb3dff5d51d812ff55a134d48c0e269323e7/btcaaron/templatehash.py)  · [`th_csfs_ik_eltoo.py`](https://github.com/aaron-recompile/btcaaron/blob/5ef4cb3dff5d51d812ff55a134d48c0e269323e7/examples/eltoo/th_csfs_ik_eltoo.py)

-------------------------

zixahuxa | 2026-11-03 20:36:35 UTC | #2

The 10-byte update leaf is a neat result, but the practical comparison would be clearer with total spend weight alongside leaf and witness sizes. The fixed Taproot control block and other witness elements can matter more than the script in a small transaction, especially when comparing these different constructions.

-------------------------

