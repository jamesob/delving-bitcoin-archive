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

