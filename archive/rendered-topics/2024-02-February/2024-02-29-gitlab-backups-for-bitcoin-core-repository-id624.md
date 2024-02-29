# GitLab Backups for Bitcoin Core repository

fjahr | 2024-02-29 16:29:34 UTC | #1

(This is a Meta post but for Bitcoin Core, so I wasn't sure where to put it. Feel free to move it but I think it might also be a good idea to have "Dev Tools" category or something similar.)

Last year, I set out to investigate using a self-hosted GitLab instance as an option to create backups of the Bitcoin Core GitHub repo which could not only persist the information but also be browsed and used as a basis to continue development quickly, no matter whether the project might want to switch any time in the future or be forced to do so.

While the setup of a GitLab server and initial trial runs looked like this project might be a matter of days, getting a full backup with all data to succeed took a lot of research, conversations with GitLab support, and also much trial & error. Earlier this week was the first time a full backup run succeeded. The result can be seen here with the latest data being from 2024-02-26: https://gitlab.sighash.org/bitcoin/bitcoin

I have documented all the relevant findings in this gist: https://gist.github.com/fjahr/9a28abefe0ab8413d96aa1dd7903c5d4 It should have all the necessary information for others to set up their own GitLab backup server. If anything is unclear or doesn't work, please send me your feedback.

I will leave the backup untouched at least until the end of next week so that people can look at the data and give feedback. I will then run a script that performs the backup regularly and persists it as well. But the downside of this will be that the data can not be viewed anymore most of the time since this is not possible while the import to GitLab is running and the import process apparently takes up to 36 hours due to GitHub API restrictions.

Last but not least, of course, GitLab can also be used to back up other repositories that are not Bitcoin Core. But in that case, you probably don't even need all of the special configurations because these are mostly needed due to the scale, contributor base and level of activity on Bitcoin Core.

-------------------------

