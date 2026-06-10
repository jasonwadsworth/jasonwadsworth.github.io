---
layout: post
title: Your One-Way Door Swings Both Ways
tags:
  - leadership
  - decision-making
---

In my years of building software, one thing I see a lot is decision paralysis. A team needs to pick a database, or a vendor, or an architecture, and the decision just sits there. Documents get written, meetings get scheduled, opinions get socialized, and weeks go by. When I dig into why a decision is stuck, the answer is almost always the same: somewhere along the way, someone decided it was permanent.

Jeff Bezos gave us one of the most useful decision-making frameworks in business for exactly this problem: one-way doors versus two-way doors. One-way doors are irreversible. Walk through, and you can't come back. Two-way doors are reversible. If you don't like what you find, you step back through and try something else. The prescription is simple. Two-way doors deserve fast decisions by individuals or small teams. One-way doors deserve deliberation.

It's a great framework. And almost everywhere I've seen it used, it's used wrong.

Not because people apply the wrong process to each door type, but because they misclassify the doors in the first place. We label decisions as one-way doors far more often than reality supports, then grant ourselves permission to deliberate, study, and stall. The framework that was supposed to accelerate decisions becomes the excuse for slowing them down.

Here's the uncomfortable truth: most of your one-way doors swing both ways. You just haven't pushed on them.

## Why do we get the classification wrong?

### Loss aversion does the labeling

We feel losses roughly [twice as intensely as equivalent gains](https://link.springer.com/article/10.1007/BF00122574), something Kahneman and Tversky showed in their work on [prospect theory](https://en.wikipedia.org/wiki/Prospect_theory). So when we evaluate a decision, our brain doesn't ask "is this reversible?" It asks "how bad would it feel to be wrong?" A decision that would be merely annoying to unwind gets emotionally tagged as catastrophic, and catastrophic feels permanent. The door gets labeled one-way not because of how it works, but because of our fear.

### Cost is not impossibility

Reversing a decision usually costs something: money, time, credibility, an awkward conversation. But cost is not the same as a welded-shut door. Migrating off a vendor is expensive, but it's not impossible. Unwinding an org structure is painful, but it's not permanent. When we say "we can't go back" we usually mean "going back would be uncomfortable", and those are very different claims.

### Ego makes reversal feel like failure

The real lock on most doors isn't technical or financial. It's identity. Reversing a decision means admitting the original call was wrong, and for many leaders that admission feels more costly than living with a bad decision. So we retroactively declare the door one-way to protect ourselves from ever having to walk back through it. This isn't just my read on it. The [escalation-of-commitment research](https://www.sciencedirect.com/science/article/abs/pii/0030507376900058) found that people invest more in a failing course of action precisely when they were personally responsible for choosing it. The irreversibility is self-imposed.

### We evaluate at the wrong altitude

"Should we adopt this architecture?" sounds like a one-way door. "Should we build one service this way and see what we learn?" is obviously a two-way door. Most big scary decisions decompose into a sequence of small reversible ones, but we insist on evaluating them at the monolithic level where everything looks permanent. The door isn't one-way. We're just refusing to open it a crack.

### Sunk cost locks the door over time

A decision that was trivially reversible on day one feels irreversible on day 400. The door didn't change. We've just invested so much that walking back feels like setting money on fire. That's the [sunk cost fallacy](https://www.sciencedirect.com/science/article/abs/pii/0749597885900494) at work. The investment is gone either way, so it shouldn't be part of the decision.

## How many true one-way doors are there?

Genuinely irreversible decisions do exist: selling your company, shipping a security breach, burning a key relationship. These deserve real deliberation.

But run the test honestly. For any decision you're agonizing over, ask "if this turns out wrong, what specifically prevents us from changing course in six months?" Not what it would cost. What prevents it. Most of the time the answer is nothing. Bezos himself flagged this problem in his [2015 letter to Amazon shareholders](https://s2.q4cdn.com/299287126/files/doc_financials/annual/2015-Letter-to-Shareholders.PDF). As organizations grow they tend to run heavyweight one-way-door processes on nearly everything, and the result is "slowness, unthoughtful risk aversion, failure to experiment sufficiently, and consequently diminished invention." The danger was never treating two-way doors carelessly. It's treating two-way doors like one-way doors and grinding decision velocity to a halt.

## What does this look like in practice?

### Default to two-way

Make reversibility the starting assumption. The burden of proof should be on the claim that a decision can't be undone, not on the claim that it can. If nobody can articulate the specific mechanism of irreversibility, it's a two-way door. Make the call and move on.

### Price the reversal, quickly

When I'm facing one of these decisions, I try to take five minutes to gut-check what unwinding would actually take. "If this vendor fails us, migration is one engineer for six weeks." That's it. No analysis doc, no committee. Suddenly the decision that felt existential is just a line item, and you can make it in an afternoon instead of a quarter.

### Decompose the monolith

When a decision feels like a one-way door, that's often a signal that it's too big, not that it's too risky. Break it into the smallest step that generates real information. Almost every "bet the company" decision contains a "spend two weeks and learn something" decision hiding inside it.

### Make reversal cheap on purpose

This is where engineering leaders have real leverage. Feature flags, incremental rollouts, abstraction layers at vendor boundaries. These aren't just good hygiene. Every investment in reversibility converts future one-way doors into two-way doors, which converts future deliberation into future speed. Notice that all of these are things you build, not things you write. The best record of a reversible decision is a system that makes the reversal easy.

### Reward the walk-back

If reversing a decision is career-damaging in your organization, you've institutionalized the misclassification. People will defend bad decisions forever rather than reverse them, and they'll demand one-way-door deliberation for everything to avoid ever being the person who has to reverse. Celebrate fast reversals as what they are: the system working.

## Why does this matter?

Decision speed compounds. And if you're worried that speed sacrifices quality, the evidence points the other way. [McKinsey's global survey on decision making](https://www.mckinsey.com/capabilities/people-and-organizational-performance/our-insights/decision-making-in-the-age-of-urgency) found that faster decisions tend to be higher quality, and the organizations that do both well are twice as likely to report superior returns from their decisions. An organization that makes reversible decisions in days instead of quarters doesn't just move faster on each decision. It runs more experiments, learns faster, and builds an information advantage that slower competitors can't close. Meanwhile, the organization that treats everything as a one-way door pays twice: once in the delay, and again in all the experiments it never ran because deciding felt too heavy.

So the next time you, or your team, are stuck on a "permanent" decision, ask what actually prevents you from changing course later. Most of the time that deliberation isn't prudence. It's fear.

Push on the door. It probably swings both ways.
