+++
title = "Is Database MCP Just Unnecessary Overhead?"
date = 2026-01-20
description = "My colleague called Database MCP 'pulling your pants down to fart.' After thinking about it, I mostly agree—but we built one anyway."

[taxonomies]
tags = ["MCP", "Database", "AI"]

[extra]
toc = true
+++

I was discussing Andy Pavlo's annual database retrospective[^1] with a colleague when we got onto the topic of MCP (Model Context Protocol). The article noted that in 2025, nearly every database vendor released their own MCP Server—including us at GreptimeDB[^2].

My colleague's assessment was blunt: **"It's like pulling your pants down to fart."**

(For non-Chinese speakers: this is a Chinese idiom meaning "completely unnecessary extra steps to achieve the same result.")

His logic: LLMs already understand SQL. Give them a database connection and documentation, and they'll get the job done. Why add another protocol layer? Why make LLMs call databases through MCP? Isn't that just unnecessary overhead?

I've been mulling this over for a few days. Here's what I think.

## The Case Against Database MCP

Let me lay out the opposing view clearly, because it actually makes sense.

**Argument 1: LLMs Already Have Built-in Database Knowledge**

Top-tier models like Claude Opus 4.5 and GPT-5.2 write SQL better than many programmers. Give them a schema, and they'll produce correct queries. Tell them PostgreSQL, and they know `LIMIT` and `OFFSET`; tell them MySQL, and they'll use `LIMIT n, m`. MCP doesn't need to teach them this.

**Argument 2: Standard Protocols Are Sufficient**

Databases already have standard protocols—PostgreSQL wire protocol, MySQL protocol, REST APIs. LLMs can write code to access databases directly through these. Why add another layer?

```
Traditional: LLM → Generate Code → Execute SQL → Results
MCP way:    LLM → MCP Client → MCP Server → Execute SQL → Results → MCP Server → MCP Client → LLM
```

That's several extra hops, and LLMs are already quite capable of writing code that calls standard protocols.

**Argument 3: Just Provide Documentation**

If an LLM doesn't know a database's special syntax, just give it the docs. Modern LLMs have large enough context windows—Claude Opus 4.5 has a 200K token context window, more than enough for GreptimeDB's entire core documentation.

**Therefore**: Database MCP is a compromise for when LLMs aren't powerful enough. Once models evolve another generation, this stuff becomes obsolete.

I have to admit, this argument makes sense. **In fact, I largely agree.**

## So Why Did We Build One Anyway?

If I also think Database MCP is somewhat redundant, why did GreptimeDB still release an MCP Server?

### Reason 1: Ecosystem Positioning

This is the most pragmatic reason.

MCP's growth has been remarkable: from roughly 100,000 downloads at launch in November 2024 to 8 million by April 2025[^3]. The MCP Registry now has over 5,800 MCP Servers and around 300 clients[^3]. OpenAI, Google, and Microsoft have all officially adopted MCP.

When every competitor has an MCP Server in the marketplace, not having one makes you fall behind. When users search for database tools in MCP clients, those with MCP Servers get discovered first.

This isn't a technical requirement—it's a **business requirement**.

Let's be honest: most Database MCP functionality can be achieved with native SQL interfaces. But "can be done" and "will users actually do it" are different things. If users are accustomed to finding tools in the MCP ecosystem and you're not there, you don't exist.

### Reason 2: MCP's Real Value Isn't in Databases

MCP's value for databases is limited, but it's different for other systems.

Think about things without standard access protocols: internal ops tools, legacy systems, private APIs, SaaS admin dashboards. LLMs haven't seen these systems, documentation might be incomplete, and there are certainly no standard protocols. MCP's value for these is far greater than for databases.

Database MCP is, in a sense, something we did "while we were at it"—since the MCP ecosystem is taking off, databases as infrastructure can't be absent.

### Reason 3: Weaker Models Were an Unexpected Win

We didn't anticipate this one.

According to Menlo Ventures' 2025 mid-year report[^4], in the enterprise LLM market, Anthropic holds 32%, OpenAI 25%, Google 20%, while Meta's Llama has only 9%. Closed-source models still lead open-source by 9–12 months in performance.

But use cases for open-source models are real. Many enterprises choose private deployment due to data sensitivity, compliance requirements, or cost control. They run Llama, Qwen, DeepSeek.

When building the GreptimeDB MCP Server, we had users of top-tier models in mind. Turns out, the people who found it most useful were a different group entirely: **enterprise users running privately-deployed open-source models**.

These models vary widely in capability. Some get basic SQL syntax wrong, let alone GreptimeDB-specific PromQL-compatible syntax or Range Queries.

For them, carefully designed tool descriptions and example templates are far more useful than dumping a pile of documentation.

The key distinction: **Documentation is for humans to read; tool descriptions are for models to use.**

Documentation aims for completeness, context, and explanation. Tool descriptions aim for precision, structure, and executability. For strong models, these don't differ much. For weaker models, the latter is significantly more effective.

## Cloudflare's Finding: Writing Code Beats Tool Calling

Here's an interesting finding from Cloudflare[^5]. They ran an experiment: instead of using tool calling directly, they exposed MCP tools as TypeScript APIs and had LLMs write code to call them. Results were significantly better.

The reason: LLMs have seen massive amounts of code, but tool calling's special tokens are synthetic training data with less exposure. So an LLM writing `db.query("SELECT * FROM users")` is more reliable than generating `<tool_call>query_database</tool_call>`.

This further supports the "Database MCP is overkill" argument: **having LLMs write code to call databases directly might work better than MCP tool calling**.

But it also points to an interesting direction—MCP can serve as a standard for tool discovery and management, while actual invocation happens through code execution. Cloudflare calls this "Code Mode."

## So Where Is MCP's Value?

If Database MCP is indeed overkill, does MCP itself have value?

Yes, but not for databases.

**MCP's real value is LSP-style ecosystem standardization.**

LSP (Language Server Protocol)[^6] decoupled editors from language support. Before LSP, N editors × M languages = N×M integrations. LSP turned this into N+M: write a language server once, and all compatible editors can use it.

MCP does the same for LLM tools. But here's the key: **this value is greater for systems without standard protocols**.

Databases already have SQL, JDBC/ODBC, and various wire protocols. MCP here is icing on the cake—or unnecessary overhead, depending on how you look at it.

But for internal tools, legacy systems, and private APIs? MCP is a lifesaver.

## My Conclusion

Back to the original question: Is Database MCP unnecessary overhead?

**Mostly, yes.**

For users with top-tier models, providing documentation and having the model write code to call standard protocols probably works better. Database MCP doesn't add much technical value.

**But we still built one, and we'll keep maintaining it.**

The reasons are simple:

1. **Ecosystem positioning is a real need**—in an ecosystem of 5,800+ MCP Servers, if you're not there, you don't exist
2. **Weaker model users were an unexpected win**—in private deployment scenarios, well-designed tool descriptions genuinely help
3. **MCP's real battlefield isn't databases**—we just built one "while we were at it"

If you're using Claude Opus 4.5 or GPT-5.2, honestly, you probably don't need the GreptimeDB MCP Server. Just give the model our documentation and let it write code to connect—that'll likely work better.

But if you're running privately-deployed open-source models, or you just like finding tools in the MCP ecosystem—the MCP Server has its uses.

**That's the reality: technically overkill, commercially necessary.**

---

## References

[^1]: [Databases in 2025: A Year in Review - Andy Pavlo](https://www.cs.cmu.edu/~pavlo/blog/2026/01/2025-databases-retrospective.html)
[^2]: [GreptimeDB MCP Server](https://github.com/GreptimeTeam/greptimedb-mcp-server)
[^3]: [MCP Statistics - mcpevals.io](https://www.mcpevals.io/blog/mcp-statistics)
[^4]: [2025 Mid-Year LLM Market Update - Menlo Ventures](https://menlovc.com/perspective/2025-mid-year-llm-market-update/)
[^5]: [Code Mode: the better way to use MCP - Cloudflare](https://blog.cloudflare.com/code-mode/)
[^6]: [Language Server Protocol](https://microsoft.github.io/language-server-protocol/)
