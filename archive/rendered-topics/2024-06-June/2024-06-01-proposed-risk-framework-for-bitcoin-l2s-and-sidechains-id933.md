# Proposed risk framework for Bitcoin L2s and Sidechains

janusz | 2024-06-01 14:27:22 UTC | #1

Hi everyone,

My name is Janusz and I am working on a project called [Bitcoin Layers](https://www.bitcoinlayers.org/) which assesses various implementations of L2s and sidechains. The new wave of sidechain protocols sparked the development of the project.

I've developed a [risk framework](https://hackmd.io/@januszgrze/bitcoin-layers-risk-proposal) which analyzes L2s and sidechains against common criteria. The goal is to clearly explain the risks associated with each implementation to users.

I've worked on refining this proposal with our [research advisor(s) and advisory board](https://x.com/januszg_/status/1795858888857665656). I'm additionally posting here to solicit feedback and see if we have any gaps in our methodology.

Let me know if you have any thoughts!

---

# Proposal for updated Bitcoin Layers risk framework
[Bitcoin Layers](https://www.bitcoinlayers.org/) currently has a risk assessment framework where scaling protocols are analyzed against four key aspects of their protocol. Each aspect is given a risk score, high, medium or low risk, after review. The scaling protocol will then receive a pie chart score, with the colors resembling its overall risk score.

We are currently assessing protocols based on the following criteria:

- Unilateral exit
- Data availability
- Block production
- State validation (settlement)

Our [docs](https://bitcoin-layers.gitbook.io/bitcoin-layers) outline why we’ve chosen these areas.

## Current challenges
The primary challenge we are currently facing with this framework is that not all protocols are designed with the same goals in mind and have varying tradeoffs. For example, a rollup is inherently different from a payment channel protocol.

## Proposed solution
Saunter (from Alby) suggested that we create a more diverse framework that assigns weights to the specific risk questions. If a protocol does not meet a specific requirement, they would be deducted points off their total score relative to the weight of that question. I’m proposing a similar model, and believe it will help create a more granular framework for the Bitcoin Layers project.

Instead of weights, we keep the high, medium, low risk categories, but we add further circumstances that could see a protocol move between the risk categories.

We also add further binary questions to the assessment that helps users better understand the risks associated with a certain protocol.

All of this information can be comprised into a risk summary that points out the key tradeoffs related to a specific protocol, and how it might potentially impact the Bitcoin network.

### Starting point
We analyze protocols on four categories: Data availability, network operators, settlement assurance (a.k.a finality), and the bridge custody (or two-way peg).

Protocols do not receive an overall score, but a risk summary is added at the beginning of every assessment to highlight risk areas that can be critical.

For example, if an optimium uses an offchain data availability solution that is a single server, then it only takes collusion between a sequencer and the operator of the DA server to steal everyone’s funds. This is extremely high risk and should be noted in the summary.

Let’s review how protocols can potentially be assessed.

#### Data availability
- Self-hosted: User can store data related to scaling protocol transactions locally - **low risk**
- Onchain: Protocol uses Bitcoin for data availability - **low risk**
- Offchain via alternative consensus protocol: Data is primarily stored on another publicly available, decentralized network and there is a mechanism to verify that the data is readily available - **medium risk**
    - If the network sees N < 21 operators participating in the network, then it is **high risk**
    - If the network sees N > 100 operators participating in the network, then it is **low risk**
    - If no mechanism to detect fraud (e.g. DA withholding attack), then **high risk**
- Offchain DAC: L2 state stored via a centralized (or permissioned) committee and Bitcoin full nodes, and users, trust attestations made by the committee that data is readily available - **high risk**
- Offchain Single Server: L2 state is stored via a centralized server and Bitcoin full nodes, and users, trust attestations made by the operator that data is readily available - **high risk**
#### Network Operators
- Network is operated peer-to-peer Protocol doesn’t rely on third party operators - **low risk**
- Network operator set is N > 21 and users can self sequence - **low risk**
    - Sequencer designs can include of this Federated, Auction, PoS, DPoS, PoW, etc
    - If network operator is N < 21 , then moved to **medium risk**
- User can’t self sequence, but operator set is N > 21 - **medium risk**
    - If operator set is N > 100 then move to **low risk**
    - Designs include Federated, Auction, PoS, DPoS, PoW, etc
- Federated operator set: Transactions are sequenced by an honest majority of participants in a permissioned, offchain consensus protocol/network - **high risk**
- Centralized operator with no self-sequencing - **high risk**
#### Settlement assurance
- Bitcoin-equivalent: Settlement happens on, or is enforced by, Bitcoin script and/or consensus participants - **low risk**
- Onchain, optimistic settlement: Settlement happens through an onchain challenge response protocol (BitVM2, Lightning) - **low risk**
    - Note: BitVM2 sees challenger/verifier role be permissionless
    - If permissioned, **medium risk** (Citrea’s zk-verifier, SNARKnado)
- Offchain via alternative, permissionless consensus mechanism: Settlement is managed offchain by a permissionless consensus protocol and/or network of nodes  - **medium risk**
- Federated: Settlement for the layer is finalized by an honest majority of participants in a permissioned, offchain consensus protocol/network - **high risk**
- Offchain via single server: Settlement for the layer is finalized by one party - **high risk**
- Edge case:
    - If the protocol has no fraud proofs, then it is **high risk**
#### Bridge Custody
- No custody with unilateral exit enforced by an L1 transaction - **low risk**
- No custody with optimistic settlement - **low risk**
- One of the BTC bridges is secured by a federated two-way peg, but anyone can participate as a watchtower/challenger and stop malicious withdrawals - **low risk**
- The two-way peg is federated where 1-N operators is required to be honest - **medium risk**
- The two-way peg is governed by an alternative, consensus mechanism - **medium risk**
    - If there is no slashing mechanism then **high risk**
- The protocol has an honest-majority federation securing its two way peg - **high risk**
- The protocol doesn’t enable unilateral exit, and its two way peg is managed by centralized parties - **high risk**
- Edge case:
    - Users can unilaterally exit, but network operators can collude with other network participants to steal funds, and there is no way to challenge fraud - **medium risk**
#### Additional security-related questions
##### Bitcoin security
- Can users unilaterally exit the l2 and/or bridge that is custodying their funds?
    - Yes/no
    - Insert how
- Does the protocol inherit security from Bitcoin miners, full nodes or BTC the asset?
    - Yes/no
    - Insert how
- Does an alternative token play a role in network security/permitting withdrawals?
    - Yes/no
    - Insert how
- Does the protocol create risks for MEV at the protocol, and base layer, level?
    - Yes/no
    - Insert how
- Does the protocol pay fees to Bitcoin miners?
    - Yes/no
    - Insert how

### Customization
This framework can be easier to customize and provide more nuance given the number of scaling solutions that are present in Bitcoin today. For example, related to block production/network operators, we can add even more scoring mechanisms based on how decentralized the network is. E.g. A network with 200 validators is better than a network with 10, and we can customize the assessment to highlight this.

### Summary

This risk assessment is an initial starting point to analyze Bitcoin scaling protocols. It is a living document and is subject to change based on community feedback and development efforts around Bitcoin scaling.

Bitcoin does not have a unified scaling roadmap. There are tradeoffs with every protocol being implemented to support Bitcoin scaling. This framework hopes to capture some of the nuance related to the various designs being proposed.

If you have comments on this framework, please consider adding them below and/or joining our [community chat](https://t.me/+8rv-1I2gkmQ4ZmJh) to discuss.

### Clarifying point(s)
We define permissionless as meaning anyone with sufficient capital and resources can participate in consensus, operating a node, block production, etc.

-------------------------

