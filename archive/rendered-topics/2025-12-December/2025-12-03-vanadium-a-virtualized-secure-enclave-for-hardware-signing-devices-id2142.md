# Vanadium: A Virtualized Secure Enclave for Hardware Signing Devices

salvatoshi | 2025-12-03 19:54:28 UTC | #1

Developing firmware applications for hardware signing devices has always been constrained by the difficulties of embedded development: tiny RAM, tiny flash, vendor-specific SDKs, slow iteration cycles, and painful debugging.

Over the past year, I’ve been building **[Vanadium](https://github.com/LedgerHQ/vanadium)**, a different approach. It isn’t production-ready yet, but it is mature enough for external developers to try. Early feedback will directly shape the roadmap.

# What is Vanadium?

Vanadium is a **RISC-V Virtual Machine** that can run in an embedded **Secure Element**, designed to run arbitrary applications (“V-Apps”) in a secure-enclave environment while **outsourcing memory and storage to an untrusted host**.

This gives developers a *virtualized secure enclave* with dramatically fewer constraints than physical secure elements normally impose.

### Core capabilities

* **Virtually unlimited memory** — Only the pages actually used at runtime are loaded. Everything else lives on the host. Page swaps are transparent.

* **Native development workflow** — Write and test V-Apps as normal Rust programs on your desktop. No vendor-specific SDK, no special toolchain, no emulator required.

* **Security of execution** — The VM runs in the Secure Element. Outsourced pages are encrypted and authenticated; the host cannot modify code or data.

By virtualizing the secure enclave, Vanadium abstracts away many challenges of embedded programming. This will hopefully lead to much faster development cycles, and help innovation in self-custody and applied cryptography reach hardware signing devices at a fraction of today’s cost and time commitments.

# What does Vanadium solve?

Vanadium makes development substantially easier, allowing fast iterations. Moreover, most of the code written for Vanadium is portable: developers write standard Rust and are able to use existing Rust libraries.

The goal is to make innovation in self-custody (and applied cryptography) happen at a much faster pace, be much more open and standardized.

# Technical deep dive

This is an architecture diagram describing the various components of the Vanadium ecosystem.

![Vanadium Architecture diagram|666x500](upload://p8ZEEkcleNXJFF61mJwY0CxY0D3.svg)

While Vanadium is a complex project, this diagram illustrates how the project aims to minimize the developer’s workload to the modules labeled as *USERLAND*. The system calls, the memory outsourcing protocol, and all direct communication between the VM and the host are encapsulated and transparent to developers.

The public interface of the V-App SDK and the V-App Client SDK are the only contacts between the V-App and its client with the Vanadium ecosystem. Therefore, porting Vanadium to a new platform is reduced to providing workable implementations of these two crates. The current implementation benefits from this segregated architecture by enabling the ***native*** compilation target where apps (and their clients) use no Vanadium, no devices, and run directly on the developer’s machine. This would also greatly simplify porting Vanadium to new hardware.

## Performance

> TL;DR <br>**The VM only executes business logic; cryptography and other heavy operations run natively via ECALLs.**

As signing devices often have rather slow CPUs (but are in some cases equipped with a cryptographic accelerator), the performance would be unacceptable if one suffered the full slow-down caused by simulating a CPU for every operation.

The solution is to provide system services that allow performance-critical operations to be executed at the native speed of the underlying device. In fact, few operations (for example, hashes, elliptic curve operations, signatures, etc.) tend to be the bottleneck in common applications of signing devices. Therefore, the severe slowdown of a VM is acceptable as long as it’s limited to the *business logic* of the app.

The standard way to introduce system services in RISC-V is via the ECALL opcode.

Here is a list of the ECALLs currently implemented in Vanadium (as an example - not final):


[details="List of ECALLs"]
```
ECALL_FATAL: u32 = 1; // abort the execution with error
ECALL_XSEND: u32 = 2; // send a buffer to the host
ECALL_XRECV: u32 = 3; // receive a buffer from the host
ECALL_EXIT: u32 = 4;  // exit the app normally
ECALL_PRINT: u32 = 5; // send a message to print to the host

// Big numbers
ECALL_MODM   // modulus
ECALL_ADDM   // modular addition
ECALL_SUBM   // modular subtraction
ECALL_MULTM  // modular multiplication
ECALL_POWM   // modular power

// Elliptic curves
ECALL_ECFP_ADD_POINT   
ECALL_ECFP_SCALAR_MULT

ECALL_DERIVE_HD_NODE: u32 = 130;         // BIP32 derivation
ECALL_GET_MASTER_FINGERPRINT: u32 = 131; // Master BIP32 fingerprint
ECALL_DERIVE_SLIP21_KEY: u32 = 132;      // symmetric key derivations

// Hash functions (streaming)
ECALL_HASH_INIT
ECALL_HASH_UPDATE
ECALL_HASH_DIGEST

// Random number generation
ECALL_GET_RANDOM_BYTES

// Signatures
ECALL_ECDSA_SIGN
ECALL_ECDSA_VERIFY
ECALL_SCHNORR_SIGN
ECALL_SCHNORR_VERIFY


// device handling, events, and UX
// ...omitted for simplicity
```
[/details]


These provide the functionalities required for the majority of Bitcoin-related use cases, and can more generally be extended to cover most cryptographic applications.

ECALLs are also used to provide communication, device-specific interactions and event handling, etc.

## Security

> TL;DR <br> **Same threat model as existing hardware signing devices, with one caveat (memory access-pattern leakage) described below.**

### Assumptions

The goal of Vanadium is to provide the same security model that hardware signing devices offer: 
* The **host is fully untrusted**, can intercept, tamper with, or abort any message.
* The **Secure Element is trusted**.
* The **display is trusted** (the host cannot forge the UI).

A successful attack is one where the host extracts secrets or tricks the device into misbehavior (e.g., signing the wrong message).

### Code Correctness / App Authenticity

Vanadium uses a **manifest** that commits to the exact code and metadata of a V-App.
During registration:

1. User approves app 'installation'.
2. Device returns an HMAC as proof of registration.
3. Only registered apps may execute.

This mirrors the security of sideloaded binaries today. Planned improvements include signed manifests for simpler verification.

### Secure Outsourced Memory

A naive outsourced memory system would be insecure. Vanadium uses standard constructions:

**Integrity & Freshness**
A Merkle tree commits to all memory pages.

* Host must provide a valid Merkle proof for every page read.
* Host must provide valid updates when committing new pages.
An invalid proof aborts execution.

**Confidentiality**
Pages are encrypted with **AES-CTR** before leaving the device.

Other approaches are possible, with different choices of tradeoffs.

This secures content—but not *everything*.

### Caveat: Memory Access Pattern Side Channel

Vanadium **does not fully hide the access pattern**: the host can observe which code and data pages are accessed and when.

This matters if:

* secrets live in memory, and
* code paths or memory lookups depend on secret bits.

Scalar-mult implementations that index lookup tables by secret bits are a canonical vulnerable case.

Most apps won’t implement cryptography directly and will rely on ECALLs instead, so most developers are not affected by this issue. But **cryptographic developers must treat this as a real attack surface**.

**⚠️ Application developers should not implement cryptography** in Vanadium apps without fully understanding the possible impact of this side channel. Existing cryptographic libraries should also be presumed vulnerable if used in Vanadium apps, unless they explicitly guarantee that their code paths and memory access patterns do not depend on secrets.

An [Oblivious RAM](https://en.wikipedia.org/wiki/Oblivious_RAM) would solve this, but bringing complexity and a significant performance impact. Simpler mitigations could also be considered, if there is demand for such cryptography-heavy applications.

## Current limitations

* It has not yet been audited.
* Performance not yet good enough for some applications (especially for large binaries).
* UX capabilities are currently rather limited.

Work is steadily ongoing to address all the above points, and I look forward to demonstrating its practicality for an increasing number of use cases.

# Developer Preview

> **TL;DR**<br>Don’t put money in it, but please do [try it out](https://github.com/LedgerHQ/vanadium)!

A working prototype of Vanadium for Ledger devices is available:<br>
&nbsp;&nbsp;&nbsp;&nbsp;→
&nbsp;[https://github.com/LedgerHQ/vanadium](https://github.com/LedgerHQ/vanadium)

-------------------------

