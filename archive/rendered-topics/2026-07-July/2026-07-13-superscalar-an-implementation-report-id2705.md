# SuperScalar: an implementation report

8144225309 | 2026-07-13 07:21:28 UTC | #1

SuperScalar puts many self-custodial Lightning clients behind a single on-chain UTXO, with no soft fork and no trust in the coordinating LSP. It is @ZmnSCPxj's design ([SuperScalar with pseudo-Spilman leaves, t/1242](https://delvingbitcoin.org/t/superscalar-laddered-timeout-tree-structured-decker-wattenhofer-factories-with-pseudo-spilman-leaves/1242)); this post reports a working implementation.

Everything below ran on regtest or signet. There has been no external audit, and mainnet is deliberately behind an explicit `--i-accept-the-risk` flag.

Code (C, MIT, at [v0.2.0](https://github.com/8144225309/SuperScalar/releases/tag/v0.2.0)): [`github.com/8144225309/SuperScalar`](https://github.com/8144225309/SuperScalar). Four binaries: `superscalar_lsp`, `superscalar_client`, `superscalar_watchtower`, `superscalar_bridge`.

## How a channel factory is built

A channel factory is one P2TR funding UTXO whose internal key is a MuSig2 aggregate of the LSP and up to 127 clients (128 signers including the LSP). Every output at every layer can be spent only by the aggregate signature of everyone beneath it, so no subset of participants, the LSP included, can move funds that are not theirs. Even the factory's expiry favors the clients rather than the LSP (the distribution transaction, below). There is no one-honest-member assumption anywhere: from the moment a client is onboarded to the moment it exits, its safety rests on its own key and on consensus-enforced timelocks, not on the honesty of the LSP or of any co-client. The mechanisms of [t/1242](https://delvingbitcoin.org/t/superscalar-laddered-timeout-tree-structured-decker-wattenhofer-factories-with-pseudo-spilman-leaves/1242), walking down the tree:

The smallest canonical shape, for orientation (8 clients, interior arity 2, k=2 pseudo-Spilman sub-factories per leaf, ZmnSCPxj's canonical t/1242 shape; the shipping default is one client per leaf, k=1; L = the LSP):

```
funding UTXO  =  MuSig2(A..H, LSP)
   |
kickoff_root     (n-of-n, no timelock)
   |
state_root       (DW, decrementing nSeq)
   |
   +-- leaf_1  (A-D)
   |    +-- A&B&L or L&CSV -> A-L, B-L
   |    +-- C&D&L or L&CSV -> C-L, D-L
   |    +-- A&B&C&D&L or L&CSV -> L-stock
   +-- leaf_2  (E-H)
        +-- E&F&L or L&CSV -> E-L, F-L
        +-- G&H&L or L&CSV -> G-L, H-L
        +-- E&F&G&H&L or L&CSV -> L-stock
```

**Decker-Wattenhofer interior.** As in the design: kickoff and state layers alternate, and the state transactions carry decrementing `nSequence`. The knobs come from the design's own tree-structuring guidance: arity per layer (`--arity 2,4,8`) and kickoff-only *static near-root* levels. The builder refuses any shape whose worst-case exit runs past BOLT's 2016-block CLTV ceiling. A plain binary tree at 128 signers runs past it (about 3456 blocks); mixed arity plus two static levels brings it down to about 864.

**Timeout-sig-trees, with the inverted expiry default.** Every state output can be spent by the N-of-N key path, or after a timeout by the LSP alone: `or(N-of-N, L&CLTV)`. At expiry the default favors the clients: at ceremony time everyone co-signs a distribution transaction that returns each client's funds, and every party keeps a copy. The LSP's daemon broadcasts it automatically once the factory expires, and sooner if its rotation retries run out; a client can broadcast the very same transaction from its own exit path, so either side can set it off. The CLN plugin does the same, holding the transaction and broadcasting it at `expiry_block` even across a restart: [`9f3e0829`](https://mempool.space/signet/tx/9f3e082943b4133261525d6e98137317535365c8f49f0535c4ae059ac9050997). A factory can also be refreshed in place, the whole tree re-signed with a later CLTV, without anyone going on-chain and without anyone migrating.

**Pseudo-Spilman leaves.** As in the design: an arity-k tree gives k sub-factories of k clients each, and a liquidity sale chains a new transaction onto the sub-factory output rather than decrementing an `nSequence`. So a leaf spends no CSV budget and has no old state to revoke. The code adds a double-sign defense so that no signer can be induced into co-signing two different spends of the same chain parent: each signer records the partial signature it gives and refuses a conflicting one for a `(parent_txid, parent_vout)` it has already signed, a refusal that survives restarts because it lives in the persist database. The cheat drivers exercise it across restart and reorg orderings.

**Laddering.** Rotating a factory is a first-class ceremony: it can run asynchronously, waiting on clients to be ready (proven at 127), and when the root counter rolls over there is a dedicated ceremony to re-sign every leaf. A mass exit at expiry is not N separate fee races. The distribution transaction returns everyone's funds in one go, and because the ladder staggers each factory's deadline, exits across the fleet do not pile onto the same blocks.

**Inside the leaves: ordinary Lightning.** Each leaf holds a normal BOLT-2/3 Poon-Dryja channel with HTLCs; PTLC support is scaffolded but not yet enabled. Per-commitment revocation uses independent flat secrets, on ZmnSCPxj's guidance in [t/1143 #34](https://delvingbitcoin.org/t/superscalar-laddered-timeout-tree-structured-decker-wattenhofer-factories/1143/34). The transport and wire are standard: BOLT-8 Noise_XK (with Tor/SOCKS5), BOLT #2/#4/#11 and #12 offer messages, LSPS0/1/2, MPP.

The rest follows the text as written: the k² sub-factory leaves (opt-in), the static near-root levels, and cut-through when a JIT operation is undone are all straight from [t/1242](https://delvingbitcoin.org/t/superscalar-laddered-timeout-tree-structured-decker-wattenhofer-factories-with-pseudo-spilman-leaves/1242). The messages that carry the ceremonies (the MuSig2 rounds, a ceremony-abort, a bundled re-sign for root rollover, a funding-reorg notice) are de facto for now; they are the raw material for the spec draft mentioned under Status.

## Security and operations

1. **Old state is punished by a poison transaction.** If the LSP broadcasts a stale state, a pre-signed transaction hands that state's L-stock to the clients, so publishing old state costs the LSP more than it can gain. Each poison is gated on a revealed, revocation-style secret, so it can only act against a state that has genuinely been revoked. It is co-signed at every state advance on all four advance paths, and it pays its own fee out of the L-stock it takes, so the cheater funds the penalty. Proven on signet with a real poison broadcast and the L-stock redistributed to the clients (21,085 sats): [`d2ae19cf`](https://mempool.space/signet/tx/d2ae19cf39547b7eb69930ef8ad92e3d7f51e683b02cd8643a74d45640d4a4f0).
2. **Signing keeps no secret nonce on disk.** If a secret nonce were kept and then reused across a ceremony abort, a restart, or a reorg, the signing key would leak. So none is ever written down, and a test fails if a secret-nonce table ever appears in the schema. A ceremony can be aborted cleanly, and a crash injected at each phase; the 20-case drill passes on regtest.
3. **Fees are attached at broadcast time, not baked in at signing.** Every pre-signed transaction carries a keyless 240-sat P2A anchor by default, and whoever broadcasts it raises the fee with a CPFP child; an uncontested exit takes its fee from the output it sweeps. A bumper escalates that child's fee toward a deadline, under set caps. The party whose funds are at risk is the one who runs the bumper and pays for it, and each client can run its own. An LSP may run one for its clients as a convenience, but that is a trust assumption, and I label it as one.
4. **Reorg handling.** If a funding transaction is reorged out, the LSP notifies the client over a dedicated message and both reset their chain state; the watchtower detects forward reorgs; confirmation depths are per-network; and reorg and breach events are journaled.
5. **All of the above can be enforced by a separate daemon that holds no secrets.** The standalone watchtower's database holds only finished, pre-signed responses (the latest state, the poison, a penalty, the per-HTLC sweeps), each handed to it at the moment its secret was already in memory. If someone takes over the watchtower, the most they can do is fail to broadcast; they cannot build a new transaction or read a secret, because the code that reads secrets is not linked into the watchtower binary at all. One `nm` command confirms it (README).

## Scale and interop, on public networks

* The full lifecycle at the 127-client limit, on public signet: a fresh factory paid through with real Lightning and cooperatively closed by all 128 signers in one transaction that spends a single shared UTXO into 128 outputs, conserving value to the satoshi: [`d1468287`](https://mempool.space/signet/tx/d1468287a30839962ca849d9b88f3f6442e9d6a357141180a401ce1b4d0dd727). A separate flagship sustained 127 independent client daemons through a 24-hour soak with 99 real routed payments: [`143471b5`](https://mempool.space/signet/tx/143471b5d1ddc0eee3ea54d74ed17081f24d48f429bb826723c8b0897e55c0e6). Cooperation is an optimization, not a requirement: every client keeps a unilateral exit, so safety never rests on all 128 staying online. These flagship factories ran the shipping default of one client per pseudo-Spilman leaf; the wider k-sub-factory shape drawn earlier is the one exercised on-chain in the poison broadcast above.
* An *unmodified* Core Lightning node, which knows nothing about factories and does not speak bLIP-56, paid into and out of factory channels through the bridge in both directions, at the same limit. Factory liquidity is reachable from the existing Lightning network with no node changes at all.
* A single client forcing its way out of a small factory at 1 sat/vB: a three-transaction chain ([`978f62f6`](https://mempool.space/signet/tx/978f62f662eb1c8180f7c617b4e7ef6a1d30eaf8fb3fb281cc21550a03fdd053), [`1fbc8f2f`](https://mempool.space/signet/tx/1fbc8f2f7b1fcf470a239c6c8f0f88ef6ab5794c226eb554e17b67b023be1ee4), [`a28bb6f7`](https://mempool.space/signet/tx/a28bb6f7de7949fe0aaacdbc561f09970592e4b35c3585c4f41d0f7f3b56a223)), with the 144-block CSV timeout proven on-chain by confirmation heights, the exit output staying unspent through the full window and spendable only at height 312510.
* We also ran cheat drivers that try to defeat each defense in turn: the LSP broadcasting stale state at several depths in the tree, cheating during a rollover, and attacks against JIT channel creation, against a liquidity purchase, and against a backup restore, plus an HTLC dust race. Each one checks that the breach is caught, and we watched the watchtower respond under restart races and reorgs.
* State is kept in SQLite with additive, versioned migrations (schema v39). Behind all of the above sit about 1,500 unit tests, the regtest and signet suites, and CI with the sanitizers and a fuzzer.

Two things in the core are not finished: the splice runtime is a stub (the BOLT-2 wire codec is in place, the state machine is not), and PTLC support is not yet enabled (the offer and settle path is built, but revoke, timeout, and watchtower breach-defense are not). Everything else described here is built and exercised; the external audit remains the gate before mainnet.

## Review

The implementation builds on standard, well-understood primitives: MuSig2 tree signing, Taproot, CLTV timeouts, Poon-Dryja channels. What's new is concentrated in two mechanisms, and they're the best place to start reading: the pseudo-Spilman double-sign defense (the persisted refusal described above) and the revealed-secret poison (proven on signet). The load-bearing crypto is compact: `src/musig.c` (the [BIP-327](https://github.com/bitcoin/bips/blob/c021a5f51ae9d3e71a41eac3dda6dc060fead35d/bip-0327.mediawiki) stateless flow), `src/tapscript.c`, `src/channel.c`, `src/noise.c`. The natural next step is an external audit.

The double-sign defense is where the whole safety rests. Pseudo-Spilman leaves are non-revocable, so the only thing stopping an LSP sockpuppet from confirming an old state is each signer's refusal to co-sign a conflicting spend of the same parent (`client_ps_signed_inputs`), the mitigation ZmnSCPxj proposed in [t/1143 #16](https://delvingbitcoin.org/t/superscalar-laddered-timeout-tree-structured-decker-wattenhofer-factories/1143/16). That refusal is persisted and survives restart and reorg in the cheat drivers; the corner most worth checking is whether it holds under arbitrary concurrent-ceremony interleavings.

Fee reserves for the zero-coin client remain an open design question. Every recourse transaction carries a keyless P2A anchor, so anyone who benefits can attach a CPFP and bump it when the output is worth more than the bump costs, with no reserve demanded up front. A client that does hold an exogenous UTXO can self-exit faster with it. The question is whether the zero-coin user this serves needs a documented reserve (like a channel reserve), or whether ZmnSCPxj's deposit-insurance construction is the better fit.

The ladder's cadence and sub-factory timing (how many factories run in parallel, how their lifetimes stagger) are still being tuned against real deployment. Signet operators are welcome: even a two-client factory running for a few days is useful evidence, and the procedure above is self-contained.

## Artifacts

* Implementation (C, MIT): [`github.com/8144225309/SuperScalar`](https://github.com/8144225309/SuperScalar)
* Explainer docs: [`superscalar.win`](https://superscalar.win). Every transaction above is annotated, with block heights and full txids, in the [live signet exhibition](https://superscalar.win/#deep-dives/signet-exhibition)
* CLN plugin (bLIP-56): [`github.com/8144225309/superscalar-cln`](https://github.com/8144225309/superscalar-cln), against the `blip-56` branch of a CLN fork
* bLIP-56 draft PR (ZmnSCPxj): [`lightning/blips #56`](https://github.com/lightning/blips/pull/56)
* A test orchestrator runs 36 multi-party scenarios on its own
* A read-only operations dashboard (stdlib Python) and a Prometheus exporter
* LSP discovery: [soup-rendezvous](https://github.com/8144225309/soup-rendezvous), a Nostr seed-list of vouched LSPs (the factory protocol itself never touches Nostr)
* Sample wallet ([superscalar-wallet](https://github.com/8144225309/superscalar-wallet), a cln-application fork): early alpha for discovery, monitoring, and demos; nothing in this report depends on it
* An exploratory covenant fork (CTV, Mutinynet): [`github.com/8144225309/superscalar-ctv`](https://github.com/8144225309/superscalar-ctv). With CTV the ceremonies go away, and with them the per-factory signer limit

## Status

[v0.2.0](https://github.com/8144225309/SuperScalar/releases/tag/v0.2.0) (2026-07-12) implements the [t/1242](https://delvingbitcoin.org/t/superscalar-laddered-timeout-tree-structured-decker-wattenhofer-factories-with-pseudo-spilman-leaves/1242) mechanism set, proven on regtest and public signet. Next: a written specification pulled out of this implementation (the wire messages, the ceremony state machines, the record-and-refuse rule, the reorg handling), and then an external audit. Mainnet stays behind the spec and the audit. Thanks to @ZmnSCPxj for the design and for his guidance. Corrections and criticism welcome: reply here, or open an issue on the repo.

-------------------------

