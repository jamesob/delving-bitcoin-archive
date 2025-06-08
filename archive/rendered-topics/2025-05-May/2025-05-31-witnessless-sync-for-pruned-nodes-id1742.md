# Witnessless Sync for Pruned Nodes

JoseSK999 | 2025-05-31 16:19:33 UTC | #1

Hi everyone. I recently wrote an analysis about the security of skipping `assumed-valid` witness downloads for pruned nodes. This possibility was first mentioned in [Segregated Witness Benefits](https://bitcoincore.org/en/2016/01/26/segwit-benefits/#efficiency-gains-when-not-verifying-signatures), but at the time `assume-valid` didn't exist. Then, two years ago, after [this BSE question](https://bitcoin.stackexchange.com/questions/117057/why-is-witness-data-downloaded-during-ibd-in-prune-mode), [a PR](https://github.com/bitcoin/bitcoin/pull/27050) was opened in Bitcoin Core to gather feedback on this, but there were some concerns about this being a security reduction.

This witnessless sync for pruned nodes would reduce bandwidth usage by more than 40%, which translates to hundreds of GBs saved during IBD. These network savings compound nicely with `assume-valid`, as the bottleneck is less likely to be the CPU, and now also the bandwidth. As shown in the PR, implementing this was relatively straightforward.

Below I will summarize what I found, but you can read the full writeup here: https://gist.github.com/JoseSK999/df0a2a014c7d9b626df1e2b19ccc7fb1

The main concern about Witnessless Sync was that we don't check the witness data availability before syncing (as we skip downloading it, for `assume-valid` blocks), but I argue it is already implicitly checked by `assume-valid`:

1. If you use `assume-valid` you trust that the scripts are valid.
2. In order for the scripts to be valid, the witnesses must have been available. Missing witness data means script evaluation fails, which we assume not to be the case because of 1.
3. Hence, you **do** know the witnesses were available at some point, because it is a premise of `assume-valid`.

Then, using this fact, you can see how Witnessless Sync follows the same data-availability model as a regular pruned node. Pruned nodes only need a one-time data availability check, performed during IBD. After that, they aren't required to download the same blocks after x months/years to verify the data is still available.

Since our Witnessless Sync node already has this one-time past availability check covered by `assume-valid`, downloading witnesses is actually checking availability twice. AFAIK this is not required for pruned nodes, even if the data availability check (IBD) was performed many years ago.

This is why I believe this change is as safe as a long-running pruned node (you know data was available at some point in the past, and that's enough). I'd love to hear any thoughts or criticisms you might have!

-------------------------

gmaxwell | 2025-05-31 20:19:40 UTC | #2

More interesting if other optimizations move bandwidth to being decisively the limit for more nodes-- there are recent proposals that seem to.

Also more interesting if the number of limited-only nodes wasn't so tiny, but perhaps that's a bit chicken and egg with reducing the bandwidth. 

FWIW just using a more efficient serialization for transactions can reduce bandwidth (and storage, if you use it there too) on the order of 1/3rd.

-------------------------

