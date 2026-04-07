# About the Protocol Design category

admin | 2024-02-22 07:49:42 UTC | #1

Bitcoin-related protocol proposals, or analysis or discussion thereof. For example, new P2P messages, new wallet techniques, methods for coinjoins or payment channels, etc.

---

Try to keep things accessible to developers -- so don't use formal academic terminology that you'd write in a paper, when your could write things more simply.

If you want to use maths, you can use LaTeX (mathjax) markup by surrounding inline formulas with `$`, for example: `$ 1+1=2 $` gives $ 1+1=2 $, or surrounding a standalone block with `$$`:

$$
1+1=2
$$

If you want to include diagrams, then [mermaid diagrams](https://mermaid.js.org/) are available, in particular [flowcharts](https://mermaid.js.org/syntax/flowchart.html) and [sequence diagrams](https://mermaid.js.org/syntax/sequenceDiagram.html). If you want to describe transactions in detail [class diagrams](https://mermaid.js.org/syntax/classDiagram.html) may work okay to separate inputs from outputs if you include brackets when naming the outputs (eg `p2pkh(A)`) but not the inputs (eg "sigA").

```mermaid height=59,auto
flowchart LR
    A ---> B <---> C o---o D
```

-------------------------

