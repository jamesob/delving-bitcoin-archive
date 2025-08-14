# [Proposal] Bitcoin Deposits: A Zero UTXO Trust-Minimized Lightning Wallet

ynniv | 2025-08-14 13:57:42 UTC | #1

Hey all! I’ve been working on an idea for scaling Lightning and it’s time to see whether it holds water. It’s more complex than I’d like, but it seems to solve tough problems that don’t currently have solutions. I’ll start with the motivation, then work through the protocol one layer at a time using a process of trust-reduction.

It’s not perfect. What I’m looking for is feedback on what I might be missing, how it could be made more durable, and whether it’s something that anyone would use. It’s a big project, and though I’m the kind of guy that wants to wait until there is running code, this one is bigger than any one person (even a vibe captain).

* @ynniv

## Bitcoin Deposits: A Layer 3 Protocol for Zero UTXO Trust-Minimized Lightning Wallets

The goals of this project are to provide wallets with:

* private key control
* Lightning payments
* no UTXO fees
* no availability requirements
* no liquidity management
* the expectation that funds will be available indefinitely

And operators with:

* a large addressable market
* LSP-level fee structures
* maintenance fees that mitigate inactivity
* the ability to operate without a public IP

This is accomplished by:

* building on Lightning channel consensus
* never providing unilateral exits
* ensuring that funds stay available in the protocol
* creating a network of operators that cross validate deposit updates
* communicating through the Lightning Network and Nostr Wallet Connect

The vision is a future with the existing 10,000 Lightning nodes supporting the next billion Bitcoiners.

## Two Honest Nodes

We’ll start with the assumption that there is a channel between two Deposits enabled nodes and that either one of them is conforming (“honest”). There is also a conforming independent auditor, Victor. These are not acceptable loopholes, but we will attempt to close them later.

### Vault

Mallory requests that Charlie add a new Validated Output to their shared channel, and that it use the BitcoinDepositsValidator. Charlie confirms, and now they have a channel with 5 ouputs (to_local, to_remote, local_anchor, remote_anchor, local_validated_output). Mallory doesn’t have existing Deposit relationships, so she requests that the well regarded, independent auditor Victor be the sole recovery on the vault, and that advertised maintenance fees be set very low.

```
channel {
  to_local      { ... amount:  50,000 sats, destination: Mallory }
  to_remote     { ... amount: 500,000 sats, destination: Charlie }

  local_validated_output {
    validator: bitcoin_deposits_validator_v1
    amount: 660 sats (nominal non-zero amount)
    address: MuSig2.KeyAgg(
      derive_key(mallory_key, 'mallory_validated_output_0'),
      derive_key(charlie_key, 'mallory_validated_output_0'))
    destination: Victor
    annualized_fee_rate: 0.1%
    annualized_fee_unit: 100 sats
    data_hash: <hash_0>
  }
}
```

### Wallet

Alice wants to buy something from Bob, who has never used Bitcoin but is curious. He decides that Alice can pay him 30,000 Bitcoin instead of using fiat. Alice recommends he try Victor’s Bitcoin PWA. Bob visits the PWA website and is presented with his new seed backup and a slider between low-fees and high-reputation. He dutifully writes down his seed backup and slides the slider to “high-reputation”. In what can only be described as a blatant plot mechanic, the only available vault is Mallory’s super fresh, low fee, YOLO-af vault. Bob takes this decision between Mallory and Mallory very seriously, and in the end chooses Mallory.

Mallory runs her node at home, as many nodes likely will be, and Bob is unable to directly connect to her address. The PWA Bob is using derives a new public key and sends a Nostr message addressed to Mallory to a few websocket relays that Mallory is said to listen to. Mallory’s node sees this request for a new wallet and sends Charlie a request to add an empty wallet to the Deposits DB. Both Mallory and Charlie maintain a local copy of this database, but only Mallory can propose changes. Charlie can only accept them, reject them, or request a database re-sync.

```
channel {
  to_local      { ... amount:  49,000 sats, destination: Mallory }
  to_remote     { ... amount: 500,000 sats, destination: Charlie }

  local_validated_output_0 {
    amount: 1000 sats
    data_hash: <hash_1>
    ...
  }
}

depositsdb {
  bob_0: {
    pubkey: <pub_bob_0>
    amount: 0
    annualized_fee_rate: 0.1%
    annualized_fee_unit: 100 sats
    maintenance_interval: 4320 blocks
    last_maintenance_fee: <current_block_number>
  }
}
```

Mallory additionally sends both this and the previous (vault opening) updates to Victor, who also validates that they conform to the protocol.

Having successfully created a new deposit, Mallory associates a Nostr Wallet Connect string with the deposit and returns it to Bob. It isn’t necessary to put this string into the channel db because it’s only used for ephemeral communication, and at any point Bob can make a new connection and prove control of the deposit by signing the connection string with the deposit’s private key.

### Payment

Bob requests an invoice for 30,000 Bitcoin that Alice can pay. He sends an NWC make_invoice to Mallory. She creates an invoice for 30,000, but before Charlie will co-sign it she needs to move funds into the vault so that the total balance is greater than 120% of all deposits (currently zero) plus the amount of the invoice. She moves 29,000 in and then asks Charlie to sign an invoice with Bob’s deposit in the metadata. Charlie confirms that the reserves are ≥ (0 \* 120% + invoice) and then uses MuSig2 to sign the invoice. Mallory returns this in the response to make_invoice.

Bob confirms that the invoice contains his pubkey in the metadata as well as signed by the deposit’s vault pubkey, and he presents it to Alice for payment. Alice pays it, and Mallory just pockets it. Bob’s wallet still has a zero balance, and Alice’s wallet is down 30,000.

### Mallory

Having received Bob’s funds in one of her other channels, Mallory initiates a cooperative close on her channel with Charlie. This requires her to first remove the Validated Output, which can only be done with a zero balance. We’re still in the timeout period for the invoice, and Charlie won’t let her remove the funds so instead she force closes the channel, sending her 30,000 to Victor.

Victor had been notified of the invoice, and because he is honest he also won’t release the funds back to Mallory until the waiting period has ended. There aren’t any deposits, so he’s done for now.

### Recovery

Bob is new to this and confused. Alice says that she paid him, but he didn’t receive the funds, and now Mallory’s node is gone. Alice tells him this can happen, and shows a QR code with the invoice she paid and the corresponding pre-image. Bob scans it and taps a “report” button that sends this to the channel partner and recovery party from his deposit, Charlie and Victor. Charlie doesn’t have a role in this vault anymore so he ignores it. Victor can see that there was a paid invoice created by the vault he received, but there’s no record of the completed HTLC in his copy of the ledger. This proves that Mallory was dishonest, and he asks Bob for an invoice that he can remit funds to.

## Chad

Bob returns to the start, to once again choose between cheap and reputable. The gods smile on him this time, and there are plenty of choices. One in particular stands out: it’s been serving close to a hundred deposits for almost a year, and has five other vaults on channels with reputable partners. Bob asks Chad for a wallet. Were they smiling or laughing?

Bob receives his wallet, makes an invoice, sends it to Victor, and receives his funds. If nothing else, opening a wallet and moving funds is cheap and fast without UTXOs. He thanks Alice for the sale and forgets about the wallet.

On the Internet no one knows that your Mallory, so perhaps Bob will be unsurprised to learn that his 30,000 now sits in a deposit on a channel between Chad and Mallory. If it seems implausible that she has a reputable, well connected node in addition to her fast and loose, flaming YOLO apocalypse, let’s just blame the writers. Mallory is finally going to get that new laptop by colluding with Chad to steal one of his vaults and splitting it 70/30. It’s a cheap laptop, and a large vault.

With both sides of the channel in agreement, it’s not even hard. Chad and Mallory update the channel to pay Chad his funds plus 70% of the vault, Mallory her funds plus 30%, and then they execute a cooperative close. Time to go shopping.

### Auditors

Chad’s other channel partners had been receiving all of the Chad/Mallory channel updates up to this point. Participation is one of the metrics used to estimate reputation. They also act as Lightning watchtowers, traditionally looking for unilateral closes, but also looking for *any* closes. Now they see that Chad/Mallory has cooperatively closed, but there’s no record that the deposits had been transferred out, or that the Validated Output was removed. Mempool shows an exit with only two parties.

Once Chad’s other channel partners have decided that he’s dishonest, there are two factors motivating them to unilaterally close their own channels with him. First, if they continue operations despite the evidence that Chad is dishonest they’re putting their own reputation at stake. It’s soft, but negative, and who knows how much longer Chad will be around anyway. Second, as part of the recovery one of them will get the future revenue that comes from continuing operations of the recovered vault.

### Over-Reserves

Respectfully capitalized vaults will have more funds in the vault output than the sum of the deposits. Keeping 100% + 100% / (number of vaults - 1) is a good start, but since channels come and go we can’t require a specific amount. And the scheme only works if you have relatively balanced vaults. If you only have three and one is twice as big as the other two, how do we keep things “reputable”?

Regardless, all of Chad’s other channel partners have full ledgers for all of his vaults. If he goes rogue, the remaining partners will force close their channels. Victor will have his hands full, but he will have 100% of the depositor funds as well as some percentage of security deposits for the other vaults. These reserves plus the individual ledgers are all that are necessary for Victor to make new vaults with the same deposits. He sends each of the vaults closed by peers to the peer they came from, and picks one of the peers at random to take the stolen vault.

It’s complicated, and underspecified, but hopefully rarely used. The goal here isn’t to prevent theft, but to make it so painful that Chad won’t even want to try. He steals gets 84% (70% of 120%) of one idealized vault, but loses 80% (20% times 4) of one plus all future operating revenue on the others.

### What About Bob?

Weeks later Bob opens his wallet only to find that Chad can’t be found. The wallet pings Victor and the previous channel partners, and finds that his funds are still available in a different channel. He uses his private key to sign a request for a new NWC connection attached to them, and buys a soda.

## Victorless

The last step is replacing Victor with a multisig of the same channel partners who will be participating in the recovery. It’s significant additional design, as coordination needs to be leaderless, and will have to make choices about who benefits from the additional vault. There are cryptographic techniques to accomplish this, but if you’ve made it this far suffice it to say that it all should be possible, if people actually start using Deposits. We can tweak it, tune it, and maybe even paper over it, as long as people actually use it.

## Links

https://deposits.ynniv.com/

https://github.com/ynniv/deposits

https://njump.me/npub12akj8hpakgzk6gygf9rzlm343nulpue3pgkx8jmvyeayh86cfrus4x6fdh

https://x.com/ynniv

-------------------------

