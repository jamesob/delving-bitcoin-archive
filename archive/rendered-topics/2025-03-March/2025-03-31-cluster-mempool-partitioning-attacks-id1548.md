# Cluster mempool partitioning attacks

stefanwouldgo | 2025-03-31 14:57:05 UTC | #1

This continues the discussion from [How to linearize your cluster](https://delvingbitcoin.org/t/how-to-linearize-your-cluster/303) on how requiring linearizations might impact mempool partitioning attacks. I have started this new topic in order not to clutter that topic too much and because it seems to me that this is interesting/problematic independently of that discussion. 

@sipa brought up the following attack as a reason against requiring linearizations/chunkings when doing RBF with non trivial mempool dependencies (I am going to assume we would only require them in this case, in order to avoid a potential resource draining attack): an attacker might rush several versions of a group of txns depending on an existing cluster to different mempools. Each version of this attacker subcluster conflicts with the others, but none can replace the others. There are multiple possible reasons for this, chief among them that these versions have corresponding fee diagrams that are either equal or incomparable. The result is that this partitions the mempools in the network with mutually exclusive versions of the cluster. This attack is possible today, though current RBF rules don’t compare mempool diagrams, and I am going to argue that that’s a good thing.

Now IIUC, the argument against requiring chunkings is that as an honest network participant I will only learn one version of the cluster and when attaching a new tx to it, I can only deliver an optimized chunking for this version. However, this chunking might not do much for other versions, because these might not overlap much. And so, my new tx will not be relayed across partition boundaries because my chunking cannot compete with the attacker chunkings. 

Now recall that I only argue for requiring linearizations when replacing existing txs using RBF, because it seems to me that this is the only scenario where we are time limited during relay. Please let me know if that’s a false assumption!

It appears to me that if the attacker constructs his txns in such a way that the different versions have incomparable diagrams, then independently of requiring linearizations, the rule that RBF requires strictly improving the diagram means that the attacker can keep me from relaying any RBF tx in this cluster, because I don’t even get to know all the versions I would have to improve upon. This is possible because for diagrams, we have only a partial order instead of the strict ordering with scalar fees. I’m referring to the RBF rules proposed here: [https://github.com/bitcoin/bitcoin/blob/20e42a4a7423d7bc62326cdb6405bc6d5900953e/doc/policy/mempool-replacements.md](https://github.com/bitcoin/bitcoin/blob/20e42a4a7423d7bc62326cdb6405bc6d5900953e/doc/policy/mempool-replacements.md)

Is this still the current proposal for RBF in a cluster mempool world? If it is, this seems like a devastating attack to me. If it is outdated I apologize for wasting everyone's time and would be grateful for a pointer to the current one.

-------------------------

