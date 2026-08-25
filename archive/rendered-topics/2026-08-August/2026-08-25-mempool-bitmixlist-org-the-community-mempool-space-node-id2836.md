# Mempool.bitmixlist.org - the community mempool.space node

Zenul_Abidin | 2026-08-25 16:34:48 UTC | #1

[![](upload://oV8Z6a9BODS9dphEFqDccYc72Ss.png)](https://mempool.bitmixlist.org)

I'm now running a production mirror of mempool.space.

This particular mirror is based on v3.3.1 \[9332d9db\], and only branding changes have been made. Otherwise, it's identical to the main site.

It is based on [mempool/electrs](https://github.com/mempool/electrs). This node has:

* 8x32 GB of DDR4 ECC RAM
* 2x Intel(R) Xeon(R) Gold 6262V  <-- This particular processor slightly lowers my memory bandwidth but it is what it is
* 2x 7.68TB NVMe SSD running xfs on an LVM configuration (got these for a huge discount!)
* 10 Gbps public Ethernet
* Layer 3/4 DDoS protection
* Separate Bitcoin node + LND (no channels)

Use it now: [mempool.bitmixlist.org](https://mempool.bitmixlist.org)

Or use the API: <https://mempool.bitmixlist.org/docs/api/rest>

Tor version: http://mempool6xjj67yzf55s2mdhhhuabxqfejfm5eh7ikxuxtmef2o5hkdyd.onion/

I2P: http://mempoolhsdlztlgq5h3phk7ifovypsxsw2puly2mkqg6ulowlypa.b32.i2p/

You can also use the Electrum server powered by this mempool at [electrum.bitmixlist.org:50002:s](http://electrum.bitmixlist.org).

-------------------------

