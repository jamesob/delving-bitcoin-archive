# Lampo v23.12-beta.1 aka Santa Claus lives in Area 51

vincenzopalazzo | 2023-12-24 19:57:40 UTC | #1

First of all, Merry Christmas!

In my free time, I have been working on a lightning node using LDK, which I've named [Lampo](https://github.com/vincenzopalazzo/lampo.rs) (Italian for lightning), for more than 1.5 years.

The initial draft was developed as a toy lightning node to support lnprototest [1] for LDK. During its development, I realized I was nearing the completion of a small, self-contained SDK for building a lightning node with LDK. So, in March, I decided to take a shot at implementing a full node using the Lampo SDK.

I have currently supported the first basic commands listed in the following part of the code [2] (admittedly, it's not ideal, but a help command is on the way), and things seem to be working. I am also close to having integration tests between c-lightning (cln) and Lampo, in addition to including lnprototest protocol tests.

Hoping to attract users and contributors, I am excited to release the first public beta version v23.12 today and begin receiving initial feedback on features most desired by the community. However, I cannot promise that I will implement them all—though patches are certainly welcome, of course.

[1] https://github.com/rustyrussell/lnprototest

[2] https://github.com/vincenzopalazzo/lampo.rs/blob/0b17902058329f5cd723411c582067da2b32c65f/lampod-cli/src/main.rs#L175-L181

[3] https://twitter.com/i/communities/1736414802849706087

-------------------------

