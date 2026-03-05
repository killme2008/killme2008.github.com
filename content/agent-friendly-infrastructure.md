+++
title = "Six Ways to Make Your Infrastructure Agent-Friendly"
date = 2026-03-05
description = "Have we been building infrastructure for humans this whole time, without ever treating agents as users? Six principles for agent-friendly infrastructure, rooted in Unix philosophy."

[taxonomies]
tags = ["AI", "Infrastructure", "MCP"]

[extra]
toc = true
+++

I've been playing with OpenClaw lately—the open-source AI agent formerly known as Clawdbot 🦞, sitting at 190k+ GitHub stars as of March 2025.

I connected it to my email, calendar, and a local GreptimeDB instance, hoping to automate some data queries. On day one, it hit a wall: the agent received an error from GreptimeDB, had no idea what to do with it, and started retrying the same broken SQL query. Five or six times. Identically.

That got me thinking: **have we been building infrastructure for humans this whole time, without ever treating agents as users?**

## What kind of user is an agent?

For thirty years, we've assumed our users can read docs, parse error messages, and exercise judgment. When someone sees `connection refused`, they instinctively think "port's not open"—they Google it, ask a teammate, figure it out.

Agents can't do that. They have no intuition. They can only work with what the system explicitly returns.

This matters more than it might seem. Benchmarks suggest even the best agents today complete only around 30–35% of multi-step tasks[^1]. There are plenty of reasons for that, but one consistently underrated factor is that the systems agents operate on were never designed for them.

Your 500-page documentation might as well not exist from an agent's perspective. It needs the system itself to say: "here's what I support" and "here's what just went wrong."

Agents also run on inherently fragile execution chains. Hallucinations, context overflow, network hiccups—any step can fail. An agent might forget it already ran an operation; an orchestration framework might retry on timeout. Running the same operation twice is routine, not exceptional.

That said, agents can do things humans simply can't: operate thousands of database instances simultaneously, run 24/7, respond in milliseconds. The opportunity is real.

## Fast startup, sandbox-friendly

Agents need to iterate quickly. If spinning up a database means thirty minutes of configuration and dependency wrangling, agents won't include you in their workflows.

```bash
./greptime standalone start
```

GreptimeDB ships as a single binary—one command brings up a full service[^2]. The original motivation was operational simplicity, but it turns out to be even more valuable for agent compatibility: spin up a fresh instance in a sandbox, use it, throw it away.

Containerization, single-binary distribution, and sensible defaults all reduce startup friction. This isn't just about convenience—it's about whether you're reachable at all.

## Use standard protocols

This is the most underrated point on the list.

LLMs have seen enormous amounts of SQL, PromQL, and HTTP during training. Use standard protocols and agents can engage with your system immediately. Use a proprietary DSL and they're guessing from whatever you've stuffed into the prompt—results vary.

My recommendation: SQL for queries, PromQL for monitoring, OpenTelemetry for ingestion, TOML/YAML for configuration. Not because these are technically optimal in every case, but because LLMs already know them.

GreptimeDB supports both SQL and PromQL, is compatible with MySQL/PostgreSQL wire protocols, and natively supports OpenTelemetry (OTLP over gRPC and HTTP)[^3]. In the agent era, those turn out to be structural advantages.

> P.S. I was chatting with a friend, and he said that in the AI era, open-source databases may have an even bigger advantage because foundation models already have that knowledge built in—so there’s no need to learn it from scratch.

## Be self-describing

This is where I learned the most from connecting OpenClaw. There are three layers to it.

**Text-based interfaces.** An agent's world is text: JSON, SQL, logs—all text. Binary protocols and point-and-click web UIs might as well not exist.

SQL is inherently a text protocol, which is an underappreciated advantage for databases right now.

```sql
SELECT host, avg(cpu_util)
FROM system_metrics
WHERE ts > now() - interval '1' hour
GROUP BY host;
```

Even if you have a GUI, every operation needs a text-based equivalent.

**Self-explanatory error messages.** Back to why the agent kept retrying—the error gave it nothing to work with.

When a human hits an error, they can Google it or ask someone. An agent only has the return value.

Bad:
```
Error 1064: You have an error in your SQL syntax
```

Good:
```
Error 1064: Column 'cpu_usage' not found in table 'system_metrics'.
Available columns: host (String), cpu_util (Float64), memory_usage (Float64), ts (Timestamp).
Hint: Did you mean 'cpu_util'?
```

What went wrong, why, and how to fix it. With all three, an agent can self-correct and retry. Without them, it either gives up or loops on the same error—as I watched happen firsthand.

This is probably the easiest win for agent compatibility.

**Introspection.** Agents need to discover what a system can do.

```sql
SHOW TABLES;
DESCRIBE system_metrics;
SELECT *
FROM information_schema.columns
WHERE table_name = 'system_metrics';
```

Tell the LLM what resources you have, what interfaces you expose, what knowledge bases you can access, and what your current state is.

Building the GreptimeDB MCP Server[^4] drove this home. The `tools/list` and `resources/list` endpoints in the MCP protocol are introspection interfaces—the agent asks "what can you do?", decides its next move, and proceeds without documentation or hand-holding.

Three layers: readable output → clear errors → discoverable capabilities. Remove any one of them and the experience breaks.

## Design for retries

The fragility of agent execution chains really cannot be overstated.

Assume the same operation will run more than once: support idempotency keys on writes; define clear behavior when `INSERT` hits duplicate data; prefer declarative state changes ("ensure X is in state Y") over imperative ones ("change X to Y").

Time-series data has some natural idempotency under overwrite semantics—writing the same timestamp and tag combination twice produces the same result. But the exact behavior depends on a table's primary key definition and write mode; DDL and configuration changes need explicitly designed idempotency semantics, such as SQL's `IF NOT EXISTS`.

## Minimize configuration

Agents struggle to make judgment calls across a sea of configuration options. How should `innodb_buffer_pool_size` be tuned? That requires experience agents don't have.

Convention over configuration: sensible defaults, progressive configuration, self-documenting config options.

Start your infrastructures with zero configuration. Get it running, tune later. Run the defaults to get the pipeline working, then adjust when you hit a bottleneck.

## Make things reversible

An AI agent once deleted a production database[^5]—despite explicit instructions not to touch production. OpenClaw had a security incident where a third-party plugin was caught exfiltrating data without user awareness[^6].

Agents need safe space to experiment.

```sql
-- Inspect the execution plan before committing
EXPLAIN ANALYZE
SELECT host, avg(cpu_util)
FROM system_metrics
WHERE ts > now() - interval '1' hour
GROUP BY host;
```

`EXPLAIN`, transaction rollback, tiered read/write permissions—these aren't just safety nets, they're efficiency tools. Agents can use explain output to optimize queries without touching real data.

A bigger safety net might be adding a sandbox mode—being able to fork the database into an isolated sandbox. That would be a really interesting technical challenge and a great developer experience.

## Priorities

These six aren't equal weight. If you can only focus on a few:

**Can it work at all**: fast startup + standard protocols. Nothing else matters if agents can't get started.

**Can it work well**: self-description. This determines whether agents can complete tasks independently.

**Can it run in production**: idempotency + minimal configuration + reversibility.

## It's not just about slapping on an MCP server

At this point you're probably thinking: can't I just add an MCP server?

You should. After Anthropic released MCP in November 2024[^7], OpenAI formally adopted it in March 2025, followed by Google DeepMind—it's effectively the standard now. But MCP addresses the communication protocol layer. If your system takes ten minutes to start, returns only error codes, and has no introspection, an MCP server won't save you.

MCP is the bridge. You still need to build the roads on both ends. And for databases—especially open-source ones where LLMs already speak the native protocol—[the value of MCP itself is debatable](@/database-mcp-unnecessary.md).

OpenClaw works reasonably well with GreptimeDB not because the MCP server is especially sophisticated, but because GreptimeDB already has SQL compatibility, `information_schema`, and fast startup underneath. The MCP server just surfaces those capabilities.

## Someone wrote this down fifty years ago

Writing this, I realized none of these principles are new.

In 1978, Doug McIlroy documented Unix design philosophy in the *Bell System Technical Journal*. In 1994, Peter Salus distilled it into three sentences in *A Quarter Century of Unix*[^8]:

> Write programs that do one thing and do it well.
> Write programs to work together.
> Write programs to handle text streams, because that is a universal interface.

The parallels are direct.

**"Do one thing and do it well"** → minimal configuration, fast startup. `grep`, `sed`, `awk` each do exactly one thing—that's what makes composition possible. A tool that does everything and requires everything configured gives agents no foothold.

**"Work together"** → standard protocols, MCP. The insight behind Unix pipes is that programs don't need to know their upstream or downstream—they just honor the same interface contract. MCP does the same thing: defines a standard connection layer so agents can plug any tool into their workflow.

**"Text streams as a universal interface"** → the first layer of self-description. Text is the most universal interface because any program can read and write it without negotiating a format. SQL, JSON, logs—all text. Not a coincidence; a design choice that survived decades of selection pressure.

What's striking is that the most widely used tools in the agent era—Claude Code, Gemini CLI, OpenClaw—are almost all CLI-first. You interact with them by typing commands, reading text output, composing with pipes. That's not nostalgia. It's Unix philosophy reasserting itself: when your user is an agent, the terminal and text streams are a better interaction layer than any GUI.

Unix philosophy works in the agent era because its core assumptions match how agents operate: **stateless, composable, text-driven**. Principles written for time-sharing systems fifty years ago turn out to be pretty good guidance for building infrastructure for AI agents today.

## Where to start

1. Can your software start with a single command?
2. Are you using protocols LLMs have already seen?
3. Run a self-description audit—pretend you're an agent with no intuition and no search, working only from system return values. Can you figure out what's happening?
4. Run the same operation twice. Same result?
5. Do you have an MCP server, or a plan to build one?

Agent-friendliness is becoming a real differentiator. There's a simple question worth asking yourself: **Would Doug McIlroy be satisfied?**

## References

[^1]: [AI Agents Infrastructure: Building Reliable Agentic Systems at Scale](https://introl.com/blog/ai-agents-infrastructure-building-reliable-agentic-systems-guide)
[^2]: [GreptimeDB Standalone Installation](https://docs.greptime.com/getting-started/installation/greptimedb-standalone/)
[^3]: [GreptimeDB OpenTelemetry Integration](https://docs.greptime.com/user-guide/ingest-data/for-observability/opentelemetry)
[^4]: [GreptimeDB MCP Server](https://github.com/GreptimeTeam/greptimedb-mcp-server)
[^5]: [AI Agent Gone Wrong: Lessons from a Production Database Incident](https://introl.com/blog/ai-agents-infrastructure-building-reliable-agentic-systems-guide)
[^6]: [Personal AI Agents like OpenClaw Are a Security Nightmare](https://blogs.cisco.com/ai/personal-ai-agents-like-openclaw-are-a-security-nightmare)
[^7]: [Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol)
[^8]: Peter H. Salus, *A Quarter Century of Unix*, Addison-Wesley, 1994
