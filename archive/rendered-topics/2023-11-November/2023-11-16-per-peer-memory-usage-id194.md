# Per-peer memory usage

amiti | 2023-11-16 03:25:56 UTC | #1

This is a thread to discuss the memory usage of peers. 

## Interesting things:
- average case / expected memory usage per peer
- difference in average case memory usage for block-relay-only peer vs full-relay peer
- worst case memory usage for a single peer 
- worst case memory usage for a block-relay-only peer vs full-relay peer 
- how different network conditions impact the expected memory usage per peer

## Memory monitoring: 
- I have written [a patch](https://github.com/amitiuttarwar/bitcoin/pull/5) to monitor the memory usage of different types of connections. It currently reports (1) current `CNode` memory usage (2) current `Peer` memory usage & (3) max `Peer` memory usage.
- Status: the broad strokes are in place, but there are some fields that are not yet accounted for (details in PR description). Also, I would love review to know if the code actually does what I think it does!
- Desired improvement: update the max peer memory usage to incorporate `CNode` memory, so it will be a better representation of the entire connection.
- This [spreadsheet](https://docs.google.com/spreadsheets/d/1VsKj0RxhLOo2m512dq1FwP6NHkA6TcKP01OIjyHUKN8/edit?usp=sharing) are the results from running the patch on my node with inbounds enabled for approximately 5 days (nov 10-15). it shows a dramatic difference between the max memory usage of block-relay-only peers vs full-relay peers.

this graph shows memory usage broken down by max & current, for connections with relay enabled or not. 
_y axis: number of bytes, x axis: different nodes._
![Screen Shot 2023-11-15 at 7.11.34 PM|690x423](upload://wYsgujkjX8Cfk9KFfyKeB3CsyiH.png)


## Warnet: 
Warnet can help us observe different network conditions. Here are some questions I'm curious about, where I think warnet can create/isolate behaviors we sometimes see:
- When [LinkingLion](https://b10c.me/observations/06-linkinglion/) is on the prowl, there is high churn of inbound connections. How much memory does each short-lived connections use? 
- Especially with ordinals/transcriptions, we sometimes see really high transaction volume. How does that impact the expected/average memory usage of our full-relay connections? 

Do you have more ideas? :)

-------------------------

0xB10C | 2023-11-16 07:05:19 UTC | #2

[quote="amiti, post:1, topic:194"]
this graph shows memory usage broken down by max & current, for connections with relay enabled or not. *y axis: number of bytes, x axis: different nodes.*
[/quote]

Thanks for sharing this! I think the labels for `max_no_relay` and `cur_no_relay` are mixed up (max is lower than current memory usage). If I'm reading this correctly it's just below 1 MB per full-relay peer? 

[quote="amiti, post:1, topic:194"]
When [LinkingLion](https://b10c.me/observations/06-linkinglion/) is on the prowl, there is high churn of inbound connections. How much memory does each short-lived connections use?
[/quote]

Curious about this too! I'm seeing more than a 100 connections per minute to a few of my nodes. I'm wondering when the memory allocations are made. When the peer connects or during connection lifetime. I think observing memory usage over time would be interesting here too!

-------------------------

ajtowns | 2023-11-16 07:08:52 UTC | #3

[quote="0xB10C, post:2, topic:194"]
I think the labels for `max_no_relay` and `cur_no_relay` are mixed up
[/quote]

The maximums in that graph counts just the `Peer` memory usage; the current counts both `Peer` and `CNode`

-------------------------

