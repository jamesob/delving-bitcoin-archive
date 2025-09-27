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

micah541 | 2025-09-26 17:44:40 UTC | #3

Setting aside the technicalities of including data, I’m a little confused how the long-term storage model is supposed to work.  I run a pruned node, so I don’t keep older blocks, but I understand that if were to go out and ask for, say block 467,000, somebody would provide it to me.  Obviously, I would need to get that from somewhere if I were to sync from genesis.  So unless we want to change the model to some checkpointing thing,  someone out there has to be ready to provide any block on demand, and if that block has something very clearly offensive in it, who is going to be willing to provide that?   Are we just hoping that someone in some untouchable jurisdiction is providing that?

-------------------------

garlonicon | 2025-09-27 10:05:51 UTC | #4

> if were to go out and ask for, say block 467,000, somebody would provide it to me

It is the case today, but it doesn’t have to be in the future. If you ask any node about any transaction or block, that node can always say: “I don’t have it, ask someone else”. And if all nodes are pruned, or if you are connected only with pruned nodes, and none of them can give you what you want, then well, you won’t have access to a given transaction or block in plaintext.

And you can easily confirm it in regtest: just enable pruning, produce a lot of data, and try to connect to your own node on localhost, and ask about past transactions or blocks. They won’t be there, and nobody will give you that data, if nobody saved it anywhere.

And then, you will have a choice: enable pruning, accept given proofs, and move forward, by building things on the same chain, or switch to a different coin.

> Obviously, I would need to get that from somewhere if I were to sync from genesis.

Again, this is true only today. But there is no consensus rule, which would require you to download everything, to perform Initial Blockchain Download. In the future, it can be done in a different way, and if the chain will be too big to handle by many nodes, then they will need other solutions. And then, the only question is, if decentralized ones will be available, or if users will use more centralized ones, like copy-pasting already synced data from their peers, when nothing better will be there.

> and if that block has something very clearly offensive in it, who is going to be willing to provide that?

Only someone, who is Tor node operator, or someone like that. Because if regular users will be sued, then they will just enable pruning, to be safe, and trust their peers.

> Are we just hoping that someone in some untouchable jurisdiction is providing that?

Yes, the current implementation simply assumes, that there will be at least one full archival node, which won’t enable pruning, and which will serve everything to the whole network. But as the chain will grow, and as some users will switch to more centralized solutions, I think it is a good idea, to think about having an option to sync a new node in a more lightweight way, just because other solutions will be even more centralized, so we will need something simpler, to convince users to run any full nodes at all, even in pruning mode, and to let others sync the chain from another pruned peer in a trustless way.

-------------------------

