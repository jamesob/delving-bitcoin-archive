# Radpool: Decentralised Mining Pool With Futures Contracts For Payouts

jungly | 2024-11-25 15:03:49 UTC | #1

Hi all,

I am looking for some feedback on, [Radpool](https://radpool.xyz), a new design to resist mining centralisation by building a syndicate of nodes acting as mining service providers, or MSPs.

The design decentralises template generation away from a centralised pools while reducing payouts for miners in a scalable manner. The detailed design is available at https://radpool.xyz 

## Independent Block Templates and Stratum Services

MSPs independently build blocks and can follow any policies for building block templates. MSPs can run stratum v1 or v2 servers, letting miners build the templates. A radpool node implementation will provide this optionality out of the box by using FOSS community implementations where possible. All MSPs can be seen as mini centralised pools that are collaborating using Radpool mining syndicate to reduce all their miner's payout variance. 

## Payout Mechanism With Unilateral Exit

The MSPs fund DLC contracts to pay miners based on the miner's hashrate. These contracts are settled at fixed, well known intervals. MSPs with enough hashrate generate attestations using FROST threshold signatures to settle these DLC contracts. The MSPs in turn get the payout from the coinbase of the pool's blocks using PPLNS. The MSPs thus absorb the risk of variance in block discovery and earn a yield for providing this service to the miners. Most importantly MSPs can't bail out of their contract as the syndicate will publish an attestation anyway to pay the miner for it's work. This results in guaranteed payouts for miners as long as the syndicate is operational  and the miner can connect to at least one MSP. See [Payout Mechanism](https://www.radpool.xyz/1/payout-mechanism.html) for details.

## Yield for MSPs

Anyone with capital for funding DLC contracts can run an MSP - they need to attract miners to earn a yield though. As the network grows, the miner's variance reduces and the MSP's return on their capital increases. We believe this does two things 1) creates an open market for pool fees and 2) encourages MSPs to grow the network. This is a classic network effect - each additional MSP or miner increases the utility for everyone else on the network.

## Design Space

The Radpool design sits in the middle of centralised pools on one end, and p2p pools on the other end where miners have to run services at the network edges. Instead, with Radpool, MSPs run an open syndicate and miners connect to an MSP just like they would connect to a centralised pool. See [FROST Membership](https://www.radpool.xyz/1/frost-federation.html#_membership) section for details. 


# The Key Features of Radpool

1. Block template construction is decentralised as each MSP builds an
   independent blocktemplate for their miners.

2. MSPs pay miners over futures contract built using DLCs. Miners get
   paid for their hashrate at fixed intervals - think of this as decentralised FPPS.

3. Radpool acts as an oracle where a threshold number of MSPs generate
   the attestation required to settle DLCs.

4. MSPs earn a yield in exchange of locking capital that funds miner
   payouts.

5. Share accounting is transparent to anyone who connects to receive
   data, making the pool fully auditable.

6. The payout mechanism allows unilateral exits for miners and MSPs.

7. Switching to Radpool from current centralised pools has zero
   friction for miners - i.e. miners don't have to run any additional
   services if they don't want to.

## Progress

My current focus is on the FROST Federation
([repo](https://github.com/pool2win/frost-federation)) for providing
the federation of MSPs. We also started work on implementing the DLC contracts using
rust-dlc - still early days - public repo coming soon.

We have a [Discord](https://discord.gg/SUbYfBzq5g), if you want to come chat. We'd like to discover as many attacks possible as early as possible :) 

Thanks

-------------------------

marathon-gary | 2024-11-25 17:41:41 UTC | #2

I enjoyed reading through the proposal and couldn't think of concrete reason it won't work.

I have several questions and comments regarding the following:

#### **Threshold Signature Schemes**

* Was ROAST considered as an alternative to Lindell's scheme? How does the robustness/rounds of ROAST compare?

#### **Data Handling and Transparency**
  
  * Recommendation: Implement time-decayed deletion of data for stratum jobs and shares. This incentivizes timely validation by miners and reduces storage costs for MSPs. Mining shares and jobs are incredibly data intensive.
  * Protocol-level data availability: Defining or recommending a baseline for data availability ensures transparency while leaving implementation details to MSPs.

**Miner Registration and Authentication:**
  * Suggestion to replace "registration" with "enrollment" when discussing the relationship between miners and MSPs.
  * The miner username should be unique across the network. 
   
    How can uniqueness be accomplished? Using MSP pub key + mining username?
  * Recommendation: specify desired authentication properties instead of specific username/password methods.

**Syndicate Protocols for MSPs:**
  * Misbehaving MSPs are removed from the syndicate and denied rewards.
    
    How exactly are dishonest MSPs identified and removed?
  * MSP hash rate validation: If an MSP fails to provide valid hash rate within 2000 blocks, they are rejected. 

     Why is 2000 blocks the threshold?

  * Syndicate's lack of visibility into miner-MSP contract rates:
    
    Does syndicate awareness of miner/MSP exchange rates break assumptions or security?

**Scaling Threshold Signatures:**
  * The syndicate must publish oracle signatures proportional to the number of miners. 
   
    Could this scale become prohibitive?

**Signed Coinbase:**
  * Each MSP retains the signed coinbase until confirmed to a 100-block depth. MSPs are incentivized to mine this quickly, potentially with zero fees.

**Malicious MSPs:**
  * Could MSPs supply “bad” nonces? What risks does this pose to payouts or pool operations?

**Small Hash Rate Miners:**
  * Roll-over mechanisms for DLC payouts could be beneficial for miners with low hash rates, preventing the creation of dust UTXOs. 
   
    Has there been other consideration for small scale miners, aside from the Roll-overs?

**FROST Federation Quorum:** 
* Benchmarks for FROST quorum performance are not present in the repository. 
  
   Could latency/throughput FROST signing be a bottleneck?

**Mining Fee Trends:** 
* [PPLNS-JD](https://delvingbitcoin.org/t/pplns-with-job-declaration/1099) seems particularly relevant as fees increasingly dominate mining revenue.

**Miner Dashboard:**
  * The dashboard allows miners to view balances, extend contracts, or switch MSPs. 
   
    Are these features explicitly defined in the protocol?

---

1. **Threshold Signature Schemes:** How does ROAST compare to Lindell’s scheme in robustness and efficiency?
2. **Dishonest MSP Handling:** What exact mechanisms are in place for identifying and removing dishonest MSPs?
3. **2000-Block Delay:** Why is this specific threshold chosen for validating new MSPs?
4. **Dashboard Specifications:** Are the dashboard functions and user interactions clearly defined anywhere?

-------------------------

mcelrath | 2024-11-27 16:35:31 UTC | #3

Some comments on this proposal.

TL;DR: The differences between this [Radpool proposal](https://radpool.xyz) and my [Braidpool proposal](https://github.com/braidpool/braidpool/blob/9fe685d2ea027814cec6766d1b27284fd17ea880/docs/braidpool_spec.md) are:
1. The use of a fixed FROST signing federation (radpool) instead of a dynamic one (braidpool).
2. Having the FROST federation be composed of share-buying entities (radpool) instead of miners (braidpool).
3. Having share-buyers in control of block templates (radpool) instead of miners (braidpool)

Everything else is identical. Kulpreet has been working on FROST signing and I have been working on the Braid accounting and payouts. I don't see any reason that this can't be one project. At any rate I intend to keep working with Kulpreet and use his FROST code for Braidpool's dynamic federation.

In more detail:

1. Using a syndicate of MSPs will result in a highly political process around allowing MSPs into the signing federation. By comparison, Braidpool will use an open dynamic federation composed of miners who have recently won blocks.
2. Since MSPs decide block templates, this presents a smaller number of entities to attack for those that want to censor transactions. Furthermore the profit motive for constructing block templates is poor -- most jurisdictions make it legally risky to perform this task, which is why OCEAN and Braidpool move block template selection to the edges (individual miners) maximizing the number of entities that must be attacked to accomplish censorship. Block template creation is not logically related to participating in the financialization of mining. Note that attempting to censor in this way is a mis-application of laws intended for banks which have three functions they perform that Bitcoin miners, nodes, and MSPs fundamentally cannot: beneficial party identification, transaction blocking, and asset seizure. We should be simultaneously lobbying against this mis-application of laws. Centralizing around MSPs will result in the same situation as currently happens in Ethereum: there will be two types of block producers, "compliant" and "MEV", neither of which we want. Share buying is a logically distinct operation from block template construction and I don't find it necessary or desirable to conflate the two. Today's centralized pools are already conflated share-buyers and block-template-deciders and we know what that looks like.
3. The use of PPLNS implies significantly higher variance than Braidpool, depending on N. One can regard Braidpool as PPLNS with N=2016 (the fixed difficulty adjustment window) which enables simple pricing of hashrate derivatives. With any other PPLNS scheme such as used by OCEAN (N=8) or Radpool, pricing and trading of options and future (share transfer to a risk-taker) is significantly muddied since different shares within the same difficulty adjustment window will have a different price. This incentivizes pool hopping -- switch to Radpool or OCEAN when the share price is high and switch away when the pool's luck drops. It also makes the creation of standardized derivatives markets nearly impossible.
4. The share accounting system was not described. Braidpool will provide this in its DAG. If these end up being separate projects, I invite Radpool to use it.

But some praise:
1. I think using DLC's for share buying is a good idea and have been investigating it for Braidpool too, to emulate FPPS. 
2. Even if @jungly wants to pursue this as a separate project from Braidpool, I think much of the FROST signing federation code and possibly share accounting (if he wants to put them in a braid) can be shared. I'd rather have two decentralized mining pool projects than zero and I will finish Braidpool.

I don't know why @jungly proposed a separate project, but I think a big part of his motivation is that he thinks miners won't want to run Bitcoin or Braidpool nodes. I've heard both opinions from miners (that running Bitcoin nodes is not a problem and "too hard"). Time will tell but I think this is important to move block template selection to the edges and OCEAN is seeing success with DATUM.

Finally let me correct some misconceptions:

"Braidpool’s UHPO model is an example where a miner needs permission from a threshold of miners to exit with their payout." -- Braidpool will not require permission to exit. All shares will be paid automatically at the end of a difficulty adjustment window and any miner in the network can broadcast this transaction. The rules around the resulting UHPO transaction are decided by consensus and proportional distribution of BTC according to submitted shares. If the pool fails to sign an Eltoo update or UHPO update, the last known-good set of payouts is already signed and can be broadcast by any miner. From that point on any hashrate pointed at the pool will fall back to solo mining.

The scale of Kaspa relative to the number of nodes on Bitcoin is not a valid comparison. This has to do with the desirability Kaspa as an asset, not the scaling constraints of their consensus protocol.

The security model of FROST signing proposed in Radpool is identical to what I proposed for Braidpool. Signing nodes are connected point-to-point by revealing their (encrypted) IP information to other miners in the signing federation (but not to anyone else). Thus all the security proofs for FROST hold.

Finally finally, there are fundamentally two different kinds of entities involved in the FROST federation: miners and share-buyers. In the presence of both, it doesn't make sense for the signing federation to be composed entirely of one or the other. Miners shouldn't be in full control of whether their contracts settle and neither should the share buyers. Therefore the logical conclusion is that the signing federation should be composed of BOTH, with miners represented by their hashrate, and share-buyers by their locked financial commitment, proportionally. I will modify the Braidpool proposal to include both. Likely using DLC's.

Lastly, if you're interested in either, please [join the Blockchain Commons meeting on Dec 4](https://developer.blockchaincommons.com/frost/) to hear Kulpreet talk about his FROST federation!

Cheers,
-- Bob

-------------------------

jungly | 2024-11-27 17:51:11 UTC | #4

There are a few differences between Braidpool and Radpool proposals

1. Consensus - Radpool does not need one, Braidpool needs consensus on transactions/beads/blocks at the base layer itself. Radpool only needs BFT broadcast that FROST federation needs anyway.
2. Timeliness concerns - Radpool does not do consensus and therefore does not have concerns about reaching the consensus in a given time frame. FWIW, I think we should build delay tolerant protocols.
4. FROST Federation - Radpool federation membership explicitly defines how [miner hashrate decides federation membership](https://www.radpool.xyz/1/frost-federation.html#_membership). Braidpool requires miners to get lucky to find a block for joining the federation. I think hashrate is enough and is more inclusive than finding blocks.
5. Payout mechanism - In Radpool a single miner can exit at any time, with Braidpool the miner has to wait two week period in the worst case, or request permission for an early exit.
6. Futures payout mechanism - Radpool's payout primitive is a future's contract using a DLC. Braidpool needs to figure out how the futures contracts will work.
7. Miner onboarding - In Radpool, a miner signs up to an MSP and is done. In Braidpool miners need to run a fair few services to get going.
8. Network Topology - Radpool ends up with a point to point network of MSPs and miners talking only to MSPs. Braidpool is a p2p network across all miners. 

For what is worth, the syndicate in *Radpool is built by MSPs who bring both the required hashrate _and_ liquidity they provide to fund DLCs.*

## Execution Risk

The biggest reason I am pitching Radpool as a separate project is execution risk with Braidpool. 

Braidpool proposes a consensus protocol that is not proven to work or to scale - especially if we don't use Kaspa's DAGKnight protocol. I get DAGKnight, my [PhD thesis](https://github.com/kulpreet/transman-phd-thesis/blob/0e8816ee270fb1ea2d596b0237c91d87b73dc97c/TCD-SCSS-PHD-2006-12.pdf) was on a DAG based consensus protocol back in 2006. I remember how much effort it took me to implement a consensus protocol. Implementing a new consensus protocol that can scale to thousands of nodes is risky. I suspect at scale DAGKnight's block rate will slow down and so will Braidpool's. What will that mean for Braidpool, where we are very focussed on fast block times, I don't know. It could work smoothly. But it is a risk.

Radpool eliminates the need for consensus. Radpool explicitly avoids new cryptography or distributed systems protocols. With Radpool, we take well understood protocols and put them together to provide a mining pool that improves blocktemplate decentralisation and offers decentralised FPPS.

## Response to Bob's Comments

> syndicate of MSPs will result in a highly political process

Syndicate membership is explicitly defined by the hashrate the MSP brings to the pool. Miners vote with their hashrate.

> smaller number of entities to attack for those that want to censor transactions

Some miners want to and can build block templates. Others don't want to. Radpool enables both. Some MSPs will run SV2, some will run SV1. Miners will choose how they want to work. MSPs will support both SV1 and SV2 - good old SRI has both!

>  pricing and trading of options and future (share transfer to a risk-taker) is significantly muddied 

Radpool's future contract is very simple and defined between two parties - MSP and Miner. MSP agrees to pay miner X BTC for Y Hashrate produced by Z date. All three parameters are independently decided for each contract by the parties involved. Braidpool's future contract system is not yet clear to me. [I tried to define something using single use seals for Braidpool a while back](https://blog.opdup.com/2021/08/18/deliver-hashrate-to-market-makers.html), but Braidpool didn't go that route.

> The share accounting system was not described.

I'll add a section on this.

-------------------------

jungly | 2024-11-27 19:30:52 UTC | #5

Hi,

Thanks for taking the time to read through the proposal. I'll include my replies inline and address other questions at the end.

[quote="marathon-gary, post:2, topic:1262"]
Was ROAST considered as an alternative to Lindell’s scheme? How does the robustness/rounds of ROAST compare?
[/quote]

Yes. In fact, we will need to support one of the optimisations suggested in the ROAST paper. Still calling it FROST Federation as the single round signing is what helps us scale to many MSPs.

> **Data Handling and Transparency**

Nice reminder. We will need to prune the data in MSPs. The time decay for shares is something I was thinking will be a cliff. The reason for the cliff is - simplicity and it makes sure the shares between two blocks found are all available, for whatever PPLNS parameters we use.

> specify desired authentication properties instead of specific username/password methods.

Yeah, I was leaving that out on purpose to make a decision just now. I think the standard usage out there is username/password on most pool servers. Therefore for keeping things easier for miners to switch I was thinking username/password might be the easiest.

> How can uniqueness be accomplished? Using MSP pub key + mining username?

Yes. The requirement is that each miner essentially is given a random string as username by the MSP, or the MSP makes sure the one provided by the user is long enough.

> Could latency/throughput FROST signing be a bottleneck?

The choice of FROST is driven by their single round signing. I did [some analysis](https://blog.opdup.com/development-updates/2024/07/09/frost-signing-for-channel-updates.html) when thinking about using Belcher's channels for payments. The message complexity is $O(n^2)$, but round complexity is 1 - which is awesome. We still need to run experiments to measure performance of FROST Federation to know scaling limits.

> **Mining Fee Trends:**

Yes. We'll need to normalise the block templates proposed by each MSP and apply weights to the various shares mining for different block templates.

### Other questions

1. Lindell's scheme is awesome. However, there is not open source implementation available,yet. I seriously toyed with rolling up my sleeves and working on it, but decided not to. Why is it awesome? Cause it is robust in the presence of failures, with three rounds. For robustness, instead, we use ROAST, which will have much higher message complexity, but easier to implement, as it is a messaging protocol, not the same as implementing cryptography :) . If we see ROAST is being a bottleneck in the future and Lindell's scheme is available, we could switch. For a functional MVP, we are going with FROST/ROAST.

2. Dishonest MSP Handling - I essentially mean, someone not running FROST correctly or sending invalid shares. FROST has identifiable aborts, so we can identify the failing MSP and kick it out the next time. For invalid shares, each MSP knows the MSP is sending bad data. For other DDoS attacks etc, again, we can block and disconnect the MSP. For an equivocating MSP, the BFT broadcast detects the behaviour and we can leave that MSP out.

3. 2000 block delay - I wanted to pick a period between 2/3 weeks. So an MSP joins, bring hash rate, but can't participate in threshold signature unless it has been working correctly for 2/3 weeks, and then I randomly chose 2000 blocks. This prevents a large miner acting as an MSP and toying with the threshold signature, leaving, coming back and so on.

4. Dashboard specifications - Not specified at all! I am thinking the first release will be a cli interface :) 

Thanks for the great questions

-kp

-------------------------

