# [Research] A clockless vardiff strands a slowing miner

gimballock | 2026-07-15 06:12:57 UTC | #1

A clockless vardiff strands a slowing miner
===========================================

> *A note on where this came from.* This began as an attempt to optimize vardiff, not to write about it. We were building a controller to track a miner more tightly, and noticed that an asymmetric response — quick to ease difficulty down, slow to push it back up — kept miner connections alive through declines that dropped the ones that adjusted evenly. We couldn't fully explain why, and following that clue is what turned into the characterization here. It was worked out through an extended back-and-forth with an AI (Claude), with ideas going both ways; the direction, the judgment calls, and the verification are mine, and the empirical claims rest on the shaping proxy in §7 and on direct review of controller source. Fuller details — the simulation framework, the earlier optimization arc — are available on request.

Picture a miner that slows down — throttled for grid demand response, cooling in the heat, curtailed on purpose; whatever the reason, its hashrate drops but its pool difficulty stays pinned where it was, set for the faster machine it used to be. Now every share is far harder than it should be, so the miner barely produces any. And here's the trap: the fewer shares it sends, the less the pool can tell what its rate actually is, and the less able the controller is to ease the difficulty back down. The miner has slowed, the difficulty hasn't followed, and the one signal that would let the controller notice (shares) is exactly the signal that just dried up.

> INFO: For those new to bitcoin mining (or PoW in general), a `share` is an "almost block" or a "weak block". It has Proof of Work but it is of insufficient difficulty for the broader network to accept as a block, while providing evidence of hashrate. Mining pools and proxies use these shares to know that a miner is connected and hashing. Shares are also used for accounting reward distribution. For the statistical math behind bitcoin mining shares and accounting, look into Poisson distribution. Hashrate is a guessing game from the chip to network level. Variable Difficulty (vardiff) is generally applied per-connection for a mining pool. From here, the article assumes familiarity with mining shares so will not elaborate further.

Some vardiff controllers dig out of this slowly. One that can only move in fixed steps may not dig out at all: if it can't build enough pressure to take a step, it doesn't move. Either way, the miner spends that whole stretch stranded at a difficulty it can't work its way down from, producing a fraction of the shares it should.

This isn't a tuning bug you can fix by picking better numbers. It's structural, and the cleanest way to see why is to notice what vardiff actually is: a feedback loop. The share stream is the sensor, the difficulty is the actuator, and the miner's hashrate is what the loop is tracking. A vardiff controller measures the share rate, compares it to a target, and moves difficulty to drive the difference toward zero — ordinary closed-loop control.

The problem is that this particular sensor fails in one particular direction. Shares arrive more slowly the harder the difficulty is relative to the miner, so when the miner slows, the feedback signal thins out — and when the miner goes quiet, it vanishes. The controller goes blind exactly when difficulty is set too high — the situation it most needs to fix. And a feedback controller that has lost its sensor, with no other way to act, does the simplest thing available: it holds its last output. It freezes. That frozen difficulty is the stranded miner.

Once you see it this way, the fix is obvious too. The controller needs a way to act without the sensor — to keep easing difficulty down even while it's blind. A clock is enough: ease on a timer, not only on shares. In control terms that's feedforward, and it's the one thing that keeps a vardiff safe when the feedback disappears. The rest of this post is about which real vardiff designs have that clock — and which are clockless.


***Overview in brief (all terms and claims explained below)***

1. **Vardiff is a feedback loop**. Shares are the sensor, difficulty is the actuator, the miner's hashrate is what's being tracked.

2. **The sensor dies on the side that matters**. Shares thin as difficulty outruns the miner, so the feedback degrades when it slows and vanishes when it goes quiet; the information floor $1/\sqrt{r\tau}$ makes this a hard limit — worse as the rate falls, hopeless at zero, and unbeatable because the information simply isn't there.

3. **A loop with no sensor freezes**. With no fresh error, a controller can act on the dead signal, hold its last output, or ease on a clock. One that acts only on shares can do nothing but hold — and if it is quantized, it may be unable to ease even then.

4. **A frozen difficulty is a stranded miner**. Held too high, the miner produces almost no shares, which keeps the controller blind, which keeps it frozen — the blind direction and the harmful direction are the same.

5. **This is a correctness problem, not a tuning problem**. No choice of gain, deadband, or step size brings back a signal that isn't arriving.

6. **The fix is to act without the sensor**. A controller that also eases on a clock (feedforward — no shares required) stays in closed loop when it can see and safe when it can't; that fallback is what separates the safe designs from the ones that strand.

7. **You can test any pool yourself**. Put a shaping proxy in front of one miner, inject a decline the pool believes but the miner never feels, and watch whether the assigned difficulty eases down (safe) or holds high (strands) — the whole claim, checkable from outside on the pool you actually mine to.


## 1. Vardiff as a control loop

Every difficulty controller, whatever its internals, is doing the same job: keeping a miner's share rate near a target by adjusting how hard each share is to find. That job has a standard shape in control theory, and borrowing the vocabulary makes the rest of this post much shorter.

A controller can run in one of two modes. An open-loop controller acts on its command alone and never checks the result. A microwave runs for two minutes because you set the timer; it never measures whether the food is hot. It's simple and predictable, but it can't correct for anything it didn't anticipate — a colder-than-expected plate comes out underheated, and the microwave has no idea.

A closed-loop controller measures its output, compares it to the setpoint, and acts on the difference — the error. A thermostat reads the room temperature, compares it to the number you dialed in, and switches the heat on or off to drive the gap to zero. It doesn't need to know in advance how cold the room is, how well the house is insulated, or that you left a window open; it discovers all of that through the measurement and corrects for it. That self-correction is the whole point of feedback, and it's why essentially every difficulty controller is closed-loop: the pool can't take a miner's word for its hashrate, so it measures — through shares — and chases what it finds.

Here is vardiff drawn as a closed loop:
![control_loop|690x345](upload://j4IAMPwyHWJRBFIwJsm7Ptj2dgI.png)

*Vardiff as a feedback loop. The target share rate is the setpoint; the controller (C) turns the error into a difficulty; the miner (P) turns difficulty into shares, disturbed by its own changing hashrate; and the measured share rate feeds back through the sensor to close the loop. When shares stop, the sensor branch goes dark and the controller has nothing to steer on.*



The mapping is one-to-one with the standard picture:

- The **plant** is the miner — the thing whose behavior we're trying to govern. 
- Its hashrate is also the **disturbance**: it changes on its own, for reasons the pool doesn't control, and the loop's job is to track those changes.
- The **actuator** is the difficulty. It's the one thing the controller can move.
- The **sensor** is the share stream. Shares are how the controller observes the miner's rate; the share rate is the measurement.
- The **setpoint** is the target share rate. The **error** is the gap between target and measured rate, and the controller moves difficulty to shrink it.

Whatever a given vardiff does inside the controller — a simple multiplier, a Proportional Integral Derivative (PID) loop, an Exponential Weighted Moving Average (EWMA), a change detector — it is some rule for turning that error into a difficulty adjustment. The families differ in the rule. They do not differ in the loop: they all sense through shares, act through difficulty, and steer by feedback.

Which is why one property matters more than any detail of the rule. A closed-loop controller is only steering as long as its measurement means something. The thermostat is useless the instant its temperature sensor stops reporting; the cruise control can't hold speed if it can't read the speedometer. Feedback control is defined by acting on the measurement — so a controller whose sensor has gone dark isn't steering at all. It has fallen out of closed-loop control entirely, and whatever it does next is no longer a feedback decision.

For a vardiff, the sensor is the share stream. So the question that decides everything is: when does the share stream stop telling the controller what it needs to know — and what does the controller do when it does? That's the next section.


## 2. The feedback can vanish — and it vanishes on the wrong side

The sensor doesn't fail randomly or symmetrically. It fails worst in the direction the controller most needs to act.

Start with what a share actually is. A miner hashes away and, every so often, finds a hash below the difficulty target; that's a share. The rate at which shares arrive isn't arbitrary — it's the miner's hashrate divided by the difficulty. Given the same hashrate, turn the difficulty up and shares get rarer; turn it down and they come faster. This is the sensor: the controller learns the miner's hashrate by watching how fast shares arrive at the current difficulty. This same principle is how the network difficulty level adjusts itself, by observing the blocks across a given time period.

Now notice the asymmetry that creates. Each share is one noisy sample of the miner's rate, and to pin the rate down you need to accumulate several of them. How many you get in a given stretch of time is set by the share rate itself. So:

- When difficulty is set *too low* for the miner, shares flood in — the controller gets samples faster than it needs them, and the rate is easy to measure. The error is loud.
- When difficulty is set *too high* for the miner, shares thin out (they're hard to find), so the controller gets fewer samples. The error goes quiet.
- When the miner goes silent (slowed to nothing, or curtailed), shares stop entirely, and the controller gets no samples at all. The error is gone.

The controller's ability to see is proportional to the very thing it's trying to control, and it's weakest where difficulty is too high. That's the trap from the introduction: the direction the sensor goes blind in is the same direction that harms the miner. Over-difficulty starves the sensor of the shares it would need to detect and fix that over-difficulty.


**How blind?**

This isn't hand-waving about "fewer samples" — the blindness is quantifiable, and the number is worth carrying because it makes the limit exact. If shares arrive at rate $r$ and you watch for a time $\tau$, you collect about $N = r\tau$ of them. Now the counting law: a total of $N$ independent, memoryless events carries a statistical wobble of about $\sqrt{N}$, so the *relative* uncertainty (the wobble as a fraction of the count) is $\sqrt{N}/N = 1/\sqrt{N}$. Writing $N = r\tau$ back in, that's a relative error of 

$$\text{relative error} \;\approx\; \frac{1}{\sqrt{r\tau}}.$$

That much is just counting. The stronger claim goes further: you cannot extract a signal the shares don't carry, so as they thin, so does what any estimator can recover from them. This is what estimation theory calls the Cramér–Rao bound. For our purposes only the shape matters. Read it as a floor on how well you can possibly know the miner's rate:
- Raise the rate $r$ (or wait longer, raising $\tau$) and the floor drops — you can know the rate better.
- Let the rate $r$ fall and the floor rises — the best-possible estimate gets worse.
- Let the rate go to zero and the floor goes to infinity — there is nothing to estimate, because nothing is arriving.

The key point is that this is a floor, not a tuning parameter. A cleverer estimator, a longer window, a smarter filter — none of them get you below it, because the information simply isn't in the share stream when the shares aren't there. And since a declining miner is by definition moving toward lower $r$, it is moving up this floor: the further it slows, the blinder the controller necessarily gets, until at silence it is completely blind.

![information_floor_wall|690x396](upload://bEoMymjwWxVFkLu3nnneEGkJOTm.png)

*The information floor as a wall. The best achievable relative error in the miner's rate is $1/\sqrt{r\tau}$; it rises as the share rate $r$ falls and diverges at silence ($r \to 0$). A declining miner moves right-to-left, up the wall, toward the blind limit — and no estimator, however clever, can recover what the shares no longer carry.*


**Why this is fatal to feedback**

Put the two halves together. §1 established that a closed-loop controller is only steering while its measurement means something — take the sensor away and it isn't steering at all. This section (§2) establishes that the sensor doesn't just occasionally fail; it degrades smoothly and predictably as the miner slows, and fails completely when the miner goes quiet — and it does both on the over-difficult side, the side where the controller is already in trouble.

So the failure isn't a rare sensor glitch you could engineer around with redundancy. It's built into what a share is. Any vardiff that steers only by shares will, on a slowing or silent miner, find its feedback signal thinning toward nothing at the moment it most needs to ease difficulty down. The loop is losing its sensor, in the one regime where losing it does damage.

## 3. A loop with no sensor freezes

The controller does not stop running when its sensor goes dark. The code still executes on every tick, the difficulty is still under its command. What's gone is the *information* — the error it computes is now built from a share stream that has thinned or stopped altogether. So the controller has to do *something* with a number it can no longer trust, and there are only three things it can do.

**Option one: act on the dead signal anyway.** Keep computing an error from whatever the share stream last produced (a stale count, a lone straggling share, a zero) and keep moving difficulty on it. This is the worst option, because the controller is now steering on garbage: a single late share after a long gap reads as a rate, an integrator winds up against a zero-error that isn't real, and difficulty lurches on evidence that means nothing. A controller in this mode doesn't freeze, it *thrashes* in the wrong direction as often as the right one.

**Option two: hold the last output.** Recognize, implicitly or explicitly, that there's no fresh information, and leave difficulty where it was when the signal was last good. This is the stable, sensible-looking default (better than thrashing on noise), and it is where a controller with no other way to act ends up when the shares stop. But notice *where* it holds. The signal went dark because difficulty was too high for the miner; **option two freezes difficulty at exactly that too-high value.** It is stable the way a stuck throttle is stable: nothing lurches, and nothing gets fixed.

**Option three: act from a model.** Do something sensible *without* the sensor, using prior knowledge in place of a measurement. For a vardiff the obvious model is: "if I haven't heard from this miner in a while, the safe assumption is that it slowed, so ease difficulty down on a schedule until it starts talking again." No share is required to take that action — it runs on a clock. It is open-loop, the controller acting blind, but it acts in the *corrective* direction, and it is the only one of the three that gets a stranded miner unstuck. This is feedforward, and it's the subject of §6; for now the point is only that it exists, and that the other two don't help.

So where a design lands turns on one thing: whether it can move difficulty *without* a fresh share. A rule whose recompute is triggered by share *arrivals* has nothing to fire on when they stop. That is option two, and it comes by default rather than by design: "hold the last value" is simply what falls out of building a controller that only a share can wake. A rule whose recompute instead runs on a *timer* is not trapped this way. It fires during the silence, reads a realized share rate of zero, and — if it treats that zero as *ease down*, an idle-decrease — lowers difficulty on its own. That is option three, reached just as quietly. So the freeze is not the fate of *reading* shares; every controller here reads shares. It is the fate of *acting only when one arrives*. (Which of the two the [Stratum V2 reference implementation](https://github.com/stratum-mining) is — and why that matters — is §6.)

The introduction flagged one controller that can't dig out *even in principle*, and the design choice behind it is common and looks harmless. It doesn't set difficulty to any value it likes; it moves along a fixed ladder (powers of two) a rung at a time, rather than sliding smoothly. To move at all it has to build enough pressure from the error to cross to the next rung; a small error just leaves it on the rung it holds.

On its own that is fine: a miner that slows a lot builds enough pressure to cross a rung, and the controller steps down, then crosses another and steps down again — following the decline in stairsteps, coarser than a smooth glide but easing. What breaks it is the tuning. Push too weakly for a given error and the pressure in the ease direction never quite reaches the next rung down: the step doesn't come at all. The controller sits on its rung, computing that difficulty ought to fall and structurally unable to lower it.

Line that up against the ordinary freeze and the difference is sharp — and it is the difference that matters for the next section. The ordinary held controller (option two) is stuck only for *want of a signal*: hand it a single share and it recomputes and recovers. This one is stuck *with* a signal. A share arrives, it registers the error, it "agrees" difficulty is too high — and it still won't step down. The escape hatch that rescues the ordinary controller is bolted shut.

(This needs the ladder *and* the adverse tuning together — free to set difficulty continuously, or tuned to push harder, the same controller eases without trouble. It's the combination that strands.)

Either way — held for want of a signal, or unable to step down even with one — **difficulty is stuck high and the miner produces a trickle or nothing.** The one hope is that a share arrives to restart it.


## 4. A frozen difficulty is a stranded miner

That one hope — a share arriving — is exactly what the situation denies. Too high a difficulty is what keeps shares from coming, so the stuck difficulty isn't a bystander to the miner's quiet; it is the *cause* of it. The very event that could break the freeze is the event the freeze prevents, and a bad spot becomes a trap.

The pieces from §2 and §3 don't sit in a line, one problem after another. They close into a loop, each one feeding the next:

![stranded_loop|690x388](upload://vIl73XucTAInMq1zL1Yf5y3NvZk.png)
*The freeze is self-reinforcing. Difficulty stuck too high makes shares rare (§2); rare shares blind the controller (§2); a blind controller makes no correction (§3); and no correction leaves difficulty stuck too high. The one exit is severed, because it needs a measurement the loop has starved away.*

A healthy loop has a way out: the controller measures the error and *eases the difficulty*, which breaks it open at the point marked "no correction." That exit is the one arrow missing here. Remove that arrow, and the remaining four don't dead-end; they feed back on each other.

What makes the loop a trap rather than a delay is that it is *self-consistent*. Every move the controller makes is locally correct: given how little it can see, holding is the right call. And holding is what keeps it seeing so little. So the freeze isn't a state the controller endures while it waits for conditions to improve; it *creates* them.

Why does *this* sensor loss trap the controller, when feedback loops lose sensors all the time without disaster? The alignment from §2. Flip it and a freeze would be a non-event, landing on the flooding side where a surplus of shares corrects it at once. Here blindness and harm are the same event, so the loop is guaranteed to seize at the one moment easing is needed.

**How tight the trap is**

The grip depends on how far the miner has fallen — a gradient, not a uniform dead-lock. A miner still producing the occasional share isn't fully sealed in: a rare share can nudge difficulty down. But *nudge* is the word — the shares are rare (the trap) and each is a poor measurement (the floor), so it digs out slowly, in small uncertain steps, on a timescale that can outlast the event; for a curtailment window measured in hours, that means stranded for the part that counts. A fully silent miner is sealed in completely — no share, no nudge, no restart until something *outside* the loop intervenes. And the quantized controller from §3 is sealed in regardless: shares or none, it will not ease; for it there is no gradient at all.

**What "stranded" costs**

This is why stranded is the right word and not merely stuck. A stuck process has stopped. This miner has not stopped: it is hashing at full effort the whole time, burning power and paying for it, while the difficulty pins its visible output to a fraction of the work it's actually doing. Its credited contribution collapses toward zero — and who absorbs that loss depends on the reward scheme: under pay-per-share the miner simply earns less, under proportional schemes its missing shares quietly redistribute to everyone else in the window. Either way the cost falls on the miner, or on its peers, but not on the pool whose controller stranded it. From the pool's side it can be indistinguishable from a miner that quietly died. And it cannot fix any of this itself, because the only lever it has (producing shares) is the lever the difficulty has taken away. Rescue comes from outside the loop or not at all: the miner gives up and reconnects (resetting its difficulty to a default and starting the whole negotiation over), or an operator notices and steps in. Left alone, the loop does not release.

The natural response to all this is to reach for the controller's settings — surely a better-tuned loop, a shorter window, a gentler step would keep it from getting stuck. It's the obvious move. It's also the wrong one.


## 5. A correctness problem, not a tuning problem

The reflex is understandable, and it rests on a category error. Every knob a vardiff exposes (how hard it reacts, how long it averages, how big a step it takes, how wide a dead zone it ignores) changes how the controller responds to shares. (That "dead zone" is really two knobs with opposite fates. An *error* deadband — ignore errors within a band — is safe: a slowing miner drives the error out of it, so the controller still acts. An *output* deadband — refuse moves smaller than δ — pins on the wrong side exactly as the quantized rung does (§3): a correct, small ease is swallowed.) But the failure in §4 isn't a bad response to shares; it is the *absence* of shares. Turning the gain up reacts harder to nothing; shortening the window reacts faster to nothing (and holds fewer samples, which by §2's floor makes each estimate noisier, not clearer). Every knob transforms the share signal; none of them is a *source* of it, and transforming a signal you don't have leaves you where you started.

The floor from §2 makes this a limit rather than an observation. Relative error is bounded below by $1/\sqrt{r\tau}$ for every estimator, and tuning is nothing but the choice of estimator — so it slides you around *on* the floor, never beneath it. As the rate falls the floor climbs without bound, to infinity at silence. No estimator can recover information the shares don't carry; at silence there is none to recover; so no tuning, of any kind, lets the controller see through a silent miner. That isn't a claim about the tunings people have tried — it's a claim about the tunings that *can exist*.

Which is what makes this a correctness problem and not a tuning problem. A tuning problem has a setting that resolves it. This one doesn't, because the deficiency isn't in *how* the controller responds — it is in *what it is allowed to respond to*. The controller acts only on shares, and shares are exactly what a declining miner stops producing.


## 6. The fix: act without the sensor

Picture the tuning space as a single dial. Turn it aggressive and the controller lurches at every stray share; turn it down and it lags through the gaps and freezes. On a real decline both ends fail (one misreads the thinning, noisy signal, the other sits too long), and once the miner falls silent, every setting freezes alike, because there is nothing left to tune against. Safety is not a point on that dial. It is a move the dial doesn't have, and that move is a clock.

First, a picture of what "safe" even means here. Plot a controller as a return map: difficulty now on one axis, the difficulty it moves to next on the other — each expressed as a multiple of the difficulty that would exactly match the miner, so that "matched" sits at ×1 (call it correct), too-hard above ×1, too-easy below. The diagonal is "no change," and wherever the map touches it, difficulty-next equals difficulty-now: the controller leaves that difficulty alone. Those touch points are its *resting places* — its fixed points, where a controller settles. You read the map as an iterated walk — the *cobweb* below shows how — and a controller comes to rest wherever that walk lands on the diagonal.

![cobweb_intro|575x500](upload://1H6KTqdJcxRico2KMe2xLNLYwEC.png)

*How to read a return map, worked on one curve. Start at some difficulty on the horizontal axis — here ×4, a deep over-difficulty — and walk:* **up** *to the curve to read the difficulty it moves to next, then* **across** *to the dashed diagonal to feed that back as the new difficulty-now, and repeat. The walk stair-steps down (×4 → ×2 → ×1.33 → ×1.11 → …) and comes to rest at the one place the curve meets the diagonal: ×1, correct. That resting place — the fixed point — is where the controller settles. This is the reading the three maps below all use.*



Decline-safety has an exact shape in this picture, and it is the whole story: **one fixed point, sitting at correct.** One resting place, at ×1 and nowhere else, means that from any starting difficulty the walk has only one place it can end — so it converges to correct from anywhere, including up out of a deep over-difficulty. Every *additional* fixed point is a place difficulty can come to rest short of correct. And the map does not care whether that place is survivable: a resting place is just where next equals now, with nothing in it asking whether the miner can still land a share there before the pool times it out. A controller can sit, perfectly content, at a difficulty that is quietly killing the miner.


![family_return_maps|690x254](upload://ifL3FW6KyZeoMQPZHEEtckvJxiI.png)

*Each vardiff family as a schematic return map: difficulty now (horizontal) against the difficulty it moves to next (vertical), both measured against correct (×1). Where a curve meets the dashed diagonal, difficulty stops moving; decline-safety is having exactly one, at correct. Only the clock-driven ease (left) does, staying below the diagonal all the way out. Share-triggered (center) rejoins the diagonal out in the dark, where its correction fades. The quantized controller (right) rests on a rung across the whole over-difficult side — even where it can still see — and in the dark rejoins the diagonal as share-triggered does.*

The same picture sorts the designs, and the sort is now something you read off the over-difficult side: where does each map leave the diagonal, and where does it come back? A **clock-driven ease** crosses the diagonal once, at correct, and stays *below* it all the way out — it never comes back, so it keeps easing however dark it gets; its one resting place is correct. A **share-triggered** controller dips below near correct, correcting sharply while the signal is strong, but as over-difficulty grows the shares thin and the correction built from them fades, so the map bends back and *rejoins* the diagonal — its resting places sit out in the dark, made by the vanishing signal, and the further out, the less remains to move it off. A **quantized controller with an adverse gain** is worse still: it rests on whatever rung it holds and can't step off, so its resting places cover the *whole* over-difficult side — even where it can still see. The three don't differ in how well they're tuned. They differ in whether the map ever rejoins the diagonal, and where. Only the clock never does.

The clock-driven ease is the move that produces that one shape. Alongside the ordinary share-triggered loop, it gives the controller a second path: if it hasn't heard enough from a miner in a while, ease the difficulty down on a timer — a step at a time, no share required to justify it. When shares are arriving the feedback loop runs and tracks the miner as well as it ever did; when shares thin or stop, the timer keeps easing. On the return map this *is* the deletion of resting places: across the whole over-difficult side the clock is always pulling difficulty down, so the map never touches the diagonal there, and every fixed point except the one at correct is gone. The controller is closed-loop when it can see and open-loop when it can't — open-loop *in the corrective direction*. This is feedforward, the "act from a model" option from §3, and the model is nothing more than "a miner I can't hear is probably one I'm holding too high."

None of this is hypothetical. The Stratum V2 reference implementation (SRI) already works this way: the pool ticks each channel's vardiff on a fixed interval rather than only when a share lands, and the reference vardiff, when that tick falls during a silence, reads a realized share rate of zero and cuts its difficulty by a factor. The idle path — the clock this section argues for — is the deployed default there, not a proposal. It also settles the question §3 left open: the designs that freeze are the ones whose recompute only a share can trigger, and the reference escapes because a timer triggers it, and a timer still fires when the shares don't.

![decline_tracking_3|690x332](upload://dX2Tth2EmowJFD1IibbYIzLIwu3.png)

*An abrupt curtailment — the miner drops to zero over minutes 60–64, and the correct difficulty (dashed) falls with it. Three vardiff designs respond — one deployed, two minimal constructed instances of a style. The SV2 reference (SRI's classic vardiff, blue), a timer-triggered design running in a real pool, holds briefly, then, as its timer keeps firing on empty and divides difficulty down tick by tick, rides all the way down to meet the miner. A share-triggered controller (red) — one whose recompute fires only on a share arrival — has nothing to fire on once shares stop, so it freezes near the top (≈0.79), barely having moved. A quantized controller built with an adverse gain (dark red) is pinned on the top rung — it computes that difficulty should fall but its correction can't clear a step, so it never actuates the ease. Two freezes — one because nothing fires without a share, one because the actuator can't step — and the timer eases past both. Curves trace a simulation of the three designs on this scenario.*

Here is why that is safe by construction rather than by careful tuning. The clock-driven ease only ever *lowers* difficulty, so ask what it costs when it fires at the wrong time — the silence was a brief fluke and the miner hadn't really slowed. The controller eases anyway, difficulty is now a little too low, and when shares resume they come *faster* than target. But a surplus of shares is the loud, visible side from §4: the ordinary loop sees it immediately and tightens straight back. The over-ease undoes itself. (Decline-safety says nothing about the ramp-up overshoot when shares return — that's a separate, loud-side *tuning* concern.) Set that against the freeze's mistake — holding difficulty too high, on the blind, silent side, where nothing corrects it and the trap closes. Both fallbacks can fire wrongly; what differs is *which way* wrong lands. The freeze fails toward the trap; the clock-driven ease fails toward the one region where errors correct themselves. The alignment that made the freeze a catastrophe in §4 now runs in the controller's favor.

The worst case is subtler than it looks. Its pin seems a wholly different failure from share-triggered's — an actuation problem, not a sensing one. §3 blamed the adverse gain: the correction never clears the half-rung (×√2) it must cross to step down, so difficulty rounds back onto the rung it holds. But out in the dark a second cause takes over. The estimate the correction is built from has died — this controller, running on the same share-starved rate as share-triggered — and rounding hides it, snapping the vanishing correction back onto the rung so the controller merely *looks* stuck where share-triggered visibly rejoins the diagonal. Only the first cause is distinctive; it pins the controller even where it can still see, which is why the quantized resting places cover the whole over-difficult side and not just the dark. Underneath, in the dark, the two share one disease.

That shared disease is the lesson. Both families are attempts at the vanishing signal of §2 and both attempt it in the same place. Each does two things — estimates the rate from shares, then decides a difficulty from that estimate — and both put the fix in the *decision*: each builds the difficulty it commands out of the rate estimate, so when the estimate dies on the over-difficult side, the command dies with it. Seen that way, the fix has two roads. One is to patch the *decision*: give it a move that needs no estimate — one fair reading of the clock-driven ease above. The other is to fix the *estimate itself* so it stops dying, and leave the decision untouched — a healthy estimate on both sides hands the return map its single fixed point at correct, no special-casing required. The decision only looked hard because we were doing estimation inside it.

And the two roads turn out to be one. Fixing the estimate sounds like the harder road: §2 put a floor under the rate error that diverges as the shares vanish. But that floor is on the *rate* — on knowing *how far* over-difficulty has run. It says nothing about the *sign*, about which way; and the sign is what silence hands you. The naive estimator discards that bit, reading only shares that arrive, of which a silence sends none. Keep the sign instead, and nothing exotic is needed: the controller acts on it exactly as it would act on any estimate, and eases down in the dark on its own. The clock is the *minimal* way to read that sign: easing on a timer *is* reading a lengthening silence as *still too high, ease more*. So the feedforward reading is true from inside the share stream — but step outside it and a deeper one shows through. The loop that looked sensorless had only lost the sensor it was watching. The lengthening silence is itself a sensor — one that says *too high* the longer it lasts, and that no information floor can switch off. What looked like acting on a model was acting on a measurement all along.

Everything here reduces to that one picture. A loop that acts only on shares is superb while they flow and helpless the moment they stop — and no *tuning* reaches the second half, because what's missing there is information. Decline-safety is not a tuning achievement but an architectural one: whether, once it can no longer measure the rate, the controller still reads the sign. The miner from the opening — throttled, cooling, curtailed — doesn't need a controller that can watch it slow. It needs one that eases off even when it can't. A clock is enough; the hard part was seeing that it's necessary — and that the whole of it was a single bit of information, dropped in where it belonged.

One caveat before we leave the fix. The safety a clock buys is real but not *prompt*. SRI's classic estimator averages cumulatively since its last change, so a converged channel's effective memory grows with its age — and the longer a miner has been healthy, the more entrenched that estimate is against the new, lower one. The ease still comes, on the timer, without a share; it just comes *slowly* on a well-established channel, in proportion to how long it was healthy. An ease that forgets — one that bounds that memory, or quickens the longer a miner stays quiet — is the natural next step.


## 7. Test it on your own pool

You don't have to take anyone's word for whether a given pool strands a slowing miner. You can measure it from the outside, with one miner and a small proxy (see below) in the path that lets you modulate the miner's *perceived* share rate on demand, and watch how the pool's vardiff responds to a decline you dial in.

You don't strictly need the proxy: pulling hashboards from a running miner drops its hashrate for real but it's manual, unrepeatable, and power-cycles hardware by hand. The shaping proxy trades that realism for a precise, programmable, repeatable decline you drive from a keyboard, which is why the method here uses it.

The idea is to inject a decline the pool believes but the miner never feels. The proxy relays the miner's shares upstream, but on command it forwards only a controlled fraction of them — so from the pool's side the miner's share rate falls exactly as if the miner had slowed, while the miner keeps hashing flat out. The trick is that the proxy acknowledges *every* share back to the miner instantly, whether or not it forwards it upstream. The miner therefore never sees a rejection, never stalls, and never runs its own difficulty logic in response — which means the only controller reacting to the injected decline is the pool's vardiff. That isolation is the whole point: you've arranged for exactly one controller to be under test.

The measurement is then direct. Let it run flat until the difficulty the pool assigns settles to a stable baseline. Trigger a step-down — forward, say, half the shares — and watch the assigned difficulty. A decline-safe pool eases difficulty down to track the apparent decline. A decline-unsafe one holds it high: the stranded-miner signature from §4, observed from the outside. Then step the rate back up; a safe controller re-tightens, and it's worth watching the recovery too, since a controller can strand on the way down *or* overshoot on the way back up.

Three cautions — one bounds what the test can tell you, one can make a safe pool look unsafe, and one, which nearly fooled us, can make an invalid run look safe:

- **A shaped decline is not a real hashrate decline, and for most controllers that doesn't matter.** The proxy drops forwarded shares; the miner is still hashing. To a controller that keys on share *rate* — which is what difficulty control is about — a shaped decline and a real one are indistinguishable. A controller that inspected finer statistics (inter-arrival timing, variance) could in principle tell them apart, so keep that in mind if you're testing something exotic. For ordinary rate-based vardiff it doesn't matter.

- **Turn off your own floor.** If the proxy enforces a minimum difficulty of its own, disable it for the test. Otherwise it can absorb the pool's ease-down, and you'll see difficulty "not dropping" and blame the pool — when it was your own floor that caught the descent, not the pool holding. That's a self-inflicted false positive for decline-*un*safety.

- **Confirm the miner is actually submitting — watch the counter, not the connection.** This is the one that nearly caught us out. An early run produced a clean difficulty descent that looked exactly like a decline-safe response — until we noticed the forwarded-share counter was frozen. The miner had gone quiet while staying TCP-connected, and what we were watching was the pool easing difficulty as it chased a *dead* miner toward its floor, not any response to our injected decline. We withdrew the result. Connection-alive is not shares-flowing; only a share counter climbing steadily over minutes tells you there's a live decline to measure. Gate every run on that.

None of this needs special hardware or cooperation from the pool. It turns the central claim of this post into something you can check for yourself, on the pool you mine to.

We built such a proxy for exactly this — an SV2 pass-through in front of one miner, with an API to modulate that miner's perceived share rate (step down, hold, ramp back up) so you can drive a clean decline against a pool and read the assigned difficulty as it responds. It's the `shape-proxy` crate under `test-tools/`, at [`shape-proxy-v0.1.0`](https://github.com/marafoundation/sv2-apps/tree/shape-proxy-v0.1.0/test-tools/shape-proxy).


## 8. Safe, but not prompt

So the headline needs both halves. The clockless designs — the ones whose recompute only a share can wake — strand a slowing miner outright; the fix is a clock, and the SV2 reference already has it, which is why it never permanently strands anyone. That is the good news, and it is real. But it is not the whole story, and read alone it invites the wrong conclusion — that the problem is solved and there is nothing left to do.

There is. SV2's safety is *asymptotic, not prompt* — and, for the reason §6 closed on, slowest where you'd least expect: not on the deepest declines, but on the miners that were healthiest longest. Recovery time scales with how long the miner was healthy before it slowed, and for a long-established channel it can exceed the curtailment itself. SV2 does the right thing; on an aged channel it does it slowly. Closing that gap — an ease that forgets — is the subject of follow-up work.


## Related work

The controllers this post treats as archetypes have deployed instances. ckpool's vardiff is a mature, widely-run share-triggered design whose estimator blends several averaging windows, switched by rate. The **Stratum V2 reference implementation (SRI)** is the timer-triggered controller analyzed throughout — the pool ticks each channel's vardiff on a fixed interval, so its estimator sees an empty window during silence and eases without a share, if not always promptly (per §8). The actuator-pinned case of §6 is the class of designs that **quantize difficulty to powers of two**: when the ease step can't clear the half-rung to the next level down, the correction rounds back and difficulty never moves. These are the mechanisms the three families embody — representatives, not a survey.

The control-theory framing — plant, sensor, actuator, feedforward — is standard vocabulary; the closed-loop/open-loop distinction §1 leans on is textbook material. §2's information-floor argument is not new either: it is the **Cramér–Rao bound** (no unbiased estimator beats the inverse Fisher information — here, that nothing recovers the rate better than the shares' own counting statistics allow) applied to a **Poisson** share stream, whose rate over a window of $N$ events carries relative uncertainty $\sim 1/\sqrt{N}$. The list below points to where those standard results are stated formally, for a reader who wants them — it is not a record of sources the argument drew on.


## Acknowledgments

Thanks to the reviewers who read early drafts and pushed on the framing, the terminology, and the claims that needed either sourcing or softening — among them **vnprc**, **average_gary**, and **based64god**. The provenance of the work — including the AI collaboration — is described in the note at the top.

-------------------------

paratoxicdev | 2026-07-15 23:09:29 UTC | #2

Instead of dropping hashrate to 0 why not drop it 99%? I think that's a more realisitic scenario for stranded hashrate. Because if a miner really drops to 0 there's no point in adjusting the difficulty down, it won't come back either way. Thinking that thought further the real test would be how fast a 99% reduced hashrate miner comes back to the target share rate for those different algos

-------------------------

