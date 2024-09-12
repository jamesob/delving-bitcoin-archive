# Mutation-core: A mutation testing tool for Bitcoin Core

bruno | 2024-09-06 19:43:09 UTC | #1

Hi, 

I’ve been working on a mutation testing tool for Bitcoin Core. It's something I've been using it for some time now and just made it public recently.

https://github.com/brunoerg/mutation-core

## Features

- Allows generating mutants only for the code touched or added in a specific branch (useful to test PRs) and avoid spending time by generating mutants for files from bench/test/doc/etc folders.

- Allows generating mutants using only some security-based mutation operators. This might be good for testing fuzzing.

    - e.g:
    ```diff
        @@ -630,7 +630,7 @@ static void ApproximateBestSubset(FastRandomContext& insecure_rand, const std::v
                    {
                        nTotal += groups[i].GetSelectionAmount();
                        selected_coins_weight += groups[i].m_weight;
    -                   vfIncluded[i] = true;
    +                   vfIncluded[i + 100] = true;
                        if (nTotal >= nTargetValue && selected_coins_weight <= max_selection_weight) {
                            fReachedTarget = true;
                            // If the total is between nTargetValue and nBest, it's our new best
    ```
    ```diff
        @@ -560,7 +560,7 @@ util::Result<SelectionResult> SelectCoinsSRD(const std::vector<OutputGroup>& utx
    
            // Add group to selection
            heap.push(group);
    -        selected_eff_value += group.GetSelectionAmount();
    +        selected_eff_value += group.GetSelectionAmount() + std::numeric_limits<CAmount>::max();
            weight += group.m_weight;
    
            // If the selection weight exceeds the maximum allowed size, remove the least valuable inputs until we
    ```
    ```diff
        @@ -4194,7 +4194,7 @@ static bool ContextualCheckBlockHeader(const CBlockHeader& block, BlockValidatio
        }
    
        // Check timestamp against prev
    -    if (block.GetBlockTime() <= pindexPrev->GetMedianTimePast())
    +    if (block.GetBlockTime() <= std::numeric_limits<int64_t>::max())
            return state.Invalid(BlockValidationResult::BLOCK_INVALID_HEADER, "time-too-old", "block's timestamp is too early");
    
        // Testnet4 only: Check timestamp against prev for difficulty-adjustment
    ```
- Avoids creating useless mutants. (e.g. by skipping comments, `LogPrintf` statements, etc).

- Allows generating only one mutant per line of code (might be useful for CI/CD). It reduces the total number of mutants significantly.

- Allows creating mutations in the functional tests. By removing some statements and method calls, we can check whether the test passes and identify buggy tests. In our case, we 
will not touch any `wait_for`, `wait_until`, `send_and_ping`, `assert_*` and other verifications.

...and, of course, there are some specific mutation operators designed for Bitcoin Core.

----------

For more informations, please see the README in the repository. Feedbacks are welcome! :slight_smile:

-------------------------

0xB10C | 2024-09-12 08:51:38 UTC | #2

Cool! 

I was wondering how this compares to the mutation testing done by https://corecheck.dev?

-------------------------

