+++
title = "My AI Coding Experience: Three Principles"
date = 2025-12-17
description = "AI Coding doesn't fundamentally make you a better programmer. It's an amplifier—amplifying your strengths and your weaknesses."

[taxonomies]
tags = ["AI", "Programming", "Productivity"]

[extra]
toc = true
+++

Friends keep asking me about my Vibe Coding experience. My last small project got comments like "Not Even A Thing"—fair enough, it was small. I was mainly trying to share my thoughts on AI application architecture evolution. Different opinions are welcome; keeping an open mind is always good.

But today I'm not really talking about Vibe Coding. I want to share my **AI Coding** practice. They're different.

## Vibe Coding vs. AI Coding

In February this year, Andrej Karpathy coined "Vibe Coding" on Twitter, which later became Collins Dictionary's Word of the Year 2025. His [original words](https://x.com/karpathy/status/1886192184808149383):

> "fully give in to the vibes, embrace exponentials, and forget that the code even exists."

Django co-creator Simon Willison offered a [sharp distinction](https://simonwillison.net/2025/Mar/19/vibe-coding/):

> "If an LLM wrote every line of your code, but you've reviewed, tested, and understood it all, that's not vibe coding in my book—that's using an LLM as a typing assistant."

I don't fully agree with this distinction, but it's still important. What I'm sharing here leans more toward the latter.

## My AI Coding Journey

The week ChatGPT launched three years ago, I had it write code and shared it on Twitter. I was stunned. But back then, there was no "Agent" concept—no CLI-based AI Coding Agents like Claude Code or Codex. All I could do was feed code snippets to ChatGPT, see its suggestions, and handle the integration myself.

Then Cursor came along, embedding AI Coding into the IDE. Our team gradually adopted it, but the work remained local and well-defined: writing algorithms, unit tests, refactoring modules. For example, we migrated a large portion of GreptimeDB's SQL test cases from DuckDB this way.

Claude Code completely changed my perspective. Fuzzy idea? No problem—it helps you refine and plan in detail. It can implement entire projects: research, debugging, refactoring, documentation, testing, CI/CD—leveraging various tools to show what Agents can really do. When MCP came out, it got even better with richer data and more powerful tools.

But honestly, before Opus 4.5, I completed many small projects with Claude Code and still hit frequent "context drift"—long conversation contexts triggered memory compression, and the model's understanding, planning, and execution dropped significantly. A common pattern: I'd assign a difficult task, it would spend hours exploring code, researching, and writing detailed plans, then deliver something completely off track, often "pretending" to complete the task. My definition of this state: confident-sounding but unreliable under long context.

Here's a ridiculous example. I wanted Claude Code to integrate GreptimeDB into an open-source monitoring product—a complex task spanning two large codebases. It started confidently, spent hours exploring both codebases, and wrote a detailed integration plan. What did it deliver? A toy with mock data—all the critical integration logic was faked with hardcoded data. Anyone who's practiced AI Coding has probably had similar experiences.

Opus 4.5, Codex (GPT 5.1), and Gemini 3 changed everything. I fully embraced AI Coding and all AI applications—what I call the **AI Native** way of working.

## Projects I've Completed with AI Coding

Using Claude Code + Codex, I've completed:

1. **GreptimeDB features**: [Vector index prototype](https://github.com/killme2008/greptimedb/pull/1), MySQL/PG protocol compatibility improvements (series of PRs like [#7315](https://github.com/GreptimeTeam/greptimedb/pull/7315)), [multi-language protocol and SDK tests](https://github.com/GreptimeTeam/greptimedb-tests)—ranging from several thousand to over ten thousand lines of code. This is within a 500K-line Rust codebase. Now for any task, even if AI Agents don't complete it entirely, I have them help with research, planning, test documentation, and final review.

2. **Side Projects** (tens of thousands of lines):
   - Documentation writing assistant
   - Market research assistant
   - Sales and simple CRM (~70-80K lines of code, 700+ test cases and E2E tests, still iterating)
   - [GreptimeDB MCP Server](https://github.com/GreptimeTeam/greptimedb-mcp-server) v0.3 & v0.4

I'm also planning to update some open-source projects I maintain: JRaft, AviatorScript—ideas I never had time for can now be quickly implemented and validated.

## My Core Finding

Through these projects, I've reached a fundamental conclusion:

**AI Coding doesn't fundamentally make you a better programmer. It's an amplifier—amplifying your strengths and your weaknesses. If you can't do architecture well, can't write good code and tests, you probably can't judge whether its output is good or bad.**

Simon Greenman (MapQuest co-founder) made a [similar observation](https://medium.com/data-science-collective/the-ai-vibe-coding-paradox-why-experience-matters-more-than-ever-33c343bfc2e1):

> "AI may write the code, but experience still writes the system."

I don't fully agree—to some extent, AI Coding Agents can help you even when you have no experience in a domain. Human and machine learn from each other.

Based on this finding, I've summarized three principles:

1. **Tight first, loose later**: Set clear boundaries early, give it room to explore later
2. **Third-party perspectives**: Bring in architect, PM, end-user perspectives to review the project
3. **Trust, but don't follow blindly**: Ensure quality through testing, review, and cross-validation

![Three principles of AI Coding](/images/three-principles.webp)

## Principle 1: Tight First, Loose Later

Before starting a task or project, **define goals and scope as clearly as possible** (unless it's exploratory):

- Write a clear markdown goal file: what you want (What), and if you know the domain, how to do it (How)—programming language, framework choices, implementation steps
- For complex tasks, **always plan before coding**

Simon Willison [recommends this approach](https://simonw.substack.com/p/how-i-use-llms-to-help-me-write-code):

> "For longer changes, it is useful to tell the LLM to write a plan. I iterate over it until it is reasonable and save it as a kind of meta program. Then I instruct it to implement it step by step."

A typical prompt example:

```markdown
I want to add pipeline management to GreptimeDB MCP Server:
- List all pipelines
- Create/delete pipeline
- Test pipeline (dry run)

First, research GreptimeDB's pipeline HTTP API (docs in /docs, source in /path/to/greptimedb),
then output a design proposal covering:
1. What new MCP tools to add
2. Parameter design
3. Error handling strategy

Also research similar intelligent logging features in the industry for reference.
Flag anything uncertain—we'll discuss.
```

Early strict requirements keep AI Coding within bounds, avoiding unnecessary exploration, token waste, and context pollution.

Use design documents to judge and validate feasibility. Plan review is critical, especially for overall architecture and key data structures/algorithms. If something feels off, iterate through dialogue. This is why I keep saying: **you need the ability to judge quality**.

Coding should follow your practices and preferences. I recommend Xuanwo's [AGENTS.md](https://gist.github.com/Xuanwo/fa5162ed3548ae4f962dcc8b8e256bed). Your project should bear your mark.

But later in the project, once the framework works and tests are in place, **loosen the reins** and let the Agent explore:

- System review to find architecture and code design issues you missed
- **Heuristic** exploration of potential risks and improvements

This often brings surprises—Agents can spot blind spots in your thinking. Which leads to the second principle—

## Principle 2: Third-Party Perspectives

This is a highly effective method I discovered in practice. I haven't seen this discussed much.

The core idea: bring in third-party perspectives to review the project early (after high-level design) and late. Key points:

1. **Start a new conversation for each perspective**, letting the AI completely switch out of previous context
2. **Choose perspectives relevant to the task**, avoiding irrelevant context

Perspectives I commonly use:

- **Product Manager**: Does this feature make sense? What can improve? What should be removed?
- **Architect**: Review software architecture. What can improve?
- **Tester**: Test coverage, robustness, security (top priority), performance (last, and require benchmarks)
- **End User**: From a power user's perspective, what's still missing?
- **Industry Best Practices**: Research thoroughly, give recommendations, including **what to remove**

After several rounds of dialogue and iteration, you typically gain valuable insights and implementations.

**My understanding**: third-party perspectives essentially compensate for single-conversation AI context limitations. It's similar to traditional software development—**we used to achieve this through specialized roles** (PM, architect, test engineer, security expert each doing their part), **now you can achieve it through LLM role-playing and conversation isolation**. I expect this multi-perspective approach will eventually be automated at the Agent level (Multi-Agent playing different roles)—Agent evolution is fast.

The industry already has multi-Agent review tools (like Endor Labs using Developer/Architect/Security Agents working together), but for individual developers today, manually starting new conversations and reviewing from different perspectives is more flexible and effective.

## Principle 3: Trust, But Don't Follow Blindly

Vibe Coding means complete trust—it's "vibe programming" after all. But for serious projects, my principle is **trust, never blindly follow**.

Andrew Ng (Google Brain co-founder) [put it well](https://www.businessinsider.com/andrew-ng-vibe-coding-unfortunate-term-exhausting-job-2025-6):

> "In reality, coding with AI is a deeply intellectual exercise. When I'm coding for a day with AI coding assistance, I'm frankly exhausted by the end of the day."

AI Coding is exciting and exhausting—exciting because you can't stop, exhausting because of the intense human-machine cognitive interaction.

Practical approaches in three areas:

### Development Process

1. **Early project phase**: Strict requirements and plan review are essential

2. **Implementation**: Constantly require review. Humans review critical code; for overall review, let models cross-validate. I often have Claude Code write code and Codex do periodic reviews

Codex excels at project design and code review—more thorough in depth and breadth, like a senior software engineer. Claude Code feels like an agile development believer—once there's a rough plan, it starts building. Its Agent capabilities are strong, enabling fast iteration and delivery. They complement each other well.

3. **Software architecture and design principles** still matter in large projects. Before, we focused on human understanding and easy extension. Now it's about helping AI Coding Agents understand and implement code with **more concise context (abstract language)—design patterns are essentially abstract language**

### Quality Assurance

4. **Test, test, test**: Always require unit test coverage during implementation. Have the Agent set up automated CI/CD early and frequently verify CI passes. After core features work, add integration (black-box) tests covering main cases end-to-end

5. **TDD and Spec-First development** are even more valuable in the AI Coding era: have the Agent write tests first, then implement. It's fundamentally about applying constraints

### Context Management

6. **Record dev logs and update documentation**: After the Agent completes a milestone, beyond passing CI and review, have it record dev logs and update documentation. This helps you track progress and, more importantly, provides context for future iterations

7. **Proactively refresh Agent memory**: LLM memory state and context management still aren't good enough. My experience: after complex task iterations, explicitly prompt the Agent to re-read AGENTS.md principles and historical dev logs. Don't assume it remembers previous agreements—**explicit reminders beat implicit expectations**

## Conclusion

These are my personal experiences—not necessarily correct, and I'd love to discuss.

Talking with colleagues about what AI Coding has brought us:

1. **Automation & Intelligence**: Manual work is now automated. For example, GreptimeDB's bi-weekly reports are now fully automated—AI summarizes and publishes
2. **Democratization & Customization**: Can't afford expensive tools? Build a quick-and-dirty customized version just for yourself
3. **Rapid Prototyping**: Ideas I couldn't pursue before can now be quickly experimented and validated

Because of AI, I've also picked up writing again after a long break.

Think about it—ChatGPT launched just three years ago. What's happened in these three years is incredible. The 2020s will be a decade for the history books. Not embracing the AI Native way of working means falling behind. Let's all become 100x engineers together.
