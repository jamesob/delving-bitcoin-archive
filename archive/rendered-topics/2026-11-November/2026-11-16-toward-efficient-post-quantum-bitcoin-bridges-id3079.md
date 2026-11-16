# Toward Efficient Post-Quantum Bitcoin Bridges

eigmax | 2026-11-16 13:19:01 UTC | #1

We have written up a garbled-circuit verifier for hash-based STARK proofs, as a post-quantum alternative to garbling a Groth16 verifier in BitVM3-style bridges, and measured what it costs on-chain and off-chain. The design is BitVM2 with the on-chain verifier replaced by a garbled one: the operator garbles the verifier once, proves the garbling correct and commits the input keys on Bitcoin.


**Paper:** Toward Efficient Post-Quantum Bitcoin Bridges, https://eprint.iacr.org/2026/2877



**Code:** https://github.com/GOATNetwork/bitcoin-stark-verifier



### Summary



- We compile the complete verifier for [Ziren]( https://eprint.iacr.org/2026/2330 )'s [WHIR](https://eprint.iacr.org/2024/1586.pdf)-based STARK proofs into a Boolean circuit and garble it with free-XOR. The proofs are over the binary tower field $\mathbb{F}_{2^{128}}$ with Blake3 commitments and transcript, so neither the verifier nor its garbling rests on pairings, discrete logarithms or lattices.

- For Ethereum mainnet block 26,206,588, recursed down to a small final proof, the final verifier reads **520,832 input bits** and has **175.4M non-free gates**, about **2.81 GB** of garbled tables. There is no cut-and-choose, so this one instance is all a challenger stores.

- Publishing that input under translated Winternitz keys takes **3.93 MvB**, in 41 standard transactions. The Schnorr adaptor baseline takes 1.11 MvB but is not post-quantum.

- Off-chain, the garbled verifier is about 15 times smaller than BitVM3's Boolean Groth16 verifier.

- On-chain, the proof input dominates: it is over a thousand times BitVM3's Assert of about 2.4 kvB. Completing the Assert takes 43 transactions over at least six blocks: one funding transaction, 41 reveals and a join that creates the completion output.



### Why a binary-field STARK



Under free-XOR only non-linear gates cost anything. Over $\mathbb{F}_{2^{128}}$ every field addition is an XOR, so the verifier pays only for field multiplications and hashes. A multiplication is 2,187 AND gates (Karatsuba over the tower, $3^k$ ANDs for a $2^k$-bit product). A Blake3 compression is 10,281 AND gates, half of bitvm-gc's circuit, because each carry is a majority with one AND per bit. Blake3 serves as the Merkle hash, the Fiat–Shamir sponge and the garbling hash, so the verifier assumes nothing beyond it.



### Getting the input down



Ziren proves the block in 49 shards and compresses them into one binary-stage WHIR proof. Four recursion levels follow, each proving the previous proof's verifier: two on Ziren's plain recursion machine, then two narrow levels on a machine with fewer trace columns. The final proof's verifier is recorded as a fixed straight-line program over $\mathbb{F}_{2^{128}}$ and compiled to a circuit with one acceptance output. Its shape does not depend on the proof, so it can be garbled at setup.



What shrinks the final verifier's input:



- aligned narrow table layouts, so one ring-switch claim covers a whole table block;

- opening only the linear forms of the columns that the constraints read, with the fixed preprocessed data specialized away;

- a closing column sumcheck that sends no opened values: 27% less input for 41% more gates;

- 200-bit Merkle digests and a Merkle cap at depth 18;

- the WHIR schedule: rate 1/1024, folds 4-5-4-4, at most 44 bits of grinding (below).



The input is an exact function of the WHIR schedule, so every setting can be costed without proving. The three costs are coupled:



- On-chain cost is linear in the input bits.

- Garbled storage is linear in the gates.

- The prover's expected grinding work, $\sum_j 2^{b_j}$ Blake3 compressions, grows exponentially with the grinding level $g$.



Each bit of $g$ saves about 5,300 input bits.



### The 4-5-4-4 schedule



The final proof commits to a polynomial in 23 variables. WHIR reduces it in stages. Each stage folds some variables away, and the verifier queries random positions of that stage's codeword. 4-5-4-4 folds 4, then 5, then 4, then 4 variables, 17 in all. That leaves a 6-variable polynomial, which the prover sends in full: it stops folding at the first stage that leaves at most six.



The fold sizes set three costs:



- **Leaf size.** A query at a stage that folds $k$ variables opens $2^k$ codeword values as one Merkle leaf. A 5-fold doubles the leaf of a 4-fold.

- **Queries.** A fold of $k$ variables lowers the next stage's rate by $k-1$ bits, because the polynomial shrinks by $2^k$ while the evaluation domain only halves. Here the log inverse rates are 10, 13, 17 and 20. At a lower rate each query buys more security, so the stages need 13, 10, 8 and 7 queries.

- **Grinding.** Each stage's folding grind rises two bits per bit of its log inverse rate and falls one bit per variable already folded: here 37, 39, 42 and 44 bits. The largest, $g=44$, is the grinding allowance: the queries then need to cover only the remaining $108-44=64$ bits of each error term.



Compare 4-4-4-5, which moves the 5 to the end. The last two stages then sit at rates 16 and 19 instead of 17 and 20, so the top grind falls to 43 bits and the expected grinding halves. In return, each of the first three stages needs one more query, adding 21,248 input bits. So 4-5-4-4 reads 3.9% fewer bits for about twice (2.1 times) the expected grinding. The schedule's input is an exact formula, so we scored all 8,252 fold sequences of two to eight variables a stage that the prover can run. At rate 1/1024, 4-5-4-4 reads the fewest bits.



### Publishing the input



The Assert is one funding transaction, then reveal transactions that spend it, then a join that spends the reveals' outputs and creates the completion output that Disprove and Take2 need. Each reveal stays under the 400,000-WU standardness limit and the 1,000-item stack limit, with witness items of at most 80 bytes. The sizes below are the reveals. Funding adds about 19 kvB. The join is not included: we serialized it only for Lamport, where it is 6.6 kvB.



| scheme | reveal transactions | size of the reveals |
|---|---:|---:|
| Schnorr adaptor, 8-bit digits (baseline, not post-quantum) | 12 | 1.11 MvB |
| Winternitz translated to Lamport (ePrint 2026/1684) | 41 | 3.93 MvB |
| Antichain Winternitz (2,16) | 66 | 5.93 MvB |
| Lamport | 87 | 8.64 MvB |



The translated-Winternitz Assert takes one block for funding, at least four full blocks of reveals (15.73M WU in all), and the join in a later block. It also needs a 953 MB off-chain translation table.



A challenged dispute needs about 46 transactions:

- Kickoff and Challenge;

- the 43 Assert transactions;

- Disprove or Take2.



An unchallenged claim needs only Kickoff and Take1.



Disprove is a 97 vB hashlock spend on the reject label. It was accepted and mined under strict Bitcoin Core 31.1 policy.



We replayed the publication in Bitcoin Core 31.1 at an earlier, larger input. A reveal near the size limit was rejected while its funding parent was unconfirmed, because the pair exceeded the cluster size limit. It was accepted once the funding confirmed. So these reveals cannot be package-relayed with their parent.



### Compared with BitVM3 and the Tiwari–Feickert estimate



The Tiwari–Feickert column is the combined example in their Section 3.5. It uses RISC Zero's verifier with Blake3 as its internal hash, an estimated 1.56 billion non-free gates, and Antichain Winternitz for the input labels.



| | BitVM3 (Groth16) | Tiwari–Feickert (ePrint 2026/2383) | this work |
|---|---|---|---|
| post-quantum verifier | no (pairings) | yes | yes |
| verifier input bits | 1,019 | ≈1.78M | 520,832 |
| input publication | ≈2.4 kvB (adaptor) | ≈14.9 MvB (Antichain Winternitz) | 3.93 MvB (translated Winternitz) |
| garbled tables per instance | 40.5 GiB | 23.3 GiB (est.) | 2.61 GiB |
| label translation table | – | 72.7 GiB | 0.89 GiB |
| garbled tables stored | 283.5 GiB (7 kept) | not reported | 2.61 GiB (1) |


- Garbled tables are 16 bytes per non-free gate in every column. In decimal units, our 2.61 GiB is the 2.81 GB above, and our 0.89 GiB is the 953 MB translation table.

- The Tiwari–Feickert publication figure leaves out fixed transaction overheads; ours include them. With overheads included, Antichain Winternitz takes 5.93 MvB at our input.

- BitVM3 keeps seven of 181 garblings after cut-and-choose. We store the one garbling that the operator proves correct. That proof is not built, and no row includes its cost.



### One garbling, proved correct



The operator garbles the verifier once and commits the table hash, the reject-label hash, the input-key hashes and the label ciphertexts that map each key secret to an input label. It then proves in zero knowledge that these come from one correct garbling. There is no cut-and-choose, so there is one garbled instance to store and one key set to reveal: the 3.93 MvB is the whole reveal cost, with no soldering across kept garblings.

### Future work



1. **A hash-based proof of correct garbling.** We will build it and measure its cost. It covers 175M non-free gates and about a million label ciphertexts (two per input bit).

2. **A publication that spans blocks.** The reveals are close to 100 kvB each and cannot be package-relayed with their funding parent. So the Assert takes at least four blocks of reveals under a timeout. We will look for a better transaction shape for a publication that long.

3. **A batch hash-verification selector.** A hash-based selector we serialized matches the adaptor's size, but only under a hypothetical opcode that verifies hashes in batch. We will work out whether that opcode is worth proposing.

4. **\Short-key label delivery.** TinyLabels (Dietz, Li and Lin, Eurocrypt 2025) gives a Ring-LWE batch-select scheme. The sender, who knows the input, publishes the input and one short key, and anyone can then decode the input's labels. At our size that is about 121 KB, or about 30 kvB in one envelope transaction, against 3.93 MvB of reveals.



Pointers on any of these are welcome.

-------------------------

