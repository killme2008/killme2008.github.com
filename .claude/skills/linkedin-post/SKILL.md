---
name: linkedin-post
description: Write a LinkedIn post for one of Dennis's blog articles, in his proven voice. Use when the user says "write a LinkedIn post for <article>", "do a LinkedIn post like the last one", references the Datadog/observability post style, or asks to promote a published blog post on LinkedIn. Includes image suggestions.
---

# Write a LinkedIn post in Dennis's voice

Turn a published blog article into a LinkedIn post that reads like Dennis wrote it, not like marketing. Two posts have worked well and are the gold references — match their structure and voice, don't invent a new one.

Output is the post text plus image suggestions. End by running the post through the **humanizer** skill's checklist (see "Final pass").

## The formula (reverse-engineered from the posts that landed)

Follow this arc. Keep paragraphs short, rhythm varied, language plain.

1. **Concrete hook + one fixation.** Open on a specific, factual scene (a dated event, a phrase everyone repeats). Then narrow to the *single word or idea you got stuck on*, set in **bold Unicode** (Mathematical Bold, the same glyphs as the examples: 𝐒𝐭𝐨𝐜𝐡𝐚𝐬𝐭𝐢𝐜, 𝐍𝐨 𝐨𝐧𝐞 𝐝𝐞𝐬𝐢𝐠𝐧𝐞𝐝 𝐢𝐭). Patterns that work: *"Most people read it for X. I got stuck on one word: 𝐘."* / *"The honest answer: 𝐘."*
2. **Plain-prose framing.** Set up the old assumption or the history. "For 20 years..." / "No committee ever decided...". This is the setup the punch lands against.
3. **The break or the non-obvious fact.** What changed, or the thing nobody noticed. Punchy, factual.
4. **The reframe.** State the subject's bet (or the twist) — often a *second* bold-Unicode phrase (𝐅𝐫𝐨𝐦 𝐌𝐨𝐧𝐢𝐭𝐨𝐫𝐢𝐧𝐠 𝐭𝐨 𝐎𝐛𝐬𝐞𝐫𝐯𝐚𝐛𝐢𝐥𝐢𝐭𝐲 𝐭𝐨 𝐀𝐮𝐭𝐨𝐧𝐨𝐦𝐲 / 𝐨𝐩𝐩𝐨𝐬𝐢𝐭𝐞). Cap it at **two bold phrases per post** — the fixation and the bet/twist.
5. **The pivot line:** a short standalone line, *"Here's the part that stuck with me."* / *"That's the part that stuck with me."* This introduces the share-worthy nugget.
6. **Honest caveat.** Name the weakness. *"Some of it is still slideware, and I poke at that in the post."* / *"This is Part 1 — I haven't gotten to X yet."* No overselling.
7. **The universal turn.** Make it bigger than the company/topic. *"It isn't really Datadog's question. Anyone building infra runs into it sooner or later."* / *"Worth remembering the next time someone tells you the current setup is just 'how it's done.'"*
8. **`Full read: <url>`** — `https://blog.fnil.net/<slug>/`. (Dennis runs it through LinkedIn's own shortener.)
9. **Five topic hashtags**, relevant to the piece (e.g. `#Observability #SRE #Monitoring #DistributedTracing #DevOps`). Always include `#Observability` if it fits.

## Voice rules

- First person, with opinions and mixed feelings. "I got stuck on", "the part that stuck with me", "I poke at that".
- Plain and direct. No corporate buzzwords, no "exciting times", no slogan endings.
- Straight quotes `"` only, never curly. No emojis. The only decoration is the 1–2 bold-Unicode emphases.
- Intellectual honesty is the differentiator — the caveat (step 6) is not optional. It's what makes readers trust the take.
- The closing must generalize: the reader should feel "this is my problem too," not "nice writeup of someone else's slides."

## Final pass (humanizer)

Run the draft against the **humanizer** skill before delivering. Watch especially for:

- **Rule-of-three overload.** Some triads are fine (especially if the topic *is* three of something), but don't stack three separate triads in a row.
- **Duplicate phrasings.** Don't use "stuck with me" and "stays with me" in the same post; pick one.
- **Self-labeled takeaways.** Cut "The takeaway that stays with me:" — just state the takeaway.
- **Round-number / corporate softeners.** "nearly a decade", "any incentive to" → tighten.

Keep the bold-Unicode emphasis through the humanizer pass — it's Dennis's deliberate LinkedIn voice, not an AI tell, even though humanizer flags boldface.

## Image suggestions (always include)

- **Lead with the blog cover as a native upload, not a link preview.** LinkedIn throttles posts with external links; native images get more reach, and the cover already carries the title + hook. Put the link in the body (proven to work) or in the first comment if throttling is a concern.
- **For more engagement, a 2-image carousel:** image 1 = cover (the hook); image 2 = a "reveal" image (a diagram from the post) with a one-line caption that detonates the twist, e.g. *"The man who drew this in 2017 proposed scrapping it in 2018. Nobody noticed."* The payoff lands when the reader swipes.
- **Don't** make a low-res or academic diagram the single lead image — no title, no hook, weak cold open.

## Gold reference posts

Match the cadence of these. They are the standard.

**Datadog post (the one that drew a lot of attention):**
> Datadog dropped a 200-page Investor Day deck in February. Most people read it for the revenue curves. I got stuck on one word: 𝐒𝐭𝐨𝐜𝐡𝐚𝐬𝐭𝐢𝐜.
>
> For 20 years, observability fought complexity. [...] But the system underneath stayed deterministic: same input, same output. [...]
>
> AI breaks the assumption. Same prompt, different answer. [...]
>
> Datadog's answer is to drag the whole field 𝐅𝐫𝐨𝐦 𝐌𝐨𝐧𝐢𝐭𝐨𝐫𝐢𝐧𝐠 𝐭𝐨 𝐎𝐛𝐬𝐞𝐫𝐯𝐚𝐛𝐢𝐥𝐢𝐭𝐲 𝐭𝐨 𝐀𝐮𝐭𝐨𝐧𝐨𝐦𝐲 [...]. Big bet. Some of it is still slideware, and I poke at that in the post. [...] That's the part that stuck with me. It isn't really Datadog's question. Anyone building infra runs into it sooner or later.
>
> Full read: <url>
>
> #Observability #AI #AIOps #SRE #Datadog

**Three Pillars history post:**
> "Metrics, logging, tracing." Anyone in observability can recite the three pillars like they're a law of nature. I went looking for where the phrase actually came from. The honest answer: 𝐍𝐨 𝐨𝐧𝐞 𝐝𝐞𝐬𝐢𝐠𝐧𝐞𝐝 𝐢𝐭.
>
> [history: built separately, three problems...] Then commerce poured concrete over the crack. [...]
>
> Here's the part that stuck with me.
>
> [Bourgon drew the Venn diagram, then 18 months later wrote the 𝐨𝐩𝐩𝐨𝐬𝐢𝐭𝐞...] The industry grabbed the first idea and quietly ignored the second.
>
> This is Part 1, on how it split apart. [caveat] [...] Worth remembering the next time someone tells you the current setup is just "how it's done."
>
> Full read: <url>
>
> #Observability #SRE #Monitoring #DistributedTracing #DevOps
