# The path to general computation on Bitcoin (with OP_CAT)

victorkstarkware | 2024-08-29 11:21:23 UTC | #1

![The path to general computation on Bitcoin|1456x819](upload://5tVi05nWVNaSzs4HwtstLetGmZM.jpeg)

## 1. Introduction

In this post, we will discuss the task of achieving general computation on Bitcoin and the various steps it entails. Indeed, the ability to compute arbitrary logic in Bitcoin script is a very compelling goal, as it opens Bitcoin to the realms of non-custodial applications and decentralized finance.

Previously, general computation on Bitcoin was deemed all but impossible due to two severe limitations: Bitcoin script length and opcode expressibility. The former prohibits transactions with anything but short scripts, and the latter limits what the script opcodes can compute. Jointly, these paint a dire picture of Bitcoin’s computation viability, as many applications are not possible to implement within these constraints.

Nevertheless, in recent years, this state of affairs has significantly improved. Indeed, the script length limitation was lifted with the 2021 Taproot upgrade, allowing for significantly longer Bitcoin scripts (in fact, Taproot scripts even allow to sometimes only pay onchain for **a** **part of the script**, but we’ll not explore this property in this post). Moreover, to deal with the opcode expressibility limitation, a single simple opcode was identified to be sufficient for effectively unlimited expressibility.

The aforementioned opcode is known as **OP_CAT**, and it has been disabled in Bitcoin script since 2010. It’s a very simple opcode that allows **concatenating elements on the stack** and requires just **13 lines of code** to be activated on Bitcoin non-intrusively [via a soft fork](https://github.com/bitcoin/bips/blob/97012a82064c7247df502a170c03b053825cdd15/bip-0347.mediawiki). Deriving so much power from this seemingly unassuming opcode is highly nontrivial, and hence, a central goal for this blog is to elaborate on this point.

In what follows, we will dive deeper into computation on Bitcoin, starting from how it’s done, and what are the limitations.

Following that, we’ll overview two tools—**covenants** and **STARK proofs**—and show that if made possible in Bitcoin script, they will enable general computation. Furthermore, we will argue in this blog post that, thanks to STARK Proofs, this computation can be bestowed with additional useful properties:

* **Scalability** – Onchain transaction processing and computation becomes feasible, with very low fees.
* **Expressibility** – The logic can be programmed in a different, more powerful, programming language than Bitcoin script. This is vital for making programming on Bitcoin ergonomic and safe.
* **Onchain safety** – No matter what the computation is, Bitcoin script is only ever verifying a mathematical proof of delegated computation.
* **Flexibility** – Computation on Bitcoin can store global state, enabling a plethora of applications.

Finally, we’ll discuss OP_CAT and how it enables the tools mentioned above. We’ll also reference some of our efforts in this direction and discuss what’s to come.

***Warning: This post simplifies some technicalities to provide a clear mental model. When it comes to actually implementing the presented ideas, “the devil is in the details.”***

## 2. UTXO model and locking scripts

In this section, we briefly outline Bitcoin’s UTXO model, where each UTXO is locked behind a locking script. If you are familiar with this, you can skip to the next section.

The first step to understanding transactions on Bitcoin is to note they are arranged and processed according to the **unspent transaction output** (UTXO) model, as in Figure 1. Bitcoin transactions can have multiple inputs and outputs, where each output represents an amount of bitcoins that can be spent by another transaction. Accordingly, a transaction’s input represents the spending of another Bitcoin transaction’s output. Naturally, we require that the cumulative amount of bitcoins in the inputs be roughly equal to the amount in the outputs (with the difference being the fee paid to the miners).

![Bitcoin transactions in the UTXO model|1456x819](upload://2jCIhM2ReIBTqKLX61VroY976rW.png)

***Figure 1**: Transactions in the UTXO model. Each input and output of a transaction has an amount of Bitcoins associated with it, where the cumulative amount in the outputs can’t exceed that of the inputs. Unspent transaction outputs (UTXOs) are colored in blue.*

Each transaction output is locked behind a **locking script** (scriptPubKey), which will either accept or reject an input (scriptSig) it is given (by the so-called **spender**). Thus if you want to transfer the funds from an **unspent transaction output** (UTXO), you need to be able to generate the correct scriptSig and include this UTXO as an input in your generated transaction.

By default, the locking script will simply verify digital signatures against a hardcoded public key, which defines the **ownership** of the locked bitcoins. More generally, whenever there is a UTXO, we say that the person who is able to produce the correct scriptSig is the **owner** of the bitcoins locked behind the corresponding locking script. Hence, the **total balance of each Bitcoin owner** is **dictated cumulatively** by all **UTXOs** associated with this owner (for example, all UTXOs requiring a digital signature from the same person).

## 3. Limitations of Bitcoin script

The above section outlines locking scripts in general, and how they interact with Bitcoin transactions and Bitcoins. In this section, we focus on the locking scripts themselves and what they consist of.

Locking scripts on Bitcoin are written in a [Forth](https://en.wikipedia.org/wiki/Forth_(programming_language))-like Stack-based language known as “Bitcoin script.” Since Bitcoin script does not have loops, we measure the cost of a script by how many opcodes it requires, as the length of a script is proportional to its running time. In Figure 2, you can see an illustration of a simple program that checks whether the sum of two inputs is 12.

![A simple program that checks whether the sum of two inputs is 12 (Bitcoin script)|1456x819](upload://tXG0xWOmShwMLg4rI1WNDlRyFqK.png)

***Figure 2:** A simple program that checks whether the sum of two inputs is 12. The execution is performed by concatenating the scriptSig with the locking script itself, after which the opcodes are processed from left to right. The scriptSig only pushes elements onto the stack. **(a)** The first step of computation. **(b)** In the middle of computation. **(c)** Final step of computation (the locking script accepts the input).*

We pointed out that Bitcoin Script **has many limitations**, which makes developing even simple logic for it extremely nontrivial, if not downright impossible. Some of these include:

* Stack size is limited to at most 1000 elements.
* No opcodes for multiplications (indeed, even a 31-bit integer multiplication needs to be emulated by [~1.4k opcodes](https://github.com/Bitcoin-Wildlife-Sanctuary/rust-bitcoin-m31)).
* The total number of opcodes that can fit inside a single transaction is at most ~4 million for [**Pay2Taproot**](https://en.bitcoin.it/wiki/BIP_0341) output types (following the Taproot upgrade), filling an entire block, and just 10k opcodes for other output types.
* Arithmetic operations (addition, subtraction) are limited to 4-byte elements.

The 4-byte limitation mentioned in the last point is especially problematic. Indeed, it implies you **can’t modify** long elements, for example, elements of the order of 20 bytes or more, which is required for **cryptographic operations**.

For instance, the very common operation of **concatenating** data together and applying a cryptographic operation is impossible, unless you **ignore the cryptographic opcodes, and emulate them as operations on data represented by 4-byte element arrays.** The latter is prohibitively expensive, as a single hash operation could require hundreds of thousands of opcodes to emulate.

As a final remark, we point out that due to the difficulty of developing in Bitcoin script, the code would naturally be more prone to bugs and vulnerabilities.

## 4. The challenge of keeping state in Bitcoin script

While the previous section demonstrated that Bitcoin scripting leaves much to be desired, we argue that the biggest limitation of Bitcoin locking scripts is the fact **they are only able to read the input they are given by the spender.** Unassuming at first, this implies two severe limitations:

* Bitcoin scripts **are not stateful**. Namely, they cannot read/write storage cell data. In analogy to Ethereum VM, there are no **SLOAD**/**SSTORE** opcode analogs, which is an essential feature for general computation.
* Bitcoin scripts **can’t enforce how bitcoins are spent** (with a few exceptions). This is solved via **covenants**, which give the locking script this ability. (You can imagine a simple application for this: **onchain vaults**, which work by restricting the address to which spending is allowed. More generally, covenants are an extremely powerful tool with a wide range of applications.)

Subsequently, we will see that constructing covenants already gives you statefulness. Intuitively, this is the case because you can use a covenant to emulate storage reads/writes in the transaction data itself.

Indeed, all of the above does not scratch the surface of the breadth of what there is to know about Bitcoin scripting and its intricacies. But suffice it to say that writing scripts for Bitcoin is not easy, to say the least. To conclude, Bitcoin scripting emerges as having very strict constraints:

* You mustn’t exceed the stack size limit of 1000 elements.
* You must emulate simple operations such as multiplications with many opcodes. In general, you must write your logic in the cumbersome Bitcoin scripting language.
* You must operate on elements encoded in 4-byte form. In particular, you cannot operate on and change the inputs to cryptographic opcodes.
* Your logic cannot hold state and must fit in a single transaction.
* Your logic cannot have control over how bitcoins are being spent once authentication occurs.

In addition, writing code subject to the above constraints is prone to bugs and vulnerabilities.

While very simple applications such as payments can meet the above constraints, anything slightly more complicated will encounter a roadblock. In the following, we will see how **covenants** and **STARK proofs** enable us to break free from these limitations. Furthermore, we will discuss how OP_CAT facilitates the possibility of the above in Bitcoin script.

## 5. Smart Contracts on Bitcoin via Covenants

In this section, we will see how covenants in Bitcoin allow logic to keep state, and, more generally, what a “**smart contract**” looks like on Bitcoin. A smart contract here means some sort of stateful logic, with a collection of functions that can be invoked to transition the state to a different, valid one, according to predefined rules.

More concretely, Bitcoin contracts need the following functionalities:

* Persistent logic and state
* Ability to control how logic/state is changed

As previously mentioned, the above can be achieved through **covenants**. Recall that by default, a locking script can only read data that is directly fed to it (scriptSig). However, through a covenant, a locking script can read **all the data fields of the spender transaction**, as well as **the data fields of the transaction where the locking script is located,** and even data from other transactions. Figure 3 illustrates the different parts a locking script can access using a covenant.

![Some of the data fields a locking script can access (Bitcoin script).|1456x819](upload://sU5Yf5KbnsQnSDgwSCEoHN0czKQ.png)

***Figure 3:** Some of the data fields a locking script (visualized as a lock) can access (shown by the red arrows). Beyond accessing the data fields of the spender transaction, the locking script can also access the data fields of the transaction where the locking script is (tx1), as well as the data fields of another transaction that is also spent by the spender transaction (tx2).*

We claim the covenant described above is sufficient to build general smart contracts on Bitcoin. Indeed, the general approach would be to encode the state in the transaction data itself and check the validity of its transitions to new values. To this end, our transaction will have two outputs:

* The first output holds the logic of the contract (via a locking script), as well as the funds locked in the contract.
* The second output holds the **current state** of the smart contract. This UTXO’s locking script only holds state and will never be spent (it locks practically 0 bitcoins).

The locking script logic will enforce the following rules:

* The spender transaction **must have the same exact form** (two outputs only, and all funds locked in the first output),
* The first output **must have the same locking script logic**
* The second output **must be a valid state**
* The input provided to the current locking script **must convince it the transition from the current state to the new state is valid** (the validity is defined by the logic designer)

You can see an illustration of this construction in Figure 4. As written [here](https://github.com/Bitcoin-Wildlife-Sanctuary/covenants-examples) by Weikeng Chen, a number of people have been suggesting a formal name for this construction, and “**state** **caboose**” stands out as a candidate. Caboose refers to a railroad car coupled at the **end** of a freight train as a shelter and working space for the crew.

![Bitcoin post figure 4|1456x819](upload://aQpDnPF8zZbSrgyZ8gSdWxPbWrm.png)

***Figure 4:** Smart contracts on Bitcoin via covenants that follow the “state caboose” design pattern. The locking script enforces that the spender transaction has the same form and locking script, and that it has a valid state S’, whose transition from the previous state S was also valid.*

As an illustrative example, you can think of a simple counting smart contract. This contract’s state is always incremented by the value provided by the spender, which effectively invokes an “accumulate” function. This counting smart contract is shown in Figure 5.

![A simple smart contract that accumulates inputs (scriptSig) into its state by adding them together.|1456x819](upload://x4O8PwmFo0Q24vMo7sBDWmMQLzv.png)

***Figure 5:** A simple smart contract that accumulates inputs (scriptSig) into its state by adding them together.*

As a final remark, we note that to “call the contract” and transition the contract to a new valid state, a user needs to create a new spender transaction for each call. In this way, **unlike Ethereum smart contracts, Bitcoin smart contracts are inherently sequential,** requiring the transaction to commit to both the current and new state. **This sequential property must be taken into account when building smart contracts on Bitcoin.** There are some possible mitigations to this, but we won’t discuss them in this post.

To conclude, we’ve seen that covenants enable smart contracts on Bitcoin, which, in theory, allows for **arbitrarily long computation** to be carried on top of Bitcoin when split across sufficiently many transactions. Nevertheless, using covenants alone still suffers from most of the limitations of Bitcoin script, namely, stack size limitations, a limited number of opcodes, and, in general, programming according to the Bitcoin script language constraints.

In the following, we will see how **STARK proof systems** can alleviate the above limitations by reducing the blockchain footprint of computation on Bitcoin and enabling scripting in an entirely different language, which dramatically improves programming efficiency and safety.

## 6. A Bitcoin STARK verifier

The goal of this section is to discuss how computation can be done on Bitcoin, while keeping the actual computing done via Bitcoin script to a minimum. Our general approach to this will be through **computation delegation**, where the computing will happen offchain (multiple entities can potentially contribute), with only verification happening onchain, in Bitcoin script.

This begs the question, how can we trust that offchain computation was carried out correctly? Indeed, a very natural candidate to solve this problem is **a proof system**. Put simply, a proof system allows Alice (a **prover**) to convince Bob (a **verifier**) of the correctness of some statement by providing a “proof,” and, crucially:

* The verifier needs to do **less work** to verify this statement, as opposed to how much he had to work without the prover’s help.
* Furthermore, the verifier is **protected**, in that the prover can’t cheat the verifier and convince them of a false statement.

The proven statement can be virtually any statement, from the solution to a difficult puzzle, to the more relevant example of computation delegation. More concretely, Alice will prove to Bob that “f(x)=y”, instead of Bob having to do the computation of “f(x)” himself, as shown in Figure 6.

![A proof system for computation delegation.|1456x819](upload://sfTqGE7IsQKwNHa1srozIqbZ1ms.png)

***Figure 6:** A proof system for computation delegation. The prover computes y=f(x) and convinces the verifier of the correctness of this statement. Crucially, if y≠f(x), the prover couldn’t have convinced the verifier of this fact.*

Hence, our approach to minimizing the blockchain footprint of computation will be to implement a proof system verifier in Bitcoin script. Thus, to invoke the smart contract, the caller will provide a proof of a correct state transition, and the Bitcoin smart contract will verify the correctness of the proof of this state transition, as opposed to checking the state transition directly (see Figure 7).

Moreover, an additional crucial benefit of this approach is that the Bitcoin script logic can stay **fixed regardless of application**, which significantly reduces the chances of bugs and eases auditing. This stems from the simple fact that the same verifier algorithm can be made to verify either a “y=f(x)” statement or a “y=g(x)” statement without needing to know the function to be computed beforehand.

![Smart contracts on Bitcoin via covenants that follow the “state caboose” design pattern, as in Figure 4, but with a verifier.|1456x819](upload://cfkBMTIgb99AC6uuFTcWxKN0eEW.png)

***Figure 7:** Smart contracts on Bitcoin via covenants that follow the “state caboose” design pattern, as in Figure 4, but with a verifier. This time, instead of checking the validity of the transition from S to S’ directly, the locking script verifies a proof that the transition was correct.*

There are many proof systems, however, we choose to use STARK proof systems (STARKs) because of their many extremely attractive features:

* Post-quantum secure
* No trusted setup
* No new cryptographic assumptions for Bitcoin
* Fastest to scale
* Battle-tested, trusted with over $1T settled

Finally, in contrast to other proof systems, STARKs are Bitcoin-friendly, namely, their verifier component can be more easily implemented in Bitcoin script. In essence, this is due to the fact STARKs are mostly hash-based, with very few algebraic operations, compared with pairing-based proofs. Since algebraic operations are very costly in Bitcoin script, this explains the Bitcoin-friendliness of STARKs. Furthermore, the [Circle-STARK](https://eprint.iacr.org/2024/278) variant is especially Bitcoin-friendly, thanks to its small field size.

Thus, since the Bitcoin verifier can relatively easily verify any computation, we **can choose which type of computation it verifies**. Remarkably, this can even be some kind of **CPU execution**. Furthermore, one can design high-level programming languages built on top of this CPU. To this end, at StarkWare, we have developed [Cairo](https://www.cairo-lang.org/), a Rust-like, ergonomic, and safe programming language **designed specifically for efficient proving and verification.** Proving the execution of Cairo on Bitcoin can be very advantageous.

To conclude, using a STARK verifier, together with covenants, we are able to achieve general computation on Bitcoin. Moreover, this computation has the following attractive additional features:

* **Scalability** – onchain transaction processing and computation becomes feasible, with very low fees.
* **Expressibility** – The logic can be programmed in a different, more powerful, programming language than Bitcoin script. This is vital for making programming on Bitcoin ergonomic and safe.
* **Onchain safety** – No matter what the computation is, Bitcoin script is only ever verifying a mathematical proof of delegated computation.
* **Flexibility** – Computation on Bitcoin can store global state, enabling a plethora of applications.

## 7. Using OP_CAT for concatenation and beyond

As mentioned above, OP_CAT is a simple, currently disabled, opcode that could allow Bitcoin script to concatenate elements on the stack. The importance of this simple operation cannot be overstated, as it simultaneously enables **covenants** and **STARKs** on Bitcoin. It does so in the following way:

* **STARKs** – In fact, this is somewhat unsurprising. This is because STARKs practically consist of just **concatenating** data together and hashing it, which leads to great savings as hashing is a native Bitcoin script operation, unlike algebraic operations. The main hashing operations in STARKs are **Merkle path verification** (see Figure 8), and the **Fiat-Shamir transform**. (Furthermore, the field size of the Circle-STARK variant is only 31 bits, so it fits in the 4-byte restriction of Bitcoin script, making it a Bitcoin-friendly algorithm.)

* **Covenants** – In 2021, Andrew Poelstra made the [nontrivial observation](https://medium.com/blockstream/cat-and-schnorr-tricks-i-faf1b59bd298) that OP_CAT can enable covenants on Bitcoin, through something called the **Schnorr trick**, where Schnorr’s algorithm is the digital signature of **Pay2Taproot** output types (for other output types, a similar **ECDSA trick** can be used, [as observed by Robin Linus](https://gist.github.com/RobinLinus/9a69f5552be94d13170ec79bf34d5e85)). To briefly describe the idea, to get covenants, we need to use OP_CHECKSIG, which is the only opcode that is capable of putting data related to the spender transaction onto the stack. It’s not entirely straightforward, but through some manipulations, you can access all the necessary data.

Regarding the last point, keeping state inside **Pay2Taproot** outputs involves some technical difficulties. However, it is possible to store the state using other output types, such as **Pay2WitnessScriptHash**. This yields the aforementioned “state caboose” technique from Figures 4 and 5, with two outputs in the transaction.

![Merkle path verification, an operation that involves hashing and string concatenation. |1464x827](upload://7HWVz9HtWeItPM7Y2AycdBUleBK.png)

***Figure 8:** Merkle path verification, an operation that involves hashing and string concatenation. The Merkle path parts (blue strings) are concatenated and hashed accordingly. Then, equality is checked with the Merkle root. **OP_CAT** operations are written as “||”.*

## 8. Present and future

In an open-source effort, led by Weikeng Chen and Pinghzou Yuan, we’re working on building the [Bitcoin Wildlife Sanctuary](https://github.com/Bitcoin-Wildlife-Sanctuary). Two primary projects there are:

* A [Circle-STARK verifier](https://github.com/Bitcoin-Wildlife-Sanctuary/bitcoin-circle-stark) with a working demo that **verifies a STARK proof of a simple statement (related to Fibonacci numbers) on Bitcoin signet,** which you can track through this [address](https://mempool.space/signet/address/tb1p2jczsavv377s46epv9ry6uydy67fqew0ghdhxtz2xp5f56ghj5wqlexrvn).
* A [framework for covenants](https://github.com/Bitcoin-Wildlife-Sanctuary/covenants-gadgets), already with an application via a [simple counting covenant](https://github.com/Bitcoin-Wildlife-Sanctuary/covenants-examples). The verifier also uses this framework.

Through this effort, we aim to bring OP_CAT-enabled smart contracts to Bitcoin that are efficient, secure, and developer-friendly. We feel that this is a promising and exciting prospect for the realm of computation on Bitcoin.

## 9. Closing words

To conclude, in this post we’ve seen that:

* **Covenants** are a powerful tool for Bitcoin Script that allows for the creation of smart contracts on Bitcoin. They expand Bitcoin’s capabilities beyond simple value transfers.
* However, even with covenants, Bitcoin script has many limitations, including restrictions on stack size and the types of opcodes that can be used. This limits the complexity and expressiveness of smart contracts that can be implemented using covenants alone.

* **STARKs** offer a promising solution to alleviate these limitations, as they can efficiently verify the execution of computations offchain. By leveraging STARKs, it is possible to perform more complex computations within Bitcoin smart contracts and minimize their onchain footprint.
* **Cairo** is a programming language specifically designed for efficient proving and verification using STARKs, making it well-suited for use in Bitcoin smart contracts. It enables the creation of more sophisticated smart contracts with increased ergonomics, functionality, and safety.
* **OP_CAT** is a Bitcoin script opcode that concatenates elements. It plays a crucial role in implementing covenants and STARKs on Bitcoin, which, in turn, brings powerful general computation to Bitcoin.

*Thanks to Weikeng Chen, Pingzhou Yuan, and the team at StarkWare for input on drafts of this post.*

-------------------------

Laz1m0v | 2025-04-06 08:26:53 UTC | #2

Hello, 
Thanks for your work and your post. 
I have several remarks to your post. 

[quote="victorkstarkware, post:1, topic:1106"]
2. UTXO model and locking scripts
[/quote]

First, you remind what is UTXO model and locking, that's cool but doesn't seems useful in this place. I hope most of readers here are aware about this.

[quote="victorkstarkware, post:1, topic:1106"]
Bitcoin scripts **are not stateful**.
[/quote]
This assertion is true but it can be done through Indexers and trackers. You don't specify in your post how a Bitcoin node should evolve to handle such states and how you'll be able to handle those evolutive states in Bitcoin (node). Do node will directly store each states? Do we would need to build covenants trackers into the node? 

[quote="victorkstarkware, post:1, topic:1106"]
what a “**smart contract**” looks like on Bitcoin.
[/quote]
The use of term **covenant** looks especially crafted to avoid the use of "smart contract" (SC) term which is not well defined especially in this case. Will you be able to trigger automatically new transactions with your implementation? I don't think so, and this implies that the "smart contract capabilities" are not fully embraced. We don't have `OP_CALL` on Bitcoin and as with OP_CAT we won't be able to trigger from the protocol level new transactions (or at least it's not discussed in your post). 

[quote="victorkstarkware, post:1, topic:1106"]
We claim the covenant described above is sufficient to build general smart contracts on Bitcoin.
[/quote]
This can't be assumed as long as there is no precise definition of a smart contract in the post and the automatic trigger of a SC is not shown. 

[quote="victorkstarkware, post:1, topic:1106"]
**Figure 6:** A proof system for computation delegation. The prover computes y=f(x) and convinces the verifier of the correctness of this statement. Crucially, if y≠f(x), the prover couldn’t have convinced the verifier of this fact.
[/quote]
You remind what UTXO model is but you don't explain the proof computation in STARK models neither a general definition of STARK which seems more important. 

[quote="victorkstarkware, post:1, topic:1106"]
Bitcoin smart contract
[/quote]
Do you mean the Bitcoin covenant? 

[quote="victorkstarkware, post:1, topic:1106"]
Battle-tested, trusted with over $1T settled
[/quote]
Where does this value come from? 

[quote="victorkstarkware, post:1, topic:1106"]
7. Using OP_CAT for concatenation and beyond
[/quote]
All the previous sections mention SC/convenants but never OP_CAT does it mean that covenants can be done without OP_CAT? 
 

[quote="victorkstarkware, post:1, topic:1106"]
However, it is possible to store the state using other output types, such as **Pay2WitnessScriptHash**.
[/quote]
Does it mean that your covenants will be implemented in P2WSH? 
How exactly the Schnorr trick compensates for the absence of address tweaking? 
How state tracking would be managed at the node level?

[quote="victorkstarkware, post:1, topic:1106"]
A [Circle-STARK verifier](https://github.com/Bitcoin-Wildlife-Sanctuary/bitcoin-circle-stark) with a working demo that **verifies a STARK proof of a simple statement (related to Fibonacci numbers) on Bitcoin signet,** which you can track through this [address](https://mempool.space/signet/address/tb1p2jczsavv377s46epv9ry6uydy67fqew0ghdhxtz2xp5f56ghj5wqlexrvn).
[/quote]
Where states can be verified? 

[quote="victorkstarkware, post:1, topic:1106"]
Through this effort, we aim to bring OP_CAT-enabled smart contracts to Bitcoin that are efficient, secure,
[/quote]
Please stop mentioning Bitcoin smart contract it's horrible and factually wrong. We have a word for this it's covenants.

-------------------------

moonsettler | 2025-04-15 08:55:58 UTC | #3

[quote="victorkstarkware, post:1, topic:1106"]
As mentioned above, OP_CAT is a simple, currently disabled, opcode that could allow Bitcoin script to concatenate elements on the stack. The importance of this simple operation cannot be overstated, as it simultaneously enables **covenants** and **STARKs** on Bitcoin. It does so in the following way:

* **STARKs** – In fact, this is somewhat unsurprising. This is because STARKs practically consist of just **concatenating** data together and hashing it, which leads to great savings as hashing is a native Bitcoin script operation, unlike algebraic operations. The main hashing operations in STARKs are **Merkle path verification** (see Figure 8), and the **Fiat-Shamir transform**. (Furthermore, the field size of the Circle-STARK variant is only 31 bits, so it fits in the 4-byte restriction of Bitcoin script, making it a Bitcoin-friendly algorithm.)
* **Covenants** – In 2021, Andrew Poelstra made the [nontrivial observation](https://medium.com/blockstream/cat-and-schnorr-tricks-i-faf1b59bd298) that OP_CAT can enable covenants on Bitcoin, through something called the **Schnorr trick**, where Schnorr’s algorithm is the digital signature of **Pay2Taproot** output types (for other output types, a similar **ECDSA trick** can be used, [as observed by Robin Linus](https://gist.github.com/RobinLinus/9a69f5552be94d13170ec79bf34d5e85)). To briefly describe the idea, to get covenants, we need to use OP_CHECKSIG, which is the only opcode that is capable of putting data related to the spender transaction onto the stack. It’s not entirely straightforward, but through some manipulations, you can access all the necessary data.
[/quote]

Curiously LNhance enables both covenants with `CTV` and `CSFS` and multi-commitments with `PAIRCOMMIT`. Yet it does not give us functional STARK proofs as far as I know. I am intrigued by the crucial piece missing. Is it the simple act of concatenation in the end?

-------------------------

