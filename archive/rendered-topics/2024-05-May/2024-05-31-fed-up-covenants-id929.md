# FE'd Up Covenants

JeremyRubin | 2024-05-31 11:37:32 UTC | #1

Linked below, a high level paper on how covenants could be introduced to Bitcoin without a soft fork using a mix of Functional Encryption and Zero Knowledge Proofs, posted here for discussion.

https://rubin.io/bitcoin/2024/05/29/fed-up-covenants/

-------------------------

AdamISZ | 2024-05-31 15:33:17 UTC | #2

Hi Jeremy,

thanks for this, looks very interesting. As I'm reading I'm seeing something that looks like a typo:

"Next, encrypt $m$ " - should that be $p$ ? and,

$UniqueSecKey(p, TX) = Add(m, CTVHash(TX))$

Same, shouldn't $m$ be $p$ here?

-------------------------

AdamISZ | 2024-05-31 18:50:11 UTC | #3

Also, a clarifying question for the end of that section:

You write:

" $\sigma \leftarrow C_F(C_p, Encrypt(M, TX))$

$\sigma$ can be used as a signature of $u$ over TX".

Would this be correct?

$\sigma = Decrypt(C_F, C_p, Encrypt(M, TX))$

... where I'm taking the definition of $Decrypt$ from the start of the paper and extending to the two-argument case, i.e. defining for an encryption public key of $M$, $Decrypt(q, c_1, c_2) = X$ where the function has been previously defined $F(a, b) = X$ and $c_1, c_2$ are ciphertexts of $a, b$ and $q$ is the output of $EncryptFunc(M, F)$.

So anyway, if I got that right, let me try to translate into English what this machine does:

The basic building block is the ability to create a decryption key that decrypts a given ciphertext to the *output* of $F$ when its plaintext is used as input.

The way you're using it here (seems pretty clever!) is that you define a function which is " $F$(private key, transaction) = the ECDSA signature of a transaction by a private key *tweaked by the hash of the transaction*". This means that given the ciphertext of that tweaked private key, and "given" the ciphertext for the transaction (anyone can generate ciphertexts; this is public key cryptography), you can pass those two arguments into the output of the "basic building block above"; you'll get a signature on that transaction. But, because of the tweak, if you try to do the same with any other transaction, it'll fail.

(Deleted earlier misunderstanding)

-------------------------

AdamISZ | 2024-05-31 18:51:12 UTC | #4

So have I got this right:

Given $C_F$, the "functional encryption" and $C_p$, it allows *anyone* to create a signature for *only the specified transaction for a given tweaked key*.

So the flow is like:

* $C_F$ and $C_p$ are created by the owner of $(m,M)$ and $(p,P)$
* $m$ and $p$ are deleted
* Then anyone could calculate a covenant output destination pubkey as a tweak of $P + Hash(tx)$
* They would know that the only way it can be signed over is by doing $\sigma = C_F(C_p, Encrypt(M, TX))$. Of course *anyone* could do that but that's no different than today, we know how to lock with additional conditions in Script

The point being that this process is repeatable, for any transaction so it doesn't require statically signing in advance, expecting certain outcomes.

-------------------------

JeremyRubin | 2024-05-31 18:54:35 UTC | #5

astute read -- yes it's a typo.

-------------------------

JeremyRubin | 2024-05-31 18:57:22 UTC | #6

yes this is correct understanding, and I think your decrypt function looks correct.

I think it's more intuitive to think of Decrypt as just the actual function call, but the normal FE papers don't do this for various reasons.

-------------------------

JeremyRubin | 2024-05-31 18:59:11 UTC | #7

Yeah that sounds right.


The whole program instance stuff is designed to handle cases where the covenant instance (the program key) and the transaction are a one-key-to-many-transaction case.

-------------------------

JeremyRubin | 2024-05-31 19:04:37 UTC | #8

@AdamISZ the deleting/editing is kind of confusing. But I think one other point to consider is normal signing federations have to actively sign this stuff, ongoing trust concern.

Whereas with FE'd Up Covenants, it's a one-time compiler build for all eternity (until quantum computers tell us we can't have fun ever again?).


My spookchains writeup is... spookily similar to this kind of setup, so maybe worth reading for a more thorough discussion (end section " Commentary on Trust and Covenantiness") https://rubin.io/bitcoin/2022/09/14/drivechain-apo/

-------------------------

