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

