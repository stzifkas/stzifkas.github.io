---
layout: post
title: "I'll See You in Seven Years"
date: 2026-06-01
categories: [open-source, space, parsers]
image: /assets/pluto-ecss-cover.png
---

<!-- TODO: drop the cover image at /assets/pluto-ecss-cover.png and screenshots inline (TUI demo, a compile output, a parse error). The playground link below is the live demo. -->

![I'll See You in Seven Years](/assets/pluto-ecss-cover.png)

In 2019 I wrote a parser for a spacecraft language and then I left it in a drawer for seven years.

Not metaphorically. There was an actual repository, with actual code, and it actually sat there.

The language is PLUTO. It's the DSL that European spacecraft operators use to write procedures — the steps you run to bring up a star tracker, to fire a parallel safety sequence, to wait for an on-board event and react to it. It's standardised by ECSS, it has an annex full of grammar, and almost nobody outside the ground-segment world has heard of it. I loved that about it.

Back then it was a Google Summer of Code sample. The whole thing parsed exactly one hard-coded script and "ran" it by walking the parse tree. It had no tests, no CLI, no packaging, and a few bugs I am still a little embarrassed about. There was a method called `setExecutionSatus`. The `s` was missing. The bug had been missing for seven years too.

I told myself I'd come back to it.

You know how that goes.

---

## The thing about abandoned projects

The problem with an abandoned project isn't the code. The code is fine. The code is patient.

The problem is the wall.

When you open something you wrote years ago, the first thing you hit is not a feature you want to add. It's a missing import. A typo. A class that calls `super.__init__` with no parentheses and would crash the moment you reached it. Three separate small failures, all standing between you and the actual interesting work.

In 2019 those felt like cliff-edges. Each one was a reason to close the tab. "I'll fix all of this *and then* I'll do the real thing" is a sentence that quietly means "I will do none of it."

So it waited.

---

## The chance, not the competition

Then GitHub ran a Finish-Up-A-Thon. A challenge specifically about going back to the thing you never finished and finishing it.

I want to be honest about what that was for me, because I don't think it was what these things are usually sold as.

It wasn't a competition. I wasn't trying to win anything. There was no leaderboard in my head, no other people's projects I was measuring mine against, no prize I was optimizing for.

It was permission.

That's the whole trick. A deadline and a theme gave me an excuse to open the drawer that I'd somehow not been able to give myself for seven years. I didn't need a contest. I needed a reason that wasn't "you should." I needed the small external nudge that turns "someday" into "this week."

That's it. That's the entire mechanism. The challenge didn't make me a better engineer. It made me an engineer who opened the file.

---

## What it is now

Once the file was open, the rest came easier than I expected.

`pluto-ecss` is now a real transpiler. You hand it a PLUTO procedure and it gives you back readable, runnable Python — not a tree walk, not eval-of-a-string, actual source you can read and debug and check into a deploy.

```bash
$ pluto-ecss run examples/01_original.pluto
[ACTIVITY] Switch on Star Tracker2
[ACTIVITY] Switch on Reaction Wheel3 of AOC of Satellite
[ACTIVITY] Switch on Star Tracker1
```

The same script that was the *only* thing the 2019 version could handle is now example number one out of sixteen.

The arc, in numbers:

| | 2019 | 2026 |
|---|---|---|
| Grammar | ~30 lines | 168 lines |
| Tests | 0 | 164 passing |
| Examples | 1 | 16 |
| CLI | none | `parse / compile / run / demo / fmt / gen` |
| Output | side effects from a tree walk | Python (sync / async / class / no-runtime) + JSON |
| Docs | one README | a real site, with a browser playground |
| ECSS coverage | a handful of constructs | most of Annex A.1 and A.3 |

There's a CLI. There's a live TUI demo that lights up a fake satellite as the procedure executes, components flipping from `OFF` to `ON` in real time. There's a Pygments lexer so the language highlights properly. And there's a playground that compiles and runs PLUTO entirely in your browser through Pyodide, no install at all.

You can go press buttons on it right now: **[the playground](https://stzifkas.github.io/pluto-ecss/playground/)**.

The code, the docs, and the before/after live here: **[github.com/stzifkas/pluto-ecss](https://github.com/stzifkas/pluto-ecss)**. The 2019 version isn't deleted, by the way — it's preserved on the `legacy/gsoc-2019` branch. You can check out exactly what I was embarrassed about.

---

## On the AI part, honestly

This revival was AI-assisted and I'm not going to be coy about it.

I used GitHub Copilot through the rebuild. It shows up in the commit history as a co-author on the release work, and I'd rather say that out loud than pretend the whole thing sprang from a lonely genius at 2am.

Where it actually helped: the grammar. PLUTO is full of multi-word identifiers — `Star Tracker2`, `Reaction Wheel3 of AOC of Satellite` — and getting an Earley parser to handle keyword priority without choking on common words is fiddly, error-prone work. The old grammar used brittle negative-lookahead hacks that broke constantly. Copilot helped me reason through the lexer priorities and surfaced an ambiguity bug that would have cost me an afternoon to find alone.

It also wrote a lot of the boring parts. The transpiler is mostly a wall of repetitive `_stmt_*` methods, one per statement kind. The runtime is threading and concurrency boilerplate. That's exactly the kind of typing an assistant should absorb so you can spend your attention elsewhere.

The decisions were mine. Transpiler instead of interpreter. The `src/` layout. Four compile targets. The test design. Those are the parts that are actually me.

But here's the thing I didn't expect, and it's the same thing the challenge did:

The assistant removed the discouragement.

The wall I described earlier — the typo, the missing import, the crash-on-reach bug — those used to be cliff-edges. With an assistant at hand, fixing them and *then* doing the real spec work felt like one continuous motion instead of three separate undertakings I'd have to psych myself up for. The friction that kept the project in the drawer for seven years was mostly the friction of starting. That friction got cheaper.

The challenge gave me permission to open the file. The tooling made the first hour of the open file not feel like punishment.

Between those two, the seven years finally ended.

---

## What I actually took from it

I'm not going to wrap this in a bow about productivity.

What I took from it is smaller and a bit more personal. There's a particular kind of guilt that lives in an abandoned project. It's quiet. You don't think about it most days. But it's there, a little background process, a tab you never closed.

Finishing it didn't feel triumphant. It felt like turning off a sound I'd stopped noticing.

The procedure waited seven years. It was always going to run. It just needed someone to switch it on.

There's another drawer, of course. There's always another drawer.

I'll see you in seven years.
