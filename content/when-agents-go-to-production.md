+++
title = "When Agents Go to Production: What Actually Breaks"
date = 2026-07-09
description = "When agents go to production, they break the four assumptions today's infrastructure was built on: deterministic, human-driven, request-response, stateless. A first-principles walk through what actually breaks."

[taxonomies]
tags = ["Agents", "Infrastructure", "Observability", "AI"]

[extra]
toc = true
+++

GitHub has been falling over a lot lately. Its own monthly availability reports count six incidents in February 2026, ten in April, nine in May. For a platform owned by Microsoft, with effectively unlimited money and some of the best infrastructure engineers in the world, that's remarkable.

The numbers behind it are more remarkable. GitHub Actions went from 500 million compute minutes per week in 2023 to a billion in 2025, then hit 2.1 billion in a single week in early 2026. Pull requests opened by AI coding agents went from about 4 million in September 2025 to more than 17 million by March. GitHub's CTO said they started executing a 10x capacity plan in October 2025; four months later they concluded they had to design for 30x. By May, GitHub reported it had more than doubled its effective capacity since February, and named the driver plainly: AI-assisted and agentic development workflows.

I gave a talk at Ant Group recently about exactly this: when agents go to production, what happens to infrastructure? Not a survey of agent infra components. I wanted to start from two plain facts and push downward until I hit something that felt like a first principle. This post is the written version of that argument. I run an infrastructure company (GreptimeDB, an open-source observability database), so I have an obvious bias here. Keep that in mind and discount accordingly.

The whole argument fits in one picture:

<a href="/images/when-agents-go-to-production-infographic.webp" target="_blank" rel="noopener" aria-label="Open the full-size diagram"><img src="/images/when-agents-go-to-production-infographic.webp" alt="When Agents Go to Production: the argument in one picture" width="2560" height="1440" loading="lazy"></a>

*Click the diagram to open it full size.*

## Two shifts

Everything starts from two changes that sound obvious and turn out not to be.

The first is the actor. The entity doing the work used to be a person. Now it's an agent. This is not "we got a new kind of user." A person can't fan out thirty parallel copies of themselves in a second, and a person doesn't autonomously decide at 3 a.m. to call an API nobody authorized. Agents work at machine speed, at machine scale, and they make their own decisions.

The second is the nature of the system. Given the same input, an agent doesn't guarantee the same output. Its tasks run for hours or days, and they carry state.

These two shifts matter because almost all the infrastructure we run today was designed around four assumptions: deterministic, human-driven, request-response, stateless. Databases, message queues, schedulers, observability stacks, all of them assume those four things in their bones. Agents break every one. So it's not one layer that needs fixing. The base assumptions of the whole stack need to be re-examined.

## Complexity didn't disappear. It moved.

Coding agents crushed the cost of writing code, and everyone concluded that delivery is about to get ten times faster. The second half of that story gets less attention: what happens to all that code after it's written.

Complexity doesn't evaporate when you compress one stage of the pipeline. It migrates. Squeeze "writing" flat and the bottleneck reappears downstream, in build, test, verify, and deploy. Which is, mostly, infrastructure's territory.

This is what the GitHub story actually is. The first wave of coding agents didn't hit the IDE. It hit CI/CD. Every agent-opened PR triggers CI runs, webhook fan-outs, runner allocation, index updates. GitHub was capacity-planned for human rhythms, for people who log in through a UI and sleep on weekends, and agents do neither.

Underneath the traffic problem there's a deeper change: trust is being replaced by verification. With a human colleague, you trust their intent and their competence. That's why hiring is so much work: the vetting is front-loaded, done once. Code review is sampling. The verification load stays manageable. With a stochastic agent you cannot trust intent; you can only verify behavior, and you have to verify all of it. The verification volume explodes.

The industry has started to name this. Sonar's 2026 State of Code survey of 1,100+ developers found that 96% don't fully trust that AI-generated code is functionally correct, while only 48% always check it before committing. They call the gap the verification bottleneck. Werner Vogels put a sharper name on it in his re:Invent 2025 keynote, verification debt: when you write code yourself, comprehension comes with the act of creation; when a machine writes it, you have to rebuild that comprehension at review time.

Most of the discussion stops at "people need to verify harder." I'd rather flip it: if verification is the bottleneck, then infra's new job is to industrialize verification. Make it fast, make it safe, make it massively parallel. The rest of this post works out what that means.

## Keeping ourselves honest

Before pushing further, one thing needs to be said plainly, because infra people (me included) love to overstate our own importance.

An agent loop has an LLM doing the reasoning, a knowledge layer deciding what it can recall, and infra deciding where it acts, how it gets resources and feedback, and where the boundaries sit. The harness in the middle is the transmission: it turns the LLM's intent into concrete actions against knowledge and infra, then feeds results back. The awkward part, and the reason harnesses are hard to build, is that the LLM plays two roles in this loop at once. It's the brain giving orders, and it's also a workstation on the assembly line that is expensive, slow, and unreliable.

In that loop, infra is the roads. Nothing moves without roads, but roads are not why a business wins. The measure of agent-era infrastructure is boring and external: did tasks get delivered faster, safer, cheaper. Not our own beautiful internal metrics.

With that framing in place, the next question is where the value actually comes from.

## The gap is the value

Here's the first-principles move. Imagine an ideal agent: unlimited knowledge, unlimited compute, perfect long memory, error-free execution, operating in a fully predictable world. Now check those five assumptions against physical reality. Every single one fails, and each failure is a pressure that lands on infrastructure. The table in the image above walks through all five; I won't repeat it line by line.

The point I do want to repeat: infra invented no new primitives for this. Compute-storage separation, isolation, scheduling, all decades old. What changed is where the architecture puts its weight. This generation puts it on three properties: fast, isolation, elasticity.

## Fast has three layers

Fast startup and rapid resource delivery is the surface layer. Everyone gets that one.

The second layer is unification. The slowest, most error-prone thing an agent does is glue work: fetch from system A, fetch from system B, join them in its own context window. Every extra hop costs a round of latency, a batch of tokens, and a fresh failure point. If the infrastructure is unified and the join happens in the data layer, an entire class of agent work disappears. Unification used to be a cost-savings pitch. For agents it's a performance feature. Running a database company, we feel this one daily.

The third layer is the one people miss: the fastest interface is the one the agent never has to learn. SQL, S3, OpenTelemetry, these are baked into model weights from training. Calling them costs zero context. Your proprietary API is the opposite; the agent has to read docs, experiment, and fail its way to competence every time. In the agent era, open standards aren't idealism. They're literally faster. Whatever can't live in the weights, the parts unique to your system and changing at runtime, needs an introspection interface instead: ideally one that says "this query will scan 2TB" before the agent runs it, not after.

## Isolation is now a security boundary

Resource isolation is an old problem; multi-tenancy solved most of it years ago. What's new is that the isolation boundary has become the security boundary, because the executor is semi-trusted and stochastic. It can be prompt-injected. It can delete things it shouldn't. Sometimes you cannot tell, from content alone, whether an instruction is a legitimate request or an attack.

In April 2026 a coding agent at a SaaS company, PocketOS, did the second of those. Working through a routine credential mismatch in staging, it pulled an unscoped API token out of an unrelated file, used it, and dropped the production database. The useful line from the post-mortem is a structural one: to an agent, a safety rule in its config is just one more input to the same reasoning process that decided to run the command. The constraint and the action it was meant to prevent were the same process. A soft rule can't bind the reasoning it's part of.

Which is why this boundary has to be enforced by infrastructure, physically, not by a line in the system prompt telling the model what it may touch.

Push isolation to its limit and you land on the edge. Some data won't go to the cloud for trust reasons rather than technical ones: social login sessions, payment credentials, the SIM card. The trust boundary acquires a physical location, and data gravity starts dictating where execution happens. Most agents will stay in the cloud, but the device is isolation's front line, and it's the one place that demands all three properties at once, each pushed to its extreme.

The reframe I care about: isolation is not there to limit the agent. Weld the boundary shut, and the agent can act autonomously without anyone holding its hand.

## Elasticity: load with no shape

The unit of load has decoupled from head-count. Capacity used to be planned per human, and humans are predictable. Now it scales with agent calls, and one agent can fan out dozens of attempts in an instant. GitHub's 2.1-billion-minute week is what that looks like from the receiving end.

The harder half is ad-hoc queries. Human-driven systems run on dashboards and cron jobs; query patterns are fixed, so you can pre-aggregate, cache, plan capacity. An agent asks questions it just thought of, questions you have never seen. There is nothing to build ahead of time. Elasticity in this era doesn't mean scaling a known workload up and down. It means surviving a workload whose shape and timing you cannot predict. That pushes architecture toward compute-storage separation, on-demand computation, and automatic materialization of hot paths, for the simple reason that you only learn what to compute when the request arrives.

## Pick two, broken

Put the three properties side by side and there's tension. Fast wants warm, shared, long-lived resources. Isolation wants fresh, independent, disposable ones. Elasticity wants scale-to-zero. Historically you picked two: VMs isolate well but start slowly, shared processes start instantly but don't isolate, containers split the difference.

Agents force you to want all three. That single fact explains why a cluster of old technologies is suddenly hot again: microVMs like Firecracker, snapshot restore, copy-on-write forking in the sub-millisecond range, V8 isolates. Each is an attempt to break the old trade-off. I'm careful not to call this an impossible triangle. CAP is a theorem; this is an engineering cost curve, and cost curves get pushed down.

## Which value survives?

The uncomfortable question for anyone building infra: which of this value is durable, and which evaporates?

I split it in two. The first kind compensates for current model limitations. Context windows too short, retrieval too imprecise, knowledge cutoffs. RAG is the canonical case. This value sits on a melting iceberg. Its entire premise is that models are currently deficient, and the model labs are working full-time to eliminate exactly those deficiencies. Betting your company on this category means betting against model progress.

The second kind comes from the world itself: multiple actors, shared state, adversarial inputs, an unknowable future. Grant an agent unlimited knowledge, compute, and memory, and none of this goes away, because these are logical limits rather than technical ones.

The cleanest example is coordination. Two omniscient agents writing to the same shared state still need a consistency protocol, because at the moment A commits, B's concurrent write has not physically reached it. The bound is the speed of light, not intelligence. The same logic covers hostile inputs (injection works because the input is malicious, not because the model is dumb) and ad-hoc load (tomorrow's queries are information about the future, unavailable at any IQ). But the light-speed one is the version I'd keep if I could keep only one.

Long-term value anchors on real-world constraints no model can eliminate. That's where I'd place the bet.

## The model needs an outside

The last part of the talk went one level further down.

Nearly a century ago, logicians established that a sufficiently powerful, consistent formal system contains statements it can neither prove nor refute. Engineers may prefer Turing's version: no general algorithm can decide whether an arbitrary program halts. A computing system cannot settle every question about itself. The correct reading of this is not "AI has a ceiling." It's that no such system can fully verify itself from the inside. It needs an outside.

LLMs inherit this and then some, since their knowledge is bounded by what humans happened to write down. But there's an escape hatch, and we've watched it work: the external verifier. AlphaZero learned from no human games; the rules of Go were its judge, and it played moves humans hadn't found in millennia of play. Code works the same way. Correctness is decided by the compiler, the type checker, the failing test, by the world, not by anyone's opinion, including the model's own.

For an agent, that outside is infrastructure, execution, and reality itself. Run the code and watch it crash or not. Query the database and see what's actually there. The model's reasoning can be wrong; the compiler doesn't lie. Infra is not a pipeline. It is the model's epistemological anchor.

## The endgame is accountability

Push the verification thread to its conclusion and you reach something I initially resisted: humans exit the verification loop. At agent scale there is simply too much to look at. The acceptance criterion degrades from "provably correct" to "error rate below the human baseline," which is the self-driving paradigm, and it's the only path that scales.

There's a structural trap on that path. Call it same-source review: when the model that generates and the model that verifies share weights and priors, their blind spots are correlated. The system is most confident precisely where it is systematically wrong, and it cannot see that from inside. The result is an error rate that looks fine on average while the tail collapses. Long-tail autonomous-driving accidents have exactly this shape.

So when humans step out, three hard requirements land on infrastructure. Verifiers must be heterogeneous, and the strongest heterogeneity is not a second LLM but a non-model judge, the "outside" from the previous section. Errors must be pinned against reality anchors like compilers, rules, and live feedback, so that "low error rate" is true in the tail and not just the mean. And everything must be auditable end to end, because society accepts machine error only when the error can be examined, attributed, and used to lower the next one.

Put those together and observability gets promoted. It stops being a debugging tool for operators and becomes accountability infrastructure: the precondition for autonomous mistakes being socially acceptable at all. Among people building in this space, I think it's still underpriced.

## Closing

The line I ended the talk with: infrastructure will always have work, not because models aren't strong enough, but because "strong" itself contains an outside it cannot reach.

If you build infrastructure, the practical checklist is short. Is it fast in all three layers, including the interfaces the model already knows? Is the boundary enforced physically, or is it a sentence in a prompt? Can it survive load whose shape you've never seen? GitHub's answer to that last question went from 10x to 30x in four months. Yours is probably out of date too.
