# Implemented Post-Quantum Cryptography (PQC) Feature into Bitcoin Core!

QbitsCode | 2024-12-13 10:57:34 UTC | #1

Greetings Bitcoin Developers,

We're glad to share with you a proposed feature 'Post-Quantum Cryptography' to permanently eliminate the future quantum advancement threats forever!

The acceptance of this issue requires Bitcoin Developers consensus as well as BIP.

Implemented Post-Quantum Cryptography (PQC) Feature into Bitcoin Core!

Highlights:

1. Core PQC algorithms (Kyber, FrodoKEM, NTRU) integrated.
2. Hybrid key management system in place.
3. PQC transaction signing enabled.
4. Flexible configuration options.

GitHub Repository:
https://github.com/QBlockQ/pqc-bitcoin

It is worth highlighting that progression of quantum computers is a problem and the latest one was yesterday when the Williow chip has been announced, which has the largest semiprime can factored!.  We believe implementing this proposed pqc feature as outlined in the attached repo and illustrated with the  integrated 3 pqc algorithms will withstand of any future quantum computers based on our expertise.

Thank you for your time to read our proposal and for your valuable understanding and feedback.

-------------------------

cryptoquick | 2024-12-13 15:28:09 UTC | #2

Great work so far. One question -- How are the new signature algorithms expected to be used? Will you use a new address type? If so, do you think you would be able to meet the P2QRH BIP? It seems you've selected the same algorithms the BIP specifies, which is good, but there are still some details that are unclear.

-------------------------

QbitsCode | 2024-12-14 13:17:30 UTC | #3

Thanks crypotquick.  What we have proposed Group 2: Kyber, FrodoKEM, and NTRU are not for digital signature is a complementary of Group 1: SPHINCS+, CRYSTALS-Dilithium, FALCON, and SQIsign 'what BIP's P2QRH proposed protection'.  Both groups are pqc, however, what is adding value in Group 2 is strong in terms of if Bitcoin evolves to require encrypted communications (e.g., between wallets or nodes).  Group 1 is exactly designed for Digital Signature Algorithms.  So, they are complement each other.  We are so glad to see the cryptography of bitcoin is quantum-safe and quantum-resistance in a permanent setup.

-------------------------

QbitsCode | 2024-12-14 17:51:00 UTC | #4

As follow up, we have just pushed to the repo adding both Groups 1 & 2 and updated PQC manager and added the suited tests.  All post-quantum algorithms are checked found ok except only one which is 'SQIsign' where we need to discuss (either to keep it or remove it) since it has not yet been standardized by NIST.

-------------------------

