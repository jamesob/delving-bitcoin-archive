# B'SST plugins for covenant opcodes

dgpv | 2023-10-04 09:23:01 UTC | #1

Hello forum,

I've recently had some free time and decided to implement a plugin for some covenant opcode - mainly as a way to look at how easy new complex opcodes can be added to B'SST (https://github.com/dgpv/bsst) and what limitations of the (totaly unstable and undefined at the moment) API it would uncover.

I don't have any preference for this or that opcode, but I have chosen `CHECKCONTRACTVERIFY` because it has plenty of rules it imposes on its arguments, and https://github.com/halseth/mattlab/ repo had some scripts that use the opcode, that I can use to test the plugin.

Writing the plugin helped to uncover some wrinkles with B'SST, which can be ironed out in the future, so I think writing plugins for other opcodes might help uncover more wrinkles.

You can find B'SST plugin for `CHECKCONTRACTVERIFY` at https://gist.github.com/dgpv/8e8e8f5fe12a9fae3133da9bd172c925 -- note that it is not fully tested, I will wait for mattlab repo to update their opcode implementation to match reference implementation, to test on their scripts.

To run `bsst-cli` with this plugin: `/bsst-cli --op-plugins=examples/checkcontractverify_op_plugin.py` (assuming the file is placed in the `examples` dir). For now, to also have access to `CAT` you need to enable Elements mode with `--is-elements=true`. In the future, I may add a setting to selectively enable opcodes.

If you think you could use B'SST to experiment with your preferred opcode and think that it would be convenient to have a plugin for it, please write below the name of the opcode and a link to reference implementation. When I have another chunk of free time, I might implement more plugins.

I need a link to the reference implementation because figuring out the constraints an opcode puts on values is much easier when looking at reference code, but it is sometimes not easy to locate a reference implementation for someone who do not track the development very closely, and I can say that I don't track that closely.

-------------------------

