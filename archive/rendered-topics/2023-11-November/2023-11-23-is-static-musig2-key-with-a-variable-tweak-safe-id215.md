# Is static Musig2 key with a variable tweak safe?

t-bast | 2023-11-23 11:06:55 UTC | #1

I'd like to create taproot addresses where the key path is using musig2 with a script path that contains a refund script, that can be spent with a single key after a delay:

- key path: `user_key` + `server_key`
- script path: `user_refund_key` + `refund_delay`

I'd like to avoid address reuse, so I want to rotate the user keys.
What I'm wondering is if it's safe to only rotate the `user_refund_key`?

That would keep the `user_key` and `server_key` static, which makes it easier to express in a descriptor used for refunds and easier to manage on the server side.

-------------------------

instagibbs | 2023-11-23 13:54:36 UTC | #2

Seems completely standard? Any reason why you'd expect it wouldn't be safe?

-------------------------

t-bast | 2023-11-23 14:43:21 UTC | #3

Not at all, I do think this is safe as well, but I wanted to have wizard confirmations before we implement and deploy this! :sweat_smile:

-------------------------

sipa | 2023-11-23 15:42:49 UTC | #4

So your proposal is to effectively use (abusing notation slightly):

    tr(musig2(user,server),[and(pk(user_refund),older(delay))])

The only potential concern I see with only rotating `user_refund` is that if you have two distinct outputs which both get spent through the script path, they can be recognized as being from the same partie(s), as the key path will be identical.

-------------------------

