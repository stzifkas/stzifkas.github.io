---
layout: post
title: "You Are the Bottleneck"
date: 2026-10-18 00:25:00 +0300
categories: [llm, software, economics]
image: /assets/you-are-the-bottleneck.jpg
---

![Adolph Menzel, The Iron Rolling Mill (Modern Cyclopes), 1872–1875: people tending machinery in a vast ironworks.](/assets/you-are-the-bottleneck.jpg)

*AI agents can save developers time. Who gets to spend it?*

A developer gives an agent a ticket, goes to make coffee, and comes back to a pull request. Implementation, tests, documentation. The whole little ceremony of software production, compressed into the time it takes the kettle to finish.

Then comes the review.

The agent has misunderstood a permission boundary. The tests confirm its misunderstanding. The implementation works beautifully for a version of the requirement nobody asked for. Somewhere inside four hundred lines of plausible code is a decision the developer now has to reconstruct.

This is a hypothetical morning. It is also a useful place to begin thinking about what agents do to development, because the productivity conversation tends to stop just before it.

The code appeared. The developer was still needed. What happened to the time?

In the August 2026 version of [*Artificial Intelligence in the Firm*](https://fion.ac/jellyfish.pdf), Fiona Chen and James Stratton examine 300 million work events across 718 firms. Agent adoption increases lines of code by about 30%, while estimated changes in finished software output are statistically insignificant. Average pull-request review time rises by 49%. This is observational working-paper evidence, dependent on assumptions about adoption timing, rather than a verdict on every agent or team.

Demirer, Musolff and Yang find a related pattern in [*Writing Code vs. Shipping Code*](https://www.nber.org/papers/w35275): gains diminish as measurement moves downstream towards released software. Their [account of the research](https://cepr.org/voxeu/columns/writing-code-versus-shipping-code-productivity-effects-across-generations-ai-coding) emphasises complementary stages of production.

There is a reasonably simple way to think about this. Suppose finished work requires both generation and verification:

$$
Q=\min\left(G,\frac{H}{h}\right)
$$

Here, $G$ is the number of candidate changes generated per day, $H$ is the developer-hours available for verification, and $h$ is the verification time each change requires. $Q$ is the number of changes that can actually be completed. Assume comparable changes and fixed requirements; this is an illustrative model, not the papers' estimated specification.

If agents generate a hundred changes and developers can verify twenty, output is capped at twenty. Doubling generation gives you a larger queue. If the additional changes are harder to understand, verification time rises and completed output can fall.

There are plenty of engineering responses. Smaller changes. Better specifications. Better tests. Agents that check their work more effectively. Some teams will find substantial gains through exactly these improvements.

But the equation also leaves something out: who decides how much human attention the system is entitled to consume.

Calling a developer a bottleneck is a perfectly legitimate description of throughput. It becomes a different kind of statement when the proposed solution is that the developer must supervise more concurrent tasks, switch contexts faster, and sign off on more work in the same afternoon.

The bottleneck is where human judgment still takes time. A production system organised around increasing throughput will experience that time as a constraint. The developer experiences it as the time required to understand what they are being asked to approve.

Both descriptions can be accurate. They carry very different priorities.

An agent can therefore make coding easier while making the job more demanding. There may be less time spent implementing a solution and more time spent reconstructing someone else's solution, except the someone else can generate another one before you finish reading the first.

Responsibility survives the automation. The failed migration still belongs to a team. The permission bug still wakes a person. The agent's confidence does not take the incident call.

This is one possible reorganisation of development: a smaller amount of direct implementation supporting a larger volume of supervision, with developers carrying responsibility for decisions they had less opportunity to form.

The papers establish a narrower empirical pattern. This account of its possible consequences is an interpretation, and the distinction matters. A review bottleneck does not by itself demonstrate intensified work, deteriorating autonomy, or declining pay. Those require their own evidence.

Still, it gives us a better question to investigate than whether the agent writes good code.

What happens to the developer's control over the working day?

Suppose the engineering problems get solved. Agents become reliable. Review gets easier. The same useful software can genuinely be delivered with fewer developer-hours, after accounting for inference, integration, maintenance and failures.

At this point the technical success is real. The distribution of its benefits remains open.

A team could work fewer hours for the same pay. It could keep its hours and tackle a useful backlog. It could finally spend time on accessibility, maintenance and the awkward architectural repair that never survives planning.

The company could also reduce headcount, leave vacancies unfilled, or raise delivery expectations until the saved time disappears into a new baseline.

These possibilities do not require different models. They require different decisions about the use of the models.

We can make that separation explicit. Let $S$ be the net economic gain from adoption over some period, after its costs. In a deliberately simplified accounting exercise, divide it between developers and owners:

$$
B_d=\beta S,\qquad B_o=(1-\beta)S
$$

$B_d$ is the benefit reaching developers through compensation or paid time released; $B_o$ is the benefit retained by owners. The parameter $\beta$, between zero and one, records the division. We are holding customer benefits and other claimants outside this small example.

This is bookkeeping, not an estimated law. Its usefulness is that it makes a missing variable visible.

A benchmark can help us understand whether a system might increase $S$. It says nothing by itself about $\beta$.

The division depends on contracts, bargaining power, ownership, hiring conditions and developers' ability to influence how work is organised. Technical indispensability may strengthen that influence. It does not automatically turn into authority over staffing, deadlines or compensation.

You can be essential to approving the output and have very little say over how much output arrives.

The same separation appears at a larger scale in Korinek and colleagues' September 2026 working paper, [*Economic Scenarios for Transformative AI*](https://www.anthropic.com/institute/econ-scenarios). Its extreme scenario combines rapid growth with a fall in labour's income share from roughly 60% to 45%. These are conditional outcomes from an Anthropic Institute model, not forecasts of developers' salaries.

The point worth carrying back to software is the distinction between producing more and receiving more.

An expanding company can afford better conditions. Affordability does not decide whether it provides them. That decision runs through institutions and competing claims on the additional income.

There is another reason developers should pay attention to this, even when the tools feel empowering.

Software expertise lives partly in individuals and partly in a shared working environment: code, documentation, conventions, review discussions, remembered failures. An agent connected to that environment can make parts of its knowledge easier to reuse. Decisions that once required a conversation with a particular developer may become available through a system.

That can be useful. It can reduce tedious repetition and let more people contribute.

It can also change who controls access to the team's accumulated knowledge. A developer's skill remains theirs. The repository, internal records, agent configuration and permission to deploy the resulting system may belong to the company. The model itself may be accessed through infrastructure controlled by another company again.

The ability to use a powerful tool during employment is a particular kind of access. Whether developers can retain that access, change its terms, take their contributions elsewhere, or participate in deciding its purpose are separate questions.

This is why the cheerful image of every developer becoming a manager of agents feels incomplete. Giving someone ten automated implementers expands their capacity to execute. It does not necessarily give them authority to choose the product, refuse a deadline, control the budget, or decide what happens to the additional revenue.

They may get a larger span of responsibility inside the same employment relationship.

None of this requires rejecting the tools. There are tasks developers would gladly stop doing, and good reasons to want software to become cheaper and easier to build. A credible argument for better working conditions should take those gains seriously.

It should also ask what changes developers can actually observe.

Has paid working time fallen? Has compensation improved? Is there room to maintain the system properly? Can the team limit incoming work when verification capacity is exhausted? Can developers challenge an adoption decision whose costs are being hidden in their evenings?

Those are concrete measures of whether saved time has become a benefit to the people doing the development.

The pull request is ready when the kettle finishes. That is an impressive technical achievement.

Whether the developer gets to drink the coffee depends on who controls the schedule.

---

*Cover: Adolph Menzel, The Iron Rolling Mill (Modern Cyclopes), 1872–1875. Alte Nationalgalerie, Berlin. Public domain, via [Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Adolph_Menzel_-_Eisenwalzwerk_-_Google_Art_Project.jpg). Resized for the web; full composition retained.*
