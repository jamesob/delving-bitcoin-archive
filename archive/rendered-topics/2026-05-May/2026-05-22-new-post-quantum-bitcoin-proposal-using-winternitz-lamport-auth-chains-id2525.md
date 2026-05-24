# New Post Quantum Bitcoin Proposal using WInternitz + Lamport auth chains

opus-lux | 2026-05-22 01:04:56 UTC | #1

Hello bitcoin community!

I am a cryptographic researcher who has been working on creating the lightest weight possible post quantum signature scheme for bitcoin. This is not just an idea but a working implementation that is currently live on 16 different EVM chains. The website URL is block_opuslux.ar.io, there you can read the security essay that explains in detail how signing with Winternitz + Lamport works along with my bitcoin proposal outline. The project along with the proposals will be made open source very soon. The contracts will be verified on chain within 24 hours.

Now that we have that out of the way let me break down why I think that Lamport + Winternitz is the right solution for creating a post quantum bitcoin. Nearly every post quantum signature that exists today uses Winternitz at its core. The problem with using winternitz signatures is that they are inherently one time signatures. This creates a problem, if you want to use Winternitz as your post quantum signing method you must build a system to make sure that the keys are one time only! This is exactly what I did using a Lamport authorization chain and some cryptography that starts from your seed phrase. Each link in the Lamport authorization chain authorizes the upload of one Winternitz public key. Then the Winternitz public key can be used to verify the winternitz signature for the message. Since Winternitz is one of the smallest possible quantum signatures it is extremely lightweight meaning that it can be added easily through Taproot. Also adjusting the w parameter in the winternitz equation to 256 instead of 16 means that the signatures are smaller and more lightweight than anything else I have seen. I would love to get some feedback on this idea and make my bitcoin proposal public. You can reach me at opusluxofficial@proton.me or opus-lux on github! Thank you for taking the time to consider this option! 

Best, Opus Lux !

-------------------------

murch | 2026-05-23 16:09:30 UTC | #2

Hi opus-lux,
thanks for sharing your idea. It sounds potentially interesting, but I don't understand your proposal well-enough, yet. Is there a more comprehensive description somewhere?

[quote="opus-lux, post:1, topic:2525"]
Each link in the Lamport authorization chain authorizes the upload of one Winternitz public key. Then the Winternitz public key can be used to verify the winternitz signature for the message.

[/quote]

I'm not sure I fully understand this part. Would output scripts based on this construction commit to exactly one Winternitz public key, or would they commit to the whole authorization chain? The former would be an issue, as we need to be able to produce more than one signature for an output script, e.g., when our first transaction attempt does not succeed, we want to be able to make a higher feerate transaction, or when we create transactions with inputs from multiple users and a participant defects.

-------------------------

opus-lux | 2026-05-23 21:03:26 UTC | #3

HI Murch,

Thanks for taking time to read my idea and respond. To answer your question regarding the authorization chain it works like this. At wallet setup, you build a Lamport chain which is a sequence of hashes rooted in your master secret. Imaginine hashing your secret to get value #1, then hashing the result of that to get value #2, and so on until you hit 50,000. You keep all of the values private except for value number 50,000 which you publish on chain. Now to verify your Winternitz public key upload you reveal value 49,999, the blockchain checks does value 49,999 hash to get 50,000. If it does then your Winternitz public key upload is authenticated. The Winternitz public key is only valid to sign one message, then its retired forever.

Also, yes there is a detailed security essay along with a bitcoin proposal outline that explain exactly how this works and fits into the existing Tapscript framework.

Security essay:  block_opuslux.ar.io/#/essay

Bitcoin outline: block_opuslux.ar.io/#/bitcoin-proposal

Please let me know if you have any other questions or want to know more! I am happy to explain this in as much detail as you want. Thanks again for taking the time to read my ideas and respond!

-------------------------

murch | 2026-05-23 22:31:11 UTC | #4

If I understand it right, there are multiple issues that I would consider problems for Bitcoin adoption:
- It sounds like it would require every node would need to keep track of all Lamport chains and all nullifiers. This would imply linear growth of the minimal state stored by every full node in the number of payments.
- The "Tapscript Leaf Structure" seems to commit to exactly one Winternitz public key. This would mean that for every UTXO we could only make a single attempt to spend it. We would be incapable of bumping fees, and would generally become unable to participate in multi-user transactions as any other participant could force us to sign more than once.
- It seems to me that this scheme reveals separate UTXOs as being owned by the same recipient upon spending. That would be a significant departure from the usual privacy expectations in Bitcoin --- address reuse is generally considered malpractice on Bitcoin.

-------------------------

