---
layout: post
title: How Much Should You Let an AI Agent Do? Exactly as Much as You Can Verify
tags:
  - ai
  - agents
  - security
---

We are about to let software spend our money, send messages in our name, and change records on our behalf, mostly while we aren't watching. A lot of the attention goes to making an agent that *can* do those things. The harder part is deciding whether it *should*, and making sure it can't overstep. That is the part I want to talk about.

Here's an example of the problem in one sentence. You tell an assistant "clean up my inbox," and it deletes nine thousand emails. It wasn't hacked. It wasn't malicious. It just did more than you meant. That gap, between what you wanted and what an eager, well-intentioned agent decides to do, is not a model problem. You don't fix it by making the model smarter. You fix it in the plumbing underneath, the part that grants or denies the ability to act. That plumbing is still being worked out.

## What's actually broken?

The obvious approach is to give the agent your credentials and let it act as you. Two things are wrong with that.

The first is that the agent inherits your entire reach. Hand it your identity and it can do anything you can do, with nothing scoping how much of that reach any single action gets to use. Security people call that ambient authority: the power is just there, available by default, the moment the agent starts working. Give a tool all of your authority and a loose reading of a vague request, and doing far more than you meant isn't an edge case. It's the natural result. The fix for ambient authority is least privilege: each action should carry only the authority it needs for that one thing, and nothing more. Most of what follows is that principle taken to its limit.

The second problem is newer. If a usable access token is sitting in the model's context, the model can reuse it. For the thing you wanted, or for something you didn't. A working credential inside a reasoning loop is a loaded gun that the reasoner can pick up at any time. Single-use tokens help, but they do nothing about the model misusing the one use it has, and they just move the question to "what hands out the next token?" The fix has to be more structural.

## Someone else holds the keys

Here is the idea the rest of this hangs on. The agent should hold no credential at all. Not a broad one, not a narrow one, not even a single-use one. It *proposes* what it wants to do. Something else decides, and if the answer is yes, that same something carries the action out. The agent never touches a key.

That sounds like a small distinction. It isn't. If the agent never holds the thing that grants the ability to act, then it no longer matters what the agent believes. Tell it "pending approval" and it cannot act on a misread, because there is nothing in its hands to act with. Tell it "approved" by mistake and it still cannot do anything, for the same reason. When the system is confused, it does nothing. That is exactly the behavior you want from anything that guards an action: the failure direction is *off*.

So there is something in the middle that holds the keys, decides whether to act on its own or ask a person, and keeps the record. Call it the gate. The agent talks to the gate; the gate does everything else. None of this depends on the agent being trustworthy. It can be eager, confused, or actively hostile. It can only ever propose.

How you actually build a gate like that, and what it costs you, is a bigger topic than one post, and the honest truth is that nothing I know of implements the whole picture today. It is one of the open problems of the agent era. For now, take the gate as a given, because the interesting question sits on the other side of it: when should the gate say yes on its own?

## Approval that can't quietly grow

Watch what this does to the classic failure. An agent gets a refund of fifty dollars approved, then decides five hundred would make the customer happier. The five hundred is a different request. The gate never approved it, and the agent holds nothing it can stretch, so the bigger refund simply doesn't happen. It goes back to the gate as what it is: a new thing to decide. You didn't write a rule against scope creep. There was never a blank check to inflate.

## Reversibility is a clock, not a flag

If you want an agent to act on its own without a person on every step, you need to know which actions are safe to just let run. The first thing that tells you is whether the action can be undone.

Reversibility isn't a yes or no. It's a clock. "Let it run, notify the person, give them a window to cancel" is a fine safeguard, but only if that window is shorter than the time it takes the action to become permanent. A five minute undo on something that's irreversible in sixty seconds is theater. And reversible actions pile up into irreversible ones. Deleting one email is easy to undo. Deleting nine thousand is not. So reversibility depends on volume, which means it's tangled up with rate limits and budgets.

That gives you a simple rule. Cheap and reversible, let it run and keep an undo handy. Irreversible or high impact, you don't get to just allow it on trust. That's where you either verify it or put a person on it. Autonomous really means autonomous inside the limits you can either undo cheaply or check.

## What is a person actually approving?

Not an action. An action *for a reason*. "Delete this email because I specifically pointed at it" is a different request than "delete this email as part of a cleanup heuristic," even though the action is identical. So make the reason a structured choice the agent selects from, not a sentence it writes. Two free-text reasons can mean the same thing without being verifiably the same, and the gate can't tell "I'm doing this for the reason you approved" from "I'm doing this for a reason that sounds like it" when the reason is prose.

Once the reason is structured, the same action can carry different friction depending on why. Delete because the user named it, fine, do it. Delete as a cleanup heuristic, always ask. Overreach almost always rides in on a reason, the proactive one, the bulk one, the clever one. Gating on the reason aims right at the problem. It also lets a person pre-approve a whole class once: "always allow refunds for a duplicate charge under fifty dollars." That's how you buy autonomy back without giving up safety.

## The whole thing comes down to one question

Here's the catch. A reason the agent *picks* is still the agent's word for it. The honest, eager agent's signature failure is sincerely picking the wrong reason, genuinely believing you asked for something you didn't. So for anything you want to approve automatically, the reason has to be backed by evidence, and that evidence has to be *checked*, not just present.

What makes evidence checkable? It has to point at something the agent could not have made up. A record in a system it can't write to, looked up by the gate, not a blob the agent hands over. (If the agent can manufacture its own proof, you've just relocated the lie.) And then the real question is whether checking it requires *judgment* or just *arithmetic*.

If a machine can verify it, you can automate it. "Duplicate charge" means two entries in the ledger, same amount, same merchant, minutes apart, neither already refunded. That's arithmetic. The gate can confirm it and make the refund itself with no human involved.

If verifying it requires interpretation, you can't. "Ugh, my inbox is a mess" does not mechanically authorize deleting nine thousand emails. Deciding whether it does is exactly the kind of judgment that produced the overreach in the first place, so you can't hand that judgment to the agent and call it verified. That one stays with a person, permanently.

Which gives you the single sentence this all reduces to. **The amount you can automate safely is exactly the amount whose justification you can check by machine against something the agent can't fake.** "How autonomous can this be?" stops being a gut call and becomes a measurable property of the work itself.

That word, safely, is the catch. You can let the agent act without verifying anything if you want to. When an action is cheap and easy to undo, that is often the right call. You skip the checking and take the small risk that comes with it. That is a choice, not a loophole, as long as you are clear that you are accepting risk rather than confirming anything. Verification is for the actions you can't afford to get wrong. For everything below that, a little risk is cheaper than a lot of ceremony.

![A decision flow with three outcomes: the agent declares an action, a reason, and grounding; if it is cheap and reversible enough you can allow it and accept the risk; otherwise, if a machine can verify the grounding the gate acts on it, and if it can't the request goes to a person]({{ site.baseurl }}/images/2026-07-09/autonomy-boundary.png)

## The honest ceiling

All of this adds up to a ceiling worth being honest about. A safe autonomous agent today is a bounded one. When it reaches for something genuinely new, against a system nobody deliberately connected it to, the system should fail toward *can't*, not toward *oops*. New capability is always a deliberate act of building, never something the agent grants itself in the middle of a task. That is the whole point, because "an agent helping itself to power nobody gave it" is precisely the thing we are trying to prevent.

So be honest about the trade. You do not get unlimited reach and safety in the same system, and anyone who promises both is selling you the bug as a feature. What you get instead is real and worth having: inside the surfaces you've connected, the agent runs on its own; outside them, it stops and asks, or it just stops.

## We're not the only ones here

None of this is happening in a vacuum. A lot of serious people are working on the same problem, and when I looked at what they are building, it lined up closely with where I had landed. [Google's Agent Payments Protocol](https://cloud.google.com/blog/products/ai-machine-learning/announcing-agents-to-payments-ap2-protocol) is built on signed "mandates" that authorize a specific purchase and that the agent can't exceed without asking again. The standards bodies are drafting [on-behalf-of flows](https://www.ietf.org/archive/id/draft-oauth-ai-agents-on-behalf-of-user-01.html) so an agent acts with a narrow, traceable slice of your authority instead of impersonating you. Researchers and vendors are converging on ["delegation beats impersonation"](https://next.redhat.com/2026/05/21/zero-trust-for-ai-agents-why-delegation-beats-impersonation/) and on permissions that can only narrow as they pass down a chain. The identity giants are racing to give every agent [its own identity](https://www.okta.com/blog/ai/okta-ai-agents-early-access-announcement/), separate from yours.

Still, one thing gets less attention than it should: what actually makes a justification *automatically* verifiable. The field talks endlessly about "intent" and "justification" while staying vague about the part that has to hold for any of it to run without a person. The line between judgment and arithmetic isn't a detail. It's the dial that decides how much an agent can ever safely do on its own.

This is still early. People who study it for a living [keep calling it an unsolved problem](https://www.resilientcyber.io/p/identity-is-the-agentic-ai-problem) of the agent era, and the failure they worry about most is people drowning in approval prompts and rubber-stamping them. The way out isn't asking more often. It's being honest about what a machine can check and what it can't, automating the first and saving our attention for the second.

So the next time someone asks how much we should let an agent do on its own, don't answer with a feeling. Answer with a question. How much of it can we actually verify? That number is your answer, and not a word more.
