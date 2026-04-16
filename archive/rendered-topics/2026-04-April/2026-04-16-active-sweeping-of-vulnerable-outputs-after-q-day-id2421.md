# Active Sweeping of Vulnerable Outputs after Q-day

show1225 | 2026-04-16 06:26:04 UTC | #1

The debate between protocol-level freezing (with seed phrase or xpriv recovery options) versus strict non-intervention (no confiscation) is heating up. Because this disagreement is rooted in fundamental philosophical differences(\*), there could be no easy solution.

Problems that “no confiscation” approach poses are:

1. No sane entity would steal vulnerable coins even if they have CRQC from legal standpoint. Only rogue actors will do.
2. Those rogue actors will not return coins to original asset holders.
3. Those rogue actors could just dump and collapse Bitcoin

Then, why don’t we set up a trust which actively sweeps vulnerable outputs using CRQC?
Of course, **it will neither be enshrined into Bitcoin protocol nor be proposed as BIP**.
The important point is that **it does not violate morals of those who oppose any form of  confiscation.**
A trustless way to return coins back to original holders is the best. But if we cannot agree on confiscation, then this could be the second best.

How the trust works:

1. It will sign treaties with as many QC companies as it can to run their QCs to save insecure coins
2. It will release coins minus operation fees when a (cryptographic) proof of ownership is provided
3. Unclaimed assets will stay in the trust forever or will be used to fund development of Bitcoin if not claimed for a very long time.

**Bitcoin protocol remains trustless, decentralized and non confiscatory**. **Those who migrate to PQ output in time will face no issues. Those who do not migrate but have not revealed their pubkeys can initiate transactions anytime** (although there is a chance that fast-clock QCs snatch your coin). Just vulnerable coins will be swept and managed by an independent, voluntary entity to prevent them from being stolen by rogues.

Whether such a trust could successfully front-run rogue actors to sweep all insecure coins is uncertain. However, as an independent, voluntary initiative rather than an official protocol mandate, they would do their best.

Let me clarify again that this is NOT the best solution, but forming such a white-hat coalition just in case could also be a good thing.

SHO

(\*)Proponents of “freezing” approach argue:

1. Although confiscation is a sad event, we should pursue maximum recovery by disabling insecure spending path and enabling a recovery method
2. Dumping after mass stealing of dormant coins and the belief that such an event is inevitable will devalue everybody else’s coins (i.e. property rights)
3. Sudden collapse of Bitcoin price might halt mining activities
4. Insecure coins after such a long period of dormant period can be considered as lost (or at least recovery with xpriv / seedphrase should be prioritized than keeping dormant coins spendable)
5. Bitcoin’s property right is based on the assumption that the underlying cryptography is secure and their keys are managed correctly

Proponents of “no-confiscation” approach argue:

1. Bitcoin must prioritize absolute property rights above all else
2. Any form of protocol-level freezing would establish a dangerous confiscatory precedent
3. We should not speculate or assume that dormant coins are truly “lost” or abandoned by their owners
4. No one can prevent anybody from doing stupid (insecure) way of storing their coins

-------------------------

