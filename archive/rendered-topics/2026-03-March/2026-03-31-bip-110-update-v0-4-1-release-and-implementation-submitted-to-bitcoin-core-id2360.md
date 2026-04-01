# BIP-110 update: v0.4.1 release and implementation submitted to Bitcoin Core

dathonohm | 2026-03-31 23:24:03 UTC | #1

# BIP-110 update: v0.4.1 release and implementation submitted to Bitcoin Core

Hi all,

Following up on the BIP-110 proposal from [October](https://groups.google.com/g/bitcoindev/c/nOZim6FbuF8), I am announcing the most recent stable release (v0.4.1) and the submission of an implementation of BIP-110 to Bitcoin Core.

## Background

BIP-110 was [initially](https://groups.google.com/g/bitcoindev/c/YO8ZwnG_ISs/m/NpQfYlGLAgAJ) [proposed](https://groups.google.com/g/bitcoindev/c/nOZim6FbuF8) in October 2025, and the first version of the [official activation client](https://github.com/dathonohm/bitcoin/releases/tag/v29.3.knots20260210%2Bbip110-v0.4.1) was released at the end of January as a [fork of Bitcoin Knots](https://github.com/bitcoinknots/bitcoin/pull/238). It was [accepted into the BIPs repo](https://github.com/bitcoin/bips/pull/2017) shortly thereafter.

For those new to BIP-110, it is a proposal to temporarily limit the size of data fields at the consensus level, in order to reject the standardization of data storage as a supported use case by consensus. It achieves this by temporarily limiting all officially supported methods of data storage (and methods that could be or become perceived as officially supported) to 256 bytes or less, while preserving all known monetary use cases.

This has several desirable effects, including but not limited to:

- Reducing demand for arbitrary data storage by frustrating attempts to monetize and gamify arbitrary data.

- Bringing consensus and policy back into closer alignment, by restoring still-popular policy limits from a few years ago in consensus.

- Protecting the decentralization of the node network by eliminating unnecessary costs and (in some cases, extreme) risks introduced by making arbitrary data storage an officially supported function of Bitcoin.

- Making Bitcoin better money by preventing Bitcoin payments from having to compete at an unfair disadvantage against gamified data spam.

## Activation client v0.4.1

The official BIP-110 activation client is currently `bitcoin-29.3.knots20260210+bip110-v0.4.1`, a fork of Bitcoin Knots 29.3. While a few bugs were found early on, they were all swiftly resolved and the code has now been stable for several weeks.

### Consensus rules (when active)

- scriptPubKeys limited to 34 bytes (83 bytes for OP_RETURN)

- OP_PUSHDATA\* payloads and witness stack elements limited to 256 bytes (exception: BIP16 redeemScript)

- Spending undefined witness/Tapleaf versions is invalid

- Taproot annex is invalid

- Taproot control blocks limited to 257 bytes (128 script leaves)

- OP_SUCCESS\* opcodes in Tapscripts are invalid

- OP_IF/OP_NOTIF in Tapscripts is invalid

- Empty witness required for P2A spends

Inputs spending UTXOs created before the activation height are exempt (this is also known as "UTXO grandfathering").

Rules expire automatically after ~1 year (52416 blocks).

### Deployment

- Signaling bit: 4

- Threshold: 55% (1109/2016)

- Start time: ~December 1, 2025

- Max activation height: 965664 (~September 1, 2026)

- Mandatory signaling enforced in the retarget period before mandatory lock-in

### Downloads (Guix reproducible builds)

**Binaries:** [GitHub Release](https://github.com/dathonohm/bitcoin/releases/tag/v29.3.knots20260210%2Bbip110-v0.4.1)

- Linux: x86_64, aarch64, arm, powerpc64, powerpc64le, riscv64

- macOS: arm64, x86_64

- Windows: x86_64

**Node platforms:**

- Umbrel: Available in the Umbrel app store ([PR #5044](https://github.com/getumbrel/umbrel-apps/pull/5044))

- MyNode: Available in MyNode ([PR #996](https://github.com/mynodebtc/mynode/pull/996))

- Start9 (StartOS v0.3.5.1): Sideload .s9pk package ([Release](https://github.com/dathonohm/knots-startos/releases/tag/v29.3.knots20260210%2Bbip110-v0.4.1))

- Start9 (StartOS v0.4): Sideload .s9pk package ([Release](https://github.com/dathonohm/knots-startos/releases/tag/v29.3.knots20260210%2Bbip110-v0.4.1-startos0.4.0-1))

- Parmanode: Available in Parmanode

- Ubuntu PPA (22.04, 24.04, 25.10, 26.04): [PPA](https://launchpad.net/~dathonohm/+archive/ubuntu/bitcoinknots+bip110)

## Bitcoin Core implementation

The activation client has proven immensely popular, and now comprises over [8% of listening nodes](https://bitnodes.io/nodes/?q=BIP110) after just 2 months. While BIP-110's mandatory signaling period is not until August, it allows for early activation if 55% of hashpower signals support, so activation could be as soon as a few weeks from now. Indeed, 4 blocks signaling for BIP-110 have already been mined during the month of March.

Due to BIP-110's rising popularity, it seems prudent to augment Core with the ability to enforce the new rules, so that once BIP-110 is activated, Core users will remain secure because their nodes will follow the valid chain. Accordingly, I have submitted an implementation of BIP-110 to Bitcoin Core, split across two PRs:

- **PR 1** - Versionbits extensions for BIP-110: [bitcoin/bitcoin#34929](https://github.com/bitcoin/bitcoin/pull/34929)

- **PR 2** - ReducedData Temporary Softfork (BIP-110/RDTS): [bitcoin/bitcoin#34930](https://github.com/bitcoin/bitcoin/pull/34930)

Both PRs were automatically closed by a bot shortly after submission. I am working to get them reopened. In the meantime, review and discussion can take place on mirror PRs:

- **PR 1** (mirror): [bip110-core/bitcoin#1](https://github.com/bip110-core/bitcoin/pull/1)

- **PR 2** (mirror): [bip110-core/bitcoin#2](https://github.com/bip110-core/bitcoin/pull/2)

The Core PRs are derived from the activation client's [Knots branch](https://github.com/bitcoinknots/bitcoin/pull/238). Some refactoring was required to accommodate recent upstream changes to Core's BIP9 implementation.

The implementation totals ~375 lines of non-test C++/H across both PRs -- roughly a third the size of CSV (BIP 68/112/113). Every commit builds and passes all tests independently.

## Review

I welcome technical review on the mirror PRs linked above, or on the upstream PRs once they are reopened.

My sincere hope is that BIP-110's activation will heal the divide in our community and allow us all to get back to building Bitcoin into history's greatest money.

Sincerely,

Dathon Ohm

-------------------------

