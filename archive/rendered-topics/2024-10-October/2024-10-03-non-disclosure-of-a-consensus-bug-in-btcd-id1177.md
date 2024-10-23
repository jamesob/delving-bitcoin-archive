# Non-disclosure of a consensus bug in btcd

AntoineP | 2024-10-03 14:19:37 UTC | #1

Niklas Gögge and I reported a consensus bug in btcd to the project maintainer back in March of 2024. This bug was fixed in [btcd v0.24.2](https://github.com/btcsuite/btcd/releases/tag/v0.24.2) and the project awarded us a bounty for this finding. Although it has a minor impact on the network overall, this vulnerability is critical for btcd users: it allows an attacker to hard fork btcd nodes using a simple standard transaction (i.e. at virtually no cost).

We want to remind btcd users of the issue and **urge them to upgrade to the latest released version [v0.24.2](https://github.com/btcsuite/btcd/releases/tag/v0.24.2)**. We had agreed on a public disclosure date (23rd of September) with the maintainer but in the last two weeks we were repeatedly asked to delay the disclosure with increasingly tenuous justifications.

Transparency is important when handling security vulnerabilities. Not only for trust in the software itself but also in the release process: users deserve to know why they have been pressed to upgrade. Unless there is a good reason not to, the announced public disclosure schedule should be respected.

We considered his concerns, the most reasonable of which had to do with the number of vulnerable nodes. There are currently about [70k Bitcoin full nodes running on the network, of which about 60 are running btcd](https://luke.dashjr.org/programs/bitcoin/files/charts/software.html). From these 60 btcd nodes, 20 are still running known to be vulnerable versions (see [Niklas' previous btcd disclosure](https://delvingbitcoin.org/t/disclosure-btcd-consensus-bugs-due-to-usage-of-signed-transaction-version/455)). Of the remaining 40 nodes, 24 upgraded to the latest version with the fix for the new consensus bug discovered. This leaves only 16 vulnerable nodes which could be reasonably expected to upgrade.

Given that these 16 nodes only represent 0.022% of the Bitcoin network and we don’t know when or if they will be upgraded, we decided this was not sufficient ground for exceptionally delaying the public announcement. However, out of an abundance of caution we will refrain from disclosing the full details until October 10th. If you haven't already, **please upgrade** to [btcd v0.24.2](https://github.com/btcsuite/btcd/releases/tag/v0.24.2)!

-------------------------

roasbeef | 2024-10-03 20:00:20 UTC | #3

I thought it would be useful to add some additional context here. It is the case that
we wanted to further delay the disclosure, as we felt that disclosing just 3
months after a patch landed was too soon. Particularly given existing
precedence set by other full node implementations to delay disclosure of
sufficiently critical issues well beyond the 6 months we requested between patch
and disclosure (observe some of the timelines carried out for the recent set of
[bitcoind security disclosures
here](https://bitcoincore.org/en/security-advisories/)).

Ultimately, Niklas and AntoineP _refused_ to delay an extra 3 months (to bring
the total age of the release to 6 months), instead deciding to prematurely
disclose themselves against the wishes of the `btcd` maintainers.

-------------------------

ariard | 2024-10-04 01:03:09 UTC | #4

[quote="AntoineP, post:1, topic:1177"]
and the project awarded us a bounty for this finding.
[/quote]

Guys, I don’t know if you had the chance to be trained by people doing infosec professionally back in school, or whatever if you’re autodidact, but personally I did. And reporting security disclosure it's kinda an art, though as soon as you start to take money from the softwares vendors and wish to give your timeline it’s all very borderline…Personally, I never took any money reward for all my past disclosures and I keep doing so, to keep the interest of the end-users as a priority, among other considerations.

To clarify for everyone in the community that could read this post, I think it would be great to have the LND vendor publishing a bug bounty program with clear rules of engagement. I believe Conner Fromknecht has been working on it for a while.

-------------------------

josibake | 2024-10-04 09:09:33 UTC | #5

[quote="roasbeef, post:3, topic:1177"]
as we felt that disclosing just 3 months after a patch landed was too soon.
[/quote]

Can you provide more context on why the patch took ~3 months to land? I can understand the desire to give users more than 3 months to upgrade, which is why I am a bit surprised at the timeline given that the public disclosure was agreed to in advance.

-------------------------

AntoineP | 2024-10-04 10:01:47 UTC | #6

[quote="Crypt-iQ, post:2, topic:1177"]
Unfortunately, it seems like there is a double standard emerging in security reporting between bitcoin implementations. If this were a bitcoind security issue, I doubt there would have been the same pressure to disclosure with such a short timeline.
[/quote]

[quote="roasbeef, post:3, topic:1177"]
Particularly given existing precedence set by other full node implementations to delay disclosure of sufficiently critical issues well beyond the 6 months we requested between patch and disclosure
[/quote]

Trying to deflect this on Bitcoin Core is not a good look. If you disagreed with the timeline, why not state so back when we reported the bug? Why confirm the schedule three months later, when you publicly committed to release the details in 90 days?

Bitcoin Core now has a disclosure policy. Had we reported it to this project and two days before the deadline maintainers requested us not to respect the schedule *with no good reason*, it would have played the same. But i doubt they would do that in the first place.

[quote="roasbeef, post:3, topic:1177"]
I thought it would be useful to add some additional context here. It is the case that we wanted to further delay the disclosure, as we felt that disclosing just 3 months after a patch landed was too soon.
[/quote]

Again, it's curious how this sudden concern only made its appearance a few days before the deadline. Not when you actually released and stated you would disclose the details 90 days later.

[quote="roasbeef, post:3, topic:1177"]
Ultimately, Niklas and AntoineP *refused* to delay an extra 3 months (to bring the total age of the release to 6 months), instead deciding to prematurely disclose themselves against the wishes of the `btcd` maintainers.
[/quote]

We indeed refused to arbitrarily make an (other) exception to the published schedule. However we still haven't released the details of the vulnerability. We asked you numerous times privately to give us a good reason not to respect the schedule. I'll restate it publicly. **We're happy to withhold the details should there be any reasonable concern about publishing them according to the initial schedule.**

-------------------------

ajtowns | 2024-10-09 06:24:15 UTC | #7

[quote="roasbeef, post:3, topic:1177"]
we felt that disclosing just 3 months after a patch landed was too soon. Particularly given existing [precedents] set by other full node implementations to delay disclosure of sufficiently critical issues well beyond the 6 months we requested between patch and disclosure
[/quote]

Vendors tend to have a very strong bias towards always feeling that disclosing is too soon; having security researchers push back against that has been a relatively recent trend, and a good one.

For example, Google's [Project Zero timeline](https://googleprojectzero.blogspot.com/p/vulnerability-disclosure-faq.html) (for public disclosure of problems they uncover) is 90 days from disclosure to the software maintainers, or 30 days after the fix is made available, as long as that's no more than 120 days after the initial disclosure. So that would presumably have resulted in public disclosure on or before 29th June (90 days after 31st March, 3 days after 0.24.2's release), or 26th July (117 days after 31st March, 30 days after 0.24.2's release). By that measure, the 10th of October is already at least 193 days / six months since initial disclosure.

Bitcoin Core still defaults to a much more conservative approach than Google does here, but even so, a "consensus bug [..that..] allows an attacker to hard fork [..] nodes using a simple standard transaction" would certainly be classed as a "critical" severity bug, and even according to core's policy that would "most likely require an ad-hoc procedure".

As far as btcd goes, I would have thought that the commitment in the release notes ("release includes **important security** fixes, with full details to be disclosed in **90 days**.") would be honoured, especially given that the project's [security policy](https://github.com/btcsuite/btcd/?tab=security-ov-file) is that only the last major version is considered supported anyway.

-------------------------

AntoineP | 2024-10-10 09:03:48 UTC | #8

Just published [the details](https://delvingbitcoin.org/t/cve-2024-38365-public-disclosure-btcd-findanddelete-bug/1184?u=antoinep) for this vulnerability, after btcd released [the security advisory](https://github.com/btcsuite/btcd/security/advisories/GHSA-27vh-h6mc-q6g8).

-------------------------

real-or-random | 2024-10-23 20:35:37 UTC | #9

[quote="roasbeef, post:3, topic:1177"]
Ultimately, Niklas and AntoineP *refused* to delay an extra 3 months (to bring the total age of the release to 6 months), instead deciding to prematurely disclose themselves against the wishes of the `btcd` maintainers.
[/quote]

Ultimately, the `btcd` maintainers shipped vulnerable software.

The idea that discovery comes with a moral responsibility to cooperate with the maintainer is flawed. People need to understand that discoverers don't owe maintainers (or users or anybody) anything. Not a timeline, not a specific method of disclosure, and not a specific audience. And particularly not that they spend their time having a debate about any of these.

-------------------------

