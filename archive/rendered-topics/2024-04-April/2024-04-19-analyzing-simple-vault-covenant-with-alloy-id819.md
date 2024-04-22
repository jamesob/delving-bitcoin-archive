# Analyzing simple vault covenant with Alloy

dgpv | 2024-04-21 22:29:25 UTC | #1

After I had a closer look at the [Basic vault prototype using OP_CAT](https://delvingbitcoin.org/t/basic-vault-prototype-using-op-cat/576/16) and discovered some problems with the implementation, I was wondering if modelling this covenant using some model checker would have helped to uncover these problems.

I have also wanted to try to model some simple contract using [Alloy](https://alloytools.org/) and see how its visualization capabilities can help with analyzing transaction inter-relations, and if Alloy syntax is suitable to model covenant-based contracts.

The resulting specification is here: https://gist.github.com/dgpv/514134c9727653b64d675d7513f983dd

The spec is commented with good detail, so please look at the comments there if interested.

`simple_vault.als` is the spec, and `simple_vault.thm` is the "theme" for the Alloy visualizer to make the instances of the model easy to examine.

If you find a problem in the spec, or have questions, please do not hesitate to comment here. I tried to make a model that is sufficient for analysis of this particular contract, but I could have missed something, of course.

The specified model demonstrates the need for the covenant script to explicitly check input index, and also for the software handling the covenant to never lock the funds at the output with index other than 0.

The model also demonstrates that there is no need to enforce the number of inputs and outputs of the current transaction, only number of outputs in the previous transaction in the 'complete_withdrawal' case.

When the covenant does not enforce the number of inputs and outputs of the current transaction, the 'cancel' case does not need its own covenant. The 'trigger' and 'cancel' cases can be handled by the same covenant. If the transaction that spends the covenant via 'trigger_or_cancel' case does not have exactly two outputs, it is effectively a 'cancel' transaction: withdrawal cannot be done with it, because the 'complete_withdrawal' case expects previous transaction to have exactly two outputs.

What became apparent is the importance for the model to reflect the underlying mechanics of the covenant. For the covenant implemented using Elements introspection opcodes, and for the covenant implemented using just `OP_CAT`, the required checks are different for the 'trigger withdrawal' covenant case - for the 'rich introspection' covenant, the 'trigger withdrawal' does not strictly need to have the current input index to be fixed at certain value. For `OP_CAT` covenant, it must be fixed. 

This is because with the `OP_CAT` covenant, the current input index and the inputs/outputs buffers that are hashed to build the signature hash are not synchronized. In Elements, these can be synchronized with `OP_PUSHINPUTINDEX` and explicit introspection opcodes that allow access to arbitrary inputs or outputs. Allowing arbitrary input index in the `OP_CAT` covenant would be too cumbersome, if at all possible, so the best way to synchronize is to just force all these indexes to be 0.

This aspect of making model reflect the particulars of the underlying covenant-enforcing mechanism is something that have to be kept in mind when modelling such contracts. Failure to reflect these details in the model can result in missing the important checks, that can lead to vulnerabilities in the contract.

Because the temporal structure of the contract is very simple, I decided to ignore the timelock aspect of the contract at all, which allowed to work only with the structure of the transactions. Although Alloy 6 has ability to model temporal progress, limiting the model to only structural analysis makes it simpler and makes checking faster.

In my opinion, Alloy syntax is more intuitive and makes the spec easier to comprehend on a basic level to wider audience, in comparison with syntax of TLA+. But to actually check the correctness of the spec, you still need a thorough understanding of the mechanisms employed by Alloy behind this syntax, so in that aspect it is not that different from TLA+.  

The visualization features of Alloy are neat, and with the right settings for the visualizer theme, the transaction structure becomes visible and demonstrative.

As an example, here is a counterexample for the `contract_holds` predicate that happens if you remove the `i.index = 0` condition in `trigger_or_cancel_cov` predicate and run the `vault` check in Alloy: 

![counterexample_1|465x348](upload://avaBftAp8ZA1O0FqJFazqYeIHl8.png)

If you remove `no_2output_envault` predicate from the `vault` check, you will get this counterexample:

![counterexample_2|690x314](upload://zA0HdKOL2gnwMyA00zN4mtY6quT.png)

I would recommend to use SAT4J solver in Alloy to explore the various instances of the model, and install [Plingeling](https://github.com/arminbiere/lingeling) (and choose it in the options) to actually check the model, because Plingeling will be much faster, but SAT4J allows to generate new instances with one click, to explore possible transaction structures.

-------------------------

