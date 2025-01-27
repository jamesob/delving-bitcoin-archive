# What happens after we "run out" of script flags?

moonsettler | 2025-01-12 19:29:48 UTC | #1

Script flags are currently represented in the bitcoin core code base as 32 bit unsigned int.

If hypothetically we were to add 11 more then it's pretty much over:
```c++
SCRIPT_VERIFY_CHECKTEMPLATEVERIFY = (1U << 21),
SCRIPT_VERIFY_DISCOURAGE_UPGRADABLE_CHECKTEMPLATEVERIFY = (1U << 22),
SCRIPT_VERIFY_DISCOURAGE_CHECKTEMPLATEVERIFY = (1U << 23),
SCRIPT_VERIFY_CHECKSIGFROMSTACK = (1U << 24),
SCRIPT_VERIFY_DISCOURAGE_CHECKSIGFROMSTACK = (1U << 25),
SCRIPT_VERIFY_INTERNALKEY = (1U << 26),
SCRIPT_VERIFY_DISCOURAGE_INTERNALKEY = (1U << 27),
SCRIPT_VERIFY_PAIRCOMMIT = (1U << 28),
SCRIPT_VERIFY_DISCOURAGE_PAIRCOMMIT = (1U << 29),
SCRIPT_VERIFY_CAT = (1U << 30),
SCRIPT_VERIFY_DISCOURAGE_CAT = (1U << 31),
```
So what do we do after? Increase flags to 64 bit?

-------------------------

AntoineP | 2025-01-12 23:08:56 UTC | #2

This is a Bitcoin Core implementation detail. The script flags mechanism itself is not consensus critical. A larger integer or another mechanism altogether could be used to instruct the script interpreter what validation should be performed.

-------------------------

moonsettler | 2025-01-12 23:32:51 UTC | #3

Yes, I know it's not consensus critical, in that sense. So it's not expected to be a breaking change for other projects? If it's fully contained to core and not a huge deal to change the type, then it's not a big deal, I guess. Right now LNHANCE only uses 3 flag bits. But since a lot of people seem to prefer C3PO, I'm looking at options for concurrent overlapping activations.

-------------------------

AntoineP | 2025-01-13 00:00:36 UTC | #4

As you already know i believe it's entirely premature to talk about activation of anything, that said i don't think anyone working on a soft fork proposal should be concerned about the "remaining number of script flags" whatsoever.

-------------------------

moonsettler | 2025-01-26 15:51:44 UTC | #5

After giving it a try, I don't think it's a super viable path to change the type. It's a lot of change that breaks easy code portability, it compiles without problems while certainly bugged. `flags_t` could be `ScriptFlags`, or whatever for better readability.

**edit:** I did experiment with using `typedef std::bitset<64> flags_t;` for script flags, it could be a good solution long term, but the imminent pain is even worse. Also doing so noticed that the flags as `unsigned int` are indeed exposed on APIs. and honestly I have no idea what to do with that.

**edit:** Context for picking up this thread again: `LNhance` + `C3PO` has enough script flags left, but I became increasingly convinced that `C4` should also be part of the conversation, because activating `CAT` without `CCV` carries the likelihood of greater annoyance with new opcodes as opposed to the immediate desire to soft-fork in further optimizations.

https://github.com/lnhance/bitcoin/compare/lnhance-26.x-deploy...lnhance:bitcoin:lnhance-26.x-deploy64

-------------------------

