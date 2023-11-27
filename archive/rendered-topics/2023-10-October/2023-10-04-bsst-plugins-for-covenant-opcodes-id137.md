# B'SST plugins for covenant opcodes

dgpv | 2023-10-04 12:01:36 UTC | #1

Hello forum,

I've recently had some free time and decided to implement a plugin for some covenant opcode - mainly as a way to look at how easy new complex opcodes can be added to B'SST (https://github.com/dgpv/bsst) and what limitations of the (totaly unstable and undefined at the moment) API it would uncover.

I don't have any preference for this or that opcode, but I have chosen `CHECKCONTRACTVERIFY` because it has plenty of rules it imposes on its arguments, and https://github.com/halseth/mattlab/ repo had some scripts that use the opcode, that I can use to test the plugin.

Writing the plugin helped to uncover some wrinkles with B'SST, which can be ironed out in the future, so I think writing plugins for other opcodes might help uncover more wrinkles.

You can find B'SST plugin for `CHECKCONTRACTVERIFY` at https://gist.github.com/dgpv/8e8e8f5fe12a9fae3133da9bd172c925

To run `bsst-cli` with this plugin: `/bsst-cli --op-plugins=examples/checkcontractverify_op_plugin.py` (assuming the file is placed in the `examples` dir). For now, to also have access to `CAT` you need to enable Elements mode with `--is-elements=true`. In the future, I may add a setting to selectively enable opcodes.

If you think you could use B'SST to experiment with your preferred opcode and think that it would be convenient to have a plugin for it, please write below the name of the opcode and a link to reference implementation. When I have another chunk of free time, I might implement more plugins.

I need a link to the reference implementation because figuring out the constraints an opcode puts on values is much easier when looking at reference code, but it is sometimes not easy to locate a reference implementation for someone who do not track the development very closely, and I can say that I don't track that closely.

-------------------------

ajtowns | 2023-10-13 11:00:24 UTC | #2

This is neat. I did run into `<n> OP_ROLL` giving an error when `<n>` wasn't static, which I guess is understanable, but also disappointing. :)

-------------------------

dgpv | 2023-10-13 13:15:11 UTC | #3

In principle, it is possible to analyze non-static `<n>` for `ROLL` or `PICK` to some degree.

I'm not sure how practical it is, though, and if it is worth it putting the effort to allow this.

Each value of `<n>` for `ROLL` will require its own execution path. `bsst` could put some upper limit on `<n>`, generate that many execution paths, analyze them one by one, and then show them in the report as 'branches' with conditions like `<n> = 1`, `<n> = 2`, etc.

But if `<n>` could happen to be above this limit, the analysis will be incomplete. The report can show a warning, something like "argument for PICK can be above the limit, analysis is incomplete". This warning will be shown in each execution path generated for each value of `<n>`.

If there are more than one such place in the script with non-static arg for `PICK` or `ROLL`, you will get a *lot* of execution paths in the report :-). I guess there needs to be an upper limit for the number of exec paths, too.

Can you give some examples of the *practical* scripts with non-static arg for `PICK` or `ROLL` ?

-------------------------

ajtowns | 2023-10-14 04:13:53 UTC | #4

[quote="dgpv, post:3, topic:137"]
Can you give some examples of the *practical* scripts with non-static arg for `PICK` or `ROLL` ?
[/quote]

I was looking at [`OP_VAULT`](https://github.com/bitcoin/bips/pull/1421) which pops `n+5` elements off the stack, one of which is `n`. (In particular: I wanted to DUP one of the items deepest in the stack, and do so automatically by DUP'ing `n` first)

Without an opcode like that, `n` is effectively static, since you need to consume all but one element from the stack, and each opcode consumes a constant number of elements. Perhaps `CHECKMULTISIG` could do it, but that's not available in tapscript, of course.

Being able to specify `$n in [1,2,3,4,5]` as a constraint in the file (ie pretending there was `DUP 3 NUMEQUALVERIFY` just prior to the `ROLL` or `PICK` for each value in the range) and just rerunning the analysis multiple times would work fine (and put the burden of the combinatorial explosion onto the user). I ended up just reframing my question to avoid the problem though, and that worked fine too.

-------------------------

dgpv | 2023-10-14 05:56:55 UTC | #5

Thanks for the example. I created an issue on `bsst` github to describe this proposed improvement of analysis: https://github.com/dgpv/bsst/issues/14

-------------------------

dgpv | 2023-11-27 14:32:30 UTC | #6

In the development version of bsst (and upcoming version 0.1.2) plugin system was reworked, and the way to run the referenced plugin slightly changed. Now it has to be run with `./bsst-cli --plugins=plugins/op_checkcontractverify_bsst_plugin.py`. The `CAT` opcode can now be enabled also via `--explicitly-enabled-opcodes=CAT`

The github gist with the plugin was also updated to work with new version.

(I would prefer to edit the first post to update the info, but it seems that I cannot)

-------------------------

