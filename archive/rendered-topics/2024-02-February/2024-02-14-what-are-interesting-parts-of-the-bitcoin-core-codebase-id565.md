# What are interesting parts of the Bitcoin Core codebase?

rodarmor | 2024-02-14 02:36:28 UTC | #1

I have a super smart friend who I'm trying to nerd-snipe into getting interested in working on Bitcoin Core. My super sneaky plan is to point him to some particularly juicy parts of the codebase to try to pique his interests.

Does anyone have any favorite parts of the Bitcoin Core codebase? They could be particularly elegant, do something particularly impressive, or be particularly interesting theoretically. I'll definitely also point him to libsecp256k1!

-------------------------

1440000bytes | 2024-02-14 03:54:27 UTC | #2

[quote="rodarmor, post:1, topic:565"]
Does anyone have any favorite parts of the Bitcoin Core codebase?
[/quote]

P2P is the most interesting part of bitcoin core

-------------------------

recent798 | 2024-02-14 11:17:16 UTC | #3

Mempool design is an interesting theoretical problem. For instance, https://delvingbitcoin.org/t/mempool-incentive-compatibility/553#summary-18

Moreover, privacy and DoS resistance in general are areas with open research questions. For instance, https://bitcoincore.academy/p2p-attacks.html

-------------------------

jb55 | 2024-02-14 18:48:28 UTC | #4

I like the EvalScript code in script/interpreter.cpp, it's a simple stack based language that you can build your own interpreter for to learn how it works. Fun exercise!

-------------------------

rodarmor | 2024-02-14 19:39:38 UTC | #5

[quote="1440000bytes, post:2, topic:565"]
P2P is the most interesting part of bitcoin core
[/quote]

What's your favorite file or function?

[quote="jb55, post:4, topic:565, full:true"]
I like the EvalScript code in script/interpreter.cpp, it’s a simple stack based language that you can build your own interpreter for to learn how it works. Fun exercise!
[/quote]

Nice, great suggestion!

[quote="recent798, post:3, topic:565"]
Mempool design is an interesting theoretical problem. For instance, [Mempool Incentive Compatibility](https://delvingbitcoin.org/t/mempool-incentive-compatibility/553#summary-18)
[/quote]

That's a great discussion! I'll definitely link him there. Anything in the existing codebase, like a file or function, which is particularly interesting?

-------------------------

sanket1729 | 2024-02-14 21:48:42 UTC | #6

I find the most interesting parts are in `validation.cpp`. In particular, the re-org logic and rolling back `ConectBlock/DisconnectBlock`, `AcceptBlock`, `AcceptBlockHeader`. 

There are several interesting cases around DoS preventions in mempool related functions. Basically, what happens when there is new block in `validation.cpp`. There is also an integration test in python `feature_block.py` that tests lots of interesting scenarios hitting codepaths in validation.cpp. 

Second, interpreter.cpp and scripting engine. miniscript.cpp is also is an interesting dive in itself.

-------------------------

1440000bytes | 2024-02-15 03:25:59 UTC | #7

[quote="rodarmor, post:5, topic:565"]
What’s your favorite file or function?
[/quote]


P2P messages sent by bitcoin nodes and service flags: https://github.com/bitcoin/bitcoin/blob/baed5edeb611d949982c849461949c645f8998a7/src/protocol.cpp

Interesting functions:

`HasAllDesirableServiceFlags()`  
`Misbehaving()`  
`MaybePunishNodeForBlock()`  
`MaybePunishNodeForTx()`  
`ProcessCompactBlockTxns()`  
`ProcessMessage()`  
`MaybeDiscourageAndDisconnect()`  
`AttemptToEvictConnection()`  
`ReattemptInitialBroadcast()`

Process used to relay transactions for better privacy in section [Message: inventory](https://github.com/bitcoin/bitcoin/blob/baed5edeb611d949982c849461949c645f8998a7/src/net_processing.cpp#L5755) is interesting as well.

-------------------------

recent798 | 2024-02-15 08:16:19 UTC | #8

[quote="rodarmor, post:5, topic:565"]
Anything in the existing codebase
[/quote]

Yes, the function would be `BlockAssembler::addPackageTxs`, which is the transaction selection algorithm for the next block in the chain, taking the presumed best transactions from the mempool.

-------------------------

bruno | 2024-03-01 14:48:24 UTC | #9

`ProcessMessage` is cool to study. You can see how P2P messages are processed, etc. From this function, you can find many other interesting functions.

-------------------------

