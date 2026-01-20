+++
title = "The AI Productivity Divide: From 10x to 100x Engineers"
date = 2025-11-27
description = "AI's real value isn't in producing the final output—it's in creating Builders. Whether you can create Builders is what separates the 2x from the 100x."

[taxonomies]
tags = ["AI", "Programming", "Productivity"]

[extra]
toc = true
+++

I've been using AI tools heavily lately, and I want to share two realizations.

**First: AI's real value isn't in producing the final output.**

Most people use AI to generate images, write code, or draft articles, then say things like "the quality is decent" or "it still needs a lot of editing." That's fine, but I think it misses the point.

The real value of AI? It's in creating Builders.

What's a Builder? A tool or workflow that can mass-produce final outputs. When AI generates one image for you, that's a one-off. But when AI writes a script that can batch-process 1,000 images? That's a Builder.

Here's an example. Say you need cover images for 100 articles.

The naive approach: have AI generate them one by one, review each, tweak each. Generate 100 times, review 100 times.

The Builder approach: have AI write an image generator — define style parameters, size specs, text overlay logic — then batch-run it. You only need to review and tune the generator itself, not 100 individual images.

The first is using AI to do grunt work. The second is using AI to build tools.

I later realized this aligns with Andrej Karpathy's "Software 3.0" concept. Karpathy — OpenAI co-founder, former Tesla AI Director, and a central figure in deep learning — argued in his [YC AI Startup School talk](https://www.ycombinator.com/library/MW-andrej-karpathy-software-is-changing-again) that we're entering a new era: natural language is becoming the programming interface, and programmers are shifting from "writing code" to "directing AI to write code." But he also emphasized that this doesn't mean you can skip understanding code — quite the opposite. **The ability to read code, design systems, and assemble components matters more than ever.**

**Second: Whether you can create Builders is what separates the 2x from the 100x.**

Same AI tools, but some people get 2x productivity gains while others get 20x. The difference? Whether you treat AI as "faster hands" or as "a tool for building tools."

This means: **programming skills remain essential.**

Because Builders are fundamentally code. Automation scripts, data pipelines, workflow orchestration — they all come down to code. AI can write it for you, but you need to read it, fix it, and assemble multiple AI-generated pieces into a working system.

People who can't code use AI to do 1+1+1+1. People who can code use AI to do 1×100.

**A personal example.** Over the past few days, I built [greptimedb-tests](https://github.com/GreptimeTeam/greptimedb-tests) — a multi-language black-box test suite for GreptimeDB. It covers drivers and clients across Java, Go, Python, and Node.js, testing compatibility with MySQL/PostgreSQL protocols as well as GreptimeDB's gRPC ingester and OpenTelemetry SDK.

The project currently has about 4.6k lines of code and 36 test cases, fully integrated into GreptimeDB's CI pipeline. It took me around three days to build the initial version. This is what a Builder looks like — a reusable framework that systematically validates protocol compatibility, rather than ad-hoc manual testing.

And it has already paid off, helping us uncover several protocol bugs:

* Fixed a MySQL driver issue with the `DATE` type ([#7291](https://github.com/GreptimeTeam/greptimedb/pull/7291))
* Addressed PostgreSQL protocol incompatibilities with `TIMESTAMP`, `DATE`, and `BINARY` columns ([#7286](https://github.com/GreptimeTeam/greptimedb/pull/7286), [#7276](https://github.com/GreptimeTeam/greptimedb/pull/7276))
* Discovered that the PostgreSQL protocol didn't set the time zone correctly, causing subtle timestamp mismatches during ingestion ([#7289](https://github.com/GreptimeTeam/greptimedb/pull/7289))

During the same period, I was also writing code for other modules, posting on social media, drafting technical articles, and attending discussions and meetings. Building the test suite was just one of several parallel threads — which is exactly the kind of throughput that becomes possible when you're creating Builders instead of producing one-off outputs.

I'm continuing to develop my own Builder, which is designed for tasks such as writing articles and documents, conducting market research analysis, and following up on leads.

Manuel Kießling, who leads a software engineering team at Joboo, captured this well in a [recent article](https://manuel.kiessling.net/2025/03/31/how-seasoned-developers-can-achieve-great-results-with-ai-coding-agents/):

> "an absolute senior when it comes to programming knowledge, but an absolute junior when it comes to architectural oversight in your specific context"

You need to provide direction, constraints, and acceptance criteria for AI to deliver value. And those are precisely the skills that come with experience.

This brings me to an old concept: the 10x engineer.

The idea was that top engineers are 10x more productive than average — through experience, deep system understanding, problem-solving intuition, and knowing which pitfalls to avoid.

Now AI has become a massive lever.

Experienced engineers know what problems to solve, roughly where the solutions lie, and which paths are dead ends. When they use AI, they can articulate requirements precisely, evaluate outputs quickly, and assemble AI-generated pieces into larger systems.

This "experience + AI" combination doesn't just multiply by 10 anymore. I think it might be 100x.

I call this the **100x engineer**.

This isn't just my speculation — there's data. [Fastly's July 2025 survey](https://www.fastly.com/blog/senior-developers-ship-more-ai-code) of 791 developers found:

> **32% of senior developers (10+ years experience) say over half their shipped code is AI-generated; for junior developers (under 2 years), it's only 13%.**

Senior developers use AI-generated code at 2.5x the rate of juniors. Why? Because experienced developers are better at catching AI's mistakes, know how to direct it, and are more confident deploying AI output to production. They treat AI as an efficient junior engineer, not as a black box.

Adlet Balzhanov wrote in his newsletter [*The True Engineer*](https://www.thetrueengineer.com/p/the-100x-engineers):

> "100x engineers are not mythical solo heroes. They're highly leveraged operators who understand the stack — technical, product, and human. They use AI like a cofounder, not a toy."

This leads to an interesting corollary: **40 isn't a cliff — it's an advantage.**

Why? Because AI amplifies what you already have.

With 10 years of experience — deep understanding of your tech stack, grasp of business context, intuition for system design — AI can't replace these, but it can amplify them. A 40-year-old veteran + AI might be tens of times more effective than a fresh grad + AI.

Stanford's digital economy research backs this up: in the industries most impacted by AI (IT and software), employment for 22-25 year olds dropped 6%, while **employment for 35-49 year olds rose 9%**. ([Source: Stack Overflow Blog](https://stackoverflow.blog/2025/09/10/ai-vs-gen-z/))

DEPT Agency's engineering lead made a similar point: as AI automates repetitive work and lowers the barrier to using frameworks, **the differentiators shift**. Boilerplate, syntax, basic implementation — these no longer set you apart. What does?

> "Judgment: Knowing what to build and how it fits into the larger picture. Trade-offs: Understanding the long-term implications of technical decisions. Quality: Spotting subtle flaws, code smells, and architectural weaknesses before they become major problems. Direction: Guiding the development process, whether the 'developer' is human or AI."

These are products of experience. ([Source](https://engineering.deptagency.com/ai-cant-replace-experience-why-senior-engineers-might-be-more-valuable-than-ever))

The old fear of "being too old for tech at 40" might actually flip into an advantage in the AI era. The key is learning to use AI as leverage, not competing with AI on who writes code faster.

---

**Takeaways:**

1. **AI's core value is in building Builders, not producing outputs.** Using AI to generate things is beginner level; using AI to create tools is advanced.

2. **Whether you can create Builders determines if you're 2x or 100x.** That's the productivity divide.

3. **Programming skills matter more than before, not less.** Because Builders are fundamentally code.

4. **Experience gets amplified in the AI era, not replaced.** Veterans have the chance to become 100x engineers.

Let's all push toward 100x.
