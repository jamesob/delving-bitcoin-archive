# Limiting OP_RETURN at consensus level

Malachi | 2025-09-24 20:08:52 UTC | #1

Dears, 

TLDR: Would it be acceptable to limit OP_RETURN at consensus level?

Why? Spam and worse can end up onchain, be it via core or any other client in a **contiguous form**. We’ve all seen the discussions. In fact, there have been that many discussions, I’m afraid we’ve woken up some sleeping dogs.

I’m a EU citizen. We see weekly in the news, reporting on CS$M. 
This goes together with the EU parliament forcing through ChatControl on all of us to “combat CS$M”. It’s of course just to get more control.

If some have gotten any notion of what is possible with OP_RETURN and that it remains onchain forever, I can see them abuse it and try to regulatory strangle Bitcoin and the Exchanges.
Companies might not want to touch it anymore, might not even be allowed to touch it anymore.

Just a few weeks ago some reporters “found” CS$M on X, it was all over the news.
I don’t believe for a second they just happened to “find” it.
They enforce platforms to remove what they deem violates law. 

Saying Bitcoin is just a protocol will not stop them from attacking it, hence I feel we should be prepared and mitigate.

I am not purposely trying to sound alarmist, I’ve just seen the EU change so much, we can expect everything. I also believe Bitcoin would survive such an attack, but it would suffer for a while.

Best Regards, 
Malachi

-------------------------

garlonicon | 2025-09-25 06:20:32 UTC | #2

> Would it be acceptable to limit OP_RETURN at consensus level?

No, because that cure would be much worse than the disease. See transaction: [5ffa5dc557d57f5f2dd45879f6dc3ba9bd7682bffe32f3d38e28c3928a0b118e](https://mempool.space/tx/5ffa5dc557d57f5f2dd45879f6dc3ba9bd7682bffe32f3d38e28c3928a0b118e)

Spammers will just wrap their spam in 160-bit hashes, or fake public keys. What then?

> in a contiguous form

Note that people could **spend** for example coins from addresses like [bc1sw50qgdz25j](https://mempool.space/address/bc1sw50qgdz25j) and make a 4 MB push, in a contiguous form. So, do you want to also block all unused Segwit versions, because of that?

> Saying Bitcoin is just a protocol will not stop them from attacking it, hence I feel we should be prepared and mitigate.

Then, just modify your node, to never broadcast these transactions. Because you don’t have to. If you stop relaying any transactions, and just accept new blocks, then it will be compatible with the current protocol, and it is something, that can be done today, by just changing your config file. And if you are worried about historical transactions, then just enable pruning. By combining these two things, you will serve to the world only the latest 288 blocks, or something like that.

And the long-term solution, is to stop broadcasting transactions in plaintext. If you can prove, that a given transaction is valid, without revealing its content, then you don’t have to send it. And if more nodes will accept such proofs, then storing the content of that transaction can be limited only to the users, who have their inputs or outputs involved. For the rest of the world, storing the transaction is never required by consensus. They have to only make sure, that the blockchain is valid, and that new blocks can be safely built on top of it.

-------------------------

