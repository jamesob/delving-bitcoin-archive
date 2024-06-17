# Differential Fuzzing of Bitcoin implementations

bruno | 2023-11-18 15:06:59 UTC | #1

Hi, all! 

I'm working on differential fuzzing of Bitcoin implementations/libraries (https://github.com/brunoerg/bitcoinfuzz). I'm starting it with miniscript, so at this moment it's fuzzing Core's and rust-miniscript. 

I'm openning this topic to raise ideas and get feedbacks about it.

-------------------------

sipa | 2023-11-29 23:01:43 UTC | #2

Nice, and it seems you've discovered something too (https://github.com/sipa/miniscript/issues/140)!

-------------------------

bruno | 2023-12-01 19:30:21 UTC | #3

Yes, I got more crashes but I'm analyzing them before publishing them.

-------------------------

bruno | 2023-12-18 18:03:14 UTC | #4

Another crash has been discussed in rust-miniscript repo: https://github.com/rust-bitcoin/rust-miniscript/issues/633#issuecomment-1856401701

cc: @sipa

-------------------------

bruno | 2024-04-15 09:37:33 UTC | #5

One more bug found by it!

https://github.com/rust-bitcoin/rust-bitcoin/issues/2681

-------------------------

bruno | 2024-06-17 16:15:50 UTC | #6

Updates! 

We recently added support for `btcd` and we discovered and could reproduce more bugs.

-------------

**rust-miniscript**: Some miniscripts were unexpectedly being rejected 
https://github.com/rust-bitcoin/rust-miniscript/issues/633

**rust-bitcoin**: During witness verification, `rust-bitcoin` doesn't check if a transaction has the witness flag but empty witnesses. 
https://github.com/rust-bitcoin/rust-bitcoin/issues/2681

**btcd**: Not a bug but an API mismatch with Bitcoin Core. When decoding a transaction, Bitcoin Core fails if the input was not entirely consumed, btcd doesn't.
https://github.com/btcsuite/btcd/issues/2195

**Bitcoin Core/rust-miniscript**: When validating miniscripts from string, there is an inconsistence between rust-miniscript and Bitcoin Core. We noticed that Bitcoin Core accepts the usage of the `+` sign, e.g. (`l:older(+1)` , `u:after(+1)` ), because of `ParseInt64` function. However, `rust-miniscript` checks the character itself, so it only accepts "1", "2", "3"..."9".
https://github.com/brunoerg/bitcoinfuzz/issues/34

**rust-miniscript**: The parser function is still recursive while Core isn't anymore. It makes some miniscripts valid for Core but rejected by rust-miniscript due to the max recursion depth. (We didn't find this, we just could reproduce by running the harness with the seed corpus from Core) 
https://github.com/rust-bitcoin/rust-miniscript/issues/696

-------------------------

