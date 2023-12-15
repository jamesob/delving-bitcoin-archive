# Warnet + Increase Tx Relay Rate

amiti | 2023-12-14 21:14:15 UTC | #1

# Overview
[PR #28592](https://github.com/bitcoin/bitcoin/pull/28592) proposes doubling the tx relay rate of bitcoin core nodes. While the code change is tiny, the crucial part is evaluating the impact to the network as a whole. Warnet seems like a good fit for trying to observe network effects, so this is a thread to brainstorm how we can set up meaningful scenarios. 

As I see it, there are two main components to represent mainnet: 
1. introducing transactions to the mempool via broadcast & removing them via blocks
2. the network setup- the number of nodes & the connection graph 

# Transactions
brainstorm of what would make the scenario useful, interested in hearing opinions: 
- we would want to observe the impact of introducing txs into the network at different rates, as well as being confirmed into blocks at different rates
- transactions and blocks should come from varied nodes on the network
- bonus would be observing impact of RBF transactions, esp with current mainnet patterns
- what else? :) 

# Network Setup
we want to mimic the real network as closely as possible, which is constrained by two main things:
1. mainnet is strongly obfuscated 
2. resource usage of what warnet can provide

but of course, we can still make representative abstractions. @pinheadmz was able to run tests with 250 nodes on docker, and with warnet support for kubernetes, we can support many more nodes. so the question remains of what number of nodes (total & reachable/non-reachable), and network graph would be helpful to depict something comparable to mainnet. 

# Desired Outcomes
After setting up scenarios, what are the metrics are important to observe? Some ideas, again, just to get the ball rolling: 
- Can we estimate expected increase in bandwidth usage? CPU usage?  
- Does increased relay rate tangibly impact mempool churn in high congestion / high fee rate situations? 
- How can we observe the impact on tx propagation with the different relay rates? 
- Is there a tangible impact on memory from having a higher relay rate? (because of internal send queue growing large for each peer with lower relay rate) 

# Next steps
I'm interested in hearing people's thoughts on: 
1. does this make sense? 
2. what's missing? 
3. what's important to prioritize?

-------------------------

