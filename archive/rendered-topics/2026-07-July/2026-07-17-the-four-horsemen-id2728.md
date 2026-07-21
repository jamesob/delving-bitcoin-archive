# The Four Horsemen

Kruw | 2026-07-21 02:23:46 UTC | #1

**[u]The Four Horsemen (of the Blockchain):[/u]** An operating system for Bitcoin coinjoin protocols, combining every existing on chain privacy mechanism into one incentive compatible, block space efficient, flexible, decentralized implementation.

**Author:** Kruw, Wasabi Wallet contributor - npub1pww7030g95nv9ptfpgfu69jpfxj6pm33xxueztsupwekce45wx4sm6en60

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
*ZeroLink's score above reflects its performance as a piece of The Four Horsemen. A wallet that implements ZeroLink by itself would have 3 Stars for privacy, 3 Stars for cost, and 1 Star for speed. "Whirlpool" manipulations of ZeroLink would have 1 Star for privacy, 1 Star for speed, and 1 Star for Cost.*

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

**Conclusion:** A client that fully deploys these tools will be able to abstract away user decisions for how to handle their UTXOs, perform the necessary privacy operations in the background for each scenario, and reduce on chain costs for users compared to regular on chain wallets. In order to comply with this spec, wallet synchronization must be performed using compact block filters or the user’s full node **by default.** A client that passively intercepts coinjoin data from users should be considered malicious. In order to reduce transaction fingerprints, wallets complying with this spec must support and prefer Taproot, which has superior privacy and scalability properties.

**Addendum:** Further scalability can be achieved by adding credential transfers or Ark trees inside of WabiSabi coinjoins, or by opening Lightning channels with ZeroLink/JoinMarket coinjoins. See [Kompaktor](https://github.com/Kukks/Kompaktor) and [Vortex.](https://github.com/ln-vortex/ln-vortex)

**Resources:**

* WabiSabi: https://eprint.iacr.org/2021/206.pdf
* ZeroLink: https://github.com/nopara73/ZeroLink/
* JoinMarket: https://github.com/joinmarket-ng/jmp/blob/main/jmp-0001.md
* Payjoin v1: https://github.com/bitcoin/bips/blob/master/bip-0078.mediawiki
* Payjoin v2: https://github.com/bitcoin/bips/blob/master/bip-0077.md

-------------------------

Kruw | 2026-07-20 11:11:29 UTC | #2

**[u]The Four Horsemen (Page 2):[/u]** An adaptive metaphor that follows the journey of any UTXO. Commoners may choose an archetype as a Ranger, Knight, Squire, or Farmer and go to the Pub. Royal Kings and Princes hail Chariots on the Street. The Upper Class (Drunks and Royalty) attend the Ritual. Any Class can trade as a Merchant, or roam the Street as a Priest, Scavenger, or Beggar. Users will not be forced into a dead end, and can opt out of privacy to transact as a Peasant. There are the 12 possible transaction operations, which loop thousands of demanded user behaviors together.

**Planets:** Possible node configurations in order of leeching/seeding network health effects.

*Sun:* ZeroSync

*Mercury:* Compact filters sync

*Venus:* Swiftsync

*Earth:* Bootstrap AssumeUTXO, verify chain, and prune

*Mars:* Verify chain

*Jupiter:* Bootstrap AssumeUTXO, verify chain

*Saturn:* Verify chain and relay Tor traffic

*Uranus:* Verify chain and serve peers

*Neptune:* Verify chain, relay Tor traffic, and serve peers

*Pluto:* Verify chain and coordinate coinjoins

**Regions:** Possible user onboarding paths.

*Wilderness:* New Wallet | Learn Fire, or catch the Plague

*City:* Recover Wallet | Coinjoin history does not carry the Plague

*Sky City:* Ascend to Second Layer | Purgatory, Flood, or Olympia

**Upgrade tree:** Client software power rankings.

*L2*: Kompaktor, Lightning, **OR** Ark

*4H:* WabiSabi, ZeroLink, JoinMarket, **AND** Payjoin

*3H:* WS/ZL/JM **>** WS/JM/PJ **>** WS/ZL/PJ **>** ZL/JM/PJ

*2H:* WS/JM **>** WS/PJ **>** WS/ZL **>** ZL/JM **>** JM/PJ **>** ZL/PJ

*1H:* WabiSabi, ZeroLink, JoinMarket, **OR** Payjoin

**Locations and Classes:** Conceptualizing where each user behavior overlaps.

*Street:* For Anyone

*Chariot:* For Royalty

*Pub:* For Spenders

*Temple:* For the Upper Class

**Operations:** Any individual behavior a user in the community may be incentivized to use.

*WabiSabi Consolidation:* Ranger who brings their own beer to the pub and tips the bartender

*WabiSabi Isolation:* Knight who buys a round for their “friends” at the pub, drinks all of it, and tips generously

*WabiSabi Payment:* Squire who buys a beer and tips the pub bartender

*WabiSabi Cut-Through:* Farmer who brews their own beer and tips the bartender

*ZeroLink Remix:* The Ritual coordinator collects an anchor output from the Upper Class, and applies the “magic” evolution to coins

*JoinMarket Taker Consolidation:* A wealthy King with an army of Chariots, burying a vault of gold in the mountain

*JoinMarket Taker Isolation:* A rich Prince in a Chariot who picks up Harlots and Scavengers off the Street

*JoinMarket Maker Consolidation:* A Harlot wearing rags on the Street

*JoinMarket Maker Isolation:* A masked Scavenger on the Street

*Payjoin Sender:* Priest, an altruistic spender with a limited budget

*Payjoin Receiver:* Merchant, a cost motivated economist

*Solo Payment:* Peasants only, a greed motivated Fool on the Street who spreads the Plague if they do not pay with coins they obtained as a Scavenger

**Currency Units:** Using randomized default values, it is even possible for the Peasant's coins flow into the King's vault or back, even if they never interact directly. WabiSabi privately consumes arbitrary value Dust, Coins, Bars, and Vaults. WabiSabi creates private Dust, Coins, and Bars, and arbitrary value payments. ZeroLink consumes and creates private Dust, Coins, and Bars. JoinMarket consumes arbitrary value Dust, Coins, Bars, and Vaults. JoinMarket creates private (for takers) and confidential (for makers) Dust, Coins, Bars, and Vaults. Payjoin consumes arbitrary value Dust for the receiver, and arbitrary Dust, Coins, Bars, and Vaults for the sender. Payjoin creates confidential Dust, Coins, Bars, and Vaults.

*Dust:* 25k sats - 10m sats | Consumed by WS, ZL, JM, PJS, PJR | Anonymized by WS, ZL, JMT | Confidentialized by JMM, PJS, PJR

*Coins:* 250k sats - 1 BTC | Consumed by WS, ZL, JM, PJS | Anonymized by WS, ZL, JMT | Confidentialized by JMM, PJS

*Bars:* 2.5m sats - 10 BTC | Consumed by WS, ZL, JM, PJS | Anonymized by WS, ZL, JMT | Confidentialized by JMM, PJS

*Vaults:* 25m sats - 100 BTC | Consumed by WS, JM, PJS | Anonymized by WS, JMT | Confidentialized by JMM, PJS

Every rational user has the possibility of being a Scavenger or a Merchant. Every Drunk or Royal has the chance to attend the Ritual. Even cheap users like Priests, Scavengers, and Beggars improve the privacy of everyone in the Community.

**TXO Labels:** Understand the cumulative privacy of a coin.

**Races:** Randomize default user behavior based on their wallet's available balance. Inert users default to Payjoin for spending, Magic users only spend coins that have been created through the Ritual, and Titan users only spend coins that have been consumed by WabiSabi, ZeroLink, and JoinMarket Taker.

*Titan -* Powerful, Generational Wealth. A deity.

*Elf -* Rich, Magic, Lives for 1,000 years. A higher life form.

*Human -* Rich, Magic, Lives for 60 years A mere mortal.

*Druid -* Poor, Inert, lives for 80 years. A kindly soul.

*Orc -* Poor, Inert, Lives for 40 years. A victim of his own poor planning.

*Dwarf -* Poor, Inert, Lives for 30 years. A victim of greed.

-------------------------

Kruw | 2026-07-20 10:54:47 UTC | #3

Reserved post for The Four Horsemen (Page 3): Server specs

-------------------------

