# Games in the head (and fraud proofs for the plebs)

salvatoshi | 2024-01-20 17:00:20 UTC | #1

While thinking about how to optimize fraud proofs implemented using MATT contracts, I came up with a quite neat approach that applies more generally to any multiple-round, 2-party game. The idea is very simple, so I wouldn't be surprised if it's not really novel, and would appreciate any literature reference.

**TL;DR:** any two party game played in a MATT contract, with arbitrarily complex game rules, can be turned into a protocol where the data overhead of each move is only about 2 hashes.

**TL;DR2:** it's possible to implement fully general fraud proofs that with an estimated cost of around 7000 vbytes and 17 total transactions. Fraud proofs applied to CoinPools of 256 user should cost 2000 vbytes and only 5 transactions.

# 2-party games

Imagine an n-step, 2-party protocol where players take turns making their "move" (for example, a fraud proof).
Such games can be represented as a sequence of $2n$ transactions that update the "state" of the contract:

- $s_1 = f_0(s_0, x_0)$
- $s_2 = f_1(s_1, x_1)$
- ...
- $s_{2n} = f_{2n-1}(s_{2n-1}, x_{n-1})$

where the state of the final transaction would be used to decide a winner.

That is, each step is revealing the current player's next move (where Alice plays all the even $f_{2i}$ moves, and Bob all the odd $f_{2i+1}$ ones). The protocol verifies that the given input $x_i$ is valid according to the rules, and the state is updated correctly.

## Direct implementation

Such protocols can be easily implemented with MATT contracts as a chain of UTXOs; each UTXO would contain a commitment to the current state $s_i$ (and is already programmed with the function $f_i$); the witness would be given the state $s_i$ (all the content, not just the commitment!), and the player's move $x_i$; the script would then verify that $x_i$ is valid, and compute the commitment to the next state $s_{i+1}$. Therefore, the witness would contain:

- the entire state $s_i$
- the content of $x_i$
- the control block, and a script with size proportional to the complexity of $f_i$

That's quite a bit of overhead if compared with just the size of $x_i$, which is the only new information revealed in a step of the protocol.

## Games in the head

We can do better: the idea is to _not_ check the current player's move at all. The move becomes just "I declare that my move is $x_i$, and the hash of the next state is $H(s_{i+1})$. The output of the UTXO commits to the previous state $s_i$, the input $x_i$, and the new state $s_{i+1}$. If the current player did the move incorrectly, then on the next move, the other player can immediately prove that and win the game!

More formally, define the initial state-in-the-head $h_0 = H(s_0)$ (where $H$ is a hash function, or any other constant-size commitment). Then, at each step, the next state-in-the-head (stored in the UTXO) is computed as:

$h_{i+1} = H((h_i || H(x_i)) || H(s_{i+1}))$

If a move is executed incorrectly (that is, either $x_i$ is invalid or it's not true that $f(s_i, x_i) = s_{i+1}$, then the other player can prove it by opening the commitment $h_{i+1}$ on its next move (that is, proving that the computation $f_{i+1}$ was incorrectly executed), immediately winning the game.

Otherwise, each moves just amounts to revealing $h_i$ (a hash), $x_i$, and $H(s_{i+1})$. All the Script does is to compute $h_{i+1}$ (a tiny Script!) and make sure that it's committed to in the next output. Therefore, the overhead amounts to just two hashes (other than the usual 65 bytes taproot control block, and other fixed transaction costs).

## Conclusions

Playing games in the head allows each transaction of the protocol to have size $|x_i| + O(1)$, independently from the complexity of the state transitions $f_i$, and the size of $s_i$.

The increase in code complexity is moderate, while it can bring noticeable savings in the total transaction size.

This could be generalized to some $k$-party games.

# Application to fraud proofs

While fraud proofs are never expected on-chain, the total cost of a fraud proof (vbytes needed, times the fee rate) should be roughly proportional to the size of the *bond*, that is, the amount that a wannabe-cheater will lose at the end of the game.

How large is this cost?

## Direct implementation
I implemented the raw version of the bisection protocol in [pymatt](https://github.com/Merkleize/pymatt), the Python framework I'm working on:
- Fraud proof [contracts](https://github.com/Merkleize/pymatt/blob/bc374a805cfa065179281202a774441100c6e253/matt/hub/fraud.py)
- [Pytest demo](https://github.com/Merkleize/pymatt/blob/bc374a805cfa065179281202a774441100c6e253/tests/test_fraud.py) of the [256-game toy example](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-November/021205.html)
- [Example transactions](https://gist.github.com/bigspider/82394963f74b4151b453778fa143e8a9) from the above pytest demo

The size of each bisection step (Alice + Bob transactions) is around 500 vbytes.
Taking $2^{32}$ steps as a plausible upper bound for the number of computational steps one could ever need, that would lead to around 32*500 = 16000 vbytes over 65 transactions.

Note that this cost is independent from *what* computation is performed, so it could for example be composed with the work that Johan's work on [elftrace](https://delvingbitcoin.org/t/verification-of-risc-v-execution-using-op-ccv/313) towards fraud proofs for a a Risc-V VM without any substantial impact on the cost estimates (as only the Script in the leaves is different for different computations).

## Bisection in the head

Applying the game-in-the-head approach to the bisection protocol, some napkin math makes me estimate a total worst case cost, for $2^{32}$ operations, around 14000 vbytes. 

A modest saving. That's because the saving does not affect the fixed per-transaction costs, that were already the dominant cost.

## Further optimizations

Improving this further requires reducing the total number of transactions. That has the added benefit of a smaller number of rounds, which is highly beneficial in practical protocols.

The key observation is that we can generalize the bisection protocol to a $k$-section protocol, building a $k$-ary Merkle tree instead of a binary tree of the computation trace. The number of $k$-section rounds for $N$ steps is $\log_k N$.

The complexity of the $f_i$ increases linearly in $k$, but thanks to the game-in-the-head approach, this cost is not paid on-chain (or it's paid at most once).

I did not yet implement this, but from some napkin math, I expect the total cost for $k = 16$ and $N = 2^{32}$ to be around 7000 vbytes, and since $log_{16} N = 8$, a total of 17 transactions for the complete fraud proof.

## Round-efficient fraud proofs in pools

I'm particularly interested in the applications to fraud proofs in pool-like constructions, for example for [affordable exit strategies](https://delvingbitcoin.org/t/aggregate-delegated-exit-for-l2-pools/297).

For a pool of up to 256 users, I estimated that $k = 16$ would bring the total cost below 2000 vbytes (compared to >4000 vbytes of the unoptimized bisection).

Most importantly: in this version ***a fraud proof requires just 5 transactions, instead of the initial 17***. Only 2 of them are made by the attacker (who might purposedly delay its response as much as possible), contrary to 8 in the unoptimized protocol.

7 transactions would support fraud proofs for pools up to 4096 users.

The reduction in the number of rounds would likely be of high practical value, especially when these pool constructions are composed with the Lightning Network (see the "protocol uplifting" of [CoinPool](https://coinpool.dev/)).

-------------------------

