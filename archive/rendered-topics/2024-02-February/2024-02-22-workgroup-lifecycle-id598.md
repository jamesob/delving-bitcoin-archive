# Workgroup lifecycle

ajtowns | 2024-02-22 05:53:20 UTC | #1

I've just closed out the cluster mempool working group, and thought it might be interesting to document the lifecycle it went through:

1. Initial private group:
   * created a wg-cluster-mempool category for posting ideas within the group, so that thoughts didn't get lost when instant messages expired, and were easier to search than random gists
   * setup the private group by the same name, and made the category private to that group
   * seeded the category by [copying](https://delvingbitcoin.org/t/cluster-mempool-rbf-thoughts/156) over a gist
   * (This was around 2023-11-01)

2. Public but read-only:
   * after a while group members wanted to share some of the discussion with others who weren't members of the group. rather than expand the group we decided to open the existing topics up to the public
   * this involved changing the category to be publicly readable, but new posts and replies were still limited to wg members
   * at this point optech discovered and [reported on](https://bitcoinops.org/en/newsletters/2023/12/06/#cluster-mempool-discussion) those discussions
   * (This was around 2023-11-30)

3. Closing out the working group
   * with the project moving from "research" to "release" (eg, it's proposed as a [priority project for bitcoin core 28.0](https://github.com/bitcoin/bitcoin/issues/29439#issuecomment-1958088049)) there's little reason to limit discussions any longer, so closing out the working group seems to make sense
   * all the discussions have been recategorised into the main #implementation category, so they're no longer restricted to the wg in any way (either for reading or writing)
   * to make it easier for future historians to still find those discussions, there's now a #wg-cluster-mempool::tag tag that's retroactively applied to all them, and that is locked to admins, so it can't be added to future discussions
   * the old category still exists, but only has an "About" post, which has a link to the tag so you can still find the old posts
   * (And here we are at 2024-02-22)

Anyway, just documenting that as it seems like it's a process that may be useful to repeat for other projects.

-------------------------

