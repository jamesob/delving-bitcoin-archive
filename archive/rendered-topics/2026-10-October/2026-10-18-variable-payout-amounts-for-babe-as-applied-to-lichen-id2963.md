# Variable payout amounts for BABE (as applied to Lichen)

AdamISZ | 2026-10-18 22:09:57 UTC | #1

Quick background for those who haven't read [BABE](https://eprint.iacr.org/2026/065).

It starts with Groth16 verification. For these purposes you just need to understand that a prover can prove that an arbitrarily complex statement is true in an  O(1) and concretely short string consisting of three group elements, traditionally written $\pi_1, \pi_2, \pi_3$ (two from one group and one from another, but we'll skip pairing details). Following (1) from the BABE paper, you can boil down that verification to something of the shape:

\[e(\pi_1, \pi_2) + e(\pi_3,x_2) = x_3 \]

(using an additive notation for the operations in the target group, for those who care about such things).

Now it so happens that witness encryption can't [1] be done for this kind of non-linear relation (nonlinear in the sense that ab +c = d is non linear if a and b are variables, whereas 2b + c = d is linear). But there is a well known technique for doing witness encryption for the linear version of a pairing relation like the one above. I saw it first in [this paper](https://eprint.iacr.org/2024/264)  for KZG commitments. It's best understood as a kind of blinding-unblinding operation. 

Before describing how to do it in this case (the main BABE idea), let's ask why it's useful: if you can encrypt a secret $msg$ to a ciphertext $ctxt$ such that it can only be decrypted by someone who has a valid SNARK proof $\pi_1, \pi_2, \pi_3$, you can arrange for a coin on bitcoin to be unlocked only be someone with such a proof. More on that in a moment.

First, I said that WE *can't* be done for the above Groth16 verification equation. BABE's main idea is to solve that problem by treating one of the first two proof elements as a value that comes out of a 2-party computation. At setup time, the verifier garbles a circuit to calculate $r\pi_1$ from a future input $\pi_1$, for a secret $r$ the verifier does not reveal (remember the $\pi_k$ are all group elements, so dlog hardness applies). Insert the normal discussion of GC in terms of size, cut and choose for malicious setting etc.

Then, at proving time, the prover posts $\pi_1$ on chain, Lamport-signed so that it's fixed. If the verifier thinks the proof is bad, she challenges by posting the garbled circuit's input labels for exactly that $\pi_1$; the script checks that they match, and if she doesn't challenge, the prover simply wins after a timeout. The prover evaluates the garbled circuit locally and gets $r\pi_1$ without learning $r$.  From this, the prover can calculate the witness decryption of the $ctxt$ into the secret $msg$, if and only if his full proof $\pi_1, \pi_2, \pi_3$ is correct (in case that's confusing: the verifier also handed over other ciphertext elements that depend on $r$, earlier, and the trick of this linear WE technique is that the value of $r$ cancels to yield the decryption). (Details of BABE's tx construction omitted, see the paper).

OK, introduction over :laughing: 

This is obviously powerful and in particular it's clever in greatly reducing what you have to do the GC evaluation over (one scalar mult.). But it has a limitation: the statement you are proving is fixed at setup time, it is not dynamic. As per the paper's motivating example in 1.1, you can imagine using this method to prove that you are entitled to spend a utxo, so it's resolving a dispute over a fixed amount of money; can work for certain applications like recovering collateral, but is limited.

Consider if you presigned a transaction tree that encoded an amount to be recovered that's variable: bit- or digit- decompose the amount over a range and have a final output in the payout transaction for each digit (as [here](https://github.com/AdamISZ/lichen/blob/a94cad1b063f286b6b68cb21745b2407767c3f0c/docs/explainers/payout.md)). Sounds nice, but it's of no use if *the statement that is being proved has to fix the payout amount at contract setup*. Getting round that by, say, having a bunch of chunks of money proved separately is ugly and limited. But ... there's another way! Assuming (as in BABE and as in [Lichen](https://github.com/AdamISZ/lichen) ) that the verifier party is pre-arranged at setup and is the same at prove/verify time, then  we can insert the amount into the proved statement. It's a surprising piece of logic that depends on an interaction between prover P and verifier V, but it works like this:

* At setup, the statement for proving used by P will be "The withdrawal of amount $a$ is valid according to the L2 (or similar) rules AND I have a signature from V's key on the same amount $a$." The WE setup is based on the circuit from this.
* P wants to withdraw; says to V "I am going to withdraw amount $a$" and commits to this in the kick-off transaction with a WOTS.
* V gives a signature (again, WOTS, because this must be in a bitcoin contract) on $a$ before a timeout, else the money goes to P. The bitcoin contract has to check that this $a$ equals the $a$ that P requested, of course.
* P constructs a valid proof (see above) using that signature.
* The BABE WE will therefore be usable from the setup **which did not include the actual value $a$**, and will give the decryption if and only if P has such a valid withdrawal claim $\pi_1, \pi_2, \pi_3$. This unlocks the onchain path that will payout P with the amount (already in the contract as $a$), following the bit decomposition technique already referred to [2].

The logic is initially surprising: V is being told to sign any arbitrary amount $a$ by the Prover, even if she is blatantly lying (V may be able to see the amount on the L2 and it could be wrong). 

Imagine it like this:

* announce amount with a one time signature
* get a signature from the counterparty (also one time and *forced* to be on the same amunt)
* prove you have that signature from the counterparty on that amount and also a valid proof for that amount

If you didn't have the second step, there'd be nothing in the ZKP that actually says what the amount is. The **usual** way to fix that is to make the amount a public input and not a secret witness, but that's what you can't do in BABE because the **public inputs are fixed at setup in this witness encryption**.

Note, withdrawals from hub-user contracts of this type are one-shot: once P chooses to withdraw, she either succeeds or not, but it closes the contract. She can't try again.

Another thing to notice is that none of BABE's cryptography changes. The garbled circuit is the same, still a single scalar multiplication on $\pi_1$, and so is the WE. Only the statement changes, but the extra signature check costs nothing externally (constant size Groth16) . What we've added is on the Bitcoin side: P commits to $a$ and V countersigns it before the usual BABE flow starts.

Finally, this analysis exists on the same topic (variable amounts in BABE): https://hackmd.io/@goatresearch/HkKp2g1Zfl ; I haven't studied it properly but it seems to involve more garbling and more crypto (see previous paragraph). At least for Lichen, and maybe for other things, I think the "countersignature" approach is a better one.

[1] Not literally 'can't', it's more that no efficient/practical technique is known and apparently there are theoretical results relating this to much harder open problems. I'll let the theoreticians explain more.

[2] The binding inside the presigned transaction chain is needed both-ways: V is winternitz-signing $a$ but so is P in her initial claim; they must be enforced the same, and the final payout transaction with bit decomposition cannot vary from it.

-------------------------

