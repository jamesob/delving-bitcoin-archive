# Inscriptionless Inscriptions

1440000bytes | 2024-04-11 05:13:11 UTC | #1

<h1>Problem</h1>

Some users and developers do not like image, text etc. stored in witness by inscriptions. Approximately 20 GB of data has been stored on chain by inscriptions until now. Bitcoin is permissionless and users should be free to inscribe different things if they pay transactions fees. However, there could be efficient ways to have fun in which data doesn't get lost because of broken links.

<h1>Research</h1>

I think bitcoin transactions already have everything you need to [decode them to some image](https://x.com/1440000bytes/status/1732580146203250731) or inscription. There is no need to add anything extra in witness data.

![UtsGcWt|690x460, 75%](upload://tLFNJXivmvuaLo2KYhzwMRhJwKY.png)

<h1>Solution</h1>

Murch has explained nLockTime in a [stackexchange Q&A](https://bitcoin.stackexchange.com/questions/23792/can-someone-explain-nlocktime/) and I had used it to build a proof of concept that generates different images using nLockTime. Images are not stored anywhere but generated off-chain based on nLockTime. In this case its just a pixelated cube but can be anything.

Initial idea was limited to ownership and gifting. It was good for privacy, could not be censored by any filters and can swim in ocean as well. These transactions would look same as any other bitcoin transactions. In fact all bitcoin transactions could be interpreted or decoded to some image.

[Johannes](https://twitter.com/HausHoppe) has [implemented](https://github.com/ordpool-space/cat-21) the idea discussed in a [tweet thread](https://x.com/1440000bytes/status/1741773728545927177) which uses a similar approach. Image is generated based on the transaction ID of the mint transaction, creating a unique pixelated cat image. A transaction is recognized as a CAT-21 mint if its [`nLockTime`](https://en.bitcoin.it/wiki/NLockTime) value is set to `21`. The seed for generating the image and traits is retrieved from the concatenated `transactionId` and `blockId` , hashed using the SHA-256 algorithm. By using this method, the characteristics of each CAT-21 ordinal can only be determined after the transaction is included in a block, thus preventing anyone from generating transactions until they get a desirable outcome. This approach ensures fairness and unpredictability in the distribution of rare traits. Images and Traits are not stored on the blockchain but are generated on-demand using the concatenated `transactionId` and `blockId` as a seed.

---

**Disclaimer:** CAT-21 is just an example for the solution suggested in this post and should not be considered an endorsement. Cats were minted for [free](https://x.com/HausHoppe/status/1777580471116771833) and I have no involvement with the project apart from one of my tweets being an inspiration.

-------------------------

