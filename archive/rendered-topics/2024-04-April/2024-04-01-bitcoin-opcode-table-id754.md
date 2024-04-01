# Bitcoin Opcode Table

doglegs | 2024-04-01 15:37:39 UTC | #1

Opcodes are the fundamental building blocks of Bitcoin's scripting language, defining conditions for spending transactions. Each opcode represents a specific instruction executed within Bitcoin's virtual machine, enabling complex scripting capabilities. Opcodes are executed sequentially to validate transactions against scriptPubKey and scriptSig, determining if inputs fulfill the conditions for spending outputs.

The scripting language employs opcodes to perform operations like data manipulation, conditional execution, cryptographic verification, and arithmetic computations. This limited set ensures predictable script evaluation, mitigating risks from malicious or indefinitely looping scripts.

Opcodes enable smart contracts and multi-signature wallets within Bitcoin. Conditional logic opcodes (e.g., OP_IF, OP_NOTIF) and cryptographic operations (e.g., OP_CHECKSIG, OP_CHECKMULTISIG) enforce specific spending conditions, such as requiring multiple signatures or locking funds until a certain block height or timestamp (via OP_CHECKLOCKTIMEVERIFY and OP_CHECKSEQUENCEVERIFY).

| Opcode   | Description | Function |
|----------|-------------|-------------------|
| 0        | OP_0, OP_FALSE | Push false |
| 76       | OP_PUSHDATA1 | Push data |
| 77       | OP_PUSHDATA2 | Push data |
| 78       | OP_PUSHDATA4 | Push data |
| 79       | OP_1NEGATE | Push -1 |
| 80-96    | OP_1-OP_16, OP_TRUE | Push 1-16 |
| 97       | OP_NOP | No operation |
| 99       | OP_IF | If conditional |
| 100      | OP_NOTIF | If not conditional |
| 103      | OP_ELSE | Else conditional |
| 104      | OP_ENDIF | End if |
| 105      | OP_VERIFY | Verify condition |
| 106      | OP_RETURN | Terminate script |
| 107-108  | OP_TOALTSTACK, OP_FROMALTSTACK | Stack transfer |
| 109      | OP_2DROP | Drop 2 |
| 110      | OP_2DUP | Duplicate 2 |
| 111      | OP_3DUP | Duplicate 3 |
| 112      | OP_2OVER | Copy 2nd pair |
| 113      | OP_2ROT | Rotate top 3 twice |
| 114      | OP_2SWAP | Swap top 2 pairs |
| 115      | OP_IFDUP | Duplicate if not 0 |
| 116      | OP_DEPTH | Stack size |
| 117      | OP_DROP | Remove top |
| 118      | OP_DUP | Duplicate top |
| 119      | OP_NIP | Remove 2nd |
| 120      | OP_OVER | Copy 2nd |
| 121      | OP_PICK | Nth item |
| 122      | OP_ROLL | Move Nth top |
| 123      | OP_ROT | Rotate top 3 |
| 124      | OP_SWAP | Swap top 2 |
| 125      | OP_TUCK | Move 3rd to top |
| 130      | OP_SIZE | Size of top item |
| 135      | OP_EQUAL | Equality check |
| 136      | OP_EQUALVERIFY | Verify equal |
| 139      | OP_1ADD | Add 1 |
| 140      | OP_1SUB | Subtract 1 |
| 143      | OP_NEGATE | Negate |
| 144      | OP_ABS | Absolute value |
| 145      | OP_NOT | Not zero |
| 146      | OP_0NOTEQUAL | Not equal to 0 |
| 147-148  | OP_ADD, OP_SUB | Arithmetic |
| 149-154  | Disabled | Disabled |
| 155-158  | OP_BOOLAND, OP_BOOLOR, OP_NUMEQUAL, OP_NUMEQUALVERIFY | Boolean logic |
| 159-160  | OP_NUMNOTEQUAL, OP_LESSTHAN | Comparison |
| 161-162  | OP_GREATERTHAN, OP_LESSTHANOREQUAL | Comparison |
| 163-164  | OP_GREATERTHANOREQUAL, OP_MIN | Minimum/Maximum |
| 165-166  | OP_MAX, OP_WITHIN | Range check |
| 167-168  | OP_RIPEMD160, OP_SHA1 | Hashing |
| 169      | OP_SHA256 | SHA-256 |
| 170      | OP_HASH160 | RIPEMD-160+SHA-256 |
| 171      | OP_HASH256 | SHA-256x2 |
| 172-175  | OP_CHECKSIG, OP_CHECKSIGVERIFY, OP_CHECKMULTISIG | Signature check |
| 176-177  | OP_CHECKMULTISIGVERIFY | Multisig verify |
| 178-181  | OP_NOP1-OP_NOP4 | No operation |
| 182-185  | OP_CHECKLOCKTIMEVERIFY, OP_CHECKSEQUENCEVERIFY | Timing checks |
| 186-189  | OP_NOP6-OP_NOP9 | No operation |
| 190-193  | OP_NOP10 | No operation |

-------------------------

