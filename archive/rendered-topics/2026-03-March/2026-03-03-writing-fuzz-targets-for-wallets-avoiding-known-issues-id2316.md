# Writing Fuzz Targets for Wallets: Avoiding Known Issues

bruno | 2026-03-03 22:32:15 UTC | #1

Fuzzing a Bitcoin wallet is very different from fuzzing a parser or a single pure function. There are many details to pay attention, otherwise you can have performance and determininsm issues. In general, wallets are stateful, persistent, have cryptographic operations, dependent on chain state, and more. Based on my experience writing wallet fuzz targets for Bitcoin Core, this post walks through the main performance and determinism pitfalls — and how to avoid them, especially to help other devs to write fuzz targets for other libraries and applications. The inspiration for writing this comes from a recent talk with a BDK maintainer, [Leo](https://github.com/oleonardolima), about adding fuzzing there, where they're having similar issues that we had in past. Please, contibute to this thread if you have any other point that has not been covered here.

--------------

### The Keypool Size

Before talking about the optimization itself, it’s important to understand what the \*\*keypool\*\* actually is and why it exists.

In a Bitcoin wallet — including Bitcoin Core — the keypool is a pre-generated pool of keys reserved for future address generation. Instead of deriving a fresh key every time the wallet needs a receiving address, a change address, or a descriptor-based scriptPubKey, the wallet derives a batch of keys in advance and stores them internally. When a call such as `wallet->GetNewDestination(type, label);` is executed, the wallet pops a key from this pool, marks it as used, and may trigger a `TopUp()` operation if the number of remaining pre-generated keys falls below the configured target size. This design improves user experience in production by making address generation fast and ensuring that backups already contain future keys.

However, in a fuzzing context this behavior becomes unnecessarily expensive. If your fuzz target generates many new destinations, exercises both external and change paths, or repeatedly invokes wallet logic that consumes keys, then each iteration may cause the wallet to check the pool size, derive additional HD keys, update internal metadata, potentially touch the database layer, and perform cryptographic operations. Even if each derivation costs only microseconds, the cumulative impact over hundreds of thousands (or millions) of fuzz iterations becomes significant. If a single iteration wastes just a few milliseconds in keypool refills, you effectively lose millions of potential executions — and fuzzing effectiveness drops accordingly.

If your target is \*\*not specifically testing keypool behavior\*\*, the simplest and most effective optimization is to reduce the keypool size to 1, as done in the wallet fuzz targets in Bitcoin Core:

```cpp
wallet->m_keypool_size = 1;
```

This drastically reduces the number of pre-generated keys, lowers the frequency of \`TopUp()\` calls, and limits the amount of HD derivation work performed during fuzzing. Importantly, this does \*\*not\*\* disable key derivation altogether; the wallet will still derive keys when necessary and behave correctly. It simply prevents aggressive pre-allocation behavior that is useful in production but irrelevant, and costly, in a fuzzing environment.


--------

### Descriptor and miniscript parsing

Descriptor wallets can easily "explode" in complexity if left unconstrained. Deep wrapper nesting such as `and_v(and_v(and_v(...)))` or `sh(wsh(multi(...)))`, large `thresh()` constructions, and highly complex Miniscript policy trees can trigger heavy type checking and expensive satisfaction weight analysis. When fuzz input is unbounded, the fuzzer often generates descriptors that are syntactically valid but computationally pathological, leading to CPU exhaustion, extremely slow parsing, etc. This happens because most execution time and branch coverage gets consumed by parsing and validation, drowning out signal from the wallet behavior you actually care about. To prevent this, constrain descriptor complexity by tracking wrapper depth, limiting the number of subfragments in `thresh()` and `multi()`, bounding derivation range sizes, and rejecting descriptors that exceed safe thresholds early in the fuzzing pipeline. Better still, use a structured input generator or custom mutator to enforce these constraints at generation time rather than filtering after the fact, so fuzzer iterations are not wasted on inputs that will be immediately discarded. The goal is to fuzz wallet behavior, not worst-case parser performance, since there is already a fuzz target for this.

This is the function Bitcoin Core uses to check if a descriptor is expensive:

```cpp
/// Deriving "expensive" descriptors will consume useful fuzz compute. The
/// compute is better spent on a smaller subset of descriptors, which still
/// covers all real end-user settings.
///
/// Use this function after MockedDescriptorConverter::GetDescriptor()
inline bool IsTooExpensive(std::span<const uint8_t> buffer)
{
    // Key derivation is expensive. Deriving deep derivation paths takes a lot of compute and we'd
    // rather spend time elsewhere in this target, like on the actual descriptor syntax. So rule
    // out strings which could correspond to a descriptor containing a too large derivation path.
    if (HasDeepDerivPath(buffer)) return true;

    // Some fragments can take a virtually unlimited number of sub-fragments (thresh, multi_a) but
    // may perform quadratic operations on them. Limit the number of sub-fragments per fragment.
    if (HasTooManySubFrag(buffer)) return true;

    // The script building logic performs quadratic copies in the number of nested wrappers. Limit
    // the number of nested wrappers per fragment.
    if (HasTooManyWrappers(buffer)) return true;

    // If any suspected leaf is too large, it will likely not represent a valid
    // use-case. Also, possible base58 parsing in the leaf is quadratic. So
    // limit the leaf size.
    if (HasTooLargeLeafSize(buffer)) return true;

    return false;
}
```

--------

### Mock the fee estimator

In `fees.cpp`, the wallet's fee estimation logic is tested in isolation by subclassing `CBlockPolicyEstimator` and overriding its two key virtual methods — `estimateSmartFee` and `HighestTargetTracked` — to return fuzz-controlled values instead of consulting a real fee histogram built from historical block data. The real estimator needs a "populated database" to produce meaningful fee rates; in a fuzz target that database would either be empty (producing only one code path) or require expensive setup to populate differently each iteration. By replacing it with `FuzzedBlockPolicyEstimator`, the fuzzer can drive `GetMinimumFeeRate` and `GetMinimumFee` through every branch that depends on what the estimator returns — conservative vs. economical modes, fee rates that are above or below the mempool minimum, edge cases — without any of that state.

```cpp
class FuzzedBlockPolicyEstimator : public CBlockPolicyEstimator
{
    FuzzedDataProvider& fuzzed_data_provider;

public:
    FuzzedBlockPolicyEstimator(FuzzedDataProvider& provider)
        : CBlockPolicyEstimator(fs::path{}, false), fuzzed_data_provider(provider) {}

    CFeeRate estimateSmartFee(int confTarget, FeeCalculation\* feeCalc, bool conservative) const override
    {
        return CFeeRate{ConsumeMoney(fuzzed_data_provider, /\*max=\*/1'000'000)};
    }

    unsigned int HighestTargetTracked(FeeEstimateHorizon horizon) const override
    {
        return fuzzed_data_provider.ConsumeIntegralInRange<unsigned int>(1, 1000);
    }
};
```

----------

### Avoid Encryption/Decryption

Wallet encryption is computationally expensive and can significantly slow down fuzzing if exercised on every iteration.  Unless your target is explicitly testing encryption behavior, passphrase derivation edge cases, or decryption error handling, you are better off using an unencrypted wallet, avoiding passphrase-related flows. Keeping costly routines out of the hot path directly translates to more iterations per second and a more effective fuzzing campaign. If you do need to cover encryption paths, consider isolating them in a dedicated target rather than mixing them into a general wallet fuzzer.

A good example for this case is that we do not cover encryption/decryption in our `scriptpubkeyman` target. E.g:

```cpp
bool DescriptorScriptPubKeyMan::Encrypt(const CKeyingMaterial& master_key, WalletBatch\* batch)
{
    LOCK(cs_desc_man);
    if (!m_map_crypted_keys.empty()) {
        return false;
    }

    for (const KeyMap::value_type& key_in : m_map_keys)
    {
        const CKey &key = key_in.second;
        CPubKey pubkey = key.GetPubKey();
        CKeyingMaterial secret{UCharCast(key.begin()), UCharCast(key.end())};
        std::vector<unsigned char> crypted_secret;
        if (!EncryptSecret(master_key, secret, pubkey.GetHash(), crypted_secret)) {
            return false;
        }
        m_map_crypted_keys\[pubkey.GetID()\] = make_pair(pubkey, crypted_secret);
        batch->WriteCryptedDescriptorKey(GetID(), pubkey, crypted_secret);
    }
    m_map_keys.clear();
    return true;
}
```


-----

### Mock the Database

Wallets are database-backed, and in Bitcoin Core this typically means SQLite. While persistent storage is essential in production, disk I/O is highly problematic in a fuzzing environment. It slows execution, introduces non-determinism due to filesystem timing and state, and can cause flaky behavior depending on the runtime environment. It may also produce unintended file-system side effects that interfere with reproducibility. For fuzzing, Bitcoin Core always uses `CreateMockableWalletDatabase()`, other applications should use an equivalent in-memory thing. Effective fuzzing requires the environment to be deterministic, fast, and isolated — and real disk writes undermine all three.

----------

### Write a “Wrapped Wallet”

Often you’re not trying to fuzz wallet creation itself, but you still need a valid wallet instance in order to exercise the code you actually care about. If every fuzz iteration creates a wallet, imports descriptors, and initializes chain-related state, then most of your runtime is spent on setup rather than on the wallet operations under test — and you also end up “spending” valuable fuzz input bytes on initialization details that don’t matter. A practical solution is to build a small \*wrapped wallet fixture\* that constructs a minimal, deterministic descriptor wallet once and then exposes helper methods that your fuzz target can call cheaply. The `FuzzedWallet` pattern below (from Bitcoin Core fuzz targets) does exactly that: it creates the wallet using a mockable database backend (avoiding disk I/O), forces descriptor mode, initializes \`last block processed\` using the current chain height/hash (so wallet logic sees consistent chain context), reduces the keypool size to 1 to avoid costly `TopUp()` behavior, and imports a small, controlled set of common descriptors (both internal/change and external/receiving) with a tiny derivation range (`range_end = 1`). As a result, later calls like `GetNewDestination()` and `GetNewChangeDestination()` become fast and stable, letting the fuzz target focus its cycles on other stuff — instead of repeatedly paying the cost of wallet setup.

```cpp
struct FuzzedWallet {
    std::shared_ptr<CWallet> wallet;

    FuzzedWallet(interfaces::Chain& chain,
                 const std::string& name,
                 const std::string& seed_insecure)
    {
        wallet = std::make_shared<CWallet>(&chain, name,
                                           CreateMockableWalletDatabase());

        {
            LOCK(wallet->cs_wallet);
            wallet->SetWalletFlag(WALLET_FLAG_DESCRIPTORS);

            auto height{\*Assert(chain.getHeight())};
            wallet->SetLastBlockProcessed(height,
                                          chain.getBlockHash(height));
        }

        wallet->m_keypool_size = 1; // Avoid timeout in TopUp()
        assert(wallet->IsWalletFlagSet(WALLET_FLAG_DESCRIPTORS));

        ImportDescriptors(seed_insecure);
    }

    void ImportDescriptors(const std::string& seed_insecure)
    {
        const std::vector<std::string> DESCS{
            "pkh(%s/%s/\*)",
            "sh(wpkh(%s/%s/\*))",
            "tr(%s/%s/\*)",
            "wpkh(%s/%s/\*)",
        };

        for (const std::string& desc_fmt : DESCS) {
            for (bool internal : {true, false}) {

                const auto descriptor{
                    strprintf(tfm::RuntimeFormat{desc_fmt},
                              "\[5aa9973a/66h/4h/2h\]" + seed_insecure,
                              int{internal})};

                FlatSigningProvider keys;
                std::string error;

                auto parsed_desc =
                    std::move(Parse(descriptor, keys, error,
                                    /\*require_checksum=\*/false).at(0));

                assert(parsed_desc);
                assert(error.empty());
                assert(parsed_desc->IsRange());
                assert(parsed_desc->IsSingleType());
                assert(!keys.keys.empty());

                WalletDescriptor w_desc{
                    std::move(parsed_desc),
                    0, 0, 1, 0
                };

                LOCK(wallet->cs_wallet);

                auto& spk_manager =
                    Assert(wallet->AddWalletDescriptor(
                        w_desc, keys, "", internal))->get();

                wallet->AddActiveScriptPubKeyMan(
                    spk_manager.GetID(),
                    \*Assert(w_desc.descriptor->GetOutputType()),
                    internal);
            }
        }
    }

    CTxDestination GetDestination(FuzzedDataProvider& provider)
    {
        auto type = provider.PickValueInArray(OUTPUT_TYPES);

        if (provider.ConsumeBool()) {
            return \*Assert(wallet->GetNewDestination(type, ""));
        } else {
            return \*Assert(wallet->GetNewChangeDestination(type));
        }
    }

    CScript GetScriptPubKey(FuzzedDataProvider& provider)
    {
        return GetScriptForDestination(GetDestination(provider));
    }
};
```

--------

### Closing Thoughts

Wallet fuzzing is not just about throwing random bytes at wallet APIs. In my experience writing wallet fuzz targets for Bitcoin Core, most improvements came not from writing more fuzz logic — but from removing hidden performance and determinism traps from our targets.

-------------------------

benalleng | 2026-03-05 12:46:24 UTC | #2

Not exactly wallet specific but I’m curious if you have explored any “process” improvements on your fuzzing workflow, whether through engine choice or cycle adjustments?

As an example I have read in some fuzzing forums about novel ways to minimize the corpus to try and increase coverage more quickly like simply randomly culling some percentage of the corpus as opposed to using the standard cmin tool built in to most fuzzing engines to force some additional variance into the fuzzer.

It’s clear that CPU cycles are the bottleneck but I wonder if there is any value in trying to get better map density and count coverage per average cycle? Or maybe it’s all a wash with enough CPU hours.

-------------------------

