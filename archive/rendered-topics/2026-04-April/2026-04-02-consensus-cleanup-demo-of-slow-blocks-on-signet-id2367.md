# Consensus Cleanup: demo of slow blocks on Signet

AntoineP | 2026-04-07 21:29:17 UTC | #1

We are going to do a demo of long-to-validate blocks on Signet on Wednesday. Here are instructions on how you can join.

The goal of this demo is to let end users see for themselves the impact of hard to validate blocks. We have crafted a series of 6 blocks that are hard to validate, but [not too much](https://groups.google.com/g/bitcoindev/c/wOVjJoLDWfA/m/zWcDG7_7CQAJ). Each should take between a handful of seconds and one minute to validate, depending on the hardware. @ajtowns is going to mine the six blocks in a row and then reorg them out, so that anyone running a Signet node at the time would see them, but they would not impose a cost to IBD forever.

We are going to do three runs of the demo so everyone has a chance to join live:
1. [date=2026-04-08 time=14:00:00 timezone=Etc/UTC]
2. [date=2026-04-08 time=22:00:00 timezone=Etc/UTC]
3. [date=2026-04-09 time=09:00:00 timezone=Etc/UTC]

## How to join the demo live

All you need to join the demo is run a Bitcoin Core node on Signet. You would be able to observe how long it took your own node to validate each block, and compare arrival time with other participants. Feel free to share your results in this thread!

Your Signet node needs to be fully-synced. The historical block chain is still pretty small, you would need about 30GiB of disk space. If necessary, you can also run in `-prune` mode. It will not affect your ability to participate in the demo.

If you are not able to run a Signet node yourself, you should still be able to observe the blocks as they arrive on [mempool.space](https://mempool.space/signet) and the fork dynamics at the [temporary peer.observer instance](https://signet.peer.observer/forks/) for Signet.

### Detailed instructions

---

If you are **using a node-in-box solution** (like **Umbrel, Raspiblitz, MyNode, Nodl, Start9**, etc..), you can still follow these instructions below to join the demo from another machine. The installation should be lightweight, using less than 50 GiB of disk space and take less than a couple hours to complete. If you prefer to follow the demo using the hardware that you use for you main node (which would be interesting), see the **instructions in the following section** instead.

---

Download Bitcoin Core if you don't have it already by following the instructions [here](https://bitcoincore.org/en/download/).

Run Bitcoin Core on the default Signet network, with some additional logging options to learn more about how your node handled the slow blocks. Specific instructions depend on your platform.

<details>
<summary>Click here for MacOS</summary>

We assume you have followed the [download instructions from the Bitcoin Core website](https://bitcoincore.org/en/download/). If you followed different instructions and have a `bitcoind` binary instead, you can simply replace `Bitcoin-Qt` by `bitcoind` in the command line below.

Open a Terminal. This can be done by pressing `Command + Space`, typing "Terminal" and hitting enter.

Type the following command that starts Bitcoin Core on Signet with additional logging about validation time and p2p messages:
```
/Applications/Bitcoin-Qt.app/Contents/MacOS/Bitcoin-Qt -signet -debug=bench -debug=net
```

If you have less than 50 GiB of disk space available, add ` -prune=10000` to the command above to reduce disk space usage to about 20 GiB.

If you have less than 30 GiB of disk space available, add ` -prune=1` to the command above to reduce disk space usage to about 10 GiB.

Enter the command.

</details>

<details>
<summary>Click here for Windows</summary>

We assume you have followed the [download instructions from the Bitcoin Core website](https://bitcoincore.org/en/download/). If you followed different instructions and have a `bitcoind.exe` binary instead, you can simply replace `bitcoin-qt.exe` with `bitcoind.exe` in the command line below.

Open Command Prompt. This can be done by pressing `Windows + R`, typing `cmd`, and pressing Enter.

Type the following command, which starts Bitcoin Core on Signet with additional logging about validation time and p2p messages:
```
"C:\Program Files\Bitcoin\bitcoin-qt.exe" -signet -debug=bench -debug=net
```

If you have less than 50 GiB of disk space available, add ` -prune=10000` to the command above to reduce disk space usage to about 20 GiB.

If you have less than 30 GiB of disk space available, add ` -prune=1` to the command above to reduce disk space usage to about 10 GiB.

Press Enter to run the command.

</details>

<details>
<summary>Click here for Linux</summary>

Instructions differ slightly depending on the Linux distribution you are running. If you know how to run Bitcoin Core on Signet you can skip this section. Here are some guidelines if you are not sure.

The [Bitcoin Core download instructions](https://bitcoincore.org/en/download/) will have you Download a `.tar.gz` archive. Extract the archive to access the `bitcoind` binary. Here is an example on Debian with version 30.2:
```
cd Downloads/
tar -xzf bitcoin-30.2-x86_64-linux-gnu.tar.gz
```

Run Bitcoin Core on Signet by adding the `-debug=bench` and `-debug=net` startup options. You may want to set the `-prune` option if you have less than 50 GiB of available disk space. Here is an example following the extraction of the archive above:
```
./bitcoin-30.2/bin/bitcoind -signet -debug=bench -debug=net
```

</details>

Bitcoin Core will synchronize with the Signet network. This could take anywhere between a few minutes and a couple hours depending on your hardware, internet connection and whether you set the `prune` option.

During the demo, you can observe block validation time as well as propagation dynamics by monitoring Bitcoin Core's log file: `debug.log`. Its location depends on your platform.

<details>
<summary>Click here for MacOS</summary>

To follow the blocks live, you can use the `tail -f` command. It will print the last lines of the `debug.log` file and automatically refresh as more are added to it. After the demo, if you are looking for a specific block, or the timestamp of some p2p message, you can use the `less` command on the `debug.log` file instead. You may also want to explore the log file using graphical tools as well, but those are out of scope for this small instructional.

Make sure Bitcoin Core is still running. (If you followed the instructions above, it must still be running in the first terminal you started.)

Open a new Terminal. This can be done by pressing `Command + Space`, typing "Terminal" and hitting enter.

Use this command to display an automatically refreshed view of the latest Bitcoin Core logs:
```
tail -n 50 -f ~/Library/Application\ Support/Bitcoin/signet/debug.log
```

To search for a past event you may use your favorite graphical tool or the `less` command:
```
~/Library/Application\ Support/Bitcoin/signet/debug.log
```

</details>

<details>
<summary>Click here for Windows</summary>

On Windows, Bitcoin Core's log file for Signet is located here:
```
%APPDATA%\Bitcoin\signet\debug.log
```

A simple way to access it is to press `Windows + R`, copy paste the above, and press enter. It will open a file browser at the location of the `debug.log`. You can then use your favourite tool to inspect its content.

*If anyone with Windows experience wants to contribute better instructions, please shoot.*

</details>


<details>
<summary>Click here for Windows</summary>

On Linux, Bitcoin Core's log file for Signet is located at:
```
~/.bitcoin/signet/debug.log
```

You can monitor it live with:
```
tail -n 50 -f ~/.bitcoin/signet/debug.log
```

You can search for specific content (block hashes, block heights, p2p messages, etc..) with:
```
less ~/.bitcoin/signet/debug.log
```

</details>

*I may add more information about what to look for in the logs later.*

### Detailed instructions for node-in-a-box solutions

<details>

<summary>Joining the demo using an Umbrel node</summary>

**NOTE**: Running the Bitcoin app on Signet will stop your mainnet node.

The Bitcoin Node app on Umbrel supports Signet. It can be configured in `Settings > Network Selection > Signet`.

![image|479x500](upload://8ZjDbHPdrOiMVReOtcxwdRX4Pzb.jpeg)

You can enable more logging about block validation time and p2p messages by customizing the Bitcoin Core configuration in the `Advanced` tab on the same panel. To do that (optional), write the following lines:
```
debug=bench
debug=net
```

![image|471x500](upload://kRfVS0Dmk6zuBq4jMUUGKmjW7oU.jpeg)

You can observe validation time and propagation dynamics of the slow blocks by monitoring Bitcoin Core's log file: `debug.log`. The most straightforward way of achieving this in Umbrel would be to run a Terminal for the Bitcoin Node app, at `Settings > Advanced > Terminal > App > Bitcoin Node`.
![image|690x425](upload://l2ldkJVfNozMxBMFr27hZhX8JZW.jpeg)

Then you can for instance poll latest events using `tail -f /data/bitcoin/debug.log`:
![image|690x388](upload://59I7ptqkX0GvbQxJGyCxACaiukK.jpeg)

Alternately, if you are looking for a past event, you can use the `less` command and search through the file by hitting `/`, typing the search query and hitting enter (hit `n` to jump to the next match).

</details>

<details>

<summary>Joining the demo using a Raspiblitz node</summary>

*h/t Openoms*

Syncing signet on an RPi5 should take less than an hour (but allow a little more time just in case).

You can start the signet bitcoind instance on a Raspiblitz by running:
```
config.scripts/bitcoin.install.sh on signet
```

You will have these aliases set in terminal for easy monitoring and configuration:
```
alias sbitcoin-cli="sudo -u bitcoin /usr/local/bin/bitcoin-cli -rpcport=38332"
alias sbitcoinlog="sudo -u bitcoin tail -n 30 -f /mnt/hdd/app-data/bitcoin/signet/debug.log"
alias bitcoinconf="sudo nano /mnt/hdd/app-data/bitcoin/bitcoin.conf"
```

The systemd service is called: `sbitcoind` so can check and restart with:
```
systemctl status sbitcoind
sudo systemctl restart sbitcoind
```

</details>

-------------------------

shinobi | 2026-04-02 20:58:49 UTC | #2

Just wanted to suggest vibe coding a basic GUI/display to pull from the logs and display to a user validation times as the blocks come in. I think without an easy visual component there isn’t much to hook many users to directly spin up a signet node and participate with their own node.

-------------------------

0xB10C | 2026-04-02 22:52:22 UTC | #3

[quote="AntoineP, post:1, topic:2367"]
* Wednesday April 8th at **2pm UTC** (7am SF / 10am NY / 4pm Paris / 1am Sydney)
* Wednesday April 8th at **10pm UTC** (3pm SF / 6pm NY / midnight Paris / 9am Sydney)
* Thursday April 9th at **9am UTC** (2am SF / 5am NY / 11am Paris / 8pm Sydney)
[/quote]

In your timezone, this is:
1. [date=2026-04-08 time=14:00:00 timezone=Etc/UTC]
2. [date=2026-04-08 time=22:00:00 timezone=Etc/UTC]
3. [date=2026-04-09 time=09:00:00 timezone=Etc/UTC]

Maybe adding this to the OP makes sense too.

-------------------------

AntoineP | 2026-04-03 16:47:41 UTC | #4

I guess this is also a good occasion for testing Bitcoin Core's latest release candidate. If you are joining the demo and comfortable with running Bitcoin Core already, consider using [the binaries from the latest "rc" for 31.0](https://bitcoincore.org/bin/bitcoin-core-31.0/).

-------------------------

ajtowns | 2026-04-03 18:08:34 UTC | #5

[quote="shinobi, post:2, topic:2367"]
a basic GUI/display to pull from the logs and display to a user validation times as the blocks come in.
[/quote]

Here's a patch for bitcoin-tui that people might find interesting to try:

https://github.com/janb84/bitcoin-tui/pull/25

This is what a 5-block reorg looks like in it (without any of the blocks being slow):

![tui-slow-blocks|690x459](upload://pMfcvaF4GvFukmM5Qk0CGq4gwxJ.png)

Enabling `-debug=cmpctblock` and `-debug=bench` logging are good and not very expensive and are required to fill out some of the columns; enabling `-debug=net` logging as well might also gather some interesting info.

-------------------------

AntoineP | 2026-04-06 18:25:00 UTC | #6

Updated OP to add instructions on how to join the demo on MacOS, Windows and Linux. These instructions are aimed at non-technical users who would like to participate and i could use review to make sure they are (correct and) clear enough.

I also added instructions on how to join using Umbrel with screenshots, courtesy of @lukechilds. I would welcome instructions on how to join using other node-in-a-box solutions.

-------------------------

AntoineP | 2026-04-06 20:43:11 UTC | #7

Openoms just shared instructions on how to join using Raspiblitz ([X](https://x.com/openoms/status/2041254429555339502), [Telegram](https://t.me/raspiblitz/153788#)). I'll update OP with the instructions.

-------------------------

shinobi | 2026-04-06 22:20:40 UTC | #8

Seems to be working perfectly.

-------------------------

ekzyis | 2026-04-06 23:15:13 UTC | #9

If you’re already synced to a different signet and don’t want to lose your datadir when switching to the default signet, you could check out [my PR](https://github.com/bitcoin/bitcoin/pull/34566).

-------------------------

AntoineP | 2026-04-07 14:41:19 UTC | #10

It's fair to plug your PR since it could have been useful in this context, but i think for a non-Bitcoin-Core-contributor a better instruction is to use the `-datadir` option rather than compiling your branch.

-------------------------

ariard | 2026-04-07 23:14:41 UTC | #11

thanks for setting up the demo, will do my best to join / observe the Thursday session.

-------------------------

0xB10C | 2026-04-08 13:52:56 UTC | #12

I'm streaming the first run on YouTube: 


https://youtube.com/live/Zx20r7QAtEM

-------------------------

Emzy | 2026-04-08 14:14:08 UTC | #13

Running Bitcoin-TUI with the patch on a thin PC:

![Screenshot 2026-04-08 at 16.12.17|690x465](upload://eY8rrTij5mPG1VneKJ4pvP13xlP.png)

-------------------------

AntoineP | 2026-04-08 14:20:08 UTC | #14

The 6 slow blocks for the first run:
- `0000000eb552c9f26e712d546c71297fd0623890299b40e7ada81d2dc32f5d0b`
- `000000002b3a132836666c18f5e1a9d93623d3797316a968ee54e47fb44c0c13`
- `00000006d34037534a517f9e5809a34766f1540c0e6817eac91b1adfee50cb5f`
- `00000014a4cae4501f98539b45c76059c706a82b77f19a9adf365b3f5e989444`
- `00000003220437cb8b5a2edef6be828c5cdad114b1b642d724ac6f3caa7f12fb`
- `000000143c97bf0134c5cf0881dfd4ef458529b7388cacf43981ffe92fb96856`

Fork has started to be produced (first conflicting block is `0000000880318f1fabf0758f470564742b41652ad17e11af4d8a2df6930233e1`), should reorg in about an hour.
![image|690x385](upload://i3IJVbSNVd2W7vCGrER4SlmCKTg.png)

-------------------------

svanstaa | 2026-04-08 14:33:55 UTC | #15

Again Bitcoin-TUI with patch, but on a rather fast PC:

![image|690x321](upload://dWWAeJmTrMyv8fb251NKdxFA4aB.png)

-------------------------

m3dwards | 2026-04-08 14:32:59 UTC | #16

Times on a M1 Mac with core v30.2:

I got an additional slow block after those 6.

| Height | Block | Time | 
|----|----|----|
| 299177 | 0000000eb552c9f26e712d546c71297fd0623890299b40e7ada81d2dc32f5d0b | 2.835s |
| 299178 | 000000002b3a132836666c18f5e1a9d93623d3797316a968ee54e47fb44c0c13 | 45.257s |
| 299179 | 00000006d34037534a517f9e5809a34766f1540c0e6817eac91b1adfee50cb5f | 4.552s |
| 299180 | 00000014a4cae4501f98539b45c76059c706a82b77f19a9adf365b3f5e989444 | 3.836s |
| 299181 | 00000003220437cb8b5a2edef6be828c5cdad114b1b642d724ac6f3caa7f12fb | 3.587s |
| 299182 | 000000143c97bf0134c5cf0881dfd4ef458529b7388cacf43981ffe92fb96856 | 29.273 |
| 299183 | 000000079e5f6f5376bd51b5d26fb2e27dd8762c4cac3936380647cc43377ac6 | 45.567s |
| 299184 | 000000040a3cdd0a545322b1d08812bd9b251cc4a49afaf8dccdb9c1ff960f49 | 0.033s |

-------------------------

xyzconstant | 2026-04-08 15:21:17 UTC | #17

I have taken this screenshot on the TUI connected to my signet node (NanoPC-T6, Ubuntu 22.04.2 LTS aarch64) with Core 31.0rc2:

![image|690x419](upload://kZF5WVdsjzes9RY5Sxc38MM7jis.jpeg)

EDIT: forgot to mention I’m running Core 31.0rc2

-------------------------

openoms | 2026-04-08 15:00:46 UTC | #18

Running Raspiblitz v1.12.1 with BItcoin Core 29.2.0 on a Raspberry Pi 5 booted from nvme ssd.

![image|690x189](upload://1spmusuKURMlooFY97OPclubsA1.png)

Have also put together a quick install intstruction to get the patched bitcoin-tui running:
```
git clone https://github.com/ajtowns/bitcoin-tui
cd bitcoin-tui
git checkout 202604-bip54blocks

sudo apt install cmake

cmake -B build
cmake --build build -j$(nproc)

sudo cp ./build/bin/bitcoin-tui /usr/bin/

bitcoin-tui --signet --user raspibolt --password PASSWORD_B
```

-------------------------

ViniciusCestarii | 2026-04-08 15:08:57 UTC | #19

Bitcoin Core version: Satoshi:29.1.0

OS: Arch Linux

CPU: Intel Core i7-13650HX (13th Gen, 20 cores, up to 4.90 GHz)

![Screenshot From 2026-04-08 11-18-15|456x500](upload://9hnfJ2PWd62gUgK2LhloCOhemPB.png)

-------------------------

AntoineP | 2026-04-08 15:13:18 UTC | #20

Each block took around 5 seconds to validate on my laptop with a performant CPU. When has everyone first seen 1) the headers 2) the content of the blocks? Was anyone stuck with peers that took a few seconds to send the block content as they were still validating it?

-------------------------

xyzconstant | 2026-04-08 15:19:22 UTC | #21

Running on Mac M3 pro with Core 31.0rc2, here are some results:

![image|690x392](upload://54qzPDnqHNy0W4R7cpukU19gXJQ.jpeg)

EDIT: forgot to mention I’m running Core 31.0rc2

-------------------------

openoms | 2026-04-08 15:24:37 UTC | #22

And they are gone:
![image|690x350](upload://puMUOwJbskZuXhIQ41zlzHEKRVw.jpeg)

-------------------------

0300dbdd1b | 2026-04-08 15:34:05 UTC | #23

Core v30.2
AMD Ryzen 5 7640U

![image|690x146](upload://sVQJB6ued2zlMoHYSgK5IMAhUxz.png)

-------------------------

andrewtoth | 2026-04-08 15:38:12 UTC | #24

Under 3 seconds for each on my i9 laptop:

```
2026-04-08T14:02:49Z [bench] - Connect block: 7.23ms [1.44s (13.11ms/blk)]
2026-04-08T14:05:47Z [bench] - Connect block: 2709.97ms [4.15s (37.41ms/blk)]
2026-04-08T14:06:42Z [bench] - Connect block: 2752.88ms [6.91s (61.65ms/blk)]
2026-04-08T14:07:26Z [bench] - Connect block: 2689.89ms [9.60s (84.91ms/blk)]
2026-04-08T14:08:32Z [bench] - Connect block: 2690.76ms [12.29s (107.77ms/blk)]
2026-04-08T14:09:31Z [bench] - Connect block: 2709.28ms [15.00s (130.39ms/blk)]
2026-04-08T14:10:59Z [bench] - Connect block: 2823.50ms [17.82s (153.61ms/blk)]
2026-04-08T14:10:59Z [bench] - Connect block: 2.29ms [17.82s (152.32ms/blk)]

```

-------------------------

jaonoctus | 2026-04-09 11:22:10 UTC | #25

Running Core v30.2 / VM (QEMU/KVM) on AMD EPYC host 6 cores 2.0 GHz

## 1st run

```
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA512

2026-04-08T14:04:50Z Saw new cmpctblock header hash=0000000eb552c9f26e712d546c71297fd0623890299b40e7ada81d2dc32f5d0b height=299177 peer=9
2026-04-08T14:05:13Z [bench] - Connect block: 22224.25ms [32.59s (264.99ms/blk)]
2026-04-08T14:06:20Z Saw new cmpctblock header hash=000000002b3a132836666c18f5e1a9d93623d3797316a968ee54e47fb44c0c13 height=299178 peer=2
2026-04-08T14:06:48Z [bench] - Connect block: 21880.74ms [54.48s (439.32ms/blk)]
2026-04-08T14:07:09Z Saw new cmpctblock header hash=00000006d34037534a517f9e5809a34766f1540c0e6817eac91b1adfee50cb5f height=299179 peer=2
2026-04-08T14:07:34Z [bench] - Connect block: 21385.88ms [75.86s (606.89ms/blk)]
2026-04-08T14:08:15Z Saw new cmpctblock header hash=00000014a4cae4501f98539b45c76059c706a82b77f19a9adf365b3f5e989444 height=299180 peer=2
2026-04-08T14:08:41Z [bench] - Connect block: 22418.28ms [98.28s (779.99ms/blk)]
2026-04-08T14:09:15Z Saw new cmpctblock header hash=00000003220437cb8b5a2edef6be828c5cdad114b1b642d724ac6f3caa7f12fb height=299181 peer=2
2026-04-08T14:09:41Z [bench] - Connect block: 21803.17ms [120.08s (945.53ms/blk)]
2026-04-08T14:10:41Z Saw new cmpctblock header hash=000000143c97bf0134c5cf0881dfd4ef458529b7388cacf43981ffe92fb96856 height=299182 peer=2
2026-04-08T14:11:07Z [bench] - Connect block: 22332.90ms [142.42s (1112.62ms/blk)]
-----BEGIN PGP SIGNATURE-----

iHUEARYKAB0WIQTru4F1HbgJ3lDdWMSsdchrbudDNAUCadZ6MwAKCRCsdchrbudD
NADeAQDYJxAczHZbltdx1XrPalJgj11rhumbZ+5YLBg8VgCo+QEAn66on4M+3odh
2WSxdMogdtva6KReACn2sIbz022vdgE=
=XP+R
-----END PGP SIGNATURE-----
```

## 2nd run

```
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA512

2026-04-08T22:03:53Z Saw new cmpctblock header hash=00000011ee1088ab58ade91da86761478799b685699b30bc649c83f13725bc6d height=299227 peer=6
2026-04-08T22:04:16Z [bench] - Connect block: 22313.95ms [24.63s (482.99ms/blk)]
2026-04-08T22:05:06Z Saw new header hash=000000037a35ebf60c59619042ece8ae71fecf4d3146098c966f820482d33955 height=299228 peer=9
2026-04-08T22:05:29Z [bench] - Connect block: 22190.23ms [46.82s (900.44ms/blk)]
2026-04-08T22:06:33Z Saw new cmpctblock header hash=0000001253d4dbf6b7b34079a70ef0357a7b12e83a38f29946effbc863f194f3 height=299229 peer=9
2026-04-08T22:06:59Z [bench] - Connect block: 21997.81ms [68.82s (1298.50ms/blk)]
2026-04-08T22:07:22Z Saw new cmpctblock header hash=0000000b4f9d1ba885832b1b9dd4f6183483a27a4762a48114c00a812f1b42dd height=299230 peer=9
2026-04-08T22:07:47Z [bench] - Connect block: 21994.78ms [90.82s (1681.77ms/blk)]
2026-04-08T22:08:10Z Saw new cmpctblock header hash=00000011240c362604a624fc4469413f36ff5cb823c0733d570d54c7916a913f height=299231 peer=9
2026-04-08T22:08:36Z [bench] - Connect block: 22012.46ms [112.83s (2051.42ms/blk)]
2026-04-08T22:09:51Z Saw new cmpctblock header hash=000000065d332b249b5fc8068177776b3dddb073d308ae5b090147c41477e351 height=299232 peer=9
2026-04-08T22:10:16Z [bench] - Connect block: 21914.79ms [134.74s (2406.12ms/blk)]
-----BEGIN PGP SIGNATURE-----

iHUEARYKAB0WIQTru4F1HbgJ3lDdWMSsdchrbudDNAUCadbXZAAKCRCsdchrbudD
NKwIAQCN3SxUjNYwssPcVZjddXCpttnQUUheNSXnuiHPxtWQSgEArZkybgTK8HaX
CeWz5aAeAfKqfNCHTSHMxmE8mZfr2g0=
=bhIT
-----END PGP SIGNATURE-----
```

## 3rd run

```
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA512

2026-04-09T09:02:48Z Saw new cmpctblock header hash=0000000db638fd84f3668c983cdf3f305a098c61d0bcbc657a15250b081efdd4 height=299296 peer=6
2026-04-09T09:03:11Z [bench] - Connect block: 22376.06ms [159.47s (1255.64ms/blk)]
2026-04-09T09:04:17Z Saw new cmpctblock header hash=00000013079c637e2d4b37cb95f019479a4e86a041e9eee162c6e2ae1758ed9d height=299297 peer=9
2026-04-09T09:04:43Z [bench] - Connect block: 22900.03ms [182.37s (1424.73ms/blk)]
2026-04-09T09:05:12Z Saw new cmpctblock header hash=0000000685879728d212116865fcf49d3b108588420f737edee8c331434bd407 height=299298 peer=9
2026-04-09T09:05:38Z [bench] - Connect block: 22418.36ms [204.78s (1587.47ms/blk)]
2026-04-09T09:06:21Z Saw new cmpctblock header hash=00000004af57cad1f5cd0563db9b4fd7dc29f4de92b48aecba7ff7e29d68e1c0 height=299299 peer=9
2026-04-09T09:06:47Z [bench] - Connect block: 22135.96ms [226.92s (1745.54ms/blk)]
2026-04-09T09:07:55Z Saw new cmpctblock header hash=0000000a0ee0a258ea2fbb2d2b0d465374885b2483ae4d175cc44aa843c63147 height=299300 peer=9
2026-04-09T09:08:21Z [bench] - Connect block: 21910.10ms [248.83s (1899.47ms/blk)]
2026-04-09T09:09:45Z Saw new cmpctblock header hash=0000001018d0038f11974c98548fca64686380c21052a856cb8c4790cf6e50ce height=299301 peer=9
2026-04-09T09:10:12Z [bench] - Connect block: 23243.72ms [272.07s (2061.17ms/blk)]
-----BEGIN PGP SIGNATURE-----

iHUEARYKAB0WIQTru4F1HbgJ3lDdWMSsdchrbudDNAUCadeLVgAKCRCsdchrbudD
NHOYAPwOTuvvy8uHmMGZCbYOXaBIvMjJmNtuToaGKb9EBjlNcQD9GMAzZpHPypNN
rPzvG4lOUADRjqjCGkITrYrmDcf3AA4=
=3lju
-----END PGP SIGNATURE-----
```

-------------------------

guibressan | 2026-04-08 16:03:59 UTC | #26

Running in an old laptop (Core i5-3337U)

```
btcd(CGO secp256k1 fork):
2026-04-08 14:11:10.466 [DBG] SYNC: processed block 299177 0000000eb552c9f26e712d546c71297fd0623890299b40e7ada81d2dc32f5d0b in 352245323324ns
2026-04-08 14:17:10.793 [DBG] SYNC: processed block 299178 000000002b3a132836666c18f5e1a9d93623d3797316a968ee54e47fb44c0c13 in 357830036109ns
2026-04-08 14:23:10.134 [DBG] SYNC: processed block 299179 00000006d34037534a517f9e5809a34766f1540c0e6817eac91b1adfee50cb5f in 357992482187ns
2026-04-08 14:29:09.598 [DBG] SYNC: processed block 299180 00000014a4cae4501f98539b45c76059c706a82b77f19a9adf365b3f5e989444 in 359395937583ns
2026-04-08 14:35:11.284 [DBG] SYNC: processed block 299181 00000003220437cb8b5a2edef6be828c5cdad114b1b642d724ac6f3caa7f12fb in 361630971097ns
2026-04-08 14:41:13.978 [DBG] SYNC: processed block 299182 000000143c97bf0134c5cf0881dfd4ef458529b7388cacf43981ffe92fb96856 in 360412155103ns

btcd(master):
2026-04-08 14:11:06.372 [DBG] SYNC: processed block 299177 0000000eb552c9f26e712d546c71297fd0623890299b40e7ada81d2dc32f5d0b in 352501623920ns
[2026-04-08 14:17:05.635 [DBG] SYNC: processed block 299178 000000002b3a132836666c18f5e1a9d93623d3797316a968ee54e47fb44c0c13 in 355806456969ns
2026-04-08 14:23:06.752 [DBG] SYNC: processed block 299179 00000006d34037534a517f9e5809a34766f1540c0e6817eac91b1adfee50cb5f in 357741226734ns
2026-04-08 14:29:10.040 [DBG] SYNC: processed block 299180 00000014a4cae4501f98539b45c76059c706a82b77f19a9adf365b3f5e989444 in 360135046981ns
2026-04-08 14:41:11.116 [DBG] SYNC: processed block 299181 00000003220437cb8b5a2edef6be828c5cdad114b1b642d724ac6f3caa7f12fb in 719982861243ns
2026-04-08 14:47:41.621 [DBG] SYNC: processed block 299182 0000000d02c0b55daa763a8e5c31083e0d93a1f3927a74ced5a250d4e4bf259a in 388900151ns

```

-------------------------

AntoineP | 2026-04-08 19:54:24 UTC | #27

Here is for each block the delay between my node first saw the block hash and when it finally received the block content.

| block hash | delay (seconds) |
|----------------|-----------------------|
| `0000000eb552c9f26e712d546c71297fd0623890299b40e7ada81d2dc32f5d0b` | 2 |
| `000000002b3a132836666c18f5e1a9d93623d3797316a968ee54e47fb44c0c13` | 44 |
| `00000006d34037534a517f9e5809a34766f1540c0e6817eac91b1adfee50cb5f` | 43 |
| `00000014a4cae4501f98539b45c76059c706a82b77f19a9adf365b3f5e989444` | 44 |
| `00000003220437cb8b5a2edef6be828c5cdad114b1b642d724ac6f3caa7f12fb` | 43 |
| `000000143c97bf0134c5cf0881dfd4ef458529b7388cacf43981ffe92fb96856` | 5 |

Here is a vibecoded Python script that takes as argument the path to your debug.log and a list of block hashes and outputs the above.

<details>

<summary>See script</summary>

```python
#!/usr/bin/env python3
import argparse
import re
from datetime import datetime, timezone


HEADER_RE = re.compile(
    r"Saw new (?:cmpctblock )?header hash=([0-9a-fA-F]+)"
)
RECEIVED_RE = re.compile(r"received: (?:blocktxn|block)\b")


def parse_ts(line: str) -> datetime | None:
    first = line.split(" ", 1)[0]
    try:
        return datetime.strptime(first, "%Y-%m-%dT%H:%M:%SZ").replace(tzinfo=timezone.utc)
    except ValueError:
        return None


def main() -> None:
    parser = argparse.ArgumentParser(
        description=(
            "For each given block hash, report the time difference between "
            "'Saw new [cmpctblock] header hash=<hash>' and the first following "
            "'received: blocktxn' or 'received: block' line."
        )
    )
    parser.add_argument("logfile", help="Path to debug.log")
    parser.add_argument("hashes", nargs="+", help="One or more block hashes")
    args = parser.parse_args()

    wanted = set(args.hashes)

    # hash -> timestamp of matching header line
    saw_times: dict[str, datetime] = {}

    # hash -> timestamp of first following matching received line
    recv_times: dict[str, datetime] = {}

    # hashes currently waiting for a following received line
    pending: dict[str, datetime] = {}

    with open(args.logfile, "r", encoding="utf-8", errors="replace") as f:
        for line in f:
            ts = parse_ts(line)
            if ts is None:
                continue

            m = HEADER_RE.search(line)
            if m:
                h = m.group(1)
                if h in wanted and h not in saw_times:
                    saw_times[h] = ts
                    pending[h] = ts
                continue

            if RECEIVED_RE.search(line):
                # assign this received line to all hashes currently pending
                for h in list(pending):
                    if h not in recv_times:
                        recv_times[h] = ts
                    del pending[h]

                if len(recv_times) == len(wanted):
                    break

    for h in args.hashes:
        saw = saw_times.get(h)
        recv = recv_times.get(h)

        if saw is None:
            print(f"{h} missing:header")
        elif recv is None:
            print(f"{h} missing:received")
        else:
            delta = (recv - saw).total_seconds()
            print(f"{h} {delta:.0f}")


if __name__ == "__main__":
    main()
```

</details>

-------------------------

AntoineP | 2026-04-08 21:40:46 UTC | #28

Next run in 20 minutes!

-------------------------

0xB10C | 2026-04-08 21:44:50 UTC | #29

Stream for 2nd and 3rd run is here:

https://youtube.com/live/gHnrAlVA35A

-------------------------

instagibbs | 2026-04-08 22:22:23 UTC | #30

time are announcement to receiving the block, and validation respectively. 

11th Gen Intel(R) Core™ i7-1185G7 @ 3.00GHz

![image|690x129](upload://rDstAAp1rrWyc2oz6LLRfv35jdl.png)

-------------------------

m3dwards | 2026-04-08 22:16:09 UTC | #31

![Screenshot 2026-04-08 at 18.14.24|690x233](upload://3iDfL9mNatL3J1oaTY9ZL31kLm6.png)

-------------------------

0300dbdd1b | 2026-04-08 22:23:47 UTC | #32

Core v30.2 AMD Ryzen 5 7640U

![Screenshot from 2026-04-09 00-20-55|690x136](upload://k6WaYIr5qaqHXa3VqsGDJfhNrjC.png)

-------------------------

xstoicunicornx | 2026-04-09 00:22:11 UTC | #33

raspberry pi 4 B - Bitcoin Core v30.2.0

from when block was reconstructed to when tip was updated

| block hash | delay (seconds) |
|----|----|
| `0000000eb552c9f26e712d546c71297fd0623890299b40e7ada81d2dc32f5d0b` | 78 |
| `000000002b3a132836666c18f5e1a9d93623d3797316a968ee54e47fb44c0c13` | 77 |
| `00000006d34037534a517f9e5809a34766f1540c0e6817eac91b1adfee50cb5f` | 77 |
| `00000014a4cae4501f98539b45c76059c706a82b77f19a9adf365b3f5e989444` | 77 |
| `00000003220437cb8b5a2edef6be828c5cdad114b1b642d724ac6f3caa7f12fb` | 74 |
| `000000143c97bf0134c5cf0881dfd4ef458529b7388cacf43981ffe92fb96856` | 76 |

is there copy/paste-able list of second run hashes?

[details="your script was broken for me so I tweaked it"]
```python
#!/usr/bin/env python3
import argparse
import re
from datetime import datetime, timezone


HEADER_RE = re.compile(
    #r"Saw new (?:cmpctblock )?header hash=([0-9a-fA-F]+)"
    r"Reconstructed block ([0-9a-fA-F]+)"

)
#RECEIVED_RE = re.compile(r"received: (?:blocktxn|block)\b")
RECEIVED_RE = re.compile(r"UpdateTip: new best=([0-9a-fA-F]+)")

def parse_ts(line: str) -> datetime | None:
    first = line.split(" ", 1)[0]
    try:
        return datetime.strptime(first, "%Y-%m-%dT%H:%M:%SZ").replace(tzinfo=timezone.utc)
    except ValueError:
        return None


def main() -> None:
    parser = argparse.ArgumentParser(
        description=(
            "For each given block hash, report the time difference between "
            "'Saw new [cmpctblock] header hash=<hash>' and the first following "
            "'received: blocktxn' or 'received: block' line."
        )
    )
    parser.add_argument("logfile", help="Path to debug.log")
    parser.add_argument("hashes", nargs="+", help="One or more block hashes")
    args = parser.parse_args()

    wanted = set(args.hashes)

    # hash -> timestamp of matching header line
    saw_times: dict[str, datetime] = {}

    # hash -> timestamp of first following matching received line
    recv_times: dict[str, datetime] = {}

    with open(args.logfile, "r", encoding="utf-8", errors="replace") as f:
        for line in f:
            ts = parse_ts(line)
            if ts is None:
                continue

            m = HEADER_RE.search(line)
            if m:
                h = m.group(1)
                if h in wanted and h not in saw_times:
                    saw_times[h] = ts
                continue

            r = RECEIVED_RE.search(line)
            if r:
                # assign this received line to all hashes currently pending
                h = r.group(1)
                if h not in recv_times:
                    recv_times[h] = ts

                if len(recv_times) == len(wanted):
                    break

    print("| block hash | delay (seconds) |")
    print("|------------|-----------------|")
    for h in args.hashes:
        saw = saw_times.get(h)
        recv = recv_times.get(h)

        if saw is None:
            print(f"{h} missing:header")
        elif recv is None:
            print(f"{h} missing:received")
        else:
            delta = (recv - saw).total_seconds()
            print(f"| `{h}` | {delta:.0f} |")


if __name__ == "__main__":
    main()

```

[/details]

-------------------------

m3dwards | 2026-04-09 00:59:35 UTC | #34

No idea why randomly my machine seems to spend a lot more time verifying. Some slow blocks take a few seconds and then I had one take 78 seconds. Not sure what can be gleaned but here are the logs:

```shell
2026-04-08T22:09:46Z Saw new header hash=000000065d332b249b5fc8068177776b3dddb073d308ae5b090147c41477e351 height=299232 peer=6
2026-04-08T22:09:46Z [net] Requesting block 000000065d332b249b5fc8068177776b3dddb073d308ae5b090147c41477e351 from  peer=6
2026-04-08T22:09:46Z [net] sending getdata (37 bytes) peer=6
2026-04-08T22:09:46Z [net] received: cmpctblock (358 bytes) peer=6
2026-04-08T22:09:46Z [cmpctblock] Initializing PartiallyDownloadedBlock for block 000000065d332b249b5fc8068177776b3dddb073d308ae5b090147c41477e351 using a cmpctblock of 358 bytes
2026-04-08T22:09:46Z [cmpctblock] Initialized PartiallyDownloadedBlock for block 000000065d332b249b5fc8068177776b3dddb073d308ae5b090147c41477e351 using a cmpctblock of 358 bytes
2026-04-08T22:09:46Z [net] sending getblocktxn (34 bytes) peer=6
2026-04-08T22:09:46Z [net] received: blocktxn (999590 bytes) peer=6
2026-04-08T22:09:46Z [cmpctblock] Successfully reconstructed block 000000065d332b249b5fc8068177776b3dddb073d308ae5b090147c41477e351 with 1 txn prefilled, 0 txn from mempool (incl at least 0 from extra pool) and 1 txn (999557 bytes) requested
2026-04-08T22:09:46Z [cmpctblock] Reconstructed block 000000065d332b249b5fc8068177776b3dddb073d308ae5b090147c41477e351 required tx ad98c0d37b53f21b4c549d7cb3c19b356c6b1d4cec0e094d2fb3732402699055
2026-04-08T22:09:46Z [bench]   - Using cached block
2026-04-08T22:09:46Z [bench]   - Load block from disk: 0.05ms
2026-04-08T22:09:46Z [bench]     - Sanity checks: 0.01ms [0.00s (0.00ms/blk)]
2026-04-08T22:09:47Z [bench]     - Fork checks: 302.56ms [1.18s (25.72ms/blk)]
2026-04-08T22:10:06Z [bench]       - Connect 2 transactions: 19052.84ms (9526.418ms/tx, 47.513ms/txin) [19.86s (431.67ms/blk)]
2026-04-08T22:11:05Z [bench]     - Verify 401 txins: 78439.78ms (195.610ms/txin) [96.80s (2104.26ms/blk)]
2026-04-08T22:11:05Z [bench]     - Write undo data: 0.51ms [0.02s (0.39ms/blk)]
2026-04-08T22:11:05Z [bench]     - Index writing: 0.01ms [0.00s (0.01ms/blk)]
2026-04-08T22:11:05Z [net] sending sendcmpct (9 bytes) peer=6
2026-04-08T22:11:05Z [bench]   - Connect total: 78743.01ms [98.00s (2130.43ms/blk)]
2026-04-08T22:11:05Z [bench]   - Flush: 40.20ms [0.26s (5.55ms/blk)]
2026-04-08T22:11:05Z [bench]   - Writing chainstate: 0.07ms [0.00s (0.02ms/blk)]
2026-04-08T22:11:05Z UpdateTip: new best=000000065d332b249b5fc8068177776b3dddb073d308ae5b090147c41477e351 height=299232 version=0x20000000 log2_work=43.631489 tx=29156012 date='2026-04-08T22:07:50Z' progress=1.000000 cache=88.0MiB(628908txo)
2026-04-08T22:11:05Z [bench]   - Connect postprocess: 0.06ms [0.75s (16.41ms/blk)]
2026-04-08T22:11:05Z [bench] - Connect block: 78783.38ms [99.01s (2152.42ms/blk)]

```

v30.2 on an M1 Mac

-------------------------

xyzconstant | 2026-04-09 09:20:56 UTC | #35

M3 pro with v31.0rc2 

![image|690x415](upload://lEpVFGHdkpkVRyzh0kWrUdhERAR.jpeg)

-------------------------

0300dbdd1b | 2026-04-09 09:21:34 UTC | #36

Core v30.2 AMD Ryzen 5 7640U

![Screenshot from 2026-04-09 11-18-54|690x125](upload://oWVQZwZGqTLUcR6ziMrxlf53xA0.png)

-------------------------

xyzconstant | 2026-04-09 09:50:01 UTC | #37

**Hardware:** FriendlyElec NanoPC-T6 (Rockchip RK3588)

* CPU: 8-core ARM (4× Cortex-A76 @ 2.4 GHz + 4× Cortex-A55 @ 1.8 GHz), aarch64
* RAM: 16 GB
* OS: Ubuntu 22.04.2 LTS, kernel 5.10.110
* Bitcoin Core: v31.0rc2 (built from source)

![image|690x417](upload://3MPe5T102TpM1XtPz7MlXnjJQNC.jpeg)

-------------------------

ajtowns | 2026-04-09 10:18:53 UTC | #38

[quote="AntoineP, post:1, topic:2367"]
@ajtowns is going to mine the six blocks in a row and then reorg them out, so that anyone running a Signet node at the time would see them, but they would not impose a cost to IBD forever.
[/quote]

For those interested in the behind the scenes (including, perhaps, future me; hi future me!) what I did was:

 1) stop the miner
 2) make sure the next scheduled block was 10+ minutes away so the backup miner wouldn't kick in too soon
 3) copy @AntoineP's proposed blocks into the custom-blocks dir targetting the next block heights (see [inq#111](https://github.com/bitcoin-inquisition/bitcoin/pull/111) for the code to support custom blocks)
 4) use just the core node to manually mine the next 6-8 blocks; 6 blocks of attack txs, then a couple of extra blocks with some tx spam so the mempool has something to do during reorg
 5) on the core node, run `invalidateblock` on the first attack block
 6) restart the normal miner, so that 7-9 blocks get mined over the course of about an hour to reorg the attack blocks out
 7) wait for my super-slow node to finish validating all the blocks (at ~5m per block, yikes. on the upside, great for catching performance regressions in normal usage!)
 8) run `invalidateblock` on `inquisition.bitcoin-signet.net` to ensure the reorg chain has good connectivity

-------------------------

0xB10C | 2026-04-09 13:17:21 UTC | #39

I’ve uploaded my detailed debug logs for the three runs for both my Bitcoin Core and Inquisition node. Both nodes ran on the same Hetzner Cloud CX43 host with 8 shared CPU cores of an AMD EPYC-Rome Processor (2019) and 16 GB of RAM.

Bitcoin Core:
* https://tmp.b10c.me/2026-04-slow-blocks-signet/core-run-1.log.gz
* https://tmp.b10c.me/2026-04-slow-blocks-signet/core-run-2.log.gz
* https://tmp.b10c.me/2026-04-slow-blocks-signet/core-run-3.log.gz

Inquisition:
* https://tmp.b10c.me/2026-04-slow-blocks-signet/inq-run-1.log.gz
* https://tmp.b10c.me/2026-04-slow-blocks-signet/inq-run-2.log.gz
* https://tmp.b10c.me/2026-04-slow-blocks-signet/inq-run-3.log.gz

Feel free to analyze them in detail, if you want.

Running `zgrep -hE “(bench|UpdateTip: new)” core-run-*.log.gz` yields something like:

```
2026-04-08T14:05:12.543864Z [msghand] [bench]   - Using cached block
2026-04-08T14:05:12.543926Z [msghand] [bench]   - Load block from disk: 0.07ms
2026-04-08T14:05:12.543962Z [msghand] [bench]     - Sanity checks: 0.01ms [0.00s (0.01ms/blk)]
2026-04-08T14:05:14.166485Z [msghand] [bench]     - Fork checks: 1622.48ms [1.96s (93.29ms/blk)]
2026-04-08T14:05:14.643331Z [msghand] [bench]       - Connect 2 transactions: 476.84ms (238.419ms/tx, 1.189ms/txin) [0.69s (32.64ms/blk)]
2026-04-08T14:06:33.383188Z [msghand] [bench]     - Verify 401 txins: 79216.70ms (197.548ms/txin) [79.43s (3782.32ms/blk)]
2026-04-08T14:06:33.385563Z [msghand] [bench]     - Write undo data: 2.39ms [0.07s (3.19ms/blk)]
2026-04-08T14:06:33.385643Z [msghand] [bench]     - Index writing: 0.11ms [0.00s (0.08ms/blk)]
2026-04-08T14:06:33.385862Z [msghand] [bench]   - Connect total: 80841.94ms [81.46s (3879.22ms/blk)]
2026-04-08T14:06:33.832999Z [msghand] [bench]   - Flush: 447.07ms [0.51s (24.52ms/blk)]
2026-04-08T14:06:33.833198Z [msghand] [bench]   - Writing chainstate: 0.27ms [0.00s (0.19ms/blk)]
2026-04-08T14:06:33.833839Z [msghand] UpdateTip: new best=0000000eb552c9f26e712d546c71297fd0623890299b40e7ada81d2dc32f5d0b height=299177 version=0x20000000 log2_work=43.630312 tx=29147716 date='2026-04-08T14:03:39Z' progress=1.000000 cache=15.3MiB(114240txo)
2026-04-08T14:06:33.833856Z [msghand] [bench]   - Connect postprocess: 0.66ms [1.57s (74.55ms/blk)]
2026-04-08T14:06:33.833868Z [msghand] [bench] - Connect block: 81290.01ms [83.55s (3978.55ms/blk)]

2026-04-08T14:06:35.161185Z [msghand] [bench]   - Using cached block
2026-04-08T14:06:35.161251Z [msghand] [bench]   - Load block from disk: 0.07ms
2026-04-08T14:06:35.161288Z [msghand] [bench]     - Sanity checks: 0.01ms [0.00s (0.01ms/blk)]
2026-04-08T14:06:36.734346Z [msghand] [bench]     - Fork checks: 1573.02ms [3.53s (160.55ms/blk)]
2026-04-08T14:06:37.065053Z [msghand] [bench]       - Connect 2 transactions: 330.68ms (165.340ms/tx, 0.825ms/txin) [1.02s (46.18ms/blk)]
2026-04-08T14:07:56.373530Z [msghand] [bench]     - Verify 401 txins: 79639.18ms (198.601ms/txin) [159.07s (7230.36ms/blk)]
2026-04-08T14:07:56.375675Z [msghand] [bench]     - Write undo data: 2.18ms [0.07s (3.15ms/blk)]
2026-04-08T14:07:56.375703Z [msghand] [bench]     - Index writing: 0.04ms [0.00s (0.08ms/blk)]
2026-04-08T14:07:56.375874Z [msghand] [bench]   - Connect total: 81214.63ms [162.68s (7394.46ms/blk)]
2026-04-08T14:07:56.847702Z [msghand] [bench]   - Flush: 471.79ms [0.99s (44.85ms/blk)]
2026-04-08T14:07:56.847883Z [msghand] [bench]   - Writing chainstate: 0.23ms [0.00s (0.19ms/blk)]
2026-04-08T14:07:56.848703Z [msghand] UpdateTip: new best=000000002b3a132836666c18f5e1a9d93623d3797316a968ee54e47fb44c0c13 height=299178 version=0x20000000 log2_work=43.630334 tx=29147718 date='2026-04-08T14:04:48Z' progress=1.000000 cache=28.7MiB(212536txo)
2026-04-08T14:07:56.848728Z [msghand] [bench]   - Connect postprocess: 0.85ms [1.57s (71.20ms/blk)]
2026-04-08T14:07:56.848747Z [msghand] [bench] - Connect block: 81687.56ms [165.24s (7510.78ms/blk)]

2026-04-08T14:07:57.814128Z [msghand] [bench]   - Using cached block
2026-04-08T14:07:57.814181Z [msghand] [bench]   - Load block from disk: 0.06ms
2026-04-08T14:07:57.814206Z [msghand] [bench]     - Sanity checks: 0.01ms [0.00s (0.01ms/blk)]
2026-04-08T14:07:58.996692Z [msghand] [bench]     - Fork checks: 1182.45ms [4.71s (204.98ms/blk)]
2026-04-08T14:07:59.229473Z [msghand] [bench]       - Connect 2 transactions: 232.76ms (116.381ms/tx, 0.580ms/txin) [1.25s (54.30ms/blk)]
2026-04-08T14:09:13.252885Z [msghand] [bench]     - Verify 401 txins: 74256.16ms (185.177ms/txin) [233.32s (10144.52ms/blk)]
2026-04-08T14:09:13.255371Z [msghand] [bench]     - Write undo data: 2.50ms [0.07s (3.12ms/blk)]
2026-04-08T14:09:13.255441Z [msghand] [bench]     - Index writing: 0.11ms [0.00s (0.08ms/blk)]
2026-04-08T14:09:13.255692Z [msghand] [bench]   - Connect total: 75441.51ms [238.12s (10353.03ms/blk)]
2026-04-08T14:09:13.636278Z [msghand] [bench]   - Flush: 380.53ms [1.37s (59.45ms/blk)]
2026-04-08T14:09:13.636471Z [msghand] [bench]   - Writing chainstate: 0.26ms [0.00s (0.19ms/blk)]
2026-04-08T14:09:13.637132Z [msghand] UpdateTip: new best=00000006d34037534a517f9e5809a34766f1540c0e6817eac91b1adfee50cb5f height=299179 version=0x20000000 log2_work=43.630355 tx=29147720 date='2026-04-08T14:06:00Z' progress=1.000000 cache=40.7MiB(310833txo)
2026-04-08T14:09:13.637150Z [msghand] [bench]   - Connect postprocess: 0.68ms [1.57s (68.13ms/blk)]
2026-04-08T14:09:13.637164Z [msghand] [bench] - Connect block: 75823.03ms [241.06s (10480.88ms/blk)]

2026-04-08T14:09:14.893852Z [msghand] [bench]   - Using cached block
2026-04-08T14:09:14.893934Z [msghand] [bench]   - Load block from disk: 0.08ms
2026-04-08T14:09:14.893995Z [msghand] [bench]     - Sanity checks: 0.03ms [0.00s (0.01ms/blk)]
2026-04-08T14:09:16.476168Z [msghand] [bench]     - Fork checks: 1582.14ms [6.30s (262.36ms/blk)]
2026-04-08T14:09:16.787262Z [msghand] [bench]       - Connect 2 transactions: 311.07ms (155.535ms/tx, 0.776ms/txin) [1.56s (64.99ms/blk)]
2026-04-08T14:10:32.440169Z [msghand] [bench]     - Verify 401 txins: 75963.99ms (189.436ms/txin) [309.29s (12887.00ms/blk)]
2026-04-08T14:10:32.442140Z [msghand] [bench]     - Write undo data: 2.00ms [0.07s (3.07ms/blk)]
2026-04-08T14:10:32.442182Z [msghand] [bench]     - Index writing: 0.07ms [0.00s (0.08ms/blk)]
2026-04-08T14:10:32.442342Z [msghand] [bench]   - Connect total: 77548.42ms [315.67s (13152.84ms/blk)]
2026-04-08T14:10:32.988139Z [msghand] [bench]   - Flush: 545.75ms [1.91s (79.71ms/blk)]
2026-04-08T14:10:32.988275Z [msghand] [bench]   - Writing chainstate: 0.18ms [0.00s (0.19ms/blk)]
2026-04-08T14:10:32.988911Z [msghand] UpdateTip: new best=00000014a4cae4501f98539b45c76059c706a82b77f19a9adf365b3f5e989444 height=299180 version=0x20000000 log2_work=43.630376 tx=29147722 date='2026-04-08T14:06:49Z' progress=1.000000 cache=55.4MiB(409132txo)
2026-04-08T14:10:32.988930Z [msghand] [bench]   - Connect postprocess: 0.65ms [1.57s (65.32ms/blk)]
2026-04-08T14:10:32.988943Z [msghand] [bench] - Connect block: 78095.08ms [319.16s (13298.14ms/blk)]

2026-04-08T14:10:34.129186Z [msghand] [bench]   - Using cached block
2026-04-08T14:10:34.129236Z [msghand] [bench]   - Load block from disk: 0.06ms
2026-04-08T14:10:34.129262Z [msghand] [bench]     - Sanity checks: 0.01ms [0.00s (0.01ms/blk)]
2026-04-08T14:10:35.431699Z [msghand] [bench]     - Fork checks: 1302.40ms [7.60s (303.96ms/blk)]
2026-04-08T14:10:35.745804Z [msghand] [bench]       - Connect 2 transactions: 314.10ms (157.048ms/tx, 0.783ms/txin) [1.87s (74.96ms/blk)]
2026-04-08T14:11:51.663648Z [msghand] [bench]     - Verify 401 txins: 76231.87ms (190.104ms/txin) [385.52s (15420.80ms/blk)]
2026-04-08T14:11:51.665985Z [msghand] [bench]     - Write undo data: 2.42ms [0.08s (3.05ms/blk)]
2026-04-08T14:11:51.666037Z [msghand] [bench]     - Index writing: 0.08ms [0.00s (0.08ms/blk)]
2026-04-08T14:11:51.666250Z [msghand] [bench]   - Connect total: 77537.02ms [393.21s (15728.21ms/blk)]
2026-04-08T14:11:52.017625Z [msghand] [bench]   - Flush: 351.30ms [2.26s (90.57ms/blk)]
2026-04-08T14:11:52.017793Z [msghand] [bench]   - Writing chainstate: 0.25ms [0.00s (0.19ms/blk)]
2026-04-08T14:11:52.018738Z [msghand] UpdateTip: new best=00000003220437cb8b5a2edef6be828c5cdad114b1b642d724ac6f3caa7f12fb height=299181 version=0x20000000 log2_work=43.630398 tx=29147724 date='2026-04-08T14:07:54Z' progress=1.000000 cache=67.5MiB(507430txo)
2026-04-08T14:11:52.018765Z [msghand] [bench]   - Connect postprocess: 0.97ms [1.57s (62.75ms/blk)]
2026-04-08T14:11:52.018785Z [msghand] [bench] - Connect block: 77889.59ms [397.04s (15881.79ms/blk)]

2026-04-08T14:11:54.078822Z [msghand] [bench]   - Using cached block
2026-04-08T14:11:54.078891Z [msghand] [bench]   - Load block from disk: 0.07ms
2026-04-08T14:11:54.078943Z [msghand] [bench]     - Sanity checks: 0.02ms [0.00s (0.01ms/blk)]
2026-04-08T14:11:55.698942Z [msghand] [bench]     - Fork checks: 1619.95ms [9.22s (354.58ms/blk)]
2026-04-08T14:11:56.025006Z [msghand] [bench]       - Connect 2 transactions: 326.06ms (163.032ms/tx, 0.813ms/txin) [2.20s (84.62ms/blk)]
2026-04-08T14:13:26.580070Z [msghand] [bench]     - Verify 401 txins: 90881.13ms (226.636ms/txin) [476.40s (18323.12ms/blk)]
2026-04-08T14:13:26.582627Z [msghand] [bench]     - Write undo data: 2.55ms [0.08s (3.03ms/blk)]
2026-04-08T14:13:26.582692Z [msghand] [bench]     - Index writing: 0.13ms [0.00s (0.08ms/blk)]
2026-04-08T14:13:26.582896Z [msghand] [bench]   - Connect total: 92504.01ms [485.71s (18681.12ms/blk)]
2026-04-08T14:13:26.965945Z [msghand] [bench]   - Flush: 382.99ms [2.65s (101.82ms/blk)]
2026-04-08T14:13:26.966145Z [msghand] [bench]   - Writing chainstate: 0.27ms [0.01s (0.20ms/blk)]
2026-04-08T14:13:26.967031Z [msghand] UpdateTip: new best=000000143c97bf0134c5cf0881dfd4ef458529b7388cacf43981ffe92fb96856 height=299182 version=0x20000000 log2_work=43.630419 tx=29147726 date='2026-04-08T14:08:56Z' progress=0.999997 cache=79.5MiB(605728txo)
2026-04-08T14:13:26.967057Z [msghand] [bench]   - Connect postprocess: 0.91ms [1.57s (60.37ms/blk)]
2026-04-08T14:13:26.967076Z [msghand] [bench] - Connect block: 92888.25ms [489.93s (18843.58ms/blk)]

2026-04-08T14:13:27.506363Z [msghand] [bench]   - Using cached block
2026-04-08T14:13:27.506425Z [msghand] [bench]   - Load block from disk: 0.07ms
2026-04-08T14:13:27.506459Z [msghand] [bench]     - Sanity checks: 0.01ms [0.00s (0.01ms/blk)]
2026-04-08T14:13:27.524771Z [msghand] [bench]     - Fork checks: 18.27ms [9.24s (342.12ms/blk)]
2026-04-08T14:13:27.563187Z [msghand] [bench]       - Connect 127 transactions: 38.41ms (0.302ms/tx, 0.182ms/txin) [2.24s (82.91ms/blk)]
2026-04-08T14:13:27.563320Z [msghand] [bench]     - Verify 211 txins: 38.59ms (0.183ms/txin) [476.44s (17645.91ms/blk)]
2026-04-08T14:13:27.564865Z [msghand] [bench]     - Write undo data: 1.53ms [0.08s (2.97ms/blk)]
2026-04-08T14:13:27.564902Z [msghand] [bench]     - Index writing: 0.06ms [0.00s (0.08ms/blk)]
2026-04-08T14:13:27.566273Z [msghand] [bench]   - Connect total: 59.84ms [485.77s (17991.44ms/blk)]
2026-04-08T14:13:27.569648Z [msghand] [bench]   - Flush: 3.33ms [2.65s (98.17ms/blk)]
2026-04-08T14:13:27.569797Z [msghand] [bench]   - Writing chainstate: 0.21ms [0.01s (0.20ms/blk)]
2026-04-08T14:13:27.572538Z [msghand] UpdateTip: new best=000000079e5f6f5376bd51b5d26fb2e27dd8762c4cac3936380647cc43377ac6 height=299183 version=0x20000000 log2_work=43.630441 tx=29147853 date='2026-04-08T14:10:21Z' progress=0.999999 cache=79.7MiB(606343txo)
2026-04-08T14:13:27.572567Z [msghand] [bench]   - Connect postprocess: 2.77ms [1.57s (58.24ms/blk)]
2026-04-08T14:13:27.572612Z [msghand] [bench] - Connect block: 66.22ms [490.00s (18148.12ms/blk)]
```

Run 2:
```
2026-04-08T22:03:58.448155Z [msghand] [bench]   - Using cached block
2026-04-08T22:03:58.448226Z [msghand] [bench]   - Load block from disk: 0.07ms
2026-04-08T22:03:58.448264Z [msghand] [bench]     - Sanity checks: 0.01ms [0.01s (0.12ms/blk)]
2026-04-08T22:03:59.508157Z [msghand] [bench]     - Fork checks: 1059.86ms [11.08s (138.46ms/blk)]
2026-04-08T22:03:59.759230Z [msghand] [bench]       - Connect 2 transactions: 251.05ms (125.526ms/tx, 0.626ms/txin) [2.90s (36.25ms/blk)]
2026-04-08T22:05:04.771640Z [msghand] [bench]     - Verify 401 txins: 65263.43ms (162.752ms/txin) [542.12s (6776.50ms/blk)]
2026-04-08T22:05:04.773967Z [msghand] [bench]     - Write undo data: 2.38ms [0.18s (2.28ms/blk)]
2026-04-08T22:05:04.774017Z [msghand] [bench]     - Index writing: 0.08ms [0.00s (0.06ms/blk)]
2026-04-08T22:05:04.774653Z [msghand] [bench]   - Connect total: 66326.43ms [553.42s (6917.71ms/blk)]
2026-04-08T22:05:05.059533Z [msghand] [bench]   - Flush: 284.84ms [3.09s (38.64ms/blk)]
2026-04-08T22:05:05.059731Z [msghand] [bench]   - Writing chainstate: 0.25ms [0.01s (0.16ms/blk)]
2026-04-08T22:05:05.060664Z [msghand] UpdateTip: new best=00000011ee1088ab58ade91da86761478799b685699b30bc649c83f13725bc6d height=299227 version=0x20000000 log2_work=43.631382 tx=29156002 date='2026-04-08T22:02:35Z' progress=1.000000 cache=79.7MiB(138957txo)
2026-04-08T22:05:05.060693Z [msghand] [bench]   - Connect postprocess: 0.96ms [1.84s (23.04ms/blk)]
2026-04-08T22:05:05.060710Z [msghand] [bench] - Connect block: 66612.55ms [558.39s (6979.87ms/blk)]

2026-04-08T22:05:06.395752Z [msghand] [bench]   - Using cached block
2026-04-08T22:05:06.395816Z [msghand] [bench]   - Load block from disk: 0.07ms
2026-04-08T22:05:06.395851Z [msghand] [bench]     - Sanity checks: 0.01ms [0.01s (0.11ms/blk)]
2026-04-08T22:05:07.881059Z [msghand] [bench]     - Fork checks: 1485.18ms [12.56s (155.09ms/blk)]
2026-04-08T22:05:08.159974Z [msghand] [bench]       - Connect 2 transactions: 278.89ms (139.447ms/tx, 0.695ms/txin) [3.18s (39.25ms/blk)]
2026-04-08T22:06:10.667317Z [msghand] [bench]     - Verify 401 txins: 62786.24ms (156.574ms/txin) [604.91s (7467.98ms/blk)]
2026-04-08T22:06:10.669543Z [msghand] [bench]     - Write undo data: 2.26ms [0.18s (2.28ms/blk)]
2026-04-08T22:06:10.669594Z [msghand] [bench]     - Index writing: 0.05ms [0.00s (0.06ms/blk)]
2026-04-08T22:06:10.669871Z [msghand] [bench]   - Connect total: 64274.06ms [617.69s (7625.81ms/blk)]
2026-04-08T22:06:10.969680Z [msghand] [bench]   - Flush: 299.76ms [3.39s (41.87ms/blk)]
2026-04-08T22:06:10.969874Z [msghand] [bench]   - Writing chainstate: 0.25ms [0.01s (0.16ms/blk)]
2026-04-08T22:06:10.970744Z [msghand] UpdateTip: new best=000000037a35ebf60c59619042ece8ae71fecf4d3146098c966f820482d33955 height=299228 version=0x20000000 log2_work=43.631403 tx=29156004 date='2026-04-08T22:03:53Z' progress=1.000000 cache=79.7MiB(237257txo)
2026-04-08T22:06:10.970773Z [msghand] [bench]   - Connect postprocess: 0.90ms [1.84s (22.76ms/blk)]
2026-04-08T22:06:10.970792Z [msghand] [bench] - Connect block: 64575.03ms [622.96s (7690.93ms/blk)]

2026-04-08T22:06:36.650042Z [msghand] [bench]   - Using cached block
2026-04-08T22:06:36.650090Z [msghand] [bench]   - Load block from disk: 0.05ms
2026-04-08T22:06:36.650115Z [msghand] [bench]     - Sanity checks: 0.01ms [0.01s (0.11ms/blk)]
2026-04-08T22:06:37.896756Z [msghand] [bench]     - Fork checks: 1246.61ms [13.81s (168.40ms/blk)]
2026-04-08T22:06:38.131776Z [msghand] [bench]       - Connect 2 transactions: 235.00ms (117.502ms/tx, 0.586ms/txin) [3.41s (41.63ms/blk)]
2026-04-08T22:07:44.046688Z [msghand] [bench]     - Verify 401 txins: 66149.91ms (164.962ms/txin) [671.06s (8183.61ms/blk)]
2026-04-08T22:07:44.048249Z [msghand] [bench]     - Write undo data: 1.60ms [0.19s (2.27ms/blk)]
2026-04-08T22:07:44.048276Z [msghand] [bench]     - Index writing: 0.04ms [0.00s (0.06ms/blk)]
2026-04-08T22:07:44.048453Z [msghand] [bench]   - Connect total: 67398.37ms [685.09s (8354.75ms/blk)]
2026-04-08T22:07:44.282301Z [msghand] [bench]   - Flush: 233.81ms [3.63s (44.21ms/blk)]
2026-04-08T22:07:44.282449Z [msghand] [bench]   - Writing chainstate: 0.19ms [0.01s (0.16ms/blk)]
2026-04-08T22:07:44.283074Z [msghand] UpdateTip: new best=0000001253d4dbf6b7b34079a70ef0357a7b12e83a38f29946effbc863f194f3 height=299229 version=0x20000000 log2_work=43.631425 tx=29156006 date='2026-04-08T22:04:43Z' progress=1.000000 cache=79.7MiB(335560txo)
2026-04-08T22:07:44.283094Z [msghand] [bench]   - Connect postprocess: 0.64ms [1.84s (22.49ms/blk)]
2026-04-08T22:07:44.283107Z [msghand] [bench] - Connect block: 67633.06ms [690.60s (8421.93ms/blk)]

2026-04-08T22:07:45.129398Z [msghand] [bench]   - Using cached block
2026-04-08T22:07:45.129478Z [msghand] [bench]   - Load block from disk: 0.08ms
2026-04-08T22:07:45.129513Z [msghand] [bench]     - Sanity checks: 0.01ms [0.01s (0.11ms/blk)]
2026-04-08T22:07:46.416482Z [msghand] [bench]     - Fork checks: 1286.94ms [15.10s (181.87ms/blk)]
2026-04-08T22:07:46.652222Z [msghand] [bench]       - Connect 2 transactions: 235.72ms (117.860ms/tx, 0.588ms/txin) [3.65s (43.97ms/blk)]
2026-04-08T22:08:56.717425Z [msghand] [bench]     - Verify 401 txins: 70300.91ms (175.314ms/txin) [741.36s (8932.01ms/blk)]
2026-04-08T22:08:56.719615Z [msghand] [bench]     - Write undo data: 2.22ms [0.19s (2.27ms/blk)]
2026-04-08T22:08:56.719653Z [msghand] [bench]     - Index writing: 0.08ms [0.01s (0.06ms/blk)]
2026-04-08T22:08:56.719836Z [msghand] [bench]   - Connect total: 71590.37ms [756.68s (9116.62ms/blk)]
2026-04-08T22:08:56.955183Z [msghand] [bench]   - Flush: 235.31ms [3.86s (46.51ms/blk)]
2026-04-08T22:08:56.955316Z [msghand] [bench]   - Writing chainstate: 0.18ms [0.01s (0.16ms/blk)]
2026-04-08T22:08:56.955853Z [msghand] UpdateTip: new best=0000000b4f9d1ba885832b1b9dd4f6183483a27a4762a48114c00a812f1b42dd height=299230 version=0x20000000 log2_work=43.631446 tx=29156008 date='2026-04-08T22:06:13Z' progress=1.000000 cache=79.7MiB(433856txo)
2026-04-08T22:08:56.955868Z [msghand] [bench]   - Connect postprocess: 0.55ms [1.85s (22.23ms/blk)]
2026-04-08T22:08:56.955881Z [msghand] [bench] - Connect block: 71826.49ms [762.42s (9185.84ms/blk)]

2026-04-08T22:08:57.756562Z [msghand] [bench]   - Using cached block
2026-04-08T22:08:57.756632Z [msghand] [bench]   - Load block from disk: 0.07ms
2026-04-08T22:08:57.756660Z [msghand] [bench]     - Sanity checks: 0.01ms [0.01s (0.11ms/blk)]
2026-04-08T22:08:58.964337Z [msghand] [bench]     - Fork checks: 1207.64ms [16.30s (194.08ms/blk)]
2026-04-08T22:08:59.232398Z [msghand] [bench]       - Connect 2 transactions: 268.05ms (134.023ms/tx, 0.668ms/txin) [3.92s (46.64ms/blk)]
2026-04-08T22:10:08.589321Z [msghand] [bench]     - Verify 401 txins: 69624.98ms (173.628ms/txin) [810.98s (9654.55ms/blk)]
2026-04-08T22:10:08.591670Z [msghand] [bench]     - Write undo data: 2.38ms [0.19s (2.27ms/blk)]
2026-04-08T22:10:08.591711Z [msghand] [bench]     - Index writing: 0.06ms [0.01s (0.06ms/blk)]
2026-04-08T22:10:08.591929Z [msghand] [bench]   - Connect total: 70835.30ms [827.51s (9851.37ms/blk)]
2026-04-08T22:10:08.921946Z [msghand] [bench]   - Flush: 329.97ms [4.19s (49.89ms/blk)]
2026-04-08T22:10:08.922151Z [msghand] [bench]   - Writing chainstate: 0.25ms [0.01s (0.16ms/blk)]
2026-04-08T22:10:08.923051Z [msghand] UpdateTip: new best=00000011240c362604a624fc4469413f36ff5cb823c0733d570d54c7916a913f height=299231 version=0x20000000 log2_work=43.631467 tx=29156010 date='2026-04-08T22:07:02Z' progress=1.000000 cache=79.7MiB(532153txo)
2026-04-08T22:10:08.923075Z [msghand] [bench]   - Connect postprocess: 0.93ms [1.85s (21.98ms/blk)]
2026-04-08T22:10:08.923096Z [msghand] [bench] - Connect block: 71166.52ms [833.59s (9923.70ms/blk)]

2026-04-08T22:10:09.915110Z [msghand] [bench]   - Using cached block
2026-04-08T22:10:09.915174Z [msghand] [bench]   - Load block from disk: 0.07ms
2026-04-08T22:10:09.915207Z [msghand] [bench]     - Sanity checks: 0.01ms [0.01s (0.11ms/blk)]
2026-04-08T22:10:11.623058Z [msghand] [bench]     - Fork checks: 1707.82ms [18.01s (211.89ms/blk)]
2026-04-08T22:10:11.934018Z [msghand] [bench]       - Connect 2 transactions: 310.94ms (155.470ms/tx, 0.775ms/txin) [4.23s (49.75ms/blk)]
2026-04-08T22:11:18.513190Z [msghand] [bench]     - Verify 401 txins: 66890.12ms (166.808ms/txin) [877.87s (10327.91ms/blk)]
2026-04-08T22:11:18.515644Z [msghand] [bench]     - Write undo data: 2.45ms [0.19s (2.27ms/blk)]
2026-04-08T22:11:18.515699Z [msghand] [bench]     - Index writing: 0.11ms [0.01s (0.06ms/blk)]
2026-04-08T22:11:18.515902Z [msghand] [bench]   - Connect total: 68600.73ms [896.12s (10542.54ms/blk)]
2026-04-08T22:11:18.815350Z [msghand] [bench]   - Flush: 299.40ms [4.49s (52.82ms/blk)]
2026-04-08T22:11:18.815486Z [msghand] [bench]   - Writing chainstate: 0.19ms [0.01s (0.16ms/blk)]
2026-04-08T22:11:18.816073Z [msghand] UpdateTip: new best=000000065d332b249b5fc8068177776b3dddb073d308ae5b090147c41477e351 height=299232 version=0x20000000 log2_work=43.631489 tx=29156012 date='2026-04-08T22:07:50Z' progress=1.000000 cache=82.5MiB(630450txo)
2026-04-08T22:11:18.816089Z [msghand] [bench]   - Connect postprocess: 0.60ms [1.85s (21.72ms/blk)]
2026-04-08T22:11:18.816102Z [msghand] [bench] - Connect block: 68901.00ms [902.49s (10617.55ms/blk)]

2026-04-08T22:12:13.129188Z [msghand] [bench]   - Using cached block
2026-04-08T22:12:13.129239Z [msghand] [bench]   - Load block from disk: 0.05ms
2026-04-08T22:12:13.129264Z [msghand] [bench]     - Sanity checks: 0.01ms [0.01s (0.11ms/blk)]
2026-04-08T22:12:13.204207Z [msghand] [bench]     - Fork checks: 74.91ms [18.09s (210.30ms/blk)]
2026-04-08T22:12:13.236787Z [msghand] [bench]       - Connect 1454 transactions: 32.57ms (0.022ms/tx, 0.021ms/txin) [4.26s (49.55ms/blk)]
2026-04-08T22:12:13.236913Z [msghand] [bench]     - Verify 1550 txins: 32.73ms (0.021ms/txin) [877.91s (10208.20ms/blk)]
2026-04-08T22:12:13.246312Z [msghand] [bench]     - Write undo data: 9.35ms [0.20s (2.35ms/blk)]
2026-04-08T22:12:13.246384Z [msghand] [bench]     - Index writing: 0.12ms [0.01s (0.06ms/blk)]
2026-04-08T22:12:13.246837Z [msghand] [bench]   - Connect total: 117.60ms [896.23s (10421.32ms/blk)]
2026-04-08T22:12:13.262386Z [msghand] [bench]   - Flush: 15.51ms [4.51s (52.39ms/blk)]
2026-04-08T22:12:13.262521Z [msghand] [bench]   - Writing chainstate: 0.18ms [0.01s (0.16ms/blk)]
2026-04-08T22:12:13.303299Z [msghand] UpdateTip: new best=00000010d89e826688208b7074202ff88b40346c300dd6bf3110ffdeeecafffd height=299233 version=0x20000000 log2_work=43.631510 tx=29157466 date='2026-04-08T22:12:03Z' progress=1.000000 cache=83.0MiB(633958txo)
2026-04-08T22:12:13.303368Z [msghand] [bench]   - Connect postprocess: 40.84ms [1.89s (21.95ms/blk)]
2026-04-08T22:12:13.303390Z [msghand] [bench] - Connect block: 174.18ms [902.67s (10496.12ms/blk)]
```

Run 3:
```

2026-04-09T09:02:52.804051Z [msghand] [bench]   - Using cached block
2026-04-09T09:02:52.804122Z [msghand] [bench]   - Load block from disk: 0.08ms
2026-04-09T09:02:52.804164Z [msghand] [bench]     - Sanity checks: 0.01ms [0.02s (0.13ms/blk)]
2026-04-09T09:02:54.348955Z [msghand] [bench]     - Fork checks: 1544.75ms [20.67s (132.52ms/blk)]
2026-04-09T09:02:54.680275Z [msghand] [bench]       - Connect 2 transactions: 331.30ms (165.652ms/tx, 0.826ms/txin) [5.07s (32.51ms/blk)]
2026-04-09T09:04:30.931279Z [msghand] [bench]     - Verify 401 txins: 96582.30ms (240.854ms/txin) [974.98s (6249.86ms/blk)]
2026-04-09T09:04:30.933638Z [msghand] [bench]     - Write undo data: 2.37ms [0.32s (2.03ms/blk)]
2026-04-09T09:04:30.933698Z [msghand] [bench]     - Index writing: 0.11ms [0.01s (0.06ms/blk)]
2026-04-09T09:04:30.933931Z [msghand] [bench]   - Connect total: 98129.82ms [996.04s (6384.89ms/blk)]
2026-04-09T09:04:31.249757Z [msghand] [bench]   - Flush: 315.78ms [5.02s (32.20ms/blk)]
2026-04-09T09:04:31.249964Z [msghand] [bench]   - Writing chainstate: 0.25ms [0.02s (0.16ms/blk)]
2026-04-09T09:04:31.251065Z [msghand] UpdateTip: new best=0000000db638fd84f3668c983cdf3f305a098c61d0bcbc657a15250b081efdd4 height=299296 version=0x20000000 log2_work=43.632857 tx=29164133 date='2026-04-09T09:02:02Z' progress=1.000000 cache=83.0MiB(179911txo)
2026-04-09T09:04:31.251099Z [msghand] [bench]   - Connect postprocess: 1.14ms [2.17s (13.93ms/blk)]
2026-04-09T09:04:31.251123Z [msghand] [bench] - Connect block: 98447.06ms [1003.32s (6431.55ms/blk)]

2026-04-09T09:04:33.338082Z [msghand] [bench]   - Using cached block
2026-04-09T09:04:33.338155Z [msghand] [bench]   - Load block from disk: 0.08ms
2026-04-09T09:04:33.338194Z [msghand] [bench]     - Sanity checks: 0.01ms [0.02s (0.13ms/blk)]
2026-04-09T09:04:34.905201Z [msghand] [bench]     - Fork checks: 1566.96ms [22.24s (141.66ms/blk)]
2026-04-09T09:04:35.252193Z [msghand] [bench]       - Connect 2 transactions: 346.98ms (173.492ms/tx, 0.865ms/txin) [5.42s (34.51ms/blk)]
2026-04-09T09:06:14.617052Z [msghand] [bench]     - Verify 401 txins: 99711.84ms (248.658ms/txin) [1074.69s (6845.16ms/blk)]
2026-04-09T09:06:14.619145Z [msghand] [bench]     - Write undo data: 2.12ms [0.32s (2.03ms/blk)]
2026-04-09T09:06:14.619188Z [msghand] [bench]     - Index writing: 0.07ms [0.01s (0.06ms/blk)]
2026-04-09T09:06:14.619392Z [msghand] [bench]   - Connect total: 101281.25ms [1097.32s (6989.33ms/blk)]
2026-04-09T09:06:14.965770Z [msghand] [bench]   - Flush: 346.33ms [5.37s (34.20ms/blk)]
2026-04-09T09:06:14.965960Z [msghand] [bench]   - Writing chainstate: 0.24ms [0.03s (0.16ms/blk)]
2026-04-09T09:06:14.966935Z [msghand] UpdateTip: new best=00000013079c637e2d4b37cb95f019479a4e86a041e9eee162c6e2ae1758ed9d height=299297 version=0x20000000 log2_work=43.632878 tx=29164135 date='2026-04-09T09:02:46Z' progress=1.000000 cache=83.0MiB(278207txo)
2026-04-09T09:06:14.966961Z [msghand] [bench]   - Connect postprocess: 1.00ms [2.17s (13.85ms/blk)]
2026-04-09T09:06:14.966984Z [msghand] [bench] - Connect block: 101628.90ms [1104.95s (7037.90ms/blk)]

2026-04-09T09:06:16.962736Z [msghand] [bench]   - Using cached block
2026-04-09T09:06:16.962803Z [msghand] [bench]   - Load block from disk: 0.07ms
2026-04-09T09:06:16.962850Z [msghand] [bench]     - Sanity checks: 0.01ms [0.02s (0.13ms/blk)]
2026-04-09T09:06:18.454208Z [msghand] [bench]     - Fork checks: 1491.12ms [23.73s (150.20ms/blk)]
2026-04-09T09:06:18.766904Z [msghand] [bench]       - Connect 2 transactions: 312.87ms (156.435ms/tx, 0.780ms/txin) [5.73s (36.28ms/blk)]
2026-04-09T09:07:56.865446Z [msghand] [bench]     - Verify 401 txins: 98411.42ms (245.415ms/txin) [1173.10s (7424.69ms/blk)]
2026-04-09T09:07:56.869922Z [msghand] [bench]     - Write undo data: 3.60ms [0.32s (2.04ms/blk)]
2026-04-09T09:07:56.869993Z [msghand] [bench]     - Index writing: 1.00ms [0.01s (0.07ms/blk)]
2026-04-09T09:07:56.870828Z [msghand] [bench]   - Connect total: 99908.03ms [1197.23s (7577.42ms/blk)]
2026-04-09T09:07:57.198163Z [msghand] [bench]   - Flush: 327.29ms [5.70s (36.06ms/blk)]
2026-04-09T09:07:57.198330Z [msghand] [bench]   - Writing chainstate: 0.22ms [0.03s (0.16ms/blk)]
2026-04-09T09:07:57.199360Z [msghand] UpdateTip: new best=0000000685879728d212116865fcf49d3b108588420f737edee8c331434bd407 height=299298 version=0x20000000 log2_work=43.632899 tx=29164137 date='2026-04-09T09:04:15Z' progress=1.000000 cache=83.0MiB(376503txo)
2026-04-09T09:07:57.199389Z [msghand] [bench]   - Connect postprocess: 1.06ms [2.18s (13.77ms/blk)]
2026-04-09T09:07:57.199408Z [msghand] [bench] - Connect block: 100236.67ms [1205.19s (7627.76ms/blk)]

2026-04-09T09:07:59.882355Z [msghand] [bench] FlushStateToDisk: find files to prune started
2026-04-09T09:07:59.882568Z [msghand] [bench] FlushStateToDisk: find files to prune completed (0.03ms)
2026-04-09T09:07:59.882732Z [msghand] [bench]   - Using cached block
2026-04-09T09:07:59.882758Z [msghand] [bench]   - Load block from disk: 0.02ms
2026-04-09T09:07:59.882791Z [msghand] [bench]     - Sanity checks: 0.01ms [0.02s (0.13ms/blk)]
2026-04-09T09:08:01.494452Z [msghand] [bench]     - Fork checks: 1611.62ms [25.34s (159.39ms/blk)]
2026-04-09T09:08:01.892695Z [msghand] [bench]       - Connect 2 transactions: 398.22ms (199.110ms/tx, 0.993ms/txin) [6.13s (38.55ms/blk)]
2026-04-09T09:09:37.308731Z [msghand] [bench]     - Verify 401 txins: 95814.25ms (238.938ms/txin) [1268.92s (7980.60ms/blk)]
2026-04-09T09:09:37.311439Z [msghand] [bench]     - Write undo data: 2.74ms [0.32s (2.04ms/blk)]
2026-04-09T09:09:37.311499Z [msghand] [bench]     - Index writing: 0.10ms [0.01s (0.07ms/blk)]
2026-04-09T09:09:37.312182Z [msghand] [bench]   - Connect total: 97429.41ms [1294.66s (8142.53ms/blk)]
2026-04-09T09:09:37.656681Z [msghand] [bench]   - Flush: 344.46ms [6.04s (38.00ms/blk)]
2026-04-09T09:09:37.656863Z [msghand] [bench] FlushStateToDisk: find files to prune started
2026-04-09T09:09:37.656919Z [msghand] [bench] FlushStateToDisk: find files to prune completed (0.03ms)
2026-04-09T09:09:37.656945Z [msghand] [bench]   - Writing chainstate: 0.32ms [0.03s (0.16ms/blk)]
2026-04-09T09:09:37.657922Z [msghand] UpdateTip: new best=00000004af57cad1f5cd0563db9b4fd7dc29f4de92b48aecba7ff7e29d68e1c0 height=299299 version=0x20000000 log2_work=43.632921 tx=29164139 date='2026-04-09T09:05:09Z' progress=0.999999 cache=83.0MiB(474799txo)
2026-04-09T09:09:37.657947Z [msghand] [bench]   - Connect postprocess: 1.00ms [2.18s (13.69ms/blk)]
2026-04-09T09:09:37.657966Z [msghand] [bench] - Connect block: 97775.21ms [1302.96s (8194.73ms/blk)]

2026-04-09T09:09:39.640412Z [msghand] [bench]   - Using cached block
2026-04-09T09:09:39.640486Z [msghand] [bench]   - Load block from disk: 0.08ms
2026-04-09T09:09:39.640535Z [msghand] [bench]     - Sanity checks: 0.01ms [0.02s (0.13ms/blk)]
2026-04-09T09:09:41.244212Z [msghand] [bench]     - Fork checks: 1603.63ms [26.95s (168.42ms/blk)]
2026-04-09T09:09:41.634262Z [msghand] [bench]       - Connect 2 transactions: 390.04ms (195.018ms/tx, 0.973ms/txin) [6.52s (40.75ms/blk)]
2026-04-09T09:11:18.479762Z [msghand] [bench]     - Verify 401 txins: 97235.54ms (242.483ms/txin) [1366.15s (8538.44ms/blk)]
2026-04-09T09:11:18.483317Z [msghand] [bench]     - Write undo data: 3.57ms [0.33s (2.05ms/blk)]
2026-04-09T09:11:18.483383Z [msghand] [bench]     - Index writing: 0.11ms [0.01s (0.07ms/blk)]
2026-04-09T09:11:18.486950Z [msghand] [bench]   - Connect total: 98846.46ms [1393.51s (8709.43ms/blk)]
2026-04-09T09:11:18.889793Z [msghand] [bench]   - Flush: 402.79ms [6.44s (40.28ms/blk)]
2026-04-09T09:11:18.889993Z [msghand] [bench]   - Writing chainstate: 0.27ms [0.03s (0.16ms/blk)]
2026-04-09T09:11:18.891177Z [msghand] UpdateTip: new best=0000000a0ee0a258ea2fbb2d2b0d465374885b2483ae4d175cc44aa843c63147 height=299300 version=0x20000000 log2_work=43.632942 tx=29164141 date='2026-04-09T09:06:19Z' progress=1.000000 cache=83.0MiB(573095txo)
2026-04-09T09:11:18.891201Z [msghand] [bench]   - Connect postprocess: 1.21ms [2.18s (13.61ms/blk)]
2026-04-09T09:11:18.891220Z [msghand] [bench] - Connect block: 99250.80ms [1402.21s (8763.83ms/blk)]

2026-04-09T09:11:21.165370Z [msghand] [bench]   - Using cached block
2026-04-09T09:11:21.165438Z [msghand] [bench]   - Load block from disk: 0.07ms
2026-04-09T09:11:21.165475Z [msghand] [bench]     - Sanity checks: 0.01ms [0.02s (0.13ms/blk)]
2026-04-09T09:11:22.743315Z [msghand] [bench]     - Fork checks: 1577.80ms [28.52s (177.17ms/blk)]
2026-04-09T09:11:23.058842Z [msghand] [bench]       - Connect 2 transactions: 315.50ms (157.751ms/tx, 0.787ms/txin) [6.84s (42.45ms/blk)]
2026-04-09T09:13:00.136665Z [msghand] [bench]     - Verify 401 txins: 97393.31ms (242.876ms/txin) [1463.54s (9090.34ms/blk)]
2026-04-09T09:13:00.139086Z [msghand] [bench]     - Write undo data: 2.47ms [0.33s (2.05ms/blk)]
2026-04-09T09:13:00.139151Z [msghand] [bench]     - Index writing: 0.10ms [0.01s (0.07ms/blk)]
2026-04-09T09:13:00.139373Z [msghand] [bench]   - Connect total: 98973.94ms [1492.48s (9270.08ms/blk)]
2026-04-09T09:13:00.547406Z [msghand] [bench]   - Flush: 407.98ms [6.85s (42.56ms/blk)]
2026-04-09T09:13:00.547570Z [msghand] [bench]   - Writing chainstate: 0.21ms [0.03s (0.16ms/blk)]
2026-04-09T09:13:00.548692Z [msghand] UpdateTip: new best=0000001018d0038f11974c98548fca64686380c21052a856cb8c4790cf6e50ce height=299301 version=0x20000000 log2_work=43.632963 tx=29164143 date='2026-04-09T09:07:53Z' progress=1.000000 cache=87.5MiB(671391txo)
2026-04-09T09:13:00.548724Z [msghand] [bench]   - Connect postprocess: 1.15ms [2.18s (13.53ms/blk)]
2026-04-09T09:13:00.548747Z [msghand] [bench] - Connect block: 99383.37ms [1501.60s (9326.68ms/blk)]

2026-04-09T09:13:02.961493Z [msghand] [bench]   - Using cached block
2026-04-09T09:13:02.961559Z [msghand] [bench]   - Load block from disk: 0.07ms
2026-04-09T09:13:02.961624Z [msghand] [bench]     - Sanity checks: 0.01ms [0.02s (0.12ms/blk)]
2026-04-09T09:13:03.087090Z [msghand] [bench]     - Fork checks: 125.43ms [28.65s (176.85ms/blk)]
2026-04-09T09:13:03.311680Z [msghand] [bench]       - Connect 2514 transactions: 224.60ms (0.089ms/tx, 0.070ms/txin) [7.06s (43.58ms/blk)]
2026-04-09T09:13:03.311769Z [msghand] [bench]     - Verify 3213 txins: 224.72ms (0.070ms/txin) [1463.77s (9035.61ms/blk)]
2026-04-09T09:13:03.327918Z [msghand] [bench]     - Write undo data: 16.10ms [0.35s (2.14ms/blk)]
2026-04-09T09:13:03.327994Z [msghand] [bench]     - Index writing: 0.12ms [0.01s (0.07ms/blk)]
2026-04-09T09:13:03.328873Z [msghand] [bench]   - Connect total: 367.31ms [1492.85s (9215.12ms/blk)]
2026-04-09T09:13:03.356049Z [msghand] [bench]   - Flush: 27.14ms [6.88s (42.47ms/blk)]
2026-04-09T09:13:03.356211Z [msghand] [bench]   - Writing chainstate: 0.21ms [0.03s (0.16ms/blk)]
2026-04-09T09:13:03.386096Z [msghand] UpdateTip: new best=0000000aa1aca52ec659275670747fb301cf3b61aadb9121d3bd0d5335a95a79 height=299302 version=0x20000000 log2_work=43.632985 tx=29166657 date='2026-04-09T09:11:00Z' progress=1.000000 cache=88.2MiB(676529txo)
2026-04-09T09:13:03.386141Z [msghand] [bench]   - Connect postprocess: 29.93ms [2.21s (13.63ms/blk)]
2026-04-09T09:13:03.386162Z [msghand] [bench] - Connect block: 424.65ms [1502.02s (9271.73ms/blk)]
```

So something between 60s and 100s for each block on that host. I'm happy to see that others had faster CPU's that seem to be mostly unaffected by this. These are only "slow" blocks, and by my understanding far from the worst blocks someone can produce...

I plan to post some stats on network propagation and validation time later.

-------------------------

AntoineP | 2026-04-09 13:45:40 UTC | #40

That's weird. Any chance you can share the full debug.log since the beginning of the event, that includes the other slow blocks that took only 2 seconds to validate?

-------------------------

AntoineP | 2026-04-09 13:59:26 UTC | #41

[quote="xstoicunicornx, post:33, topic:2367"]
is there copy/paste-able list of second run hashes?
[/quote]

```
00000011ee1088ab58ade91da86761478799b685699b30bc649c83f13725bc6d 000000037a35ebf60c59619042ece8ae71fecf4d3146098c966f820482d33955 0000001253d4dbf6b7b34079a70ef0357a7b12e83a38f29946effbc863f194f3 0000000b4f9d1ba885832b1b9dd4f6183483a27a4762a48114c00a812f1b42dd 00000011240c362604a624fc4469413f36ff5cb823c0733d570d54c7916a913f 000000065d332b249b5fc8068177776b3dddb073d308ae5b090147c41477e351
```

-------------------------

