# Proof-of-micro-burn – Burning BTC while minimizing on-chain block space usage

RubenSomsen | 2022-08-24 14:30:25 UTC | #1

# Proof-of-micro-burn

Proof-of-micro-burn is a possible way to revitalize the original hashcash use case, e.g. by only accepting emails which have an SPV proof of some burned sats attached, or any other place where spam is an issue.

What you need is a basic merkle sum tree (not sparse), so if e.g. you want to burn 10, 20, 30 and 40 sats for separate use cases, in a single tx you can burn 100 sats and commit to a tree with four leaves, and the merkle proof contains the values. E.g. the rightmost leaf is 40 and has 30 as its neighbor, and moves up to a node of 70 which has 30 (=10+20) as its neighbor, totaling 100.

The leaf hash needs to commit to the intent/recipient of the burn, so that way you can't "double spend" the burn by reusing it for more than one purpose.

You could outsource the burn to an aggregating third party by paying them e.g. over LN but it won't be atomic, so they could walk away with your payment without actually following through with the burn (but presumably take a reputational hit).

While an op_return is needed (or rather, preferred) to burn, you don't necessarily have to put the hash there and can thus save some bytes. One possible place to commit the root hash is in a double taproot commitment in the change output. So while taproot is `Q = P + hash(Q||mast)*G`, you'd commit the root in `P` such that `P = N + hash(N||burn_tree_root)*G`. What's important is that the location is fully deterministic, in order to ensure there isn't more than one tree (which would be yet another way to "double spend").

Finally, you can perform the burn on a [spacechain](https://gist.github.com/RubenSomsen/c9f0a92493e06b0e29acced61ca9f49a#spacechains) (basically a "sidechain" that has burned BTC as its native token) in order to pretty much avoid using mainchain block space altogether, so it should scale much better. It's worth noting that this fully supports SPV proofs, so the third party you're proving the burn to doesn't have to run a full node (though SPV may not be safe enough for big amounts).


## More details about the sum merkle tree

The goal is to burn multiple amounts (10, 20, 30, 40) in a single `OP_RETURN` (100) and specifically indicating how much of the total is intended for what use case. A merkle sum tree achieves this.


```
(1a)  100      (1b)  ABCD       (2a)  100     (2b)  ABCD
    /    \          /    \          /    \         /    \
  30      70      AB      CD      30      70     AB      CD
 /  \    /  \    /  \    /  \    /  \           /  \
10  20  30  40   A  B    C  D   10  20          A  B
```


So while in a normal merkle tree (1a) you hash e.g. A and B to get AB, with a sum tree (1b) you also hash 10 and 20 to get 30.

When you verify the full merkle sum proof (2a + 2b, combined in a single tree), you verify that 10 (A) + 20 (B) add up to 30 (AB), and 30 (AB) + 70 (CD) add up to 100 (ABCD), else the root hash won't match.

This ensures that you can't create a valid tree with commitments that add up to more than the burned amount (essentially a "double spend").

[graphviz]
digraph {
ABCD -> AB
ABCD -> CD
AB -> A
AB -> B
CD -> C
CD -> D
A [label="A = H(A')"]
B [label="B = H(B')"]
C [label="C = H(C')"]
D [label="D =  H(D')"]
AB [label="AB=H(A, 10, B, 20)"]
CD [label="CD=H(C, 30, D, 40)"]
ABCD [label="ABCD=H(AB, 30, CD, 70)"]
}
[/graphviz]

The original posts on bitcoin-dev can be found [here](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-July/020746.html) and [here](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-July/020756.html).

This topic was also discussed in the Bitcoin Optech Twitter space, which can be listened to [here](https://twitter.com/bitcoinoptech/status/1552324101649334273).

-------------------------

naumenkogs | 2023-08-30 08:33:28 UTC | #2

[quote="RubenSomsen, post:1, topic:21"]
You could outsource the burn to an aggregating third party by paying them e.g. over LN but it won’t be atomic, so they could walk away with your payment without actually following through with the burn (but presumably take a reputational hit).
[/quote]

I started wondering if this if this is possible without finalizing the payment? "Here, I locked 10$ in an HTLC to show that I'm real, you can see through your previous hop I'm not gonna resolve it".

This kinda overlaps with the [ln-jamming](https://jamming-dev.github.io/book/) issue, meaning that I wouldn't want to promote even more liquidity abuse while it's not possible to compensate for it.

-------------------------

RubenSomsen | 2023-08-30 09:19:39 UTC | #3

Thanks for reading.

Yeah, in principle timelocking coins with another party would suffice for proving to them (not to others) that you made a sacrifice. I agree that doing this over LN has issues, liquidity wise.

The nice thing about burning is that nothing has to resolve. I personally think it's the cleanest and simplest way, but you can equally make this scheme work by letting the aggregating third party provably lock up coins for some period of time and allowing them to take the coins back in e.g. a year so no coins are lost.

-------------------------

