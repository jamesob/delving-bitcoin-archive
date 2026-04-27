# Simple off-chain future soft fork aware testing framework in go

never | 2026-04-27 14:37:46 UTC | #1

A lot of Bitcoin off-chain R&D: statechains, vaults, eltoo prototypes, channel constructions gets written in Go, and the iteration loop is pretty consistent: spin up bitcoind in regtest, drive it via `bitcoin-cli` or hand-rolled JSON-RPC, write the protocol against that. Ethereum has
  [anvil](https://book.getfoundry.sh/anvil/) for that loop a local test node that's fast to start, easy to embed, and exposes the chain features your code actually needs.

  I wanted something close to that for Bitcoin, specifically against Inquisition so I could prototype with CTV / APO / CAT / CSFS / INTERNALKEY active rather than mocking them in. Packaged what I had:

  https://github.com/neverDefined/go-regtest

  The whole point is that the harness gets out of the way:

  ```go
  rt, _ := regtest.New(nil)   // PATH auto-detect: Inquisition first, fallback to Core
  rt.Start()
  defer rt.Stop()

  if ok, _ := rt.SupportsBIP(regtest.BIP119); !ok {
      t.Skip("requires bitcoind-inquisition")
  }
  // your statechain / vault / channel test code here.
  // CTV, APO, CAT, CSFS, INTERNALKEY are all live.
```

  Where this is meant to land:

  - A statechain implementation verifying its withdrawal templates against a live CTV-active node before publishing the design.
  - A vault prototype exercising APO-based recovery paths with real signatures rather than mocked verification.

  What made it worth its own library rather than a wrapper file copied between repos: the same test suite runs against either bitcoind-inquisition or stock bitcoind, with SupportsBIP skipping cleanly on the Core-only path. That matters for CI matrices, and for downstream libraries that want a fork-gated subset of
  tests already wired up so they light up automatically once activations happen on your local chain.

  Pre-1.0, MIT, No Docker. README has the Inquisition cmake recipe and a troubleshooting section for the friction points I hit along the way.

source: https://github.com/neverDefined/go-regtest

-------------------------

