# Mutual exclusiveness of op_codes

PierreRochard | 2024-05-21 03:35:59 UTC | #1

Doing a very surface-level readthrough of current softfork proposals, a good amount of narrative seems to be "X is better than Y because..." implying a mutually exclusive predicament, that we have to choose one proposal over another. Without picking a side of whether that's true or not, I'd like to at least understand the philosophical arguments on both sides. 

Some initial thoughts:

* Needing to maintain more code. In some cases where the op_code is trivial like OP_CAT this seems inconsequential, in other cases like Simplicity it's a very substantive point. 
* Attack / bugs / vulnerability surface area, not always correlated with lines of code. 
* Limited number of op_code "slots" available to take.
* Activation politics, bundling could increase contention but separating might exhaust participants' energy too.
* Ecosystem complexity, multiple similar options to accomplish the same outcome.                                                                                                                                                                                                      

I just philosophically like the idea of letting developers and users opt in to the op_code they want even if there's a 90% overlap between two different op_codes. They might find one easier to grok or have a very particular use case. As long as node resource usage limits are respected and protocol maintainers are not overwhelmed, it seems fairly innocuous. Interested in hearing your thoughts!

-------------------------

