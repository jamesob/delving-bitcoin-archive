# Tapsim: Bitcoin Tapscript Debugger

halseth | 2023-12-14 13:11:52 UTC | #1

Hi, I want to announce a tool I've been working on, that has been tremendously helpful in testing out new script primitives and debug bitcoin scripts: [Tapsim](https://github.com/halseth/tapsim)

> # Tapsim: Bitcoin Tapscript Debugger
> Tapsim is a simple tool built in Go for debugging Bitcoin Tapscript transactions. It's aimed at developers wanting play with Bitcoin script primitives, aid in script debugging, and visualize the VM state as scripts are executed.

It is useful for visualizing the execution environment during script execution, and stepping forwards and backwards through the script.

```bash
$ ./tapsim execute --script "OP_HASH160 79510b993bd0c642db233e2c9f3d9ef0d653f229 OP_EQUAL" --witness "54"
Script: OP_HASH160 79510b993bd0c642db233e2c9f3d9ef0d653f229 OP_EQUAL
Witness: 54
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 script                                  | stack                                   | alt stack                               | witness
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 OP_HASH160                              | 01                                      |                                         |
 OP_DATA_20 0x79510b993bd0c642db23...f229|                                         |                                         |
 OP_EQUAL                                |                                         |                                         |
>                                        |                                         |                                         |
------------------------------------------------------------------------------------------------------------------------------------------------------------------------


script execution verified
```

It is based on the `btcd` script interpreter, which makes it easy (IMO, as a Go developer) to add additional opcodes to play around with. Currently I've added support OP_CAT and OP_CHECKCONTRACTVERIFY.

Hopefully more people working with Bitcoin script can find it helpful!

All feedback or contributions welcome of course :smile:

-------------------------

