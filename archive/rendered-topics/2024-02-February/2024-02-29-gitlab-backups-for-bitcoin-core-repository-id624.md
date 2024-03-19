# GitLab Backups for Bitcoin Core repository

fjahr | 2024-02-29 16:29:34 UTC | #1

(This is a Meta post but for Bitcoin Core, so I wasn't sure where to put it. Feel free to move it but I think it might also be a good idea to have "Dev Tools" category or something similar.)

Last year, I set out to investigate using a self-hosted GitLab instance as an option to create backups of the Bitcoin Core GitHub repo which could not only persist the information but also be browsed and used as a basis to continue development quickly, no matter whether the project might want to switch any time in the future or be forced to do so.

While the setup of a GitLab server and initial trial runs looked like this project might be a matter of days, getting a full backup with all data to succeed took a lot of research, conversations with GitLab support, and also much trial & error. Earlier this week was the first time a full backup run succeeded. The result can be seen here with the latest data being from 2024-02-26: https://gitlab.sighash.org/bitcoin/bitcoin

I have documented all the relevant findings in this gist: https://gist.github.com/fjahr/9a28abefe0ab8413d96aa1dd7903c5d4 It should have all the necessary information for others to set up their own GitLab backup server. If anything is unclear or doesn't work, please send me your feedback.

I will leave the backup untouched at least until the end of next week so that people can look at the data and give feedback. I will then run a script that performs the backup regularly and persists it as well. But the downside of this will be that the data can not be viewed anymore most of the time since this is not possible while the import to GitLab is running and the import process apparently takes up to 36 hours due to GitHub API restrictions.

Last but not least, of course, GitLab can also be used to back up other repositories that are not Bitcoin Core. But in that case, you probably don't even need all of the special configurations because these are mostly needed due to the scale, contributor base and level of activity on Bitcoin Core.

-------------------------

0xB10C | 2024-03-06 14:36:26 UTC | #2

Thanks for looking into this and sharing what you've found. Sad to hear that a continues import between GitLab and GitHub isn't possible.

Would it be possible to run two instances where one is importing while the other is displaying and then switch them over once the other is done importing and so on? Could also do this on a 48h timer (you mentioned the import takes 36h).

---

I've also set up some GitHub backup and mirroring of Bitcoin related repositories last year. I think putting some links here makes sense in case someone is looking for backups or a mirror. Sorry for hijacking your post.

I have the backups and a mirror on https://mirror.b10c.me. For example, [https://mirror.b10c.me/bitcoin-bitcoin](https://mirror.b10c.me/bitcoin-bitcoin) shows a GitHub like read-only bitcoin/bitcoin repository mirror. I'm also rewriting issue/PR links to not link back to GitHub. There's also a Tor hidden service for the mirror if anyone happens to need it: e3y5vky4v7snefqyhbn6kcmyl5fo4cnk3a2irh2ttvwua46ww5ubl6qd.onion.

I also push the backups to GitHub (duh) and GitLab:
- https://github.com/bitcoin-data/github-metadata-backup-bitcoin-bitcoin
- https://github.com/bitcoin-data/github-metadata-backup-bitcoin-core-secp256k1
- https://github.com/bitcoin-data/github-metadata-backup-bitcoin-core-gui
- https://github.com/bitcoin-data/github-metadata-backup-bitcoin-bips
- https://gitlab.com/0xb10c/github-metadata-backup-bitcoin-core-gui
- https://gitlab.com/0xb10c/github-metadata-backup-bitcoin-core-secp256k1
- https://gitlab.com/0xb10c/github-metadata-backup-bitcoin-bips
- https://gitlab.com/0xb10c/github-metadata-backup-bitcoin-bitcoin

Code for the backups can be found in https://github.com/0xB10C/github-metadata-backup and for the mirroring in https://github.com/0xB10C/github-metadata-mirror. I've also written a few words on it here: https://b10c.me/projects/021-github-backups-mirror/

-------------------------

fjahr | 2024-03-14 19:55:23 UTC | #3

Thanks for the comments!

[quote="0xB10C, post:2, topic:624"]
Would it be possible to run two instances where one is importing while the other is displaying and then switch them over once the other is done importing and so on? Could also do this on a 48h timer (you mentioned the import takes 36h).
[/quote]

Yes, that would be possible, we wouldn't even need to switch, we could have one that constantly clones from Github and then every time it is done the second one can get a GitLab to GitLab clone which is a lot faster because it's just dumping the databases and not using any APIs. This should be more comfortable for viewers because they would only have one go-to url. Though I am not sure if it's worth the effort honestly. I don't expect a lot of people to want to look at the data before we actually need it. Did you have specific use cases in mind for making the data viewable? I guess we should somehow be checking that the backup still works and there is no garbage data coming in but I think that should also be doable via a script.

-------------------------

fjahr | 2024-03-14 20:00:11 UTC | #4

Noting here as well that I talked about this at the Optech podcast and there were some good questions from the audience that are interesting considerations for next steps, so definitely worth a listen: https://bitcoinops.org/en/podcast/2024/03/07/

-------------------------

0xB10C | 2024-03-15 08:07:32 UTC | #5

> I don’t expect a lot of people to want to look at the data before we actually need it.

The mirroring and using that mirror is part of my process to make sure the backups work. My [bitcoin-bitcoin mirror](https://mirror.b10c.me/bitcoin-bitcoin) is my go to place on mobile to check when wanting to stay up to date with new PRs and issues. A script should also work, yes.

-------------------------

0xB10C | 2024-03-15 09:54:03 UTC | #6

It might make sense to add a section on existing backups in https://github.com/bitcoin-core/bitcoin-devwiki/wiki/GitHub-alternatives-for-Bitcoin-Core. What do you think?

-------------------------

fjahr | 2024-03-15 17:03:20 UTC | #7

Yeah, definitely. I had forgotten about that page honestly :D Let me know if you want to give it a go first and ping me for review or if I should make a first draft.

-------------------------

0xB10C | 2024-03-19 13:17:31 UTC | #8

I've added a section on backups and tooling to the wiki: https://github.com/bitcoin-core/bitcoin-devwiki/wiki/GitHub-alternatives-for-Bitcoin-Core#repository-backups-and-tooling. Feel free to add your's too.

-------------------------

