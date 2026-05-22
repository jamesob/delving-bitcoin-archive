# New Post Quantum Bitcoin Proposal using WInternitz + Lamport auth chains

opus-lux | 2026-05-22 01:04:56 UTC | #1

Hello bitcoin community!

I am a cryptographic researcher who has been working on creating the lightest weight possible post quantum signature scheme for bitcoin. This is not just an idea but a working implementation that is currently live on 16 different EVM chains. The website URL is block_opuslux.ar.io, there you can read the security essay that explains in detail how signing with Winternitz + Lamport works along with my bitcoin proposal outline. The project along with the proposals will be made open source very soon. The contracts will be verified on chain within 24 hours.

Now that we have that out of the way let me break down why I think that Lamport + Winternitz is the right solution for creating a post quantum bitcoin. Nearly every post quantum signature that exists today uses Winternitz at its core. The problem with using winternitz signatures is that they are inherently one time signatures. This creates a problem, if you want to use Winternitz as your post quantum signing method you must build a system to make sure that the keys are one time only! This is exactly what I did using a Lamport authorization chain and some cryptography that starts from your seed phrase. Each link in the Lamport authorization chain authorizes the upload of one Winternitz public key. Then the Winternitz public key can be used to verify the winternitz signature for the message. Since Winternitz is one of the smallest possible quantum signatures it is extremely lightweight meaning that it can be added easily through Taproot. Also adjusting the w parameter in the winternitz equation to 256 instead of 16 means that the signatures are smaller and more lightweight than anything else I have seen. I would love to get some feedback on this idea and make my bitcoin proposal public. You can reach me at opusluxofficial@proton.me or opus-lux on github! Thank you for taking the time to consider this option! 

Best, Opus Lux !

-------------------------

