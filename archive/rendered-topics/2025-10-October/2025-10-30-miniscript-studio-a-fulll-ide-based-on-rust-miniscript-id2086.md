# Miniscript Studio - a fulll IDE based on Rust Miniscript

adys | 2025-10-31 00:31:15 UTC | #1

I built an online **Miniscript Studio**, a full IDE powered by the Rust Miniscript crate, with a strong emphasis on learning.

[https://adys.dev/miniscript](https://adys.dev/miniscript)

Started it at BTC++ hackathon just for better error handling on my Miniscript expressions, but it snowballed into packing in all the features I ever dreamed of:

* Policy & Miniscript editors: indenting, clearing, syntax highlighting, and more;
* Script area: resulting scripts/addresses across contexts (Legacy/SegWit/Taproot), testnet/mainnet support;
* Key Variables: auto-generate from hex, or map keys to custom names;
* HD Wallet Descriptors: xpubs/tpubs, range descriptors;
* Lift Functionality: from raw Script to Miniscript, and from Miniscript to Policy;
* Taproot Support: x-only keys, multi-branch policies, script/key paths;
* Spending cost analysis (worst case);
* Compile debug info display;
* Packed with examples & detailed descriptions.

Plus a settings section to customize your experience; Policy/Miniscript references; save/load/share expressions; tips section and more.

Full details in my blog post: [https://adys.dev/blog/miniscript-studio-intro](https://adys.dev/blog/miniscript-studio-intro)

Here is a Miniscript expression from Liana wallet:
![|585x500, 100%](upload://JNMAanoN2fjSNPnSnZt4mytomk.png)

Try it in Studio:
[https://adys.dev/miniscript#example=miniscript-liana_wallet](https://adys.dev/miniscript#example=miniscript-liana_wallet)

Feedback is much appreciated :folded_hands:

-------------------------

bruno | 2025-10-31 11:54:23 UTC | #2

Super helpful, nice project. Is it open-source?

-------------------------

adys | 2025-10-31 12:14:03 UTC | #3

Thanks! 

Sure, it has a link in the upper right corner of the page.
Here’s the repo:

[https://github.com/adyshimony/miniscript-studio](https://github.com/adyshimony/miniscript-studio)

-------------------------

