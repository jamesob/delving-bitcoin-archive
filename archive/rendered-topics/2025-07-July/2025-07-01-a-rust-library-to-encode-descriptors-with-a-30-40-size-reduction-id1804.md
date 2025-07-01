# A rust library to encode descriptors with a 30-40% size reduction

josh | 2025-07-01 19:59:48 UTC | #1

## TLDR

[descriptor-codec](https://github.com/joshdoman/descriptor-codec) is a rust library for encoding and decoding Bitcoin wallet descriptors, achieving a 30-40% size reduction. This can be useful when sharing a descriptor as a QR code, over NFC, or in a data-constrained environment.

This library is partially inspired by [this comment](https://github.com/bitcoin/bitcoin/pull/32788#issuecomment-2996271309) by @sjors, where he suggested that wallets may seek to reduce QR code size. It is based on the encoder I built for [descriptor-encrypt](https://delvingbitcoin.org/t/rust-descriptor-encrypt-encrypt-any-descriptor-such-that-only-authorized-spenders-can-decrypt/1750), with additional support for descriptors with private keys.

## Features

* **Compact Encoding**: Tag-based and variable-length encoding and the avoidance of bech32 and base58 reduces descriptor size by 30-40%

* **Complete Coverage**: Supports all descriptors including those with complex miniscript and private keys

* **QR Code Friendly**: Smaller encodings improve QR code reliability and scanning

* **NFC Compatible**: Fits descriptors within NFC byte limits for hardware wallet communication

## Usage

The library has a simple API with two `encode` and `decode` functions. For demonstration purposes, it can also be used on the command-line with the built-in CLI.

```rust
use descriptor_codec::{encode, decode};

let descriptor = "wsh(sortedmulti(2,\
    03a0434d9e47f3c86235477c7b1ae6ae5d3442d49b1943c2b752a68e2a47e247c7,\
    036d2b085e9e382ed10b69fc311a03f8641ccfff21574de0927513a49d9a688a00,\
    02e8445082a72f29b75ca48748a914df60622a609cacfce8ed0e35804560741d29\
))#hfj7wz7l";

let encoded = encode(descriptor).unwrap();
let decoded = decode(&encoded).unwrap();
assert_eq!(descriptor, decoded);
```

## Example
Below is a 2-of-3 multisig descriptor, which is 457 characters, or 457 bytes if encoded as ASCII:
```
wsh(sortedmulti(2,[3abf21c8/48'/0'/0'/2']xpub6DYotmPf2kXFYhJMFDpfydjiXG1RzmH1V7Fnn2Z38DgN2oSYruczMyTFZZPz6yXq47Re8anhXWGj4yMzPTA3bjPDdpA96TLUbMehrH3sBna/<0;1>/*,[a1a4bd46/48'/0'/0'/2']xpub6DvXYo8BwnRACos42ME7tNL48JQhLMQ33ENfniLM9KZmeZGbBhyh1Jkfo3hUKmmjW92o3r7BprTPPdrTr4QLQR7aRnSBfz1UFMceW5ibhTc/<0;1>/*,[ed91913d/48'/0'/0'/2']xpub6EQUho4Z4pwh2UQGdPjoPrbtjd6qqseKZCEBLcZbJ7y6c9XBWHRkhERiADJfwRcUs14nQsxF3hvx7aFkbk3tfp4dnKfkcns217kBTVVN5gY/<0;1>/*))#e7m305nf
```

Encoded the descriptor is ~37% smaller, at 289 bytes:
```
050902032a2404610101050201000102302a2404610101050201000102302a2404610101050201000102303abf21c80488b21e04021f6d1080000002abcdaf8fd02bbec97fec5fd0efb74d259fb0f1ec74ddc81ada76e44c24cd552a039ecdadf9f67914b544825d8ccfa7e09a8a4a069960aef0ba454695b0b3cd53c1a1a4bd460488b21e0435113bd2800000026cf687ad7652cc0141acb9b0dcbd2ee508b47820b99924db919d7f2ee4c611300374ad54281f46da81639ff0556a58126f01cbd0a29f158e0d321478776b92e9e2ed91913d0488b21e0476a1d0b68000000235ec244d7c80352bbce7aa1c2b376b38e56ebe88286b4d458b850cbc6b49269d03d909eec00cb2ab8f120cb7e2ceb8896bbb09010e1e33be8696227c21eed43b05
```

## Algorithm

The encoder splits the descriptor into two parts that are concatenated: a structural **template** and a data **payload**.

### Template and Payload

The encoding separates the descriptor's structure from its raw data.

* **Template**: This part defines the logical structure of the descriptor. It is a sequence of single-byte tags that represent script components (like `wsh`, `pk`, `older`) and structural information. It also contains variable-length encoded integers for derivation paths and multisig `k`/`n` parameters.

* **Payload**: This part contains the raw data values from the descriptor, concatenated in the order they are referenced by the template. This includes items like public keys, private keys, key fingerprints, hashes, and timelock values.

When decoding, the template is read first to understand the structure, which then dictates how to parse the subsequent payload data.

### Variable-Length Encoding

To save space, unsigned integers are encoded as variable-length LEB128 integers. This is used for:

* Absolute and relative timelocks (`after`, `older`).
* The `k` (threshold) and `n` (total keys) values in multisig (`multi`, `sortedmulti`) and threshold (`thresh`) scripts.
* The length of derivation paths and the individual child numbers within them.
* Hardened child numbers are encoded as $2c+1$, where $c$ is the child number. Unhardened child numbers are encoded as $2 c$.

### Tags

Each component of a descriptor is represented by a single-byte tag.

|Tag Name | Hex Value | Description|
|:--- | :--- | :---|
| `False` | $0x00$ | Miniscript `false` operator. |
| `True` | $0x01$ | Miniscript `true` operator. |
| `Pkh` | $0x02$ | Top-level Pay-to-Public-Key-Hash descriptor. |
| `Sh` | $0x03$ | Top-level Pay-to-Script-Hash descriptor. |
| `Wpkh` | $0x04$ | Top-level Witness-Pay-to-Public-Key-Hash descriptor. |
| `Wsh` | $0x05$ | Top-level Witness-Pay-to-Script-Hash descriptor. |
| `Tr` | $0x06$ | Top-level Pay-to-Taproot descriptor. |
| `Bare` | $0x07$ | Top-level Bare Script descriptor. |
| `TapTree` | $0x08$ | A Taproot script path tree or leaf. |
| `SortedMulti`| $0x09$ | A sorted multisig script. |
| `Alt` | $0x0A$ | Miniscript `alt` operator. |
| `Swap` | $0x0B$ | Miniscript `swap` operator. |
| `Check` | $0x0C$ | Miniscript `check` operator. |
| `DupIf` | $0x0D$ | Miniscript `dupif` operator. |
| `Verify` | $0x0E$ | Miniscript `verify` operator. |
| `NonZero` | $0x0F$ | Miniscript `nonzero` operator. |
| `ZeroNotEqual`| $0x10$ | Miniscript `zeronotequal` operator. |
| `AndV` | $0x11$ | Miniscript `and_v` operator. |
| `AndB` | $0x12$ | Miniscript `and_b` operator. |
| `AndOr` | $0x13$ | Miniscript `andor` operator. |
| `OrB` | $0x14$ | Miniscript `or_b` operator. |
| `OrC` | $0x15$ | Miniscript `or_c` operator. |
| `OrD` | $0x16$ | Miniscript `or_d` operator. |
| `OrI` | $0x17$ | Miniscript `or_i` operator. |
| `Thresh` | $0x18$ | Miniscript `thresh` operator. |
| `Multi` | $0x19$ | Miniscript `multi` operator. |
| `MultiA` | $0x1A$ | Miniscript `multi_a` operator. |
| `PkK` | $0x1B$ | Miniscript `pk_k` (CHECKSIG). |
| `PkH` | $0x1C$ | Miniscript `pk_h` (CHECKSIG from hash). |
| `RawPkH` | $0x1D$ | Miniscript `raw_pkh` (raw public key hash). |
| `After` | $0x1E$ | Miniscript absolute timelock (`after`). |
| `Older` | $0x1F$ | Miniscript relative timelock (`older`). |
| `Sha256` | $0x20$ | A SHA256 hash. |
| `Hash256` | $0x21$ | A double-SHA256 hash. |
| `Ripemd160` | $0x22$ | A RIPEMD-160 hash. |
| `Hash160` | $0x23$ | A HASH160 (SHA256 then RIPEMD-160) hash. |
| `Origin` | $0x24$ | Indicates a key has an origin (fingerprint + path). |
| `NoOrigin` | $0x25$ | Indicates a key has no origin. |
| `UncompressedFullKey` | $0x26$ | An uncompressed public key. |
| `CompressedFullKey` | $0x27$ | A compressed public key. |
| `XOnly` | $0x28$ | An x-only (Taproot) public key. |
| `XPub` | $0x29$ | An extended public key (`xpub`). |
| `MultiXPub` | $0x2A$ | An `xpub` with multiple derivation paths. |
| `UncompressedSinglePriv`| $0x2B$ | An uncompressed private key. |
| `CompressedSinglePriv` | $0x2C$ | A compressed private key. |
| `XPriv` | $0x2D$ | An extended private key (`xprv`). |
| `MultiXPriv` | $0x2E$ | An `xprv` with multiple derivation paths. |
| `NoWildcard`| $0x2F$ | No wildcard `/*` in a derivation path. |
| `UnhardenedWildcard` | $0x30$ | Unhardened wildcard `/*` in a derivation path. |
| `HardenedWildcard` | $0x31$ | Hardened wildcard `/*h` in a derivation path. |

## Use Cases

* Sharing complex multisig configurations via QR codes
* NFC communication with hardware wallets
* Reducing bandwidth in wallet coordination protocols
* Storing descriptors in constrained environments

## Future Work

This library aims to reduce QR code size wherever descriptors need to be shared. The goal is for it to be used by hardware wallets importing and exporting descriptors and by users that wish to backup their descriptor by printing out a QR code.

In the future, I plan to replace the encoder in `descriptor-encrypt` with this library. If new miniscript operators, witness programs, or descriptor types are added in the future, new tags will need to be added to provide support.

## Links

Github: [https://github.com/joshdoman/descriptor-codec](https://github.com/joshdoman/descriptor-codec)

Docs: [https://docs.rs/descriptor-codec/latest/descriptor_codec](https://docs.rs/descriptor-codec/latest/descriptor_codec)

-------------------------

