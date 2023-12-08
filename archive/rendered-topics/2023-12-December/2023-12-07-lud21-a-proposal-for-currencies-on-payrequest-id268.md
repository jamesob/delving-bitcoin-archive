# LUD21: A proposal for currencies on payRequest

lorenzolfm | 2023-12-07 15:24:46 UTC | #1

Hi all,

I'm working on a proposal to add a currency list to the payRequest LNURL. This allows some cool features such as:

- Allowing users to create invoices denominating amounts in other currencies based on the exchange rate of target wallets;
- Allowing receivers to signal intent to convert the received BTC into another currency upon payment
- Allowing senders services to inform of exchange rates if another currency besides btc is used on the sender side.

In the most general use case, we can have a remittance-like experience powered by Lightning and LNUrl.

Enabling wallets to create invoices based on the exchange rate of the target wallet is also very useful.

I would appreciate feedback on the [PR](https://github.com/lnurl/luds/pull/251).

We're working on a implementation at Bipa (see bipa.app/.well-known/lnurlp/lorenzolfm). If any other service/wallet is interested in enabling support please reach out.

-------------------------

