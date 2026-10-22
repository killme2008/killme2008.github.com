+++
title = "Is Software Dead in the Age of AI?"
date = 2026-10-22
description = "Partly yes: AI agents will replace a lot of application software that sells UI and workflow. Partly no: enterprises with private data still need B2B software, and infrastructure software takes years in production to get right."

[taxonomies]
tags = ["AI", "Software", "Infrastructure", "Open Source"]

[extra]
toc = true
social_media_card = "/images/is-software-dead-cover.jpg"
+++

![Is Software Dead in the Age of AI?](/images/is-software-dead-cover.webp)

There's a lot of talk lately that software is dead. Here is my take.

To some extent, the answer is yes. Model vendors and their agents will replace a large share of consumer and business software, especially for individuals and small businesses. Bookkeeping, drawing, writing, data analysis, reports, professional consulting: a lot of needs that used to mean buying software can now be handled by just asking an AI for it. What gets replaced is mostly application software that sells UI and workflow, and per-seat SaaS gets hit first. This may be the best time ever for free software: real freedom to build it, share it, and change it.

On the other side, the answer is no.

First, enterprises and use cases with private data, strict compliance, and high business standards still need models deployed on-prem and tight control over their data. Model quality varies, data quality varies, a company's own data is often messy and needs ongoing governance, and few companies have enough in-house AI talent. They still need professional B2B software and vendors to help them adopt and implement AI. That's why forward deployed engineer (FDE) roles are in such demand, and why Palantir has become the poster child for an AI-era company.

Second, infrastructure software and the systems that hold core data are still a long-term bet, because AI itself runs on infrastructure software. And the more software AI generates and the more agents run, the more storage, compute, and observability load sits underneath. Infrastructure software is also being rebuilt for agents: the workload patterns, interfaces, and data types are different from before. Model vendors, for marketing reasons, claim they built an operating system, a database, or a programming language in a few hours. Trust me, nobody is going to run those in production. Infrastructure software takes long-term, continuous investment. It doesn't happen overnight. A database an AI writes in one night can run a demo, but edge cases, failure recovery, version compatibility, and data correctness in production get worked out over years of running and fixing. Infrastructure software keeps taking in use cases from all kinds of users, keeps iterating, keeps fixing bugs and getting better. It has a long life, even as more and more of its development relies on AI.

AI also lowers the bar for writing infrastructure software. There will be more competitors, and the moat will come more from stability built up over years in production and from the community around it. How infrastructure software, and open source in particular, builds a business in the AI era is a separate question. I'll leave it for another time.

That's my view on whether AI kills software. Happy to discuss.
