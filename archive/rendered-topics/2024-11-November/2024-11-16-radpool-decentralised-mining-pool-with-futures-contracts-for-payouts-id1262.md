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

