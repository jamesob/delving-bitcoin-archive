# Node Attestation Protocol

bkeroack | 2026-11-21 02:11:05 UTC | #1

Historically, miner signaling has been the primary mechanism by which the network is surveyed for support of proposed network changes. There are valid reasons for this: mining is inherently Sybil resistant, miners are a key economic participant in the network, and miner signaling has a reliable mechanism available. 

However, miners are highly centralized and are not the only significant network constituency. A broader survey of the Bitcoin network would be helpful when contentious proposals are being debated. The Node Attestation Protocol (NAP) is a new protocol to allow a broader network survey, particularly of what has been described as "economic nodes". That is, non-mining network participants that control a meaningful proportion of the economic value of the network.

NAP allows node operators, and participants more generally, to publish free-form key-value **attestations** attached to cryptographic proof of control of specific UTXOs. These attestations are published via a new backwards-compatible P2P message pair and service bit. Supporting nodes serve attestations on demand. An **aggregator** surveys the network for all attestations, validates them against the current UTXO set, and publishes the aggregated results sorted by coin weight. Each coin is counted exactly once per key and is de-duplicated by the aggregator, resulting in Sybil resistance. Attestations are weighted by aggregated coin value behind them.

Attestations are simple string-type key value pairs. These key value pairs are arbitrary and can be anything from the trivial (*"bestmoney=bitcoin*", "*favoritecolor=blue*") to the consequential ("*bip9999=yes*"). The protocol itself puts no structure or restriction on these keys or values, aside from max length (128 bytes for each) and character set (printable ASCII only). BIP authors who wish to request NAP attestations for their proposal are encouraged to publish the canonical key name and values within the BIP document.

Attestation signatures use BIP 322 or legacy `signmessage` depending upon outpoint type. Nothing is locked or spent via the attestation alone. Attestation messages are human-readable so they are transparent when signing and displayed by hardware signing devices. Attestations backed by spent coins are no longer considered valid and drop out of any aggregation. Holders can change their mind by republishing a new attestation backed by the same coins, which supersedes the previous attestations. An empty value withdraws the statement for that key.

NAP is intended to be strictly non-binding. No BIP or other network decision should be mechanically gated on some attestation threshold or vote proportion. NAP is intended to be a purely human-judged signal of the popularity of a proposal or level of support among economic nodes, in addition to miner signaling and any other data points that are relevant.

Attestations are time bound by FRESHNESS_WINDOW which is 26,208 blocks (\~6 months). Beyond this window, the attestation is considered invalid and must be re-signed by the holder. A coin backing an attestation must have MIN_COIN_AGE of 4,032 blocks (\~4 weeks) and MIN_COIN_VALUE of at least 10k sats. Test networks have shorter values for minimum coin age to allow easier NAP testing.

A potential weakness of NAP is that attestations reveal what coins the attestor controls, which is a privacy cost. This is mitigated by the fact that attestations are pseudonymous and bear no identifying information. Conceivably, public nodes could allow anonymous submission of valid attestations which then would break any link to an individual or organization. For P2PKH, P2WPKH or script output coins, signing reveals the public key or script, which holders should consider. A future version of NAP might explore the possibility of zero knowledge proofs for attestations, but this is out of scope of the current proposal.

Another weakness is coins held by custodians, where the beneficial owner does not control the keys. This both prevents the owner from attesting on their own behalf, and also allows the custodian to attest for them. Custodians could allow some mechanism for owners to express what attestations they want to publish for their coins, but nothing stops a custodian from producing attestations purely for their own self-interest.

Coin-weighted signaling has prior art: Bitcoinocracy, launched in 2015 during the block size debate and relaunched by Bitcoin.com in 2017, tallied signed messages by address balance. HODL.voting (2017) had voters time-lock coins and weighted them by amount and duration. Ethereum's CarbonVote was used to set the default for the 2016 DAO fork. NAP addresses weaknesses of these attempts: attestations are served by any node that opts in, and anyone with a UTXO set can collect and verify them, so there is no trusted tallier. BIP 322 extends signing to segwit and taproot outputs. An attestation expires after about six months unless re-signed, stops counting the moment its coin is spent, and requires the coin to have been confirmed for about four weeks. Participation is still self-selected, custodians can sign with coins they hold for others, and signatures can be bought.

NAP is fully implemented in [satd 0.7+](https://github.com/epochbtc/satd), a full Bitcoin node implementation in Rust, and the protocol spec is currently a draft. Interested users can run the node and begin publishing attestations immediately. A reference aggregator is also included.

[Full protocol specification](https://github.com/epochbtc/satd/blob/1a9a943bba9b6a083866272dcd09667e565f58de/docs/nap/protocol.md)

Please read and provide feedback. More implementations in other nodes would be welcome when the spec has been reviewed and is more finalized.

-------------------------

