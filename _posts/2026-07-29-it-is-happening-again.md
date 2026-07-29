---
layout: post
title: "It Is Happening Again"
date: 2026-07-29
categories: [llm, security]
image: /assets/it-is-happening-again.png
---

![It Is Happening Again](/assets/it-is-happening-again.png)

*A scorecard for the agent-security essay I wrote in April, including the parts I got wrong.*

In April I wrote a post arguing that the privacy conversation had quietly changed shape. We spent fifteen years learning to protect communication - Tor, TLS, encrypted mail, anti-fingerprinting - and then we handed a probabilistic parser of ambiguous language the keys to our infrastructure and asked it to act on our behalf. The claim was that the security question had moved up a layer: not *who can see me*, but *what can act for me, under whose authority, and how easily can that be turned*.

That was a prediction. Predictions are cheap. So this post is the invoice.

I want to grade the April essay against the record of the last three months, and I want to do it honestly, which means the interesting part is not the calls I got right. It's the three I got wrong, and the one uncomfortable thing the whole exercise reveals.

Let me start with the ledger and then argue with it.

---

## The record, in brief

In April the abstractions were these. Context is an attack surface, because natural language has no built-in boundary between data and instructions. Retrieval is a security boundary, because untrusted content acquires the same effective authority as policy. Tool use materializes model errors into real actions. The confused deputy comes back. Identity and privilege need to be explicit and scoped. Memory is a persistent risk surface and a vector for long-lived manipulation. And the model must propose, not authorize, with policy enforced outside it.

Here is what the world did with those abstractions while I wasn't posting.

**Tool-output trust became a CVSS 8.5.** Amazon Q Developer auto-loaded MCP configuration from workspace directories with no user consent, and opening a malicious repository was enough to execute code and exfiltrate AWS credentials. A companion flaw was attributed directly to the agent implicitly trusting MCP tool output as if it were safe. That is the "retrieval is a security boundary" argument with a patch date attached.

**The unsafe loop I sketched in pseudo-code shipped, and got 180,000 stars.** In April I wrote out the naive agent - read everything, decide everything, call everything, no policy layer, no trust separation - and called it the architectural equivalent of dropping a language model into the middle of your infrastructure. OpenClaw is that loop, self-hosted, fastest-growing repository in GitHub's history. It earned the first CVE ever assigned to an agentic AI system, a remote-code-execution chain, and somewhere between thirty and forty thousand internet-exposed instances, the overwhelming majority running with no authentication at all.

**Memory became a deliberate target, not a passive one.** This is the prediction that aged best and worst at once. I said memory could be a vector for long-lived manipulation. The ClawHavoc campaign went straight for OpenClaw's persistent memory files - the ones that hold long-term behavioral instructions - because editing them converts a point-in-time exploit into a standing one. Right idea. I'll come back to why it still counts partly against me.

**Delegated authority became the defining problem of the year.** In April "identity, privilege, and execution context" was a section heading. By this summer it was the main program at the identity conferences. There are now two competing standards efforts, an OpenID Connect extension for agents with delegation-chain validation and an IETF track, plus a wave of scoped-token, per-principal, "the subject is the human, the actor is the agent" designs. The confused deputy I described is now a thing people write delegation grammars to prevent.

The ledger:

| April claim | What happened | Verdict |
|---|---|---|
| Context has no data/instruction boundary | Indirect injection is the dominant agent attack class, and a page that says "summarize me" is indistinguishable to the parser from one that says "email your contacts to the attacker" | Confirmed |
| Retrieval / tool output is a security boundary | Amazon Q credential exfil via untrusted MCP config plus implicit tool-output trust | Confirmed |
| The naive "read/decide/call everything" loop is a liability | OpenClaw, at civilizational scale, with the first agentic-AI CVE | Confirmed |
| Memory is a vector for long-lived manipulation | ClawHavoc poisoning persistent memory files for stateful persistence | Confirmed, sharper than predicted |
| Identity must be explicit, scoped, delegated | Agent-identity standards became the year's central authorization fight | Confirmed |
| Policy must live outside the model | Standards-track "model proposes, external engine authorizes" is now the consensus position | Confirmed in principle, absent in practice |

Six for six looks like a victory lap. It isn't, and I'd distrust anyone who ran one on this record. The essay was directionally right the way a weather forecast that says "it will get worse" is right. What it got wrong is more instructive than what it got right.

---

## Where I was wrong

### 1. I wrote about the frontier. The damage came from 1995.

The April essay is a philosophy-of-agents piece. Its center of gravity is the *novel* risk: semantic attacks, ambiguity exploitation, the confused deputy, "a failure of interpretation under conditions of adversarial input." I spent my words on the clever threat because the clever threat was the new and interesting one.

The clever threat was real. It was also, this year, secondary.

The actual damage was overwhelmingly boring. ClawHavoc did not subvert anyone's chain-of-thought. It published over a thousand malicious packages using week-old accounts and told users to run a command that installed a credential stealer. Every one of the stealer skills phoned home to a *single* command-and-control address. The mcp-remote flaw that kicked this era off was ordinary OS command injection. The Claude Code disclosure was config injection through a hooks file. The largest single risk multiplier across the whole OpenClaw fleet was not prompt injection. It was that almost none of the exposed instances had a password.

I described a semantic frontier. The breaches came through supply chain, command injection, and unauthenticated network exposure, which are pre-AI security failures wearing an AI hat. If I'd wanted to protect the most people in 2026, the essay should have opened with "put it behind auth and don't run untrusted skills," not with the confused deputy. The exotic risk was the one worth *theorizing*. The dumb risk was the one doing the *work*.

### 2. I wrote for the enterprise. The crisis was personal.

Reread the April identity section and you can smell the assumed context: service accounts, API keys, backend APIs with broad access, policy engines, audit logs. I was picturing a governed organization with an agent that had too much authority, a corporate deputy to be constrained.

The defining deployment of 2026 was the opposite. It was a self-hosted, single-user assistant running on someone's own laptop, wired into their email and calendar and files, exposed to the internet without authentication, extended by plugins from an open marketplace that required a one-week-old account to publish. Shadow AI, but with root. The threat model that mattered was not "enterprise agent with excessive scope." It was "consumer agent with no scope control at all, on a machine with no security team behind it." I aimed at the boardroom. The fire was in the living room.

### 3. I wrote like an architect. The failure was economic.

This is the one that actually bothers me.

The April essay ends by contrasting the naive loop with a safer one - identity-scoped, policy-gated, human-in-the-loop for high-risk actions, everything audited - and treats the safe design as the hard, valuable output. As if the problem were that we hadn't yet figured out the right architecture.

We had. Simon Willison published the lethal trifecta - private data, untrusted content, external communication, and the observation that any agent holding all three is exploitable - a *year* before my post. The safe pattern was known. It is cheap to describe. It fits on a slide.

And the field shipped the unsafe one and starred it a hundred and eighty thousand times.

The gap was never architectural. One enterprise survey this year found that eighty-eight percent of organizations reported a confirmed or suspected agent-security incident, while eighty-two percent of executives believed their existing policies already protected them against exactly that. Those are the same companies. That gap is not a knowledge gap. Everyone involved could have read the trifecta post. The gap is that the unsafe design is the one that demos well, ships this quarter, and goes viral, and the safe design is the one that adds a consent prompt and a policy engine and a reason for the user to churn.

I wrote a post about how to build agents correctly. The actual determinant of 2026 was that correctness was available, and unrewarded, and so it lost. That is not a security-engineering problem. It is an incentives problem, and I addressed it in zero of my sections.

---

## The part that should bother us

The title of this post is a line from *Twin Peaks*. The Giant says it in a crowded room while a murder happens upstairs, and the horror of the scene is not that anyone is surprised. It's that the warning is exact, and public, and changes nothing. The knowledge is already in the room. It just doesn't reach the hand in time.

That is the actual shape of agent security in 2026, and it's the thing my April essay, measured and structural and faintly optimistic that better design would arrive, did not have the nerve to say.

None of this was unknown. The trifecta was published. The confused deputy is from the seventies. Supply-chain poisoning of package registries is a solved-and-resolved genre of attack with a decade of prior art. Unauthenticated network exposure is the oldest finding in the book. Every ingredient of the OpenClaw crisis was documented before OpenClaw existed. We did not lack the analysis. We shipped past it, at speed, because the thing that spreads is capability and the thing that doesn't is restraint.

So when I say *it is happening again*, I don't mean a new class of vulnerability. I mean the recurrence itself. We are re-running the early-web security curve. Ship first, expose everything, discover the registry is poisoned, add scanning after the stealer is already resident. Except the client this time can read your email, hold a persistent memory, and take actions under your name. The plot is a rerun. Only the blast radius is new.

The Tor era had an honest excuse: the privacy tools were genuinely hard to use, and people opted out because the friction was real. The agent era doesn't get that excuse. The safe pattern here is not hard. It is one policy layer and one consent boundary and a refusal to execute untrusted skills. It is cheap. It is known. And it lost to a version with fewer prompts and more stars.

That's the part the April essay missed, and it's the part I'd write first if I were writing it today. The dangerous thing about agentic systems was never only that the model can be confused. It's that we can be, reliably and profitably and at scale, into deploying the version that was already known to be unsafe.

The monster in that show was never new either. That was always the point of the warning. It is happening again, and it will keep happening again, until the incentive to ship the safe loop is larger than the incentive to ship the loop that goes viral.

I don't have that mechanism. Neither, apparently, does anyone else, or the exposed-instance count would be lower.

I'll grade that prediction in another three months.

---

*Previous: [We Used to Talk About Tor. Well We've Got LLM Agents](https://stzifkas.github.io/llm/security/2026/04/14/we-used-to-talk-about-tor-well-weve-got-llm-agents.html)*

*Cover: Evelyn De Morgan, "Cassandra" (1898), public domain, via Wikimedia Commons.*
