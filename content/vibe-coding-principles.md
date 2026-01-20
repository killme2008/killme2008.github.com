+++
title = "Rethinking Software Design Principles in the Age of Vibe Coding"
date = 2025-11-28
description = "When AI can rewrite an entire module in seconds, do the design principles built for 'humans maintaining code' still apply?"

[taxonomies]
tags = ["AI", "Programming", "Software Design"]

[extra]
toc = true
+++

Last week, I spent half an hour arguing with a colleague over a PR.

The PR was mostly written by Claude Code under my guidance. The code had some duplication—several functions with similar logic. My colleague said, "We should abstract this into a generic method." I pushed back: "These functions will evolve independently. Abstraction would become a burden."

We didn't reach a conclusion. But it got me thinking: when AI can rewrite an entire module in seconds, do the design principles built for "humans maintaining code" still apply?

## Starting with Vibe Coding

In February 2025, Andrej Karpathy (OpenAI co-founder, former Tesla AI director) coined "Vibe Coding" in a tweet[^1]:

> "There's a new kind of coding I call 'vibe coding', where you fully give in to the vibes, embrace exponentials, and forget that the code even exists."

Karpathy's description was extreme: fully trust AI-generated code, hit "Accept All" without reviewing diffs, copy-paste error messages for AI to fix, even "randomly change things until the problem goes away." He explicitly said this is "for throwaway weekend projects."

But the industry is sliding in this direction. Y Combinator's Winter 2025 batch shows 25% of startups have codebases where 95% of code is AI-generated[^2]. GitClear found code duplication increased 8x in the AI era[^3].

Simon Willison made an important clarification[^4]:

> "If an LLM wrote every line of your code, but you've reviewed, tested, and understood it all, that's not vibe coding in my book—that's using an LLM as a typing assistant."

This distinction matters. This article isn't about "abandon all review." It's about a practical question: when AI becomes the primary code producer, what design principles do we need?

## The Core Assumption Behind Traditional Principles

SOLID[^6], DRY[^7], KISS[^8]—these principles were born from one assumption: **code is written by humans for humans, and modification is expensive**.

Sandi Metz described a common scenario in her 2016 article *The Wrong Abstraction*[^5]:

1. Programmer A spots duplicate code and extracts an abstraction
2. Programmer B needs new functionality and adds parameters and conditionals
3. Programmers C, D, E keep stuffing in more conditionals
4. Eventually, the abstraction becomes "twisted and convoluted, hard to understand"

Her conclusion:

> "Duplication is far cheaper than the wrong abstraction."

This insight is widely accepted in microservices. Sharing code across services introduces coupling. Sometimes you'd rather duplicate than have two independent teams depend on the same component. Some even say "DRY is the enemy of decoupling."

Here's the question: if AI can rewrite any code in seconds, shouldn't we recalculate the cost of duplication versus abstraction?

## How the Cost Structure Changed

What has AI changed?

**Costs that dropped**: generating new code (hours to seconds), rewriting existing code (nearly zero), generating tests and docs.

**Costs that haven't dropped**: understanding semantics and business logic, verifying correctness, making architecture decisions, debugging cross-module issues.

This means: **defensive abstractions may no longer be necessary, but semantic clarity matters more than ever**.

AI doesn't fear duplicate code. It fears not understanding your intent.

## Five New Principles

Based on this cost shift, I propose five design principles for AI collaboration. To keep this concrete, I'll use a real project as a case study: `greptime-content`—a content creation automation system I built with Claude.

### From Separation of Concerns to Semantic Separation

Traditional Separation of Concerns (SoC) divides modules by human responsibilities: Model handles data, View handles presentation, Controller handles logic.

AI doesn't understand code this way. For AI, what matters is: **what is this code's intent?**

Here's the architecture of `greptime-content`:

```
prompts/       ← Instructions for AI ("this is my system prompt")
projects/      ← Work state ("this is my current context")
glossary.yaml  ← Constraints ("these terms must translate this way")
workflow_next()← Direction ("what should I do next")
```

This isn't MVC. It's layering by **semantic intent AI can understand**:

| Module | Meaning to AI |
|--------|--------------|
| `prompts/` | "These are my instructions" |
| `projects/` | "This is my current context" |
| `glossary.yaml` | "These are translation constraints" |
| `workflow_next()` | "This is my next direction" |

The design doc summarizes it well:

> "Prompts are files—version controlled, easily editable, team-shareable. State lives in the filesystem—not dependent on Claude's memory. MCP is the glue—letting Claude read and write the filesystem."

This is semantic separation: dividing not by human responsibilities, but by **intent boundaries AI can understand**.

### From DRY to Replaceability over Reusability

Traditional DRY (Don't Repeat Yourself) pursues reuse. In the AI era, pursue **replaceability** instead.

Look at how `greptime-content` organizes prompts:

```
prompts/zh/
├── research.md        # Research prompt
├── write-zh.md        # Chinese writing prompt
├── translate.md       # Translation prompt
├── polish.md          # Polishing prompt
└── social/
    ├── linkedin.md    # LinkedIn post
    ├── twitter.md     # Twitter post
    └── reddit.md      # Reddit post
```

Traditional DRY would say: abstract these social media prompts into a template system with variables for platform differences.

The actual choice: one independent file per platform. Why?

1. **Independent evolution**: LinkedIn and Twitter styles will diverge. Abstraction becomes what Sandi Metz calls "the wrong abstraction"
2. **Context isolation**: Changing one file doesn't affect another. No worrying about breaking shared modules
3. **Zero rewrite cost**: AI can rewrite an entire prompt in seconds

The `prompt_update()` function reflects this:

```python
@mcp.tool()
async def prompt_update(name: str, content: str, lang: str = None) -> str:
    """Update a prompt template (creates backup of old version)."""
    # Replace the entire file, not modify template parameters
```

One-liner: **Duplication is the price of stability; rewriting is the engine of evolution.**

### From Open/Closed Principle to Prompt-Oriented OCP

The traditional Open/Closed Principle (OCP): open for extension, closed for modification.

In AI collaboration, this shifts from "code" to "prompts":

- **Closed: core prompts**. `write-zh.md` defines fundamental writing principles (technically accurate, opinionated, short paragraphs). These rarely change
- **Open: task extensions**. Need new functionality? Add new prompt files, don't modify existing ones

The workflow task list shows this:

```python
WORKFLOW_TASKS: list[str] = [
    "translate-to-en",
    "translate-to-zh",
    "polish-zh",
    "polish-en",
    "challenge-zh",   # Added later for review
    "challenge-en",
    "cover-image",
    "social",
    "social-image",   # Added later for social images
]
```

When we needed article review, we didn't modify the writing prompt—we added `challenge.md`. Core stays stable, boundaries stay open.

### From Clarity over Cleverness to Semantics over Structure

Traditional wisdom: "Clarity over Cleverness." In the AI era, go further: **Semantics over Structure**.

For AI, elegant design patterns don't matter. Inferable semantics do.

Look at the directory structure:

```
projects/{slug}/
├── meta.yaml           # Metadata + history
├── 00-idea.md          # Initial idea
├── 01-research.md      # Research findings
├── 02-draft-zh.md      # Chinese draft
├── 02-draft-zh-v2.md   # Polished version
├── 03-draft-en.md      # English translation
└── social/             # Social media content
```

Filenames are semantic:
- `00-`, `01-`, `02-`, `03-` prefixes tell AI the workflow order
- `-zh`, `-en` suffixes tell AI the language
- `-v2`, `-v3` suffixes tell AI version relationships

The constant definition:

```python
STAGES: dict[str, str] = {
    "idea": "00",
    "research": "01",
    "draft-zh": "02",
    "draft-en": "03",
}
```

This isn't about elegance. It's about AI being able to "read" intent. Naming is documentation; AI understands intent through naming.

Docstrings follow the same principle:

```python
@mcp.tool()
async def project_goto(slug: str, target: str) -> str:
    """
    Jump to a specific task in the workflow.

    Use this when you want to perform a specific task without following
    the linear workflow. Works for both imported and regular projects.

    Args:
        slug: Project identifier
        target: Target task, one of:
            - 'translate-to-en': Translate Chinese draft to English
            - 'translate-to-zh': Translate English draft to Chinese
            ...
    """
```

Who is this docstring for? AI first. It needs to know when to use this, what parameters mean, what options exist.

### From Observability to Explainability and Co-agency

Traditional Observability focuses on monitoring and logging. In AI collaboration, we need more: AI should **understand** system state, not just **read** it.

The History Message system in `greptime-content`:

```yaml
# meta.yaml structure
history:
  - stage: idea
    time: 2025-11-26T10:30:00
    message: "Initialize idea"
  - stage: draft-zh
    time: 2025-11-26T15:00:00
    file: 02-draft-zh.md
    message: "First draft with benchmark data"
  - stage: draft-zh
    time: 2025-11-26T17:00:00
    file: 02-draft-zh-v2.md
    message: "Polish: improve intro section"
```

This isn't just logging. It's "AI-conversable state":
- AI can query history: `project_status(slug, include_history=True)`
- AI can understand what happened last time
- AI can explain why this version exists

`workflow_next()` goes further—AI knows what to do next:

```python
@mcp.tool()
async def workflow_next(slug: str, show_all: bool = False) -> str:
    """Get guidance on the next step for a project."""
    # Returns detailed instructions based on current stage
```

This is co-managing with AI—not just passive execution, but understanding the workflow and actively guiding.

## Where Are the Boundaries?

These principles aren't universal. Traditional principles still apply to:

- **Performance-critical paths**: Database engine core loops, HFT hot paths—every line needs careful human thought
- **Security-sensitive code**: Encryption, permission checks, input validation—requires rigorous auditing
- **Long-stable infrastructure**: Core components running for years, well-tested—"if it ain't broke, don't fix it"

The new principles work better for: fast-iterating business code, tools and scripts, config templates, frequently-adjusted UIs.

## Summary

| Traditional Principle | New Principle | Core Change |
|----------------------|---------------|-------------|
| Separation of Concerns | Semantic Separation | Layer by AI intent, not human responsibilities |
| DRY | Replaceability over Reusability | From "reuse" to "regenerable" |
| Open/Closed Principle | Prompt-Oriented OCP | Core prompts closed, task extensions open |
| Clarity over Cleverness | Semantics over Structure | Naming and comments serve AI understanding |
| Observability | Explainability & Co-agency | Let AI understand and participate in evolution |

These principles don't replace traditional ones. They're an evolution for a world where AI collaboration is the norm.

We used to write code for humans. Now we write code for humans and AI together.

The goal isn't "perfect architecture" anymore. It's systems that can be understood, rewritten, and continuously evolved.

---

**Easter egg**: This article was written by the `greptime-content` assistant, which researched and analyzed the `greptime-content` project as reference material. Meta-circular, indeed.

## References

[^1]: [Andrej Karpathy's tweet on Vibe Coding](https://x.com/karpathy/status/1886192184808149383)
[^2]: [TechCrunch: A quarter of startups in YC's current cohort have codebases that are almost entirely AI-generated](https://techcrunch.com/2025/03/06/a-quarter-of-startups-in-ycs-current-cohort-have-codebases-that-are-almost-entirely-ai-generated/)
[^3]: [GitClear: Code Quality in the AI Era](https://www.gitclear.com/coding_on_copilot_data_shows_ais_downward_pressure_on_code_quality)
[^4]: [Simon Willison: What is vibe coding?](https://simonwillison.net/2025/Mar/19/vibe-coding/)
[^5]: [Sandi Metz: The Wrong Abstraction](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction)
[^6]: SOLID: Five principles of object-oriented design—Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion
[^7]: DRY: Don't Repeat Yourself—a principle to avoid code duplication
[^8]: KISS: Keep It Simple, Stupid—a design principle favoring simplicity
