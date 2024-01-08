# DoS disclosure: Channel open race in CLN

morehouse | 2024-01-08 19:01:42 UTC | #1

CLN versions between [23.02](https://github.com/ElementsProject/lightning/releases/tag/v23.02) and [23.05.2](https://github.com/ElementsProject/lightning/releases/tag/v23.05.2) are susceptible to a DoS attack involving the exploitation of a race condition during channel opens.
If you are running any version in this range, your funds may be at risk!
Update to at least CLN [23.08](https://github.com/ElementsProject/lightning/releases/tag/v23.08) to help protect your node.

# The Vulnerability

The vulnerability arises from a race condition between two different flows in CLN: the channel open flow and the peer connection flow.  When the race occurs, CLN attempts to launch a `channeld` daemon twice, triggering an assertion failure and a crash.

The crash can be reliably triggered in 30 seconds using a [fake channel DoS attack](https://morehouse.github.io/lightning/fake-channel-dos/).

# Protecting Your Node

Update your node to at least [v23.08](https://github.com/ElementsProject/lightning/releases/tag/v23.08) to prevent it from crashing due to the race.

# More Details

For a full discussion about the vulnerability, its root causes, and how it could have been prevented, see my [blog post](https://morehouse.github.io/lightning/cln-channel-open-race/) about it.

-------------------------

