# Proposal: Limit ScriptPubKey length, with one exception

billymcbip | 2025-12-10 12:48:52 UTC | #1

I propose that we limit ScriptPubKey length for new UTXOs to 260 bytes, unless the transaction happens in a block where `block_height % 256 == 0`. This would be a consensus rule, activated in a soft fork.

**Why limit ScriptPubKey length?**

To make it more expensive and more cumbersome to embed arbitrary data. We cannot prevent steganography, but we can impose limits on contiguous data storage. Each ScriptPubKey in a serialized transaction is accompanied by a ScriptPubKey length field and an amount field.

The highest ScriptPubKey length that is encoded as a single byte is 252, so the optimal chunk size would be 251 or 252 bytes (depending on whether OP_RETURN is used). Each chunk would be accompanied by 8 bytes for the amount, 1 byte for the ScriptPubKey length, and potentially another byte for the OP_RETURN opcode. That’s around 4% minimum overhead per arbitrary data byte.

**Why exempt every 256th block?**

To avoid invalidating pre-signed transactions that create UTXOs with longer ScriptPubKeys. To be honest, a scenario where a soft fork without this exception would lead to real-world loss of funds is quite contrived, but it’s good to set a precedent that we will go above and beyond to prevent confiscation. This exception could be removed in another soft fork.

**Why 260 bytes?**

Consider that number a placeholder. 260 bytes would include all standard scripts, but I’m not set on this value.

Credits to @portlandhodl who suggested a 520 byte limit. He abandoned the idea due to confiscation risk, but I think the exception I propose mitigates that risk.

I’m looking forward to your feedback, especially on the block height exception.

-------------------------

Rob1Ham | 2025-12-09 23:56:27 UTC | #2

FYI - not my proposal, it was @portlandhodl who came up with this originally.

-------------------------

billymcbip | 2025-12-10 12:49:33 UTC | #3

Thank you for the correction. Edited.

-------------------------

