# Debuggable Lisp Scripts

ajtowns | 2024-10-29 12:14:36 UTC | #1

Longer delay since my [last post on the topic](https://delvingbitcoin.org/t/btc-lisp-as-an-alternative-to-script/682) than I'd hoped. Most of it was spent thinking about this bit of feedback (particularly in the context of how annoying I was finding it to write btclisp code):

[quote="prozacchiwawa, post:2, topic:682"]
If I had one piece of advice, it’s to nail down a great system for mapping high level lisp to low level code for debuggers and interpreters early. Chia is behind in this (hoping to catch up) and it’s probably the biggest negative feedback about chialisp development.
[/quote]

I spent a bunch of time trying to figure out if using [racket-lang](https://racket-lang.org/) would be a better approach -- it's a lisp dialect that's designed for creating your own DSLs, it has a nice IDE, and it has some nice "introduction to computer science" tooling, that in theory should work great for deeply understanding small bits of code. But that didn't seem to really pan out: the debugging stuff still becomes complicated when macros are involved, customising the debugging interface seems to require getting your hands very dirty, and there aren't nice easy libraries for doing all the cryptographic operations, so implementing tx parsing and secp maths would've been a bunch of work.

The conclusion I eventually came to are that macros represent something of a false idol -- they're a really great way of *building* a language feature, because they let you say "when I write this expression, I want you to treat it as if I'd done some other more complicated thing". But the problem is that you're leaving implicit the meaning of the expression, so when you try to debug things, the computer can only do what you told it and perform the mechanical translation and continue on from there, which confuses things immensely, because you're no longer debugging the code that you wrote, but some complicated equivalent expression that you didn't write. Which sucks: if you'd been happy dealing with the complicated expression, you never would have bothered writing the macro in the first place! So, they're great for building a language, but not actually sufficient once you want to use the language.

Another aspect of the "lisp approach" that was bugging me is the conflict between having an "eval" function, and translating from a high-level to a low-level. In particular, if you have a high-level eval, how do you translate that to the low-level? If you do it natively, then your low-level language essentially is your high-level language; and if you don't, then you have to implement the entire high-level language in the low-level one in order to implement eval.

Thinking about [miniscript](https://bitcoinops.org/en/topics/miniscript/) finally brought these ideas together in a way that allowed progress: if you treat the high-level language as a friendly variation of the low-level language (as miniscript does with script), then it perhaps makes sense to have two interpreters (one that keeps everything friendly and high-level, and one for the low-level language), and just have a separate translator/compiler to go from one to the other. That's a little frustrating, in that it introduces the possibility of the high-level interpreter coming up with a different answer to what the compiler and low-level interpreter would produce; but it seems well worth the tradeoff, and that problem is potentially eventually solvable by formal analysis tools.

So jumping on that idea, I've rewritten almost everything into that new model, with a bit more of a focus on debugging, which means I've now got three things: a basic lisp language which I'm calling "bll", that's designed to be able to be slotted into consensus, a higher level language that's designed to be both easier to program in and able to be compiled into bll in a straightforward way, which I'm calling "symbll" for "symbolic bll", and a REPL for developing in those languages, called "bllsh" for "bll shell", and which is pronounced "bullish", because what's the point of a multi-month diversion if you can't come out of it with some pun-based AI art?

![bllsh - a bull of shells, before a new dawn in scripting|690x172, 100%](upload://xFqM6rYD13KsH8M3T4htPy0C4yO.jpeg)

https://github.com/ajtowns/bllsh

In essence, bll isn't very different from [btclisp](https://delvingbitcoin.org/t/btc-lisp-as-an-alternative-to-script/682) as it was in March; the biggest difference is that it's no longer lazily evaluated, so all arguments to an operation are evaluated (unless the script aborts), though there are other tweaks, such as some opcodes being implemented (`partial`), some being dropped (`/%`) and some being changed (`c` becomes `rc` and builds the list in reverse). The actual code implementing it is quite different, though, mostly due to adopting [continuation passing style](https://en.wikipedia.org/wiki/Continuation-passing_style).

Here's what a simple factorial calculator looks like in bll:

```
$ ./bllsh
>>> blleval (a (nil a 2) (rc 1 (b (nil a (i 3 (nil * 3 (a 2 (rc (b (- 3 (nil . 1))) 2))) (nil nil . 1)))))) 5
120
```

Writing code in symbll is somewhat easier, provided you're familiar with functional programming:

```
$ ./bllsh
>>> def (FR N) (if N (* N (FR (- N 1))) 1)
>>> eval (FR 5)
120
```

There is also some fairly straightforward ability to do debugging, either printf-style:

```
$ ./bllsh
>>> def (FR N) (if (report N 'FR-called) (* N (FR (- N 1))) 1)
>>> eval (FR 5)
report: (5 <FR-called>)
report: (4 <FR-called>)
report: (3 <FR-called>)
report: (2 <FR-called>)
report: (1 <FR-called>)
report: (nil <FR-called>)
120
```

or step-through debugging:

```
$ ./bllsh
>>> def (FR N) (if N (* N (FR (- N 1))) 1)
>>> debug (FR 5)
   -- 1. FN(fn_symbll_eval,nil)    [<FR> 5]
>>> step
   -- 1. FN(fn_userfunc,[[<if> <N> [<*> <N> [<FR> [<-> <N> 1]]] 1] [<N>]])    (5)
>>> step
   -- 2. FN(fn_symbll_eval,nil)    5
   -- 1. FN(fn_userfunc,[[<if> <N> [<*> <N> [<FR> [<-> <N> 1]]] 1] [<N>]])    nil
>>> step
   -- 2. FN(fn_fin,nil)    5
   -- 1. FN(fn_userfunc,[[<if> <N> [<*> <N> [<FR> [<-> <N> 1]]] 1] [<N>]])    nil
>>> step
   -- 1. FN(fn_userfunc,[[<if> <N> [<*> <N> [<FR> [<-> <N> 1]]] 1] nil [5 . <N>]])    nil
>>> cont
Result: 120
```

There's also the ability to compile symbll into bll:

```
$ ./bllsh
>>> def (FR N) (if N (* N (FR (- N 1))) 1)
>>> program FR
(1 (nil 1 2) (6 1 (10 (nil 1 (5 3 (nil 25 3 (1 2 (6 (10 (24 3 (nil . 1))) 2))) (nil nil . 1))))))
```

(which is just the original bll program we had above, with the opcodes represented by their numbers, rather than their symbol). The compiler logic is essentially simple macro rewriting, so there's plenty of potential for optimisation there eventually, but it already seems to work pretty well, and being able to run and debug the high level code natively is (at least in my opinion) a huge win.

Anyway, the point of this post was just to update on the language design; hopefully having done that it won't be too hard to move onto discussing actual interesting things you can do in future posts.

-------------------------

