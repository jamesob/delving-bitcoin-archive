# Lightning Cheques

andyschroder | 2024-09-24 21:23:53 UTC | #1

Tangential to 
https://delvingbitcoin.org/t/privately-sending-payments-while-offline-with-bolt12/1134

I'd like to make another proposal: `Lightning Cheques`. `Lightning Cheques` are a paper instrument where the front side is a BOLT12 `invoice_request` for a predetermined amount of satoshi and the back side is an `offer`.

The idea is that you could have some software at home that connects to your always on lightning node and generates a bunch of `invoice_request` before you leave home and then formats them automatically into a QR code that stores a URI in the form of `bitcoin:?lnr=lnrXXXXXXXXXX`. The back side has a QR code that stores a URI in the form of `bitcoin:?lno=lno1XXXXXXXX`. Many of these can be printed on a standard sheet and a paper cutter can be used to cut them all out.

A merchant can withdraw from your lightning node using the `invoice_request` and then return change to you back using the `offer` on the back side. Just like paper money, these can be denominated in different units. You risk that the merchant will actually give you change, but the same thing happens with paper money. The main drawback with `Lightning Cheques` is you don't necessarily know if you've been ripped off and never got the change until you go back home.

Because you can create as many of these as you want, you can create a number of denominations and then pay a merchant with the closest combination of `Lightning Cheques` to minimize risk that you won't actually be paid your change. You can overprint `Lightning Cheques` just so that you can ensure that you have a closer combination of denominations for many purchases.

After a `Lightning Cheque` is used, it is worthless and it can be discarded. At any time, you can also destroy `Lightning Cheques` if you are worried they could be stolen. When you are back home, you can also invalidate a `Lightning Cheque` if you lost it.


`Lightning Cheque` could also be written to cheap NFC cards and reused, but I'm not sure if you could place the refund offer on the same card, you may need a separate card for that. Also, you may only want to transport them in a Faraday Bag.

You can use `Lightning Cheques` as a way to gift money, for young kids to pay for their lunch at school, as a backup when traveling in case you get robbed or your phone gets smashed, or if you just don't like mobile phones or can't afford one.

-------------------------

