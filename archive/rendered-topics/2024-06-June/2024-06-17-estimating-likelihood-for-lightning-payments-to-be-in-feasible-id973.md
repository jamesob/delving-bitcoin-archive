# Estimating Likelihood for Lightning Payments to be (in)feasible

renepickhardt | 2024-06-17 18:27:44 UTC | #1

Dear fellow Lightning Network developers, 

as some of you are already aware (through out of band communication) I am currently developing and finalizing a paper with a mathematical theory of payment channel networks. For the paper I created a small example that shows how to apply the rather technical and abstract geometric concepts to describe the Lightning Network. I thought this standalone example / application might be of interest for you. 

The following is a summary of an ipython notebook from my [Github reopository in which I develop the theory and research paper](https://github.com/renepickhardt/Lightning-Network-Limitations/blob/46e6fce28510be1db646dc8967ed2299e19243d8/likelihood-of-payment-possability/Likelihood%20of%20Payments.ipynb
) (Caution: The current latex file of the paper in the repository is a very early / incomplete and wrong draft. I would not recomand to read it yet. I expect a much more complete and accurate version to be updated later this month):

## Abstract 

In [prior research we have provided a method of estimating the liquidity in payment channels](https://arxiv.org/abs/2103.08576). This was used to compute success probabilities for payments. The proposed model has several flaws:

1. It assumes the liquidity distributions in channels are independent of each other
2. it uses a uniform distribution to model the uncertainty

The second issue is being addressed through the [introduction of bimodal models](https://lightning.engineering/posts/2024-05-23-pathfinding-2/). However [we believe the observed bimodal distributions emerge due to the geometry of payment channel network](https://github.com/renepickhardt/Lightning-Network-Limitations/blob/906b3a4d691cbfb30c2f385c6697b4fe2bb81676/Limits%20of%20two%20party%20channels/Geometric%20Argument%20for%20Bimodal%20Liquidity%20Distribution%20in%20Payment%20Channels%20of%20the%20Lightning%20Network.ipynb). This happens in particular if the first assumption is dropped. 

## Contribution
This notebook provides a tiny examples to demonstrate how one can compute the payment success rate between two peers on the lightning network for a payment amount even if no information about the liquidity distribution is availble. We do however assume that the sender owns more coins that he wishes to send (Though we could easily drop that assumption).

For this we start from the assumption that all feasible wealth distributions (not channel states!) in the given topology are equally likely to occur. As payments are changes in the wealth distribution **we just have to test for all feasible wealth distributions if the change induced by the payment is still a feasible wealth distribution**. The fraction of payments for which this is possible is the expected success probability of a payment. 

### Math formulation

Let $w=(w_1,\dots,w_n)$ be a wealth vector. Furth let $b_1,\dots,b_n \in \mathbb{Z}^n$ be unit base vectors that span the space of feasible wealth distributions. Then a payment is the new wealth vector $w'$ which can be computed via the following:

$w'= w -amt\cdot b_{src} + amt\cdot b_{dest}$

if $w'$ is feasible then the payment was feasible.


**Caution** this assumes that all feasible wealth distributions are equally likely. This is neither the same as assuming that the liquidity in channels is uniformly distributed nor the same as assuming that the wealth is equally distributed. The reason is that the network topology makes many wealth distributions infeasible. Thus despite assuming uniformity among the feasible wealth distributions this assumption is hopefully close enough to match the reality of a skew power law distributed wealth.

**Important**: The probabilities for a payment to be feasible is not related to attempt probabilities! Instead the probabilties computed with this method estimate weather it is likely that with sufficient attempts a payment is feasible and thus eventually succesfully being deliverd.
 
## Results

Take the following Network
![A mathematical theory of payment channels|690x388](upload://8k4qTANUgedCZKwtMiKS5Yjaxol.png)

We get the blue curve that depicts how likely it is that a payment of a certain amount can be sent from any peer to any other peer. In comparison we depicted the likelihood of the min cost flow. Of course the min cost flow probability has to be lower as the payment could still be feasible while the min cost flow fails: 
  
![comparisionWithMCF|640x480](upload://sQQcc4NnL6pjwxEGaEQPqmVkfKp.png)

### Example
Let us look at the likelihood of node `0` being able to send `5` coins to node `1`.  Given our methodology the likelihood that this payment is feasible on this network is: `39.22%`

Using the methods of probabilistic pathfinding with independend and uniform liquidity distributions we see:
```
P1: s=0.00% (0)--- 3---->(1)

P2: s=21.87% (0)--- 7---->(2)---11---->(1)
```
This means of course node `0` cannot send `5` coins on `P1` to node `1` as the path has only one channel with capacity 3. 

The likelihood on the send path is `21.87%`

The minmum cost flow would however split the payment and attempt to send `coin` along `P1` and `4` coins along `P2`. This results in a `25%` chance to settle this MPP attempt. 

If on the other hand node `1` wanted to send `5` coins to node `2` then we get the following likelihoods:

```
5 coins from (1) to (2)		 expected success rate = 64.05% 		 
P1: s=58.33% (1)---11---->(2)
P2: s=0.00% (1)--- 3---->(0)--- 7---->(2)
Not both P1 and P2 fail: 58.33%
MCF: s=43.75% 4 to P1 and 1 to P2
```
The increase makes sense as node `1` and `2` have access to more liquidity and thus should be more likeli to be able to conduct a payment. 

## Conclusion

We can produce a likelihood that estimates how likely a payment is feasible on the lightning network. We have seen that it is not necessary to estimate the liquidity in channels in order to do so. In particular there are always feasible states from which the payment is not feasible. (for example the receiving node doesn't have sufficient inbound liquidity)

**Thus Lightning Network payments cannot work in 100% of the time.**


Also we have seen that optimally reliable payment flows always have a lower success probability than the probability that the payment is feasible. This is to be expected. However relying purely on flows and paths it seemed tricky to estimate the feasibility of a payment as one would have to simulate the payment loop. 

Of course extending this method to a larger network is a bit more work as testing feasibility and sampling feasible wealth distributions is a bit more tricky (but possible in polynomial time as will be explained in detail in the paper)

I am curious about your thoughts, feedback or questions. 

with kind regards Rene Pickhardt

-------------------------

harding | 2024-06-25 19:29:50 UTC | #3

Thanks for this research, Rene!  I was wondering if this is something you imagine being incorporated into user and business software.  For example:

- Businessperson Bob typically receives payments up to 0.05 BTC.  His node management software occasionally runs a background job that calculates the average likelihood of feasibility of a 0.05 BTC payment from every node on the network to his node.  The node management software also looks at current liquidity advertisements and simulates what would happen if Bob had a channel with the advertiser, calculating a hypothetical alternative average likelihood of feasibility.  If the hypothetical alternative with a new channel has a significantly higher average likelihood of feasibility, Bob's node management software automatically accepts the liquidity advertisement and opens the new channel.

- User Alice makes a regular monthly bill payment set up through BOLT12 offers.  She's configures her wallet to start to try paying 5 business days before the due date.  The first try and first few automatic retries don't succeed.  Before her wallet marks the payment attempt as a failure or takes other steps, it checks the likelihood of feasibility.  If it's low but still practical, it will keep retrying at lengthening intervals for another few hours or days before finally marking the payment attempt as a failure.

-------------------------

renepickhardt | 2024-06-26 07:34:41 UTC | #4

[quote="harding, post:3, topic:973"]
Businessperson Bob typically receives payments up to 0.05 BTC. His node management software occasionally runs a background job that calculates the average likelihood of feasibility of a 0.05 BTC payment from every node on the network to his node. The node management software also looks at current liquidity advertisements and simulates what would happen if Bob had a channel with the advertiser, calculating a hypothetical alternative average likelihood of feasibility. If the hypothetical alternative with a new channel has a significantly higher average likelihood of feasibility, Bob’s node management software automatically accepts the liquidity advertisement and opens the new channel.
[/quote]

This is a very interesting use case which I haven't thought of so far and it certainly could be done in this way. I do have a few additional thoughts on this use case:

1. If a user was interested in the likelihood of feasibility of a payment from all other nodes he might want to look at the histogram of amounts that he could receive from any user. So for a given random wealth distribution (which can be [sampled with this library](https://pypi.org/project/drs/) as [explained in this notebook](https://github.com/renepickhardt/Lightning-Network-Limitations/blob/906b3a4d691cbfb30c2f385c6697b4fe2bb81676/Limits%20of%20two%20party%20channels/A%20Mathematical%20Theory%20of%20Payment%20Channel%20Networks.ipynb)) he could use [this method to compute a feasible network state](https://github.com/renepickhardt/Lightning-Network-Limitations/pull/2). From here one could use [Gomory-Hu Trees](https://en.wikipedia.org/wiki/Gomory%E2%80%93Hu_tree) to compute the all pair max flow more efficiently than computing a max flow problem from every user to Bob. Similar to [this notebook one could compute the distribution of max flows / min cuts](https://github.com/renepickhardt/Lightning-Network-Limitations/blob/906b3a4d691cbfb30c2f385c6697b4fe2bb81676/likelihood-of-payment-possability/An%20upper%20Bound%20for%20the%20Probability%20to%20be%20able%20to%20successfully%20conduct%20a%20Payment%20on%20the%20Lightning%20Network.ipynb)  and have a more precise view. This is because besides the percentile of nodes that are below the amount which Bob wishes to receive one would also get confidence intervals (if for example this method was repeated for several random wealth distributions). **Important**: Assuming random liquidity in channels as I have done in the notebook to compute the min cut distribution is probably not as precise as starting from random wealth distributions. 

2. Of course when sampling random wealth distributions Bob could take into account that he ownes `x` coins in his channels. Furthermore as Bob knows the capacity and state of his channels he could also know what wealth his peers own at least. putting these constraint to the polytope of wealth distributions from which to sample can be done with the above mentioned library.

3. Similar to 2. Bob could use his local knowledge see the receiving problem as max flow problem with his peers as sinks (assuming he has enough inbound liquidity in those channels) This knowledge would in particular be taken into account for the liquidity advertisement which he uses in his simulation. (Those are just som engineering / modelling optimizations. I am not sure how much improvement they will bring)

4. The biggest issue is that with larger lightning networks it will be always harder to successfully sample feasible wealth distributions from where to start the above described computation. That is because it seems that the [likelihood for sampled Bitcoin wealth distributions to also be feasible on the lightning network declines when the network grows](https://github.com/renepickhardt/Lightning-Network-Limitations/blob/906b3a4d691cbfb30c2f385c6697b4fe2bb81676/Limits%20of%20two%20party%20channels/A%20Mathematical%20Theory%20of%20Payment%20Channel%20Networks.ipynb) in size. Furthermore the test of feasibility is also rather costly. As discussed (out of band) with Stefan Richter (who was first to state this in our conversation) testing if a sampled wealth distribution has a feasible state cannot only be solved through linear integer programming but it boils down to solving a particular multi source multi sink max flow problem.

[quote="harding, post:3, topic:973"]
User Alice makes a regular monthly bill payment set up through BOLT12 offers. She’s configures her wallet to start to try paying 5 business days before the due date. The first try and first few automatic retries don’t succeed. Before her wallet marks the payment attempt as a failure or takes other steps, it checks the likelihood of feasibility. If it’s low but still practical, it will keep retrying at lengthening intervals for another few hours or days before finally marking the payment attempt as a failure.
[/quote]

Yes absolutely. Knowing the likelihood of feasibility for a payment and being able to decide how often to attempt a payment and when to give up to make an on chain transaction to either pay someone on chain or open a new channel is exactly the one application I had in mind.

-------------------------

stefanwouldgo | 2024-06-26 14:33:24 UTC | #5

First of all, I want to congratulate @renepickhardt for pioneering this promising approach.

If I am allowed a little nitpicking that doesn't in any way detract from the achievement, I believe that comparing the min cost flow probability (over the probability space of independently uniform channel balances) and the probability of feasibility calculated here (over the probability space of uniformly chosing a feasible wealth distribution) doesn't make much sense, because these are two different models that give incomparable results. 

In particular, if I am not mistaken, this observation here is not true in general:

[quote="renepickhardt, post:1, topic:973"]
Of course the min cost flow probability has to be lower as the payment could still be feasible while the min cost flow fails:
[/quote]

To see this, consider the example network that Rene gave, but make it a little easier to compute by setting the capacity of channels 01 and 02 to 1, and 12 to 2. Then there are only 12 possible overall channel states. Of these, there are 3 states that lead to the wealth distribution 121 (0 has 1, 1 has 2, 2 has 1 coin), and 3 states that lead to 112, so overall there are 8 feasible wealth distributions. 

Now let's ask ourselves how probable is it that node 1 can pay 1 coin to node 0 as well as 1 coin to node 2. This is feasible whenever node 1 has at least 2 coins, node 0 has at most 1 and node 2 has at most 2. There are exactly 3 wealth distributions that apply: 121,022, and 031. So this is feasible with probability 3/8.

Now observe that there is only one flow that can achieve these two payments at the same time: 1 coin each directly from 1 to 0 as well as from 1 to 2. This is the min cost (=most probable) flow, and there are 5 network states where it will succeed (the same as the ones coming from the feasible wealth distributions above, but 121 is counted 3 times), So its success probability is 5/12, which is strictly greater than 3/8.

So the min cost flow in this example is more probable than the feasibility of these two payments, which makes no sense but is an artefact of the different probability spaces that are compared.

-------------------------

