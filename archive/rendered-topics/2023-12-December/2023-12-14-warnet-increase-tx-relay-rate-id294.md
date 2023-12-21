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

ajtowns | 2023-12-18 02:43:32 UTC | #2

[quote="amiti, post:1, topic:294"]
* we would want to observe the impact of introducing txs into the network at different rates, as well as being confirmed into blocks at different rates
* transactions and blocks should come from varied nodes on the network
* bonus would be observing impact of RBF transactions, esp with current mainnet patterns
[/quote]

I was thinking something like this.

 1. have three parameters:
    * size of the network (100 nodes total, 90 listening, 10 non-listening)
    * tx creation rate (5tx/s to 70tx/s)
    * tx confirmation rate (3.75tx/s, 7.5tx/s, 15tx/s)
 2. start the nodes
    * run 26.0 out of the box consistently everywhere
    * start each node with its own wallet
    * maybe reduce `MAX_OUTBOUND_FULL_RELAY_CONNECTIONS` from 8 to 5 (to simulate the greater distance between nodes in a network of 10k-100k nodes, despite only having 100 nodes)
    * set `-blockmaxweight` to 999000, 1998000 or 3996000 depending on the target tx confirmation rate
 3. start mining blocks
    * every 10 minutes, select a random peer, mine a block
    * once a block reward matures, create a tx splitting the reward into 0.1 BTC outputs at a 500sat/vb feerate
 4. generate small txs at the nominated rate:
    * 1-in, 1-out taproot tx is 111 vbytes. if signatures are annoyingly slow, make it p2wsh where the script is "<61B push> OP_DROP OP_TRUE", which should also be 111 vbytes
    * 1-in, 0-out tx to burn funds: ie, just an 0-value 32 byte OP_RETURN output when the input amount is below 0.05 BTC perhaps?
    * output should just be a new change address
    * fee rate should `estimatesmartfee 5` multiplied by `exp(random()*0.2-0.1)`
    * do this randomly amongst all peers, eg get 5tx/s overall by having 100 nodes generate 1tx every 20s

The easiest thing to report is the performance of compact block relay, so run with `debug=cmpctblock` look at the "Successfully reconstructed block" lines, eg:

```text
2023-12-18T02:01:46.628563Z [cmpctblock] Successfully reconstructed block
   00000000000000000001083bc8296739b4a65f1264f69104a52f16962a24af5b
   with 1 txn prefilled, 3513 txn from mempool (incl at least 9 from extra pool)
   and 7 txn requested
```

In particular, my expectation is that if you increase the sustained tx creation rate above 18 tx/s you'll start to get inconsistent mempools and the "txn requested" number may rise. Having those inconsistencies appear at the top of the mempool probably requires you to be doing RBF or to have chains of unconfirmed transactions, particularly involving CPFP.

-------------------------

amiti | 2023-12-20 21:54:22 UTC | #3

overall this proposal makes a lot of sense to me. thanks @ajtowns :) 

a couple thoughts & questions: 
> maybe reduce `MAX_OUTBOUND_FULL_RELAY_CONNECTIONS` from 8 to 5 (to simulate the greater distance between nodes in a network of 10k-100k nodes, despite only having 100 nodes)

we can test out more than 100 nodes, but reducing this number still make senses to me. I'd be curious to see both 4 and 8 to see if the patterns match our expectations. so, I'm proposing adding this as one of the initial parameters. 

> once a block reward matures, create a tx splitting the reward into 0.1 BTC outputs at a 500sat/vb feerate

is the point of this suggestion to have an arbitrary but deterministic technique for creating plenty of UTXOs to support whatever general transaction patterns we are interested in?

> In particular, my expectation is that if you increase the sustained tx creation rate above 18 tx/s you’ll start to get inconsistent mempools and the “txn requested” number may rise.

this sounds like a good way to test-the-test: with whatever initial setups are chosen, keep increasing the tx creation rate until we see the tx requested number rise, to identify when functionality is starting to deteriorate. 

one aspect I’m still trying to grok is the asymmetry of rates between inbound & outbound peers. the current patch proposes 14tx/s for inbound, and we have the 2.5x multiplier for outbound peers. how do we compare these node level values with whatever we find to be reasonable at the network level?

-------------------------

m3dwards | 2023-12-21 11:23:16 UTC | #4

I'm putting this test together with warnet but I have a question regarding:

[quote="ajtowns, post:2, topic:294"]
every 10 minutes, select a random peer, mine a block
[/quote]

Is there any merit in mining blocks quicker to get faster feedback, perhaps while "testing the test"?

Can you help me understand how does the block production rate affects the test?

-------------------------

