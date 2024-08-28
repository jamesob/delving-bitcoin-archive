# PPLNS with job declaration

Fi3 | 2024-08-28 14:21:15 UTC | #1

I'm sketching up an sv2 extension, that allow miners to verify pool's payouts when miners are allowed to select transaction hence mining jobs with different fees.

The extension is based on a system proposed [here](https://github.com/demand-open-source/pplns-with-job-declaration/blob/bd7258db08e843a5d3732bec225644eda6923e48/pplns-with-job-declaration.pdf) this system is build on top of PPLNS.

You can find the extension spec [here](https://github.com/demand-open-source/share-accounting-ext/blob/281c1cbc4f9a07b21a443753a525197dc5d8e18c/extension.md)

[Here](https://github.com/demand-open-source/share-accounting-ext/tree/master/src) instead I implement the extension messages using SRI libraries.

[Here](https://github.com/demand-open-source/demand-cli) I started implementing a miner translator proxy that use the extension.

Everything above is still work in progress and need reviews.

-------------------------

