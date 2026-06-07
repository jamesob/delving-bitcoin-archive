# Encoding art into the satoshi value field, and the unfilterable floor

softmatsg | 2026-06-07 06:33:05 UTC | #1

This picture is made of money. Not a picture of money, not sold for money. The image is encoded into the satoshi values of spendable Bitcoin outputs. Each dot is a real payment. The amounts, read in order, are the image. The artwork is the denominations. We built this to ask what happens when art is made of money. The answer is that money's rules govern it. A dust floor forbids the faintest mark. A fee is charged to place the work and again to move it. A large enough piece becomes too expensive to sell. The artwork traps its owner.
![art\\_preview (1)|500x500](upload://vQUkk1EottW24NKWctyjiXjqOYw.png)

The encoding is value equals offset plus data byte, with offset at the P2WPKH dust floor of 294 satoshis, so each output carries one byte of compressed image data in its amount. Output count equals byte count, decode order follows vout index, and the image is CCITT Group 4 compressed before being chunked. The metadata, namely a magic byte 0x50, a codec identifier, and the square side, fits in the four bytes of nLockTime using the Lockchain Protocol layout, packed so that the resulting value is a past timestamp and the transaction is therefore final. The construction needs **no OP_RETURN, no witness payload, no script encoding**, and no consensus changes. All transactions are fully standard and relay on default nodes.

Transfer preserves the value vector exactly. The transaction spends the previous data outputs together with a funding input, recreates the same values to a new canvas address, and pays the fee entirely from the funding input since reducing a data value would corrupt a byte. With relay limit at 100,000 virtual bytes, single transaction transfer bounds at about 1010 outputs, a sweep plus re-mint path raises it to about 1470, and a sharded sweep plus re-mint reaches about 3225. Above that a piece must be minted across several transactions. The consensus limit near a million virtual bytes is reachable only through direct miner inclusion, which is the relevant point for very large pieces.

This cannot be filtered. Because the data lives only in the amount field, no relay policy can reject it without also constraining which satoshi values users are permitted to send. A policy that said some amounts are invalid is not a data filter, it is a payment restriction, and it is incompatible with permissionless money. Freedom money entails freedom art. The value field therefore is the hard floor for data filtering, one that policy cannot pass without abandoning the payment semantics the network depends on. This does not mean data filtering is impossible elsewhere on the chain. It means **there is a place data filtering cannot reach**, and that place is the carrier every payment must use.

We arrived at it by making art and watching money's rules govern it. Yves Klein reached a related question in 1962 by selling empty space for gold and throwing the gold in the Seine, locking his work in immobility through irreversible destruction. We arrived here by locking value that stays recoverable, the same stillness by a gentler road. The technical property and the conceptual one are the same property, that once something is made of money it obeys money's rules, and those rules have a floor that cannot be legislated away.

Code, signet verification, and a live piece are at [\[repo\]](https://github.com/softmatsg/thulge-philargyry).
Reach to me if you would like to sponsor or participate in developing philargyry.com

-------------------------

