# A Comprehensive OP_RETURN Limits Q&A Resource to Combat Misinformation

tidwell | 2025-05-12 17:31:37 UTC | #1

My goal is to share a consolidated list of questions about OP_RETURN limits that we've answered on Stacker News, as the original thread has become unwieldy with over 200 comments. We began compiling this list about a week ago. I've frequently shared individual links and received very positive feedback. I hope this resource helps us work from a common set of facts and reduces misinformation. I hope you find this as a valuable resource.

Special thanks to @murch for encouraging me to ask questions and for taking the time to field many of them.

I'll list the questions in order of activity and tips received. I've removed duplicates, rephrased some statements as questions, and ignored completely irrelevant questions.

1. Users should be given clear configurable options to decide what's in their mempool, why were these options taken away? [link](https://stacker.news/items/971277/r/tidwell?commentId=971285)
2. Won't spammers abuse large OP_RETURNs to bloat the blockchain and make IBD take longer? [link](https://stacker.news/items/971277/r/tidwell?commentId=971284)
3. A similar PR was proposed by Peter Todd 2 years ago, why was it rejected then? What has changed since then, why would this get approved now? [link](https://stacker.news/items/971277/r/tidwell?commentId=971297)
4. Shouldn't we be fighting spam, why are we making policies less strict, shouldn't we be making them more strict? [link](https://stacker.news/items/971277/r/tidwell?commentId=971287)
5. How would someone get around the standardness policy currently for OP_RETURN size? [link](https://stacker.news/items/971277/r/tidwell?commentId=971281)
6. What does "standardness mean" in reference to OP_RETURNs? [link](https://stacker.news/items/971277/r/tidwell?commentId=971280)
7. Will more than 1 OP_RETURN per transaction be possible if this PR gets merged? [link](https://stacker.news/items/971277/r/tidwell?commentId=971280)
8. What are the current OP_RETURN limits and what restrictions are being lifted? [link](https://stacker.news/items/971277/r/tidwell?commentId=971278)
9. Are current relay and mempool policies effective for filtering out spam transactions? [link](https://stacker.news/items/971277/r/tidwell?commentId=971288)
10. Is it true that this type of update could affect Bitcoin's decentralization? [link](https://stacker.news/items/971277/r/tidwell?commentId=971934)
11. Is it possible to stop the abuse of payment outputs (i.e., bare multisig, fake pubkeys, and fake pubkey hashes) that are used to embed data, thereby creating unprunable UTXOs that bloat the UTXO set? [link](https://stacker.news/items/971277/r/tidwell?commentId=971325)
12. What was the main reason /concern to add this PR? ... What will happen if we do nothing? [link](https://stacker.news/items/971277/r/tidwell?commentId=971824)
13. If OP_RETURN still cannot stop all the garbage, why is so important to remove it? Does it affect future development / improvements for LN? [link](https://stacker.news/items/971277/r/tidwell?commentId=971309)
14. What will be the worst case scenario if users still could set their own limits for OP_RETURN? [link](https://stacker.news/items/971277/r/tidwell?commentId=971311)
15. Shouldn't we debate the controversy of this PR on Github since it's where the code gets merged to make these changes? [link](https://stacker.news/items/971277/r/tidwell?commentId=971330)
16. What does it mean when someone says "Fix the Filters"? [link](https://stacker.news/items/971277/r/tidwell?commentId=971289)
17. Will this open the flood gates and drown out all legitimate onchain activity? [link](https://stacker.news/items/971277/r/tidwell?commentId=971391)
18. What can we do to stop spam at the consensus layer of Bitcoin? [link](https://stacker.news/items/971277/r/tidwell?commentId=971326)
19. Will Taproot wizards and other spam companies and projects start using OP_RETURN to put jpegs on the blockchain? [link](https://stacker.news/items/971277/r/tidwell?commentId=971293)
20. If we prevent these transaction from going into our mempools doesn't that prevent or delay these spam transactions from being mined therefore discouraging the spammers? [link](https://stacker.news/items/971277/r/tidwell?commentId=971291)
21. Is it possible to stop abuse of witness data? If so, how? (i.e ordinal theory inscriptions, "jpegs"). [link](https://stacker.news/items/971277/r/tidwell?commentId=971321)
22. Is there any conflict of interest with Bitcoin Core and companies like Citrea, in ref to this PR? [link](https://stacker.news/items/971277/r/tidwell?commentId=971302)
23. Is there any estimation on how much would this affect fees for the average user, considering external projects (like Citrea) using it? Any possibility that this could saturate the mempool and boost fees beyond reasonable? [link](https://stacker.news/items/971277/r/tidwell?commentId=971403)
24. Was this PR initially proposed because of Citrea BitVM needs? If so don't they only need a slight bump in OP_RETURN size, why is it being proposed to make the size unrestricted? [link](https://stacker.news/items/971277/r/tidwell?commentId=971300)
25. What makes a UTXO unprunable? Which projects are making unprunable UTXOs? [link](https://stacker.news/items/971277/r/tidwell?commentId=971296)
26. Why would a spammer use OP_RETURN if it's cheaper to use Witness data to store arbitrary data? [link](https://stacker.news/items/971277/r/tidwell?commentId=971294)
27. Won't large OP_RETURNs allow people to spam the mempool with 100kb transactions and mess up bitcoin for everyone by bloating the mempool and not allowing legitimate transactions in the mempool? [link](https://stacker.news/items/971277/r/tidwell?commentId=971292)
28. If relaxing op_return standardness limit seeks to make 'spam' prunable, then what are proponents of this change assuming about the long-term feasibility of running a 'full' (unpruned) bitcoin node? [link](https://stacker.news/items/971277/r/tidwell?commentId=971592)
29. Is allowing standardness for larger OP_RETURNs a slippery slope? If we allow this won't we continue to allow things that make bitcoin less for money and more for arbitrary data? [link](https://stacker.news/items/971277/r/tidwell?commentId=971299)
30. Won't removing the OP_RETURN cap reduce fee market pressure by allowing senders to consolidate arbitrary data into a single transaction? [link](https://stacker.news/items/971277/r/tidwell?commentId=972132)
31. Could this PR be the beginning of reducing other mempool restrictions? [link](https://stacker.news/items/971277/r/tidwell?commentId=971912)
32. Culture is what protects Bitcoin from external forces, shouldn't non-technical arguments be valid when considering these types of changes? [link](https://stacker.news/items/971277/r/tidwell?commentId=971331)
33. What's the difference between UTXO set, mempool, and blockchain, and how do larger OP_RETURN or witness data affect node resource usage? [link](https://stacker.news/items/971277/r/tidwell?commentId=971314)
34. What is the difference in defining a transaction as valid versus defining a transaction as standard and why do we need this difference? [link](https://stacker.news/items/971277/r/tidwell?commentId=971308)
35. If you're happy with your viewpoint on consensus and mempool rules, is not upgrading Bitcoin Core until it makes sense to you a valid action to take right now? [link](https://stacker.news/items/971277/r/tidwell?commentId=972071)
36. Why didn't this PR get a BIP number? [link](https://stacker.news/items/971277/r/tidwell?commentId=973745)
37. Why is core rushing this change? [link](https://stacker.news/items/971277/r/tidwell?commentId=973744)
38. If there will be a hard fork resulted from this PR (split chain like in 2017), what will happen with existing LN channels? Will exist on both chains with 2 LNs? [link](https://stacker.news/items/971277/r/tidwell?commentId=971840)
39. Isn't this all moot in a (almost guaranteed) future where fees are very high? [link](https://stacker.news/items/971277/r/tidwell?commentId=977652)
40. What is this controversy about, and what is it *really* about? [link](https://stacker.news/items/971277/r/tidwell?commentId=974539)

-------------------------

