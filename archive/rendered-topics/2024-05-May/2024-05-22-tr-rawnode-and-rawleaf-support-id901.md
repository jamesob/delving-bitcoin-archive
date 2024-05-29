# Tr(): rawnode() and rawleaf() support

Eunovo | 2024-05-22 15:25:38 UTC | #1

Previous discussions on [Issue 24114](https://github.com/bitcoin/bitcoin/issues/24114) and [Partial Descriptors Gist](https://gist.github.com/sipa/06c5c844df155d4e5044c2c8cac9c05e#partial-descriptors) suggests two new descriptors for `tr()`:
- `rawnode(HEXHASH)` which allows the specification of a branch using the merkle hash
- `rawleaf(HEXSCRIPT,[HEXLEAFVER])` which allows the addition of raw scripts with different leaf versions

Arguments for `rawnode()` include:
- Adding support for Need-to-Know-Branches (N2KB) see [Jeremy's comment on 21365](https://github.com/bitcoin/bitcoin/pull/21365#issuecomment-793027851)
- Allowing users to specify TR trees with omitted information
- Allowing users to simplify TR trees  
As an example, this descriptor:
```
tr(
  5a1e61f898173040e20616d43e9f496fba90338a39faa1ed98fcbaeee4dd9be5,{
    multi_a(
      2,
      tprv8btqJtFL5n1drr8jP95PBgTitQiCJyxDb27gGhbXM2vacGt3gV2Bfs5PHSBt6LbpPs5fUb4hGhD5TXudLrisjd14qtM8EZ3nVSv31jhLuop/*,
      415f59b2ce1ad8f714fd40b71865937dd911ad4742724cead5db936f73029d1e,1a3569fb509ababa09cf896c0a66619d0a01fd235d26b479a42503670beb9754
    ),
   {
     {
       and_v(
         v:older(4320),
         multi_a(
           2,
           ae3f20aee8d8798ceafb1671bf5607a45794676498e2edfac042958f7407ee1c,
           tpubD8asTJHaE9hJkKAXGnjyb67qTSE8UK98AKiTZDdpmJiySm8pJsqmrMhFTZV9cQyCdisA71G9bjGCZXcjXgFWCfjK5rfxVYmbnA5XikwzEHt/*
         )
       ),
       and_v(
         v:older(4320),
         multi_a(
           2,
           ae3f20aee8d8798ceafb1671bf5607a45794676498e2edfac042958f7407ee1c,
           415f59b2ce1ad8f714fd40b71865937dd911ad4742724cead5db936f73029d1e
         )
       )
    },
    {
      and_v(
        v:older(4320),
        multi_a(
          2,
          ae3f20aee8d8798ceafb1671bf5607a45794676498e2edfac042958f7407ee1c,
          1a3569fb509ababa09cf896c0a66619d0a01fd235d26b479a42503670beb9754
        )
      ),
      and_v(
        v:older(12960),
        pk(ae3f20aee8d8798ceafb1671bf5607a45794676498e2edfac042958f7407ee1c)
      )
    }
  }
})#vz6zss2j
```
has been reduced to:
```
tr(
  5a1e61f898173040e20616d43e9f496fba90338a39faa1ed98fcbaeee4dd9be5,
    {
        multi_a(
          2,
          tprv8btqJtFL5n1drr8jP95PBgTitQiCJyxDb27gGhbXM2vacGt3gV2Bfs5PHSBt6LbpPs5fUb4hGhD5TXudLrisjd14qtM8EZ3nVSv31jhLuop/*,
          415f59b2ce1ad8f714fd40b71865937dd911ad4742724cead5db936f73029d1e,
          1a3569fb509ababa09cf896c0a66619d0a01fd235d26b479a42503670beb9754
        ),
        rawnode(8a62dc0a100c4156cc2a4b7c2a97747ce0dfe90562673fd662678aaae93121fb)
    }
)
```
See [test case verifying this](https://github.com/Eunovo/tr-partial-descriptors-test/blob/2449ccddbe7132fa20c2f607e2a807542f07f875/src/main.py#L17-L59)

While it would seem that `rawleaf` offers no advantage since we have `rawnode`, @sipa makes the argument that since the script and leaf version can be communicated via PSBTs, then descriptors should also be able to pass this information [See Sipa's comment on 24114](https://github.com/bitcoin/bitcoin/issues/24114#issuecomment-1038441246)

There's an unanswered question about whether `rawleaf(HEXSCRIPT,[HEXLEAFVER])` should just be `raw()` that takes an optional `HEXLEAFVER` under `tr()` but I prefer `rawleaf` to avoid overloading the `raw()` descriptor.

## Implementation
I have created a PoC here [wip-tr-raw-nodes](https://github.com/Eunovo/bitcoin/tree/wip-tr-raw-nodes), see [diff](https://github.com/bitcoin/bitcoin/compare/master...Eunovo:bitcoin:wip-tr-raw-nodes).  I created two new `DescriptorImpl` subclasses, `RawNodeDescriptor` and `RawLeafDescriptor`. Both are only valid under `P2TR` context and can be inferred by `InferDescriptor`. 

### Descriptor Inference
- If no taptree is inferred, and the Merkle root is not null, then `tr(internal_key,rawnode(merkle_root))` is returned instead of the previous  `rawtr(tr_public_key)`. I am not sure if this should be the case. *What advantage does inferring `tr(internal_key,rawnode(merkle_root))` have over inferring `rawtr(tr_public_key)`?*
- `rawleaf` is inferred after no standard scripts could be inferred under a `P2TR` context.

-------------------------

josibake | 2024-05-22 15:28:00 UTC | #2

cc @ajtowns @sipa @AntoineP , since this work is based off your prior discussions.

-------------------------

sipa | 2024-05-28 15:49:08 UTC | #3

Thank you for working on this!

[quote="Eunovo, post:1, topic:901"]
What advantage does inferring `tr(internal_key,rawnode(merkle_root))` have over inferring `rawtr(tr_public_key)`?
[/quote]

It provides strictly more information: the internal key. Philosophically, I think, the idea of partial descriptors is that any subset of relevant information should be conveyable, so that would be sufficient justification for having it. But practically, imagine a signer that recognizes the internal key, but does not recognize the tweaked output key; having the `internal_key` and `merkle_root` would enable them to locate the correct key.

-------------------------

Eunovo | 2024-05-28 17:37:26 UTC | #4

Makes sense. I didn't see it that way. Thanks! @sipa

-------------------------

Eunovo | 2024-05-28 17:41:55 UTC | #5

@sipa I wonder if you have any ideas about how N2KBs are supposed to be used. I imagine that other parties of the transaction will fear hidden spend paths.

Would be cool to get @JeremyRubin take as well

-------------------------

ajtowns | 2024-05-28 21:20:57 UTC | #6

[quote="Eunovo, post:5, topic:901"]
@sipa I wonder if you have any ideas about how N2KBs are supposed to be used. I imagine that other parties of the transaction will fear hidden spend paths.
[/quote]

Could be useful for inheritance/escrow type services: "this isn't my money, so I don't care about other spend paths, but in an emergency, the spend path I do know about can be used to help that person recover their money even if they've lost all their keys".

Could be useful for places where you're using multiple wallets for some funds -- maybe your main wallet understands some super clever contracts that you want to put in a tapscript, but your backup (hardware? cold?) wallet doesn't and can only spend via the key path; in that case all the backup wallet needs to know is the ipk and a rawnode leaf.

-------------------------

Eunovo | 2024-05-29 08:00:27 UTC | #7

Thanks for the clarification @ajtowns !

-------------------------

