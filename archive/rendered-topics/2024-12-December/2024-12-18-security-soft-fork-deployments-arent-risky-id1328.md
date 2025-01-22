# Security soft fork deployments arent risky

Chris_Stewart_5 | 2024-12-19 20:54:18 UTC | #1

I think its useful to think of soft forks in 3 categories

- Feature soft forks
- Security soft forks
- Both Security and Feature Soft Forks

### Feature soft forks

Here are examples of feature soft forks.

- [BIP16](https://github.com/bitcoin/bips/blob/8e59f7414bef6203809deaad972a3d7a3a0c2f7a/bip-0016.mediawiki)
- [BIP65](https://github.com/bitcoin/bips/blob/8e59f7414bef6203809deaad972a3d7a3a0c2f7a/bip-0065.mediawiki)
- [BIP66](https://github.com/bitcoin/bips/blob/8e59f7414bef6203809deaad972a3d7a3a0c2f7a/bip-0066.mediawiki)

When people mention risk associated with soft forks. They usually mean chainsplit risk associated with deploying a new feature. Potentially non-upgraded node and an upgraded node could disagree with the validity of a given transaction mined in a block. We don't want chainsplits - thus we require widespread consensus for deploying feature soft forks

### Security soft forks

Here are some examples of what I call "security soft forks"

- [BIP42](https://github.com/bitcoin/bips/blob/8e59f7414bef6203809deaad972a3d7a3a0c2f7a/bip-0042.mediawiki)
- Worst case block validation time (part of GCC)
- 64 byte txs (part of GCC)
- Time warp attacks (part of GCC).

### Both security and feature enhancement soft forks

- [BIP141](https://github.com/bitcoin/bips/blob/8e59f7414bef6203809deaad972a3d7a3a0c2f7a/bip-0141.mediawiki)
- [BIP143](https://github.com/bitcoin/bips/blob/8e59f7414bef6203809deaad972a3d7a3a0c2f7a/bip-0143.mediawiki)
- [BIP341](https://github.com/bitcoin/bips/blob/8e59f7414bef6203809deaad972a3d7a3a0c2f7a/bip-0341.mediawiki)

These BIPs were used to fix some sighash vulnerabilities that existed before bitcoin. To fix these security issues, we required allocating a new witness version. This introduced deployment risk because we were enabling new features (v0,v1 segwit).

### The takeaway

Security soft forks do not have much chainsplit risk as feature and bundled ('Both security and feature') soft forks. 

If someone was to try to trigger a chainsplit while a security soft fork is being deployed, by definition they are participating in behavior that is hostile to the network. The key insight here is **we want to fork people exploiting the protocol off the network**

Lets look at the Timewarp attack. This means the miner is attempting to increase block velocity and claim more rewards for themselves. If someone is actively attempting to exploit this, we want to fork them off the network via soft fork. If no one is exploiting the timewarp vuln, then we wont have a chain split. I.e. there is no risk to deploying this.

The only deployment risk I can think of related to security soft forks is implementation risk (i.e. a bug in the implementation). 

Maybe this is self evident to other developers, but it was a realization I had this morning when reading through the [Great Consensus Cleanup](https://delvingbitcoin.org/t/great-consensus-cleanup-revival/710/63).

I think we should encourage deploying security soft forks independent of feature soft forks whenever possible to encourage deployment of security fixes as soon as the fix can properly be reviewed.

Am I missing something?

-------------------------

1440000bytes | 2024-12-18 20:42:14 UTC | #2

[quote="Chris_Stewart_5, post:1, topic:1328"]
Am I missing something?
[/quote]

I think every change in the consensus code is risky and needs the same level of review.

Example: https://github.com/bitcoin/bitcoin/pull/9049

-------------------------

Chris_Stewart_5 | 2024-12-18 20:54:02 UTC | #3

I agree

>The only deployment risk I can think of related to security soft forks is implementation risk (i.e. a bug in the implementation).

What do you think about the deployment risk?

-------------------------

1440000bytes | 2024-12-18 20:57:54 UTC | #4

Soft forks with bug fixes surely have less deployment risk.

-------------------------

harding | 2024-12-20 18:39:49 UTC | #5

I would've categorized BIP66 as a security soft fork (see the [disclosures](https://github.com/bitcoin/bips/blob/2caa8e27b80c76ef581780f4da1039f106dde032/bip-0066.mediawiki#user-content-Disclosures) section at the end of the BIP or [this](https://bitcoinops.org/en/topics/soft-fork-activation/#2015-ism-and-validationless-mining-the-bip66-strict-der-activation)).

In that case, I think the BIP66 activation problems related to spy mining provide a clear historical example of deployment risk in security soft forks.

Additionally, I think it's important to remember that all soft forks work by making the rules more strict, so they all have a risk of confiscation.  That confiscation risk can be subtle and can apply even to forks that seem to only affect miners.  For example, there's the widely held belief (that I share) that Bitmain resisted segwit because they were privately using covert asicboost to increase their hardware's effectiveness; blocks containing segwit transactions would not allow them to use that capability and (arguably) "confiscated" part of their capabilities.

More definitively (and with less emotional baggage), an early proposal for BIP141 required the `OP_RETURN` wtxid commitment be placed in the final output of the coinbase transaction; this would allow the creation of small constant-sized proofs using SHA256 midstate compression.  However, it was [realized](https://gnusha.org/pi/bitcoindev/CAAS2fgTcU-Svd5S3F-xA9+pjYSihdh7jtS6LU4k5enR-8OPESQ@mail.gmail.com/) that about 1% of hashrate at the time was mining on an ASIC that required the final output be a P2PKH payment to a fixed address.  If BIP141 had been deployed as previously envisioned, that mining hardware would have been incapable of producing blocks containing segwit transactions, which would've reduced their profits compared to other miners.

-------------------------

Chris_Stewart_5 | 2024-12-23 15:09:58 UTC | #6

[quote="harding, post:5, topic:1328"]
Additionally, I think it’s important to remember that all soft forks work by making the rules more strict, so they all have a risk of confiscation. That confiscation risk can be subtle and can apply even to forks that seem to only affect miners. For example, there’s the widely held belief (that I share) that Bitmain resisted segwit because they were privately using covert asicboost to increase their hardware’s effectiveness; blocks containing segwit transactions would not allow them to use that capability and (arguably) “confiscated” part of their capabilities.
[/quote]

After reading this passage, it seems like there are 2 phases to softfork development

- Design 
- Deployment

I would put analyzing confiscation risk under the design phase. I agree there can be subtleties to (potentially) confiscatory soft forks. However I believe that risk is isolated in the design phase rather than the deployment phase. 

**If consensus is reached during the design phase** -- for instance if there are certain Scripts could many nodes on the network -- that risk is already baked into the cake in the deployment phase. We are disallowing something that is harmful to network operation. We should use this fact to our advantage when deploying a security soft fork.

-------------------------

