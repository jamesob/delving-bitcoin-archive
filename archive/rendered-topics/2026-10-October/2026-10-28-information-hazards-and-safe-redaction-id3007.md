# Information hazards and safe redaction

laissez-faire | 2026-10-28 02:23:51 UTC | #1

Hi all, and thanks for having me, it's good to be here. 

Before I even start I need to list some assumptions. If you agree with the assumptions, then this topic is for you. If you disagree, then please say so.

* This post is about information hazards. 
* Specifically, it's about subjective information hazards. We'll never get sufficient consensus on a list of objective universal information hazards.
* Information hazards (the subjective kind) exist.
* It is possible for an information hazard to be written to the blockchain.
* For some people, the blockchain already contains information hazards today.
* For some people, the blockchain doesn't contain information hazards today, but it might tomorrow.
* There is currently no (effective, general) defence against information hazards being written to the blockchain. Any information hazard you can think of today, might be written to the blockchain tomorrow (up to some size limit).
* For a person who wishes not to store information hazards on their computer, and not to share and perpetuate them, current options are limited, and require trusting or relying on others. That may include backing up the blockchain, sharing UTXO data, or validating blocks.

I want to start a conversation about safely redacting information hazards, and creating a new option for users who care about this issue. An option that maintains as much of the functionality and as many of the benefits of a full archival node as possible while also giving the user the option of safely redacting any data that they consider to be an information hazard.

If you know of anyone else actively working in this space, please let me know. I know there has been work done (I'll get to that in more detail later) but there's nothing active and current happening today, to my knowledge. If there is, I'd like to know about it, so that I can join in.

I'll come back to this topic and share some more of my ideas and work and research soon. But for now, please share your thoughts on the topic in general, if any.

Laissez faire, laissez passer.

Let it be, let it go.

----

**Edit 1:** I should note that my intention here is to research, discuss and agree what can be done without putting restrictions on the publication of arbitrary data into new blocks. Such restrictions may or may not be a viable approach to protecting users from information hazards - it's just not what this topic is about.

**Edit 2:** Information hazards can also be shared as unconfirmed transactions in the mempool, even when those transactions never get mined. It's a potential use case to deal with information hazards here too, but I think this is notably easier to deal with, and so slightly less interesting. You can reject or drop or choose not to share a transaction from your mempool much more easily than you can from your blockchain.

-------------------------

