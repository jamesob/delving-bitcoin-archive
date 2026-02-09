# BIP 119 CTV activation client

1440000bytes | 2026-02-09 20:27:15 UTC | #1

Since there were no objections to the activation parameters shared in the [last email][0], I have updated the [website][1] with the repository link for activation client and added some FAQs.

Repository: https://github.com/ctv-activation/activation-client

```
// Deployment of CTV (BIP-119)
consensus.vDeployments[Consensus::DEPLOYMENT_CTV].bit = 5;
consensus.vDeployments[Consensus::DEPLOYMENT_CTV].nStartTime = 1774809000; // 30 March 2026
consensus.vDeployments[Consensus::DEPLOYMENT_CTV].nTimeout = 1806345000; // 30 March 2027
consensus.vDeployments[Consensus::DEPLOYMENT_CTV].min_activation_height = 1001952; // May 2027
consensus.vDeployments[Consensus::DEPLOYMENT_CTV].threshold = 1815; // 90%
consensus.vDeployments[Consensus::DEPLOYMENT_CTV].period = 2016;
```

Cc: @simul @JeremyRubin 

[0]: https://groups.google.com/g/bitcoindev/c/HC2bn4QOR-M/m/TF8qJidzAAAJ
[1]: https://ctv-activation.github.io/

<details>
<summary>Archive</summary>

https://archive.is/o6QgT

</details>

-------------------------

