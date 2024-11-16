# Radpool: Decentralised Mining Pool With Futures Contracts For Payouts

jungly | 2024-11-16 14:58:04 UTC | #1

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

We have a [Discord](https://discord.gg/Kg3YR6z4), if you want to come chat. We'd like to discover as many attacks possible as early as possible :) 

Thanks

-------------------------

