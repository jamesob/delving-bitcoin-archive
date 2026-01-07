# Consensus bug on NBitcoin: out-of-bound issue in `Remove()`

bruno | 2025-11-24 19:03:42 UTC | #1

We recently integrated `NBitcoin` into the `script_eval` target of bitcoinfuzz.
This target performs *differential fuzzing* of Bitcoin script evaluation logic by comparing the behavior of EvalScript (or equivalent) implementations across different projects.
Before adding NBitcoin, this target was already fuzzing Bitcoin Core and btcd.

Shortly after starting the fuzzing campaign — thanks to an already good corpus — we encountered a crash with the following script:

```
Hex: 000360670000005b770100000000005b770100000000000008777760779f00000877776077
Asm: 0 26464 0 0 11 OP_NIP 0 0 0 0 0 11 OP_NIP 0 0 0 0 0 0 777760779f000008 OP_NIP OP_NIP 16 OP_NIP
```

When debugging the run output, we observed that NBitcoin returned **false** while Bitcoin Core returned **true**. Since this represented a clear consensus discrepancy, we manually reviewed the script to verify whether it was valid. The analysis confirmed that the script is valid, meaning that Bitcoin Core’s behavior is correct — and that the discrepancy originated from a bug in NBitcoin.

Initially, this was surprising, since the only opcode in the script is `OP_NIP`, which removes the second item from the top of the stack — a relatively simple operation.

NBitcoin’s implementation of this opcode is shown below:

```csharp
case OpcodeType.OP_NIP:
{
    // (x1 x2 -- x2)
    if (_stack.Count < 2)
        return SetError(ScriptError.InvalidStackOperation);

    _stack.Remove(-2);
    break;
}
```

To better understand the issue, we created a test case and stepped through `EvalScript`.
During execution, an `IndexOutOfRangeException` was thrown when processing the third `OP_NIP`.

Further investigation revealed the following:

> When the underlying array is at full capacity (e.g., 16 elements), and \_stack.Remove(-2) is called, the Remove operation must delete the item at index 14 and shift the subsequent elements down. During this shift, the implementation may attempt to access \_array\[16\], which does not exist, leading to the IndexOutOfRangeException.

Interestingly, this bug was only discovered through differential fuzzing, because the discrepancy in return values between `Bitcoin Core` and `NBitcoin` highlighted the problem.
Since the exception occurs within a try/catch block (preventing a full crash), fuzzing by itself might not catch it.

Timeline:

* 2025-10-23 - Bruno Garcia reports the issue to Nicolas Dorier
* 2025-10-23 - Nicolas Dorier confirmed the issue
* 2025-10-23 - Nicolas Dorier opens [#1288](https://github.com/MetacoSA/NBitcoin/pull/1288) to fix it
* 2025-10-23 - NBitcoin 9.0.3 is released

*Note: There is no known node implementation using NBitcoin, so there is no risk of chain split. That's why we're making it public.*

-------------------------

