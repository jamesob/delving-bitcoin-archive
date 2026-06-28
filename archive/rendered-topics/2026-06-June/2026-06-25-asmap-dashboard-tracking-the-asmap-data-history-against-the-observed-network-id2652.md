# ASmap dashboard: tracking the asmap-data history against the observed network

jorisstrakeljahn | 2026-06-25 15:07:19 UTC | #1

Hi all,

Sharing an early version of an ASmap dashboard I've built as a Summer of Bitcoin project, mentored by fjahr and jurraca.

Live: https://jorisstrakeljahn.github.io/asmap-dashboard/

It walks the full bitcoin-core/asmap-data history: how much address space drifts to a different operator each release, how the entry count and operator coverage grow over time and a diff explorer for exactly what changed between any two builds.

The part I'd most like eyes on is the Network tab. Instead of looking at the .dat files in isolation, it scores each build against actually observed nodes, so you can see how well a given build would have grouped the peers a node really meets  and how that decays as a build ages.

I want to keep iterating, so feedback is very welcome:

* Do the metrics make sense and feel useful?
* Anything confusing, or a view/metric you'd want that's missing?
* Any bugs?

Credit for the node data goes to the KIT DSN group, b10c and the BitMEX Research team.

Thanks!

-------------------------

bruno | 2026-06-26 12:54:29 UTC | #2

Nice dashboard, thanks for working on it! 

I think a good metric to have on https://jorisstrakeljahn.github.io/asmap-dashboard/#diff (diff) is to track which ASNs was drastically moved. I mean, many moves are just big companies (Google, Amazon, etc) rearranging "internally". Is there a way to track "weird" moves?

-------------------------

jorisstrakeljahn | 2026-06-26 20:25:36 UTC | #3

Thanks and great point! 

I dug into it and think it's very doable, purely from the build history.

The Top Movers table already bundles individual prefix moves into edges: when lots of IP ranges shift between the same two networks, that's really one move from network A to B.

```
edge {AS14618, AS16509}:  Amazon ↔ Amazon
edge {AS3999,  AS19905}:  Penn State University ↔ Vercara
```

The trick for "weird vs. routine" I think is recurrence. For each edge, count in how many of the historical release steps it shows up. So for the current 15 ASmaps, something like this:

```
Amazon ↔ Amazon        [XXXXXXXXXXXXXX]  14×  → routine, internal reshuffle
Penn State ↔ Vercara   [.............X]   1×  → first time ever, interesting
```

In the "Top Movers table" I'd imagine this as a small recurrence indicator plus a "new moves only" filter. The table already shows the size of each move, so combined with recurrence you can surface the moves that are both large and new.

Before I build it, is that the kind of "weird" you had in mind (rare in the history), or were you thinking more about the type of jump, e.g. a move that crosses orgs?

That second one is trickier, since ASmap itself doesn't know which ASNs belong to the same organisation. I'd need to pull in external ownership data for that, so it'd be more of a best-effort heuristic than something exact.

-------------------------

fjahr | 2026-06-28 10:45:37 UTC | #4

Identifying a move as relevant to highlight ("weird") should also be looked at through the Bitcoin Network lens. If there is a move but no nodes are/were ever hosted in it, it's not that interesting probably. But if the change contains IPs which host nodes, especially a lot of them, then it should be more interesting to check out.

-------------------------

