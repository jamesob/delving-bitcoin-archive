# Implicit Deletions and Improvements in Utreexo IBD

Davidson | 2026-09-05 02:56:25 UTC | #1

Utreexo is a dynamic accumulator that allows the entire TXO set to be represented with only a couple of hashes. It does so by keeping the data structured as a forest of perfect Merkle trees,
allowing nodes to keep only the top part, which takes $O(log(n))$ hashes. In Bitcoin, `n` is the number of TXOs ever added to the accumulator^[In Utreexo, we don't add same-block TXOs or provably unspendable ones. Therefore, the number of leaves inside the accumulator is smaller than the total number of TXOs that have ever existed.]. This allows for space- and I/O-efficient clients
that can run in constrained environments.

To determine whether a UTXO exists, an inclusion proof must be appended to a transaction or block, and Utreexo nodes must verify that proof before accepting it. This data must be transmitted along
with blocks and transactions, creating an extra bandwidth requirement. This is the main trade-off behind Utreexo and is unavoidable given how the accumulator works. During IBD, a node that
only keeps the roots — called a Compact State Node, or CSN — must download a batched proof for all TXOs spent in that block. We compute one big proof instead of a per-TX proof because
inside the block, there may be "overlaps": something that's needed to prove `A` is also required to prove `B`, so we can send it only once, thereby saving bandwidth.

Until recently, we thought this was unavoidable and focused on a caching strategy to improve bandwidth usage. With no optimization, the average overhead caused by a Utreexo proof is
100% with respect to the block size. Therefore, a 1 MB block will have a \~1 MB proof attached to it. The total amount of data required during IBD is roughly twice the current chain size, about 1.3 TB
for the current chain. However, heuristically, we know that UTXOs tend to be spent a couple of blocks after being created. Usually, 80-90% of all TXOs spent in a block were created in the previous 100 blocks. With
a small amount of RAM, you might be able to save 60-70% of the required bandwidth for positions that have changed within the last few blocks. The P2P messages defined by the draft BIP-183^[ https://github.com/utreexo/biptreexo/blob/917bdf3d344e69bfe387af5c0d761c93a91bbd95/bip-0183.md] allow
requesting nodes to specify which positions they need. However, given the fairly large size of today's chain, even 30% is still \~200 GB. Given that the global average internet speed is \~118 Mbps, this
can still be a big problem. Removing this entirely without need for the entire forest would be ideal, since keeping the entire forest would defeat the whole purpose of Utreexo.

As it turns out, it is possible to do so — and in a very simple and elegant way! We can place leaves exactly where they will end up at, without ever needing to delete. This post explains the new method and potential future optimizations.

### Utreexo Additions and Deletions

Before I can show the new algorithm, I'll give a quick recap on how Utreexo works.

As defined by BIP-181, the Utreexo accumulator exposes three different operations: `modify`, `prove`, and `verify`. Let's take a look at the `modify` one. Under the hood, it applies both additions
and deletions to the forest. We don't expose both because order matters. If you `add` and then `delete`, that will generate a different forest than if you `delete` and then `add`. Internally, we always remove
spent TXOs before adding the newly created UTXOs. Doing so consistently is key to staying in consensus.

The forest initially holds all leaves on the bottom row and then hashes them up to the roots. Leaves are distributed in such a way that they always add up to a power of two at the bottom. We then organize
those trees in decreasing order based on how many leaves there are. A 10-leaf forest would look like this^[I've used a sequential numbering system, but this is not how we number things. Check BIP-181 for more details.]:

```
14
|---------------\
12              13
|-------\       |-------\
07      08      09      10      11
|---\   |---\   |---\   |---\   |---\
00  01  02  03  04  05  06  07  08  09
```

Here, `11` and `14` are roots. A CSN would prune everything but `11` and `14`, relying on proofs to learn about the branch and leaf nodes. A proof is an ordinary Merkle proof within the tree
containing that leaf. So the proof for `01` is `00`, `08`, and `13`. You then compute the hash of `00` and `01` to get `07`, `07` and `08` to get `12`, and, finally, `12` and `13` to get `14`. Compare
the computed root with what you currently hold, and that will tell you whether the proof is valid.

However, since Utreexo is **dynamic**, we don't really know how the forest will look at a given height. So the addition process must be a bit smarter than just joining things together and hashing them like
we do with static Merkle trees.

#### Utreexo Additions

To add a new element, we must use a destroy-and-move cycle that works as follows:

1. Let `i` start at 0, and let `curr_add` be the hash of a UTXO being added to the accumulator.

2. If the `i`-th root is empty:

   1. Move `curr_add` to that position and increase the leaf count by one.
   2. Break.

3. Pop what is at that position, replacing it with an empty value.

4. Set `curr_add` to the hash of the concatenation of that root and `curr_add`.

5. Increase `i` by one.

6. Go back to step 2.

Visually, let's assume a forest with five leaves. I'm adding the index of each tree below it; each tree will have $2^n$ leaves, where $n$ is that index. To avoid confusion, forest nodes now use letters, and they are lettered in the same order in which they are created. By the end, you should understand why they are created in this zig-zaggy way.

```
G
|-------\
C       F
|---\   |---\
A   B   D   E             H
————————————————————————————
      02          01      00
```

When adding a new `I`, you first look at the zeroth root and check whether it is empty. Here, `H` occupies it, so we "destroy" it by removing it from there and creating a new root by computing
`J = H(H || I)`. We are now adding `J`.

```
G
|-------\
C       F              J
|---\   |---\          |---\
A   B   D   E          H   I
————————————————————————————
      02          01      00
```

Now we try to add `J`. The next root, `01` is empty, so we add it there.

```
G
|-------\
C       F       J
|---\   |---\   |---\
A   B   D   E   H   I
————————————————————————————
      02         01      00
```

This is our final forest. We now have roots `G` and `J`, and a CSN can get rid of everything below them. One very important thing: **addition operates solely with roots**; we don't need to know
anything below them. This is very efficient and requires only `O(log(n))` hashes per addition, but on average, the number of hashes will be much smaller since you will destroy the smaller trees much more frequently.

#### Deletions

Deletions are even simpler. When deleting node `B`, we just move its sibling up to where their parent was. So for our forest above, when deleting `B`, we just move `A` to where `C` is.

```
G
|-------\
A       F       J
|---\   |---\   |---\
-   -   D   E   H   I
```

We then rehash up to the root, updating the value of `G` to `H(A || F)`. If you are a CSN, you can still do that, but you need the proof to learn about siblings and all positions needed for hashing
up to a root. In this case, your proof is \[`A`, `F`\].

#### Implicit deletions

It turns out that you can update the addition algorithm to already take into account the fact that a TXO is spent. If you know that beforehand, instead of hashing it with
a root when adding, **you simply move the root up!** This very simple change is enough to completely remove the need for an explicit deletion phase afterward.

To visually represent it, imagine we want to arrive at the forest above, but we start with an empty forest. I'll show you each step. Keep the algorithm I told you in mind.

Adding `A` — Root `00` is empty, so I leave it there.

```
                          A
————————————————————————————
      02         01      00
```

Adding `B` — Root `00` is populated, so I have to destroy it. However, I know `B` was deleted, so instead of taking `H(A || B)`, I just move `A` up to root `01`.

```
                A
                |---\
                -   -
————————————————————————————
      02         01      00
```

Now we add `D` — `00` is empty, so we leave it there.

```
                A
                |---\
                -   -    D
————————————————————————————
      02         01      00
```

Adding node `E` — We have to destroy both `00` and `01` and move them to `02`.

```
G
|-------\
A       F
|---\   |---\
-   -   D   E
```

Finally, we add the two missing nodes — nothing changes from the addition we are used to.

```
G
|-------\
A       F       J
|---\   |---\   |---\
-   -   D   E   H   I
```

And voilà! We have the exact same forest we had above, but we didn't explicitly delete anything!

Notice, however, that this will only work up to a certain height. After the last hinted height, all UTXOs are where they should be for that specific height. If something is deleted afterwards, you won't be able to remove it without a proof, since you already added it.

After finishing IBD, you go to the normal Utreexo business using proofs and regular deletion. This approach is an improvement for IBD only.

#### How to know something is already spent

One important question is: how do I know a TXO is really spent without having to trust anyone? Someone could DoS me by claiming a UTXO is spent when it actually isn't. This would cause my accumulator
to drift from the network, making me reject valid proofs.

The way we do this is by using Swift Sync^[ https://gist.github.com/RubenSomsen/a61a37d14182ccd78760e477c78133cd]. Our spentness oracle is Swift Sync's hintsfile, a file that tells you whether a TXO is spent. And to verify the hintsfile, I keep a hash aggregate that
should zero out by the end if the hintsfile is balanced — all inputs were removed, and all STXOs were added. No malicious entity can force me into a diverging accumulator.

### Performance improvements

Aside from reducing bandwidth usage, this design allows us to make the process considerably more concurrent. Our WIP implementation for AssumeValid Swift Sync^[ https://github.com/getfloresta/Floresta/pull/1115] processes blocks in whichever
order we receive them. No slow peer can slow our progress because we process blocks that our faster peers give us. We can use all the resources we have. In all the tests we ran, with internet bandwidths
ranging from \~30Mbits/s to 2 Gbps, we were **always** bandwidth-constrained, even on a cheap Raspberry Pi. This shows great potential for fast and cheap IBD. The only sequential
part of this process is applying the changes to our accumulator. We have a thread that does just that: it receives a list of what has changed, holds those that can't be applied because their ancestors
are not yet available, and applies them whenever possible. This thread, however, has an extremely small resource footprint and uses a negligible amount of CPU for its job.

A non-AssumeValid version is under active development, but I'll write about that later on. However, we do have an idea for how to make it fairly concurrent and bandwidth-efficient.

### Future research

We have some ideas about precomputing the addition concurrently, making the sequential thread's job even easier. We can also precompute the root set from deletions and have the sequential thread only
check its validity and apply it to the current accumulator. That would make most of our workload concurrent, even during normal Utreexo operation.

### Acknewlegements

Special thanks for Vinteum^[https://vinteum.org/], 2140^[https://2140.dev/] and HRF^[https://hrf.org/] for both supporting the people who came up with this, but also hosting in-person events where we discussed these.

This work was developed by Tadge Dryja, Calvin Kim and Ruben Somsen, me and some other people who helped reviewing our ideas and code.

-------------------------

