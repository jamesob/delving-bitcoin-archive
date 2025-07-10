# Exploring Extended Relative Timelocks

pyth | 2025-07-10 04:50:02 UTC | #1


I’ve been discussing about feedbacks from Liana users with some folks at Wizardsardine about Bitcoin’s 65535-block limit for relative timelocks. Some users are curious why this limit exists and have expressed interest in longer timelocks. I’ve been experimenting with an idea to extend timelocks and would love your thoughts—*not* advocating for a soft fork, just exploring the concept to gauge its feasibility and implications.

## Context and Experiment
I looked into extending relative timelocks to ~10 years by:
- Using bit 21 in `nSequence` to signal a "long" timelock.
- If bit 21 is set and bit 22 is unset, the timelock unit is interpreted as 8 blocks, multiplying the duration by 8.
- This seems to require minimal changes to Bitcoin Core (though I’m not deeply familiar with the Core codebase, so I could be missing complexities). It would be a soft fork, as older nodes would ignore the flag and enforce a timelock 8 times shorter.

I also created a tool to craft addresses and transactions for a simple "anyone can spend after timelock" policy to test this. you can check out the experimental code and tool here:
- Code change: https://github.com/bitcoin/bitcoin/commit/a9187117ccf7224297a74f3dfebc2354ad9fe6e2
- Tool: https://github.com/pythcoiner/csv2

## Discussion Points
- **Use cases**: Are there compelling use cases for ~10-years timelocks?
- **Risks**: What risks or edge cases might arise? Are there consensus or compatibility concerns I’m overlooking?
- **Alternatives**: Can we address the (potential) need for longer relative timelocks without a soft fork ?
- **Implementation**: Does the bit 21 flag with 8-block units seem reasonable, or are there better ways to do this?

To be clear, I’m not pushing for a soft fork—just curious about the community’s take on this idea. I don’t have extensive knowledge of the Bitcoin Core codebase, so I’d especially appreciate feedback on the technical side. Thanks for any insights you can share!

-------------------------

