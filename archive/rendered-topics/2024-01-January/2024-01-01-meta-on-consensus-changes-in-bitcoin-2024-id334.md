# [Meta] On consensus changes in bitcoin 2024

reardencode | 2024-01-01 16:05:12 UTC | #1

There has been much conversation on X and elsewhere recently of how, if, and when bitcoin consensus rules should be changed. I will try to briefly summarize some of the viewpoints, open the question of how consensus changes to bitcoin should be selected, and then provide my own thoughts on that question.

## Ways to change consensus

### Ossification

On the one hand, this perspective cannot be taken seriously, but it currently seems to hold sway with many in the bitcoin ecosystem.

This is the view that bitcoin consensus rules are fine as they are. It's perfect money already and does not require any improvements. There seem to be a few underlying reasons that some folks hold this view:

* Inscription/ordinal derangement: Some folks hold the opinion that previous changes to bitcoin enabled inscriptions/ordinals and that changes are therefore bad.
* General lack of understanding: For the non-technical, bitcoin seems to work by magic. Any change to a magical system could upset the delicate balance of magical properties that enable it.

### Trusted Leaders

There seem to be a group of bitcoiners who will only consider a change to the bitcoin protocol if they are supported or proposed by the correct people. This view is essentially similar to the magical ossification view. Bitcoin is a delicately balanced magical system and only the anointed saints can make the correct decisions on its future.

### Rough Consensus

This is the view that changes to bitcoin consensus should *roughly* follow the IETF rough consensus model. Here I'll quote a few sentences from the [IETF doc](https://datatracker.ietf.org/doc/html/rfc7282) on the topic:

> Requiring full consensus allows a single intransigent
   person who simply keeps saying "No!" to stop the process cold.

> Consensus doesn't require that
   everyone is happy and agrees that the chosen solution is the best
   one.

> all that they've done is capitulated; they've simply given up
   by trying to appease the others.  That's not coming to consensus [...]
> Even worse is the "horse-trading" sort of compromise

> Rough consensus is achieved when all issues are addressed, but not
    necessarily accommodated

> While counting
   heads might give a good guess as to what the rough consensus will be,
   doing so can allow important minority views to get lost in the noise.

#### How would rough consensus even work in bitcoin?

In the rough consensus model there is always a chair for any specific change discussion, and it is this person's responsibility to determine when rough consensus has been reached. We don't have chairs (because we sold them all for sats), so we lack a critical element of the rough consensus process.

## How _should_ consensus changes be chosen?

Are there other ideas for how consensus changes should be chosen? Can we follow one of the above models? This seems to be the most pressing question in bitcoin development as we roll into 2024. Historically, bitcoin consensus changes have followed some mix of *Trusted Leaders* and *Rough Consensus*. Our trusted leaders have wisely chosen not to lead consensus changes any further, and it seems like rough consensus got a bad name when certain people tried to use it as a bludgeon during the blocksize war.

## Where to?

First, I think we must accept that right now we do not have the ability to reach consensus. We lack Trusted Leaders and we lack Chairs. Without these we are afloat in our various pods of bias, preconception, personal history, etc. Not only can we not reach consensus, we don't even have the ability to bring the various factions of bitcoin developer mindshare together to find out if there are technical objections to any particular change. Consensus changing code is rarely proposed, rarely reviewed, etc.

Some have claimed that publishing or promoting a signaling / activation client for a consensus change is an attack on bitcoin, but I would counter that doing so may be the only way to discover consensus. An activation client makes the economic nodes the chairs of the rough consensus process. Each economic node, being their own chair, can evaluate whether consensus has been reached for themselves when deciding what software to run.

I'd suggest that for bitcoin to continue along the massively successful path that it has walked to date we do need to discover how to continue developing consensus changes, and that this is an important focus in the year ahead.

My proposal is that the bitcoin core project themselves begin publishing a client which supports validation of a variety of changes with separate signaling and let the signals fall where they may. Exact details of these activation clients are not my area of expertise, but I suspect that for this to be successful configuration options for signaling bits and lock-in on time would be necessary.

-------------------------

