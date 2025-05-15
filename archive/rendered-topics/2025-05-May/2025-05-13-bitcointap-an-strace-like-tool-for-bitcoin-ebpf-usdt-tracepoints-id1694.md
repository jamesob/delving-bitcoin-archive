# Bitcointap: an strace-like tool for bitcoin ebpf USDT tracepoints

jb55 | 2025-05-13 17:43:50 UTC | #1

I thought I'd share this tool I put together, based off of @0xB10C 's peer_observer project. bitcointap is a rust library and cli tool for tapping into bitcoin-core's ebpf-usdt tracepoints. I have a demo video here to show how it works:

https://cdn.jb55.com/s/bitcointap-demo.mp4

https://github.com/jb55/bitcointap

If you're new to tracing in core I suggest checking out the tracing docs:

https://github.com/bitcoin/bitcoin/blob/8309a9747a8df96517970841b3648937d05939a3/doc/tracing.md

Let us know what traces you would find useful so we can start making this tool more powerful!

-------------------------

willcl-ark | 2025-05-15 16:23:11 UTC | #2

Nice work jb55!

I actually got halfway through writing a similar type of tool for peer-observer a few months back: https://github.com/willcl-ark/peer-observer/tree/tui

It is in somewhat of a working state; `cargo run --bin tui --release -- --nats-address my-peer-observer-host:4222` will get you something like this: https://asciinema.org/a/eMHaUhfX537iXR2IZBlulXHBx

Ultimately I only gave up on it because ratatui ended up having pretty bad performance when left long-running (I was bumping into something like this issue: https://github.com/ratatui/ratatui/issues/1338) and I didn't have the time to re-write it.

Looking forward to testing out your tool as a replacement. We need more tracepoint-oriented tools :heart:

-------------------------

jb55 | 2025-05-15 20:23:27 UTC | #3

yeah I wanted something a bit more hackable that doesn't depend on nats. The library just exposes an mpsc channel that you can use to read events from. Using that you should be able to build anything on top of it, including realtime visualizations.

I was trying to build a realtime cluster mempool visualizer at the bitcoin++ mempool hackathon and realized I needed this tool first.

-------------------------

