# Emulating curve point scalar multiplication with OP_CAT

RobinLinus | 2024-01-06 16:43:16 UTC | #1

[Here's a short writeup](https://gist.github.com/RobinLinus/8890ded496c9c12796dc6a65c196a147) of how to emulate curve point scalar multiplication with OP_CAT. This brings us quite close to computing TapTweaks, enabling recursive covenants.

-------------------------

ajtowns | 2024-01-08 02:10:16 UTC | #2

Copypasta:

---
> 
> # OP_CAT Enables Scalar Multiplication for EC Points
> CAT can reduce curve point scalar multiplication to a subtraction in the scalar field. 
> 
> Subtraction of field elements can probably be emulated in less than 250 (?) opcodes. For now, let's assume we had an (emulated) opcode, `op_scalar_sub`, for subtracting two elements of the scalar field of secp256k1.
> 
> 
> Given secp's generator `G`, we want to compute for some scalar `r` the point `R = rG`
> 
> That is possible by hacking it into a Schnorr signature `(R,s)` for the key `P = xG = 1G = G`
> 
> The Script performs the following steps:
> 1. Verify the signature `(R,s)` for the committed key `P = G`. That's possible with op_checksig.
> 2. Get the sighash `M` onto the stack using the Schnorr+CAT trick (requires a second signature)
> 3. Compute `c = Hash(R | P | M)` using op_cat, op_sha256
> 4. Compute `r' = s - c` using op_scalar_sub 
> 	  - this works because `s = r + c * x`, and `x = 1`
> 5. Verify `r == r'`
> 
> This proves that `R` is `r * G`, which is as good as computing the scalar multiplication ourselves. However, unfortunately, this works only for scalar multiplications with the generator point `G`. Still, that's useful.

---

I don't think that works though? If I want to falsely claim that `r*G = X`, then I create a valid signature using `s*G = x + H(X,G,m)*1`, then I make up a random value `z` that differs from `m`, calculate `c = Hash(X,G,z)`, and `r = s-c` and tell you that `r*G = X`. Without some way of verifying that the `z` I give you matches the tx msg hash used by the CHECKSIG operation, I don't think your procedure gives you any way to tell you that I'm lying? On the other hand, it doesn't give me a way of choosing the value of `z`, so perhaps that's good enough in some situations?

If you had CSFS you could force `m=z` directly; if you had both CTV and APO and they had a common tx msg hash function between them, you could use that to indirectly force `m=z` (`m DUP CTV DROP` checks `m` is correct and leaves `m` on the stack).

(If you're introducing a new secp256k1-specific 256 bit opcode `op_secp256k1_scalar_sub` anyway, I don't see why you wouldn't just introduce an `secp256k1_mul` opcode directly, though)

-------------------------

RobinLinus | 2024-01-08 03:46:20 UTC | #3

Not sure I understand you correctly. This procedure should enforce the message to be the
[quote="ajtowns, post:2, topic:370"] tx msg hash used by the CHECKSIG operation.[/quote]


- In step1 it is obviously the case.
- In step2 m=z is enforced with Andrew's CAT trick. It can be used as a primitive to push the sighash digest onto the stack. (Of course, you have to check here that the signature is exactly 64 bytes, implicitly enforcing SIGHASH_ALL.)

I don't see how your attack breaks that.


-----


[quote="ajtowns, post:2, topic:370"]
If you’re introducing a new secp256k1-specific 256 bit opcode `op_secp256k1_scalar_sub`
[/quote]

That's a misunderstanding. I do not want to introduce any opcode. 

My main point is that OP_CAT can reduce scalar multiplication for *curve points* to a single subtraction of *field elements*. That in itself is an interesting result.

Furthermore, I estimate that (given CAT) we probably can already implement subtraction of field elements in less than 250 opcodes. Definitely seems to be trivially possible when using kilobytes of Script. That's why I thought it's fair to assume `op_secp256k1_scalar_sub` as given, to then show the main point.

-------------------------

ajtowns | 2024-01-08 03:41:26 UTC | #4

Ah, you're right, I just can't read!

[quote="RobinLinus, post:3, topic:370"]
That’s a misunderstanding. I do not want to introduce any opcode.
[/quote]

I mean, you're introducing `OP_CAT` at a minimum :slight_smile:

-------------------------

RobinLinus | 2024-01-08 03:54:16 UTC | #5

[quote="ajtowns, post:4, topic:370"]
you’re introducing `OP_CAT`
[/quote]

The goal was rather to explore what's possible with CAT.

-------------------------

ajtowns | 2024-01-08 04:58:48 UTC | #6

[quote="RobinLinus, post:5, topic:370"]
The goal was rather to explore what’s possible with CAT.
[/quote]

Sure. Just feels like asking a handyman "If you could only have one tool to do your job, what would it be? A knife, a hammer, a screwdriver or a handsaw?" Fun thing to debate over beers or whatever, but seems a crazy limitation to apply in reality?

-------------------------

