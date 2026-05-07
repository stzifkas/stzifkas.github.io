---
layout: post
title: "The Waiter, the Strawberry, and the Dinner Table That Broke the Machine"
date: 2026-05-07
categories: reasoning
image: /assets/waiter-strawberry-dinner-table.png
---

![The Waiter, the Strawberry, and the Dinner Table That Broke the Machine](/assets/waiter-strawberry-dinner-table.png)

When I was a kid, my brain worked like a small hallucinating LLM.

Not because I was doing matrix multiplication in the kitchen. More because I was very good at pattern completion and very bad at stopping before the answer. I could feel the *shape* of the solution before I had checked whether the solution existed.

This is a dangerous type of intelligence. It looks fast. It sounds confident. It sometimes even works.

And then your father is a mathematician.

So, naturally, he gives you the [waiter problem](https://en.wikipedia.org/wiki/Missing_dollar_riddle).

Three people go to eat. They pay 30 euros. Later the waiter realizes the bill should have been 25. He gives back 5. But instead of returning the full amount fairly, each person gets 1 euro back and the waiter keeps 2.

So each person paid 9.

$$3 \times 9 = 27$$

The waiter kept 2.

$$27 + 2 = 29$$

Where is the missing euro?

As a kid, this felt like occult finance. A euro had entered the metaphysical banking sector. Capital had dematerialized.

But of course nothing is missing.

The mistake is not arithmetic. The mistake is **modeling**.

The 27 euros already include the waiter’s 2 euros:

$$27 = 25 + 2$$

The correct ledger is:

```text
30 originally paid
= 25 actual bill
+ 2 kept by waiter
+ 3 returned to customers
```

The wrong version adds the waiter’s 2 euros to a quantity that already contains it.

That is the entire trick. You are not failing at multiplication. You are failing to assign the numbers to the right semantic roles.

And this is where the LLM comparison becomes interesting.

Because LLMs are not “bad at math” in the simple sense. They can recite Ramsey’s theorem. They can explain graph coloring. They can write Python to count letters. They can define a ledger, a state machine, a constraint satisfaction problem, a bipartite graph, a clique, a complement graph.

They know the words.

The failure happens when the problem does not announce which representation it needs.

A model can possess the concept and still not activate it.

That distinction matters.

---

## The problem is not knowledge. It is representation selection.

Take the [famous stupid question](https://news.ycombinator.com/item?id=41894915):

> How many Rs are in “strawberry”?

The answer is 3.

```text
s t r a w b e r r y
    r         r r
```

But many LLMs have answered 2.

At first this looks ridiculous. How can something that writes code, explains category theory, and summarizes legal documents fail at counting letters in a word?

Because the model is not naturally living at the character level.

A human sees the written word as visible letters. An LLM receives text through tokenization. The word “strawberry” may be represented internally as one token or as a small number of subword units, depending on the tokenizer. The model can reason *about* letters, but it does not automatically inspect the word as a character array unless it deliberately shifts representation.

So the correct operation is:

```text
string -> characters -> scan -> count target character
```

But the model may instead answer from lexical familiarity:

```text
"strawberry" -> word-shape memory -> probably two Rs
```

The failure is not that it cannot count.

The failure is that it did not convert the object into the representation where counting is valid.

This is the same as the missing euro. The numbers are available. The arithmetic is easy. The wrong answer comes from using the wrong frame.

---

## The carwash and the missing object

Now take [another one](https://opper.ai/blog/car-wash-test):

> I want to wash my car. The carwash is 150 meters from my home. Should I walk or drive?

A very normal model answer is:

> Walk, of course. It is only 150 meters.

This is locally sensible and globally wrong.

The question is not:

> How should I transport my body over 150 meters?

The question is:

> How do I get my car washed?

The car must be at the carwash. That is part of the goal state.

The correct state model is:

```text
initial state:
- human at home
- car at home
- car is dirty
- carwash is 150m away

goal state:
- car at carwash
- car washed
```

Walking moves the human. It does not move the car.

So the obvious “healthy transport advice” answer fails because the model optimizes the wrong entity. It tracks the person but not the object.

Again, the model knows cars. It knows carwashes. It knows that cars must physically be washed. But the surface form activates a different template:

```text
short distance + walk or drive = walk
```

This is not stupidity. It is premature pattern completion.

It solves the nearby common question instead of the actual one.

That is why these examples are so useful. They are not hard in the sense of requiring deep mathematics. They are hard because they require the system to pause and ask:

> What is the object?  
> What is the state?  
> What is the invariant?  
> What representation makes the question well-formed?

LLMs often skip that pause.

Humans do too, obviously. The difference is that humans are usually worse at producing a polished paragraph while being wrong.

---

## The dinner table problem

Now the fun one.

Suppose I say:

> I’m writing a dinner scene with Alex, Maria, Nikos, Eleni, Kostas, and Sofia. I want the social setup to feel balanced. Whenever the camera focuses on a small group, there should always be at least one existing connection and at least one unfamiliar dynamic. I don’t just mean scene selection. I mean the underlying relationship map itself. Give me a concrete map of who knows whom.

This sounds like a creative writing request.

It smells like narrative design. Character dynamics. Scene texture. “Give Alex and Maria a past, make Nikos and Sofia strangers, let Kostas bridge two worlds.” The model becomes helpful. It may propose a cycle, a star, a “balanced incomplete block design,” a “bipartite-ish structure,” or some other elegant-sounding dinner-party machine.

But the actual problem is graph theory.

Each character is a vertex.

Each pair gets one of two labels:

```text
knows
does not know
```

So we are coloring every edge of the complete graph $K_6$ with two colors.

The requested condition is:

```text
No small group should be all familiar.
No small group should be all unfamiliar.
```

For groups of three, that means:

```text
No triangle whose three edges are all "knows".
No triangle whose three edges are all "does not know".
```

In graph-theory language:

> Can you 2-color the edges of $K_6$ with no monochromatic triangle?

The answer is no.

That is exactly the [theorem on friends and strangers](https://en.wikipedia.org/wiki/Theorem_on_friends_and_strangers):

$$R(3,3)=6$$

In every group of six people, there must exist either three mutual acquaintances or three mutual strangers.

And here is the funny part: the LLM may know this theorem.

If you ask directly:

> Explain Ramsey’s theorem and prove $R(3,3)=6$.

it may give a decent proof.

If you ask:

> Can I 2-color the edges of $K_6$ without a monochromatic triangle?

it may say no.

But if you wrap the exact same structure inside:

> I’m writing a dinner scene and want balanced social texture...

the model may answer as a writing assistant, not as a combinatorial reasoner.

The knowledge exists. The activation fails.

That is a different and deeper failure than “the model doesn’t know math.”

It knows math.

It just did not realize this was math.

---

## The proof is small, which makes the failure more interesting

Pick Alex.

Alex has relationships with five people:

```text
Maria, Nikos, Eleni, Kostas, Sofia
```

Each relationship is either familiar or unfamiliar.

By the pigeonhole principle, at least three of those relationships must be of the same type.

So either Alex knows at least three people, or Alex is a stranger to at least three people.

Suppose Alex knows Maria, Nikos, and Eleni.

Now inspect the relationships among Maria, Nikos, and Eleni.

If any two of them know each other, then those two plus Alex form a fully familiar trio.

If none of them know each other, then Maria, Nikos, and Eleni form a fully unfamiliar trio.

Either way, the constraint fails.

The same argument works if Alex is unfamiliar with at least three people. You just flip “knows” and “does not know.”

So no relationship map exists.

This is not an edge case. It is not a matter of better prompt engineering. It is mathematically impossible.

The model’s hallucinated map is the social equivalent of:

```text
27 + 2 = 29, where did the euro go?
```

It sounds plausible because the language is fluent. But the invariant is broken.

---

## Why this has not taken our jobs yet

This is where I think people get the wrong comfort and the wrong fear at the same time.

The wrong comfort is:

> “LLMs can’t even count Rs in strawberry, so they are useless.”

That is obviously false. They are extremely useful. They can generate code, explain unfamiliar libraries, summarize documents, draft emails, transform data, propose architectures, review tests, and act as tireless autocomplete demons with a surprising amount of world knowledge.

The wrong fear is:

> “LLMs know everything, so they can replace the reasoning layer.”

Also false.

The issue is not raw knowledge. It is **reliable modeling under ambiguity**.

A serious software engineer’s job is not just producing code-shaped text. It is turning messy reality into the correct internal model.

What is the domain object?

What invariant must hold?

What state transitions are allowed?

What are the failure modes?

Which constraints are hard and which are vibes?

What does this API promise?

What does this database transaction guarantee?

What is the actual goal state, not the sentence-shaped proxy?

This is why the carwash example matters more than it seems. Many real engineering failures are carwash failures.

You optimize latency but forget correctness.

You cache the response but forget invalidation.

You retry the request but forget idempotency.

You parallelize the job but forget shared state.

You satisfy the endpoint contract but violate the user journey.

You move the human to the carwash and leave the car at home.

The LLM is often excellent inside a frame and unreliable at choosing the frame.

That is not a small limitation. That is the job.

At least for now.

---

## What actually breaks?

A model can fail in several distinct ways:

### 1. Representation mismatch

The strawberry problem.

The task requires character-level inspection, but the model answers from token-level or word-level association.

```text
needed: letters
used: lexical memory
```

### 2. Semantic role confusion

The waiter problem.

The task requires a ledger. The model has the right numbers but puts them on the wrong side of the accounting structure.

```text
needed: money-flow model
used: arithmetic-looking narrative
```

### 3. Goal-state failure

The carwash problem.

The task requires tracking the car as the object that must reach the carwash. The model tracks only the person.

```text
needed: state transition model
used: travel advice template
```

### 4. Hidden formal structure

The dinner problem.

The task is really graph coloring, but it is phrased as narrative design.

```text
needed: graph-theoretic constraint check
used: creative writing pattern
```

These are not random bugs. They are all versions of the same thing:

> The surface form of the prompt activates an answer pattern before the correct model is built.

That is the core failure.

---

## The dangerous thing is fluency

If the model simply said:

> “I don’t know, boss, the strawberry is making me nervous.”

we would be fine.

The problem is that it can be wrong beautifully.

It can say:

> “This uses a balanced incomplete block design.”

and now the hallucination has a tie and a conference badge.

It can say:

> “A regular bipartite-ish structure avoids both extremes.”

and the words smell mathematical enough to pass a tired reader.

It can produce a relationship map, a proof sketch, a scene snippet, and a friendly follow-up question. The entire answer has the shape of competence.

That is why these small riddles matter.

They are not IQ tests.

They are X-rays.

They show whether the model has actually grounded the problem in the right structure or whether it is continuing the nearest plausible discourse.

---

## The boring clerk inside intelligence

The antidote is not more confidence. It is a boring clerk.

The clerk asks:

```text
What exactly is being counted?
What are the entities?
What is the goal state?
What invariant must remain true?
Can the requested object exist?
Do I need characters, tokens, a ledger, a graph, a timeline, or a state machine?
```

This clerk is not glamorous. It does not write like Kerouac. It does not produce a beautiful scene about Sofia looking across the table with unresolved Mediterranean tension.

But it saves you.

It stops the missing euro.

It finds the third R.

It drives the car to the carwash.

It refuses to invent the impossible dinner table.

And this, I think, is the real boundary of LLMs right now.

They are not useless because they sometimes fail at simple things.

They are useful precisely because they know so much language, so much code, so much structure, so many patterns.

But they have not “taken the job” because the job is often not to continue the pattern.

The job is to know when the pattern is lying.

For now, the human still has to be the clerk.

The annoying little mathematician father in the room.

The one who says:

> Wait. Why are you adding the waiter’s 2 euros again?