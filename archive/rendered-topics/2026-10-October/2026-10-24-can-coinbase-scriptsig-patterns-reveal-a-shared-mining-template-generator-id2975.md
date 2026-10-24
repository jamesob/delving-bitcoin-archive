# Can coinbase scriptSig patterns reveal a shared mining template generator?

satoshi | 2026-10-24 14:51:33 UTC | #1

I am investigating five Bitcoin blocks whose coinbase transactions pay to the same address. I noticed recurring byte patterns in the coinbase scriptSig and would appreciate a technical assessment of whether they could indicate a shared mining template generator or mining infrastructure.

**Payout address:**

`bc1qevfvr7n7hqu56eu5cqunk22nuwds0dkjgqqxe6`

**Coinbase transaction IDs:**

* `9cdc98072e6f7534a279988bd23b1771633eabe783b5ab53ff0c6508268b1927`

* `5594e383f4038618694fa09959f22cc9b1d51d68011f28e1b6f3cb3d5f03fb96`

* `ad32ea99f1c0c6fa083af58425d54051a7989184fec1c8e49e641d670c98845a`

* `25b985fbc8bafdf615481445c910ecce0809247c41663d94a408f4c5510688b6`

* `2e9f8d0cef299d57cacf0c6d78a8ce690c011d27241c61856b2a7112efa2ce5f`

**Observed pattern**

After the BIP34-encoded block height, three of the transactions share the byte prefix `bd c4 9a 9d`. In those same three transactions, the bytes `d1 97` also occur at the same offset.

The other two transactions do not share this exact pattern.

All five transactions pay to the same output address, but I understand that this alone does not prove common ownership or establish whether the blocks were mined solo or through a pool.

**Questions**

1. Could these recurring bytes be associated with a known mining implementation, coinbase construction method, or block-template generator?

2. How likely is this pattern to be coincidental, given the number of transactions observed?

3. Which additional fields or on-chain characteristics would be most useful to compare? For example, block version, timestamp, nonce, transaction selection, witness commitment, or coinbase structure.

4. Is there a reliable way to distinguish solo mining from pool mining using public block data alone in a case like this?

I would appreciate any reproducible analysis, references to relevant source code, or suggestions for further investigation. I am particularly interested in evidence that could be independently verified.

If this topic would be more appropriate in another category, please let me know and I'm sorry. I would appreciate any guidance on where to direct this question.

Thank you for your time.

-------------------------

