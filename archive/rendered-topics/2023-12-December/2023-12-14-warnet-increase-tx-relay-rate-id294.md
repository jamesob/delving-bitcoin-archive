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

amiti | 2023-12-21 18:03:33 UTC | #5

>> every 10 minutes, select a random peer, mine a block

> Is there any merit in mining blocks quicker to get faster feedback, perhaps while “testing the test”? [...] Can you help me understand how does the block production rate affects the test?

the rate of block production is essentially the rate at which transactions are being removed from the mempool. so in theory, we can mining block faster - eg, every 5 minutes - and scale the tx creation rate results by a factor of 2. however, bitcoin core has some tx relay values that wouldn't automatically scale, and thus skew results - [some examples](https://github.com/bitcoin/bitcoin/blob/44d8b13c81e5276eb610c99f227a4d090cc532f6/src/net_processing.cpp#L96-L103). 

but maybe mocktime fixes this?

-------------------------

ajtowns | 2023-12-22 03:04:11 UTC | #6

I think it would be possible to get accurate results faster with mocktime, but I think it'd require a lot of care to get that right, and it would probably be easier to have a real-time baseline to compare against.

-------------------------

m3dwards | 2024-01-19 23:56:02 UTC | #7

I have put together most of this described test into a [Warnet scenario](https://github.com/bitcoin-dev-project/warnet/blob/957aec9e0a286893afe72cf0dfc6780ac35f1da2/src/scenarios/double_tx_relay.py) and have run a few simulations of 100 nodes on one large vm for a few hours each.

The first thing the simulation does is mine a block every few seconds to give each node a starting balance of 50 BTC which is split up into 499 x 0.1 BTC taproot outputs. The following charts do not show that setup and instead start from when each block is produced at a normal rate and the transactions start being sent at a set rate. I implemented a simple block timing function that assumes an exponential distribution of block times with an average of 600 seconds to try and make the test more realistic.

Simulation without PR applied (using tag 26.0)
![Screenshot 2024-01-19 at 17.18.14|690x406](upload://wWEzwfUPoTHuSpuXXS7GYC5WGb3.jpeg)

Both simulations were with a 3.75 tx/s confirmation rate and a 7 tx/s creation rate. On a single box with higher transaction rates I have been struggling to keep the simulation stable.

The top box on both screenshots shows how many transactions are being requested to reconstruct the block. The middle chart is the size of the mempool and bottom chart isn't very useful for this test but just shows number of outbound connections.

I can't see any improvement (or much of a difference) with the PR enabled vs not but perhaps this would become apparent at much higher transaction rates? This simulation also does not RBF or CPFP or construct any chains of unconfirmed transactions. Perhaps this is needed?

Some things I'm a bit hazy on:

1) What is the relationship between relay rate and number of transactions requested? Is it that a higher relay rate should cause more churn in the mempool? As the fees are not all the same I would expect the mempool to keep the best transactions and be fairly similar between all nodes?

2) How would RBF / CPFP / chains of unconfirmed transactions impact this test?

I would like to move these simulations over to Warnet running with a kubernetes backend as I think it should be a lot more stable with 100+ nodes and higher transaction rates. On one box things tend to get a bit network / IO bound.

-------------------------

m3dwards | 2024-01-19 23:48:45 UTC | #8

Simulation with PR applied (Delving would only allow me to post one image per reply)
![Screenshot 2024-01-19 at 19.59.02|690x405](upload://f1Cb2uHFNKq4dPCQfmYxdXpjXJ5.jpeg)

-------------------------

ajtowns | 2024-01-20 03:46:40 UTC | #9

[quote="m3dwards, post:7, topic:294"]
[
Screenshot 2024-01-19 at 17.18.141920×1131 167 KB
](https://delvingbitcoin.org/uploads/default/original/1X/e6e6ecd21c06f1b103f1ca6934c3fd2317023a15.jpeg)

Both simulations were with a 3.75 tx/s confirmation rate and a 7 tx/s creation rate. On a single box with higher transaction rates I have been struggling to keep the simulation stable.
[/quote]

At a 7 tx/s creation rate, I think there should be no problem with mempools staying in sync prior to this PR, so your results make sense to me, for what it's worth.

Do you know what's causing the simulation to be unstable? Too much CPU signing/verifying txs? Or do the wallets get too large/slow, so tracking which utxos to spend is slow? Or is performance okay, and you just start getting inconsistent results between runs?

-------------------------

m3dwards | 2024-01-29 13:01:26 UTC | #10

Good to hear the results so far make sense.

Re the instability, I was finding that nodes were dropping peers until they became disconnected. CPU utilisation seemed ok, so I don't think it was too much time signing. I believe that it's more related to volume of network connections.

We are working on improving the scalability and stability of the simulations in Warnet. I will report back when we can push the tx rate higher.

In the meantime, could you help me understand the following a bit better?

[quote="m3dwards, post:7, topic:294"]
Some things I’m a bit hazy on:

1. What is the relationship between relay rate and number of transactions requested? Is it that a higher relay rate should cause more churn in the mempool? As the fees are not all the same I would expect the mempool to keep the best transactions and be fairly similar between all nodes?
2. How would RBF / CPFP / chains of unconfirmed transactions impact this test?
[/quote]

-------------------------

amiti | 2024-01-29 22:41:00 UTC | #11

@maxedwards here are my thoughts from trying to wrap my head around these questions. Thanks to AJ for helping steer me in the right direction. Let me know if this generally makes sense, or if you have any additional questions. 

The crux of our question is - “what should the tx relay rate be“. To answer this question we want to know- what is safe & what is efficient. 

For each transaction that a user submits to the network, asymmetric bandwidth usage is required for the transaction to propagate to the entire network. Without mitigations, network bandwidth could be intentionally abused or accidentally overused. The minimum fee rate helps mitigate these problems by preventing “free relay” attacks and ensuring that bandwidth usage has a minimum cost in satoshis. The tx relay rate is also important, because it essentially limits the multiplier of each node on incoming traffic to outgoing traffic. Currently, every transaction we see will INV every other node (which would change if we adopt erlay). 

One one side: if the tx relay rate didn’t exist or was too high, an attacker can pay the minimum fee rate to use greater than [num nodes on network] x [size of txn] amount of bandwidth on the whole network. 

On the other side: if the tx relay is too low, transaction propagation is laggy, and there will be dramatic differences in the mempools of different nodes. The rate being “too low” is in relationship to the (tx creation - block creation) rate, aka how many txs are hanging out in the mempool over time. 

We care the most about the top 1MB of the mempool being synchronized, so miners can have the information to confirm transactions paying the highest fee rate. The relay rate is effective because it is implemented with topological fee rate sorting, prioritizing which transactions are worthy of using the network bandwidth based on what is most likely to make it into a block. 

When trying to test & evaluate if we should increase the tx relay rate, it seems very challenging to observe what “too high” looks like. However, it seems more tangible to observe “too low”, by seeing the top-of-mempools be significantly out of sync. One way to do this would be to have each mempool calculate a block and compare the contents, but this would be resource intensive. So a simpler heuristic is looking at the compact block logging because this indicates how many transactions need to be requested to complete the block aka indicators of top of mempool synchronization. 

Since the PR is proposing increasing the relay rate, the strongest support this strategy can offer is evidence that the current tx relay rate is too low for certain network conditions - especially since we have seen more mempool activity in recent months. So, I believe this is what we should be aiming for.

Two additional notes on warnet implementation strategy: 

- if transactions are being created with a random fee rate distribution, that is probably a bit different than what we naturally see out in the wild. on mainnet we usually see tx fee rate rising as the mempool getting more full. this is relevant because we’ve identified *synchronization of the top of mempool* as a key metric. txs with random fee rates will organically lead to a stronger synchronization of top-of-mempool in a short amount of time, while increasing fee rates will see stronger divergences in short time intervals.
- RE RBF: I believe this is a strategy to create many transactions with a limited set of UTXOs. I don’t think we need RBF transactions if that’s not an issue.

-------------------------

m3dwards | 2024-01-31 12:43:52 UTC | #12

Thanks @amiti for the detailed answer.

Regarding the random fee, I have implemented the fee calculation as suggested by AJ: `estimatesmartfee 5` multiplied by `exp(random()*0.2-0.1)` so it will not be completely random but will be estimatesmartfee +- 10% for a small amount of variability.

-------------------------

