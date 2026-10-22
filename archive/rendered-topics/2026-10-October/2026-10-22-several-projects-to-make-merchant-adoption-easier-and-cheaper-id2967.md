# Several projects to make merchant adoption easier and cheaper

andrewclicks | 2026-10-22 01:00:04 UTC | #1

I would like to share with you several projects I have been working on over the past 2 years to make it easier to onboard merchants, particularly to lightning. Everybody knows there is a lot of rough UX around lightning payments that make it painful for everyday merchants to adopt, which causes centralization around large custodial providers. So, end to end, I have been working to chip away at these UX issues.

**Project 1: Electrum CLINK plugin**

You have a customer who wants to make a payment but you don't want to have to open ports on your router to expose your wallet machine to the world for incoming connections (for LNURL). You also don't want to have to manually make an invoice each time. CLINK solves this problem by routing payment requests via nostr relays. Now you can receive unsolicited payments to your Electrum wallet. https://github.com/BareBits/electrum_clink

**Project 2: Electrum liquidity manager plugin**
Everybody knows obtaining inbound liquidity in lightning can be difficult and that there is a lot of centralization around liquidity providers. Electrum has a built-in decentralized mechanism to do this via submarine swaps, but actually using it is painful because Electrum's routing isn't great (many failed payments) and random swap providers are not particularly reliable. This plugin automates the process so you always have inbound liquidity. It goes through every possible swap provider until a swap is sucessful, often taking dozens or hundreds of attempts. Imagine doing that manually!

https://github.com/BareBits/electrum_liquidity

**Project 3: Electrum Swap GUI**
Electrum comes with a build in swap server so you can offer swaps to other users, but it requires using the CLI which puts if out of reach for most users. This plugin adds a GUI so you can control Electrum's swapserver without using the command line. This also enables you to earn a small yield on your BTC. https://github.com/nothing-stops-this-train/electrum_swapgui

**Project 4: BareBits merchant software**

This is where is all comes together. Merchants are typically suggested to use BTCPay Server for self-custody, but this requires a VPS which is a significant expense and even with a pruned node you are looking at $10-$25 USD per month. Self-custody shouldn't require this much overhead. BareBits does much of what BTCPay server does but it runs on any simple web hosting and can even be one-click installed onto a Windows desktop.

For lightning payments, it connects to your wallet via CLINK, NWC, or LNURL, so you can have payments from your customers go directly to Electrum. It's also fully BTCPay compatible, so that means any plugin or software which is designed to talk to BTCPay can also talk to BareBits, just use a different url and it won't be any wiser. This means BTCPay's extensive integration list (woocommerce, magneto, etc) can all work natively with BareBits. This software does come with a 1% fee, but it's fully open source, private, and self-hosted. And if you don't want to fuss with inbound liquidity management, instead of or in addition to your Electrum or other wallet, you can also use a cashu mint which is periodically emptied.

https://github.com/BareBits/cashupayserver

-------------------------

Anzus_GemWallet | 2026-10-22 12:45:02 UTC | #2

Thanks for sharing this. Merchant setup is where a lot of people give up on Lightning. One question from the customer side: if the merchant’s Electrum wallet happens to be offline when someone tries to pay through CLINK, what does the customer see? Does the payment just fail, or can it fall back to an on-chain address or the cashu mint automatically?

-------------------------

