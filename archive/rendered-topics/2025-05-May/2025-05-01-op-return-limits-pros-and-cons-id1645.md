# OP_RETURN limits: Pros and Cons

1440000bytes | 2025-05-01 09:21:48 UTC | #1

What are OP_RETURN limits?

- 83 bytes per output 
- Only one output per transaction


<table border="1" cellpadding="5">
  <tr>
    <th></th>
    <th>With limits</th>
    <th>Without limits</th>
  </tr>
  <tr>
    <td>UTXO set</td>
    <td>⚠️</td>
    <td>✅</td>
  </tr>
  <tr>
    <td>Fee estimates</td>
    <td>⚠️</td>
    <td>✅</td>
  </tr>
  <tr>
    <td>Compact block relay</td>
    <td>❓</td>
    <td>✅</td>
  </tr>
</table>

Since OP_RETURN limits exist, some users prefer other ways of storing data. Data is mostly required by protocols that use bitcoin. This affects UTXO set and the full nodes resource usage. If users directly submit transactions to mining pools without propagation they wont exist in other mempools hence fee estimates and compact block relay might be affected.

![image|690x360](upload://sLookVyWwjJqK1AfYrcr6Yp8AdN.jpeg)


Alternatives for users who support limits: 

1. Core w/ patch
2. Older versions that include bug fixes
3. Other implementations

Alternatives for users who want to remove limits:

1. Libre relay
2. Slipstream

Related mailing list thread: https://groups.google.com/g/bitcoindev/c/d6ZO7gXGYbQ

Note: This post is inspired by a Github [comment](https://github.com/bitcoin/bitcoin/pull/32381#issuecomment-2840165283). I want to keep this post technical (not political) and I will add more relevant information based on my research, other posts, reviews etc.

-------------------------

1440000bytes | 2025-05-01 09:22:15 UTC | #2

https://bitcoin.stackexchange.com/questions/122321/when-is-op-return-cheaper-than-op-false-op-if

-------------------------

1440000bytes | 2025-05-01 09:35:53 UTC | #3

A solution proposed by Warren Togami: 

> Now I propose adding an additional expectation of fee for each output where the value is "small". For example a shitcoin in 2011 added 1,000 penalty bytes to the fee expected by MempoolAccept for each output that was too small. I always thought that was a good idea that would benefit Bitcoin because it would make any UTXO set expansion extra costly. I first proposed this for Bitcoin ~2014 because I was very angry about SatoshiDice spam. I still think charging for too-small outputs would have been better than the dust limit.

Link: https://x.com/wtogami/status/1917289539703280006

-------------------------

1440000bytes | 2025-05-12 19:16:11 UTC | #4

OP_RETURN and Coinjoin: https://uncensoredtech.substack.com/p/op_return-and-coinjoin

-------------------------

