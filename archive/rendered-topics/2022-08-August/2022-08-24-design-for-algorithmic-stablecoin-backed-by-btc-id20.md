# Design for algorithmic stablecoin backed by BTC

ajtowns | 2022-08-24 12:41:48 UTC | #1

The idea is to have an on-chain market maker that issues two tokens, ***STB*** (a stablecoin) whose value is automatically backed by BTC, and ***VOL*** which captures any excess value from the backing BTC. If successful, this allows a stablecoin to be purely on-chain, without requiring assets to be stored externally, allowing greater auditability and automation. 

The market is parameterized with two variables, $p$ representing the current target value of 1 BTC in STB tokens, and $m$ which gives a slowly changing prediction of the lowest expected future value for $p$. The market maker then aims to ensure that $p$ STB tokens can be traded for 1 BTC, at least provided that $p \ge m$, and that $m$ does not decrease. Further, if those assumptions are violated, failure should be graceful rather than catastrophic.

At any point in time, we will denote the number of BTC held backing the issued STB and VOL tokens as $b$, the number of issued STB tokens as $s$ and the number of issued VOL tokens as $v$. Call the price of 1 STB token $\alpha$ BTC (thus the goal is $\alpha = p^{-1}$), and the price of 1 VOL token $\beta$ BTC. This then gives the budget constraint:

$$
b = \alpha s + \beta v
$$

since selling all the issued STB and VOL tokens at the current price in BTC should give you exactly the value of all the backing BTC.

We define two constraints: the ceiling for $\alpha$ should be $\frac{1}{p}$ and the floor for $\beta$ should be some constant $\bar{\beta}$ and at all times one or both of these conditions should be met. This gives two cases:

$$
\alpha, \beta =
\begin{cases}
p^{-1}, \frac{b-sp^{-1}}{v} & \text{if } b \ge sp^{-1} + v \bar{\beta} \\
\frac{b - v\bar{\beta}}{s}, \bar{\beta} & \text{otherwise}
\end{cases}
$$

When $b \ge sp^{-1} + v\bar{\beta}$ we say the peg holds, and when that is not the case, we say the peg is broken. Note that when the peg is broken, $\alpha \lt p^{-1}$, and hence selling STB will net a payment of less than the target price.

Now that we know the prices at which tokens might be traded, to determine when a trade should be allowed we need to take into account possible changes in the price $p$. In particular, at this point we start to make use of $m$ and set the goal $b \ge sm^{-1} + v\bar{\beta}$.

We setup two scoring rules:

$$
\begin{align}
\gamma_p &= b - sp^{-1} - v\bar{\beta} \\
\gamma_m &= b - sm^{-1} - v\bar{\beta}
\end{align}
$$

Note the following properties:

 * The peg holds precisely when $\gamma_p \ge 0$

 * Provided $m \lt p$, $\gamma_m \lt \gamma_p$, and conversely.

 * When the peg holds:
    * buying VOL with BTC increases $\gamma_p$ and $\gamma_m$
    * trading STB for BTC does not change $\gamma_p$,
    * provided $m \lt p$, buying STB with BTC decreases $\gamma_m$
    
 * When the peg is broken:
     * trading VOL for BTC does not change $\gamma_p$ or $\gamma_m$.
     * buying STB with BTC decreases $\gamma_p$
     * if $\gamma_m \lt 0$, then buying STB with BTC decreases $\gamma_m$

The peg can only be broken directly by external movements in $p$, but such movements are expected.
In order to maintain the peg, we limit trades that may weaken the peg indirectly:

 * We block trades that would cause $\gamma_m$ to become negative (some purchases of STB or sales of VOL while the peg holds)
 * We block trades that would cause $\gamma_p$ to become more negative (purchases of STB while the peg is broken)

This concept is mostly due to discussions with Rusty Russell, and is obviously similar to the Dai stablecoin.

-------------------------

stevenroose | 2023-10-05 15:03:29 UTC | #2

Isn't this what BitMatrix is on Liquid?

-------------------------

ajtowns | 2023-10-06 02:22:38 UTC | #3

BitMatrix is an automated market maker -- so given two assets, it'll let you swap one for the other. But you have to have the assets in the first place for this to work -- so you need a "USDTrent" token that's backed by USD in Trent's bank account, eg. An algorithmic stablecoin lets you establish a USDX token that's backed by BTC instead; with the obvious caveat that if the real value of BTC plummets the token can't maintain par value.

-------------------------

