# zkBitcoin: zero-knowledge proof apps on Bitcoin

mimoo | 2024-01-23 02:47:33 UTC | #1

Hello hello.

I guess this is my first post, just wanted to let you know that Ivan and I have been working for a few weeks on a minimal L2 that would allow people to build zkapps (zero-knowledge applications) on Bitcoin. It does not verify zero-knowledge proofs directly on Bitcoin (unless we have new opcodes it's just not possible). Instead it relies on a multi-party computation between nodes to manage a wallet, which tracks on-chain zkapps and their state (in the case they are stateful). The nice thing about the design is that MPC nodes don't need to know about the canonical chain, so they're really light to run, and could even easily run in trusted execution environment to provide some more defense-in-depth. We're looking for well-known ZK individuals to run nodes in order to maintain this service and have a large threshold parameter (the larger, the more nodes an attacker would need to compromise to compromise the L2). For now we're just running on testnet. As far as I know something like this didn't exist until today, so now you can use protocols like zkLogin or zkMail to lock your Bitcoin in zkapp and unlock them with a proof of login on Google or a proof that you received some email ![Smiley](upload://cWbugZPJ3QDP63GwL2a9hWZe4HR.gif)

You can check the CLI here: https://github.com/sigma0-xyz/zkbitcoin

-------------------------

22388o | 2024-01-23 10:52:22 UTC | #2

Good idea.

What's difference of this ZK and ZeroSync? :thinking:

-------------------------

