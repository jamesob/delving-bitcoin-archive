# Bitcointap: an strace-like tool for bitcoin ebpf USDT tracepoints

jb55 | 2025-05-13 17:43:50 UTC | #1

I thought I'd share this tool I put together, based off of @0xB10C 's peer_observer project. bitcointap is a rust library and cli tool for tapping into bitcoin-core's ebpf-usdt tracepoints. I have a demo video here to show how it works:

https://cdn.jb55.com/s/bitcointap-demo.mp4

https://github.com/jb55/bitcointap

If you're new to tracing in core I suggest checking out the tracing docs:

https://github.com/bitcoin/bitcoin/blob/8309a9747a8df96517970841b3648937d05939a3/doc/tracing.md

Let us know what traces you would find useful so we can start making this tool more powerful!

-------------------------

