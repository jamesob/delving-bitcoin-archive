# Guardian Address + GASPv1

guardian | 2025-09-22 15:47:18 UTC | #1

I’ve already submitted this as 2 \* draft PR BIPs and announced it on the mailing list (though I believe it’s in moderation). I’m posting here to open up the discussion to a wider audience and get more detailed technical feedback.

\*\*Overview\*\*

We propose two Bitcoin Improvement Proposals to address the growing threat of theft against bitcoin users: Standards Track BIPs defining the Guardian Address Signal Protocol and wallet implementation. Both share the same abstract and motivation due to their complementary roles in a novel security mechanism. We seek community feedback to refine these drafts before submission.

\*\*Abstract\*\*

This proposal introduces the concept of a Guardian Address and defines a standard signalling mechanism that allows bitcoin wallets to become locked in response to an activation event. A single external control address triggers a security lockdown across one or more unrelated wallets without requiring any on-chain linkage between them. The goal is to prevent theft of bitcoin by enabling users to broadcast a standardized on-chain lock that causes cooperating wallets to enter a restricted mode, disabling the ability to spend UTXOs under duress.

The design allows a separation of key material between the user's spending wallet and a Guardian Address; a discrete identity that signals lock state changes via a transaction embedding data in an \`OP_RETURN\`. This enables emergency responders, user level software, and wallet applications to recognize a distress signal without exposing user spending address(es) or balances.

Adoption requires minimal overhead for wallet developers. This approach does not alter spending rules. It is a voluntary signalling protocol that requires adoption by wallet and custodial software to be effective. BIP compliant wallets will be able to offer this security mechanism without compromising privacy or usability. These standards are intended to be optional and without breaking compatibility for existing wallets or nodes.

\*\*Motivation\*\*

Bitcoin users are increasingly the targets of physical threats including robbery and coercion\[^1\]. A non-exhaustive list is maintained with details of physical attacks on bitcoin users\[^2\], which provides some insight into the prevalence and severity of attacks. Notably the incidence of attacks is also increasing. Security controls have been implemented in some self-hosted wallets as a means to prevent theft of bitcoin. One such is a decoy wallet, which presents a wallet with a smaller balance of bitcoin when a duress PIN is entered. However, this comes with two significant downsides:

\- An assumption is made that the attacker does not know about or understand the purpose of a decoy wallet. If a sophisticated attacker is able to link an address to the real world identity of the user, they may already know the true balance of the bitcoin holder. If the attacker does not know the balance of the user they are attacking, they may still suspect the user has unlocked a decoy wallet given the lower than anticipated balance.

\- In the case that the attacker does not know the wallet opened is a decoy wallet, the attack still results in the loss of bitcoin for the user.

Current self-custody solutions do not provide a safe way to respond under physical duress without risking loss of funds. In addition, participants in the Bitcoin ecosystem commonly use both self-hosted and centralized services\[^3\]. There's no mechanism that currently exists that can act as a self-sovereign "kill switch" for both user scenarios of a self-hosted wallet or a user with a self-hosted and centralized wallet.

This proposal introduces an interoperable mechanism to:

\- Allow users to trigger a wallet lockdown using a separate device or operation.

\- Preserve privacy by decoupling the Guardian Address from wallet addresses.

\- Enable wallet software to observe chain state and mempool to react defensively.

\- Protect a multiple wallet user (e.g., self-custodial wallet, exchange account, institutional wallet, custodian) with a single on-chain emergency trigger.

\- Allow businesses or multi-user custodial setups designating a Guardian Address to coordinate responses and align with risk management frameworks.

\*\*BIP Structure and Request for Feedback\*\*

BIP A: Guardian Address Signal Protocol (Standards Track, Applications Layer): Defines a lightweight (\~141 vBytes) OP_RETURN based protocol for \`Lock\`/\`Unlock\` signals using a Guardian Address, with nonce based replay protection. It requires no consensus changes and preserves privacy. [https://github.com/bitcoin/bips/pull/1973](https://github.com/bitcoin/bips/pull/1973)

BIP B: Guardian Address Wallet Implementation (Standards Track, Applications Layer): Defines integration for wallet developers, including state machines, polling mechanisms, and attack scenarios to ensure adoption across both self-custodial and custodial wallets. [https://github.com/bitcoin/bips/pull/1972](https://github.com/bitcoin/bips/pull/1972)

Both BIPs share this abstract and motivation to align with the goal of protecting users from theft. The split keeps BIPs concise and ensures focus. BIP A standardizes the protocol while BIP B standardizes implementation. We welcome feedback on the approach, split rationale, and technical details. We will address comments before formal submission.

Wallet implementation: [https://github.com/bitcoinguardian/electrum](https://github.com/bitcoinguardian/electrum)

Guardian CLI tool: [https://github.com/bitcoinguardian/GASPv1-draft](https://github.com/bitcoinguardian/GASPv1-draft)

Demo wallet screenshots with testnet transactions: [https://github.com/bitcoinguardian/electrum/tree/master/demo](https://github.com/bitcoinguardian/electrum/tree/master/demo)

\*\*References\*\*

\[^1\]: Investigating Wrench Attacks, DOI: 10.4230/LIPIcs.AFT.2024.24

\[^2\]: [https://github.com/jlopp/physical-bitcoin-attacks](https://github.com/jlopp/physical-bitcoin-attacks)

\[^3\]: [https://river.com/learn/how-many-people-use-bitcoin/](https://river.com/learn/how-many-people-use-bitcoin/)

-------------------------

