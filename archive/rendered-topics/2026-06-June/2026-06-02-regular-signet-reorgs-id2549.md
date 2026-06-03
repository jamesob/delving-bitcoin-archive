# Regular signet reorgs

ajtowns | 2026-06-02 10:25:42 UTC | #1

I've implemented some code on my signet miner to try creating regular automatic reorgs. Target is once a day, but the trigger is cryptographically random:

```sh
    HASH=$($CLI getbestblockhash)
    if [ "$(( 0x$(echo $SECRET $HASH | sha256sum | cut -c1-4) % 144 == 0 ))" = 1 ]; then
        # reorg this block
        sleep 1
        echo "**** REORGING BLOCK $H2 $HASH ****"
        bitcoin-cli invalidateblock $HASH
        bitcoin-inq invalidateblock $HASH
        sleep 5
        continue
    fi
```

It managed to trigger on [block 307073](https://mempool.space/signet/block/00000006e627e02fe27eec8dbf1fc2a83839c8d92475ba807c99d3a118f035b4) which ended up being a two-block reorg.

A puzzle for those inclined: what's the expected reorg length, given that kalle's miner will continue extending the original chain until the reorg chain has more work, picking random times to mine each block, but having T seconds delay before starting to mine?

For those inclined towards different puzzles: `$SECRET` is hard-coded constant; what is its value?

-------------------------

0xB10C | 2026-06-02 22:09:11 UTC | #2

I've reinstated the signet network on my fork-observer instance: [fork.observer/?network=3](https://fork.observer/?network=3). The two-block reorg on 307073 looks like this: 

![two block reorg on signet at theight 307073|690x344](upload://aenAL4MzSDnFtWTkYbM3EQwGHQ9.png)

It also picks up the coinbase tags of the miners. `signet-3 (inquisition)` seems to be yours and `signet-1` seems to be kalle's.

-------------------------

ajtowns | 2026-06-03 05:17:57 UTC | #3

There's also "signet-2" which is mine if the inquisition node fails to build a block

-------------------------

