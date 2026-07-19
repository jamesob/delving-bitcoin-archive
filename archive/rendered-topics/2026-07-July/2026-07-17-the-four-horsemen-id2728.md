# The Four Horsemen

Kruw | 2026-07-19 12:38:19 UTC | #1

**[u]The Four Horsemen (of the Blockchain):[/u]** A unified spec for Bitcoin coinjoin protocols, combining every existing on chain technology into one incentive compatible, block space efficient, flexible, decentralized implementation.

**Author:** Kruw - npub1pww7030g95nv9ptfpgfu69jpfxj6pm33xxueztsupwekce45wx4sm6en60

**Abstract:** Different approaches have been taken to improve on chain privacy for Bitcoin using the coinjoin primitive. Each protocol has unique trade offs and fragmented liquidity. However, a single client that assembles every technology can passively adapt to the user’s needs, access an overlapping liquidity pool, and enable default privacy with lower fees than solo payments.

**Subprotocols:** The name of each "Horse", and what functionality it adds to the spec. Implementing any single Horse improves privacy independently, but combinations of two, three, or four Horses each fill UX gaps.

1. **WabiSabi** - For consolidating UTXOs without revealing common ownership, creating the ZeroLink standard denomination inputs, and privacy maximized spending
2. **ZeroLink** - For block space efficient remixing and DoS protected rounds
3. **JoinMarket** - Provides on demand liquidity for whales, and provides free (or profitable) remixing for low time preference users
4. **Payjoin** - Enhances third party privacy for on demand payments with supporting wallets, block space efficient, steganographic spending

**Strengths, weaknesses, and costs:**

WabiSabi uses keyed verified anonymous credentials and lets users make every spend a coinjoin. The drawback is that it has poor on chain scalability, and slow intervals between transactions. The flagship WabiSabi implementation (Wasabi Wallet v2.0+) uses \~600-700 vBytes per user, per transaction. There is an additional "dust" cost (\~100 sats per user, per tx at 1 sat/vByte) that users must discard to create matching value outputs. Maximum round size is limited by performance (Tor network stability is the bottleneck). Sybil dilution is disincentivized since users pay for their own mining fees. UTXO probing is prevented by random round consistency verification. Failure-to-sign DoS protection is enhanced by increasing the minimum value for eligible inputs.

ZeroLink uses Chaumian Blind Signatures and lets users create fixed sized denomination outputs from equal sized or larger sized inputs. The drawback is limited amount flexibility that leaves users with untouchable toxic change that shouldn’t be consolidated. The flagship ZeroLink implementation (Wasabi Wallet v1.1+, now discontinued) used a base denomination of 0.1 BTC. As a piece of the wider spec, ZeroLink costs \~100 vBytes per user, per transaction. Maximum round size is limited by available liquidity. Sybil dilution is disincentivized since users pay for their own mining fees. UTXO probing is prevented by limiting registrations to a single input. Failure-to-sign DoS protection is enhanced by only whitelisting WabiSabi or JoinMarket outputs for ZeroLink inputs to guarantee a sunk mining fee cost for the attacker.

JoinMarket makers provide passive liquidity, and takers pay their mining fees when consuming liquidity for coinjoin transactions. The drawback is that the taker is a trusted coordinator that learns the input-output mappings of makers. A JoinMarket coinjoin with 10 makers uses \~4,000 vBytes (makers will create two outputs each, which will eventually lead to consuming two inputs later). Maximum round size is limited by diminishing marginal returns for takers. Sybil dilution is avoided by makers’ commitments to fidelity bonds. UTXO probing is prevented by takers’ commitments to Proof of Discrete Log Equivalence (PoDLE). Failure-to-sign DoS protection is not a concern since makers are acceptors of altruistic taker fees.

PayJoin is an opportunistic coinjoin between senders and receivers that can confuse third party analysis. The drawback is that both counterparties are single points of failure, and wallet/behavioral fingerprints may leak additional information. The payjoin sender subsidizes the mining fee for the receiver’s first input, paying for \~210 vBytes. Failure-to-sign DoS and UTXO probing attacks are prevented by the sender presigning a fallback payment to the recipient. Payjoins are assumed to be “Sybiled” by each counterparty automatically.

![Four Horsemen Stars|656x500](upload://1IpTdXGlL5QRf4SEUq9mrh1of8c.png)
*ZeroLink's score above reflects its performance as a piece of The Four Horsemen. A wallet that implements ZeroLink by itself would have 3 Stars for privacy, 3 Stars for cost, and 1 Star for speed.*

**Incentives and UX integration:**

This is a matrix style comparison that shows how the subprotocols complement each other to fix their individual weaknesses.

Users have an incentive to use WabiSabi to enhance privacy before and after ZeroLink coinjoins, and before and after Payjoins. This protocol is capable of providing end to end privacy by itself, but cost-sensitive users will benefit from these integrations:

* ZeroLink enables highly scalable remixing with very low fees
* JoinMarket gives WabiSabi's whales access to a huge liquidity pool, and gives passive participants the chance to gain free remixes
* Payjoin gives users a faster and cheaper alternative compared to sending their payment directly in a WabiSabi coinjoin, and Payjoins between WabiSabi users further increase their privacy

Users have an incentive to use ZeroLink in between WabiSabi coinjoins to save fees when remixing. This protocol provides incomplete privacy by itself, but is made whole with an additional WabiSabi integration:

* WabiSabi feeds ZeroLink participants their even sized inputs and handles change outputs created by ZeroLink payments, creating a larger liquidity pool
* JoinMarket gives ZeroLink participants the chance to gain free remixes
* Payjoins between ZeroLink users further increase privacy

Users have an incentive to use JoinMarket in between their other coinjoins to passively enhance privacy. This protocol provides complete privacy by itself, but is extremely costly for the taker initiating it:

* WabiSabi is more scalable than JoinMarket, but can be a slow experience for whales
* ZeroLink is less flexible and less liquid than JoinMarket, but is far more scalable
* Payjoins between JoinMarket users further increase privacy

Users have an incentive to use Payjoin when receiving payments in between their other coinjoins. This protocol provides incomplete privacy by itself, but is enhanced by any underlying coinjoin integration:

* WabiSabi can be performed before/after a payjoin to fully protect both users
* ZeroLink can be performed before a payjoin to protect the receiver’s source of funds
* JoinMarket gives Payjoin participants the chance to gain free remixes

**Conclusion:** A client that fully deploys these tools will be able to abstract away user decisions for how to handle their UTXOs, perform the necessary privacy operations in the background for each scenario, and reduce on chain costs for users compared to regular on chain wallets. In order to comply with this spec, wallet synchronization must be performed using compact block filters or the user’s full node **by default.** A client that passively intercepts coinjoin data from users should be considered malicious.

**Addendum:** Further scalability can be achieved by adding credential transfers or Ark trees inside of WabiSabi coinjoins, or by opening Lightning channels with ZeroLink/JoinMarket coinjoins. See [Kompaktor](https://github.com/Kukks/Kompaktor) and [Vortex.](https://github.com/ln-vortex/ln-vortex)

**Resources:**

* WabiSabi: https://eprint.iacr.org/2021/206.pdf
* ZeroLink: https://github.com/nopara73/ZeroLink/
* JoinMarket: https://github.com/joinmarket-ng/jmp/blob/main/jmp-0001.md
* Payjoin v1: https://github.com/bitcoin/bips/blob/master/bip-0078.mediawiki
* Payjoin v2: https://github.com/bitcoin/bips/blob/master/bip-0077.md

-------------------------

Kruw | 2026-07-18 19:14:14 UTC | #2

Reserved post for The Four Horsemen (Page 2), with intermediary integrations, and a fully defined UX path.

-------------------------

