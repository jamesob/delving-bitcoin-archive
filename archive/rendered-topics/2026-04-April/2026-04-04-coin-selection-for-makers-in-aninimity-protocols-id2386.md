# Coin selection for makers in aninimity protocols

cmp_ancp | 2026-04-05 18:55:32 UTC | #1

# Introduction

I always liked the idea of market makers for anonimity protocols like coinjoins, and recently a protocol that I am very interested about, coinswap. Without the market makers, these protocols have a huge UX flaw, that is to depend on the interest of other users at the same time (something that could take hours sometimes).

However, I also think that makers could easily deanonimize takers if bad coin selection algorithms were to be used. That originates from the old toxic change problem, and makers, by making more frequent transactions, could be the greater deanonimization factor in the system.

## The toxic change problem

For those not used to the chainanalisys heuristics, the change is the main pillar behind deanonimization in bitcoin. When you have a UTXO greater than the amount that you need to send (that is, almost always, as it is very unlikely that you have a UTXO with the exact round amount that you want to send), you need to create a change UTXO sending funds back to yourself. The change tracking is the basis on clustering transactions and histories, associating different activities to the same wallet.

Input →TX → change → TX → change

When different users want to work together in order to mix their chain footprint, we always step in the problem of participants having different amounts, and these numbers are capable of giving information on who owns which UTXO post interaction.

In the case of coinjoin, all outputs need to have the same size in order to not be related to inputs. The point I am bringing to discussion is the maker coin selection, because the maker will hardly have UTXOs the same size as the takers, and as they consolidate the output liquidity in order to make more interactions on market, it is possible that previous takers could be severely affected.

Even though coin selection is a main issue in makers job, I never heard anyone talking about strategies or algorithms for makers not to scramble all privacy takers ought to obtain.

## Coinswap

Coinswap is a protocol I got very interested in since last year, it proposes a decentralized system of same chain swaps for bitcoin, not working on mainchain yet.

Just like joinmarket, the idea is to have market makers with liquidity for takers. Makers would participate in circuits organized by takers. E.g.:

Taker → A → B → C → Taker

Where A, B and C are makers that earn fees on swap hop.

My main concern is that, if the maker simply consolidates received UTXOs with the change from the operation, we could deanonimize the entire swap. The taker is the only one that gives up an entire UTXO when swapping, A, B and C have UTXOs with different amounts, and therefore create change.

If it is known that a cluster belongs to a maker (by cluster I mean, a set of UTXOs and transaction history, that could be associated together as own by the same user), and the new UTXO enters the same cluster from the fund sent in that swap circuit, we could associate the inflow and outflow as being part of the same operation, and if all makers commit this same sin, we could trace the entire circuit.

# My (maybe naive) solution

I think a simple algorithm for preserving anonimity is to makers have multiple unlinked clusters. When in a circuit, the maker receives funds in one cluster and sends from another cluster, making sure that clusters are never consolidated. If transaction hops interact with different clusters, these cannot be easily associated with the same circuit, and a chain analist cannot trace the circuit begin to end.

But what if a maker wants to consolidate the funds anyway? Maybe to use them for private needs. Well, then the maker turns into a taker theirself, and swap-in a cluster into another.

This cluster separation obviously brings some burden to the maker, as he needs to be concerned to liquidity management in order to participate in circuits. Liquidity would flow between clusters, and he is limited by wich circuit amount he could participate based on liquidity distribution.

I was very interested on this topic recently, and wanted to know if I’m missing something, or if it is an already tackled problem.

-------------------------

