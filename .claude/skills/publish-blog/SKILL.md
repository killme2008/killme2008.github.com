---
name: publish-blog
description: Publish an English blog post to this Zola site from a greptime-content project draft. Use when the user says "publish <project-name>", gives a project slug like "2026-05-28-observability-unification-history-1", or asks to turn a content-project draft into a blog post and push it. This blog is English-only.
---

# Publish a blog post from a content project

Turn the final English draft of a content project into a Zola blog post in this repo, wire up images, build, and push. This site (`blog.fnil.net`) publishes **English only** — never publish a Chinese draft.

The user invokes this by naming a project, e.g. `publish 2026-05-28-observability-unification-history-1`. The project name is the directory under `~/Documents/greptime-content/projects/`.

## Steps

### 1. Locate the project and final English draft

- Project dir: `/Users/dennis/Documents/greptime-content/projects/<project-name>/`
- Final draft: `03-draft-en-vN.md` with the **highest** N (e.g. `03-draft-en-v5.md` beats `03-draft-en.md` and `-v4`). List the dir and pick the max version — do not assume.
- Read `meta.yaml` for `date`/`created`, `authors`, `tags`. `tags` is often empty there; if so, derive 3–4 tags from the topic.

### 2. Decide the slug and metadata

- **Slug**: a clean kebab-case name, NOT the dated project folder name. E.g. project `2026-05-28-observability-unification-history-1` → slug `observability-three-pillars-history`. Base it on the title.
- **Date**: use today's date (publish date), unless the user says otherwise.
- **Description**: the draft usually opens with a `> blockquote` summary — reuse it (trimmed) as `description`.
- **Tags**: Title Case (`"Observability"`, `"Open Source"`), not lowercase/kebab.

### 3. Convert and place images

The user typically provides a cover and optional illustrations (often by image paths in `~/Downloads/`).

- Convert to webp with `cwebp` into `static/images/`:
  ```bash
  cwebp -q 88 "<cover.png>" -o static/images/<slug>-cover.webp
  cwebp -q 90 "<illustration.png>" -o static/images/<slug>-<name>.webp
  ```
- Cover goes at the very top of the post: `![<title>](/images/<slug>-cover.webp)`
- **Illustrations that come from a source MUST be attributed.** Put an italic caption line right below the image:
  ```markdown
  ![alt](/images/<slug>-foo.webp)

  *Source: Author, [Title](url) (year).*
  ```
  Trace the real origin (e.g. a Venn diagram from a referenced blog post → cite that post, matching the references list).

### 4. Create the post

Path: `content/<slug>.md` — **`content/`, NOT `content/blog/`.** (The repo CLAUDE.md says `content/blog/` but actual posts live in `content/`; trust the file structure.)

Front matter (match existing posts exactly):
```toml
+++
title = "..."        # keep "(Part 1 of 2)" etc. if present
date = YYYY-MM-DD
description = "..."

[taxonomies]
tags = ["Tag1", "Tag2"]   # Title Case

[extra]
toc = true
+++
```

Body: drop the leading `# Title` line from the draft. Keep the rest verbatim (the draft is already final/polished — do not rewrite content unless asked).

### 5. Fix references rendering (common gotcha)

Drafts often write a `## References` list using footnote-**definition** syntax (`[^1]: ...`) with **no inline `[^1]` markers** in the body. pulldown-cmark (Zola's engine) drops unreferenced footnote definitions, so the section renders **empty**. Convert them to a plain numbered list:

```markdown
## References

1. First reference...
2. [Linked reference](url): ...
```

(Only do this when the body has no inline `[^n]` markers. If it does, leave footnotes as-is — Zola renders them at page bottom.)

### 6. Build and verify

```bash
zola build
```
Must report `0 orphan`. Optionally confirm the references list rendered (e.g. grep the built HTML in `public/<slug>/index.html` for `<li>` / a known reference string).

### 7. Commit and push

This repo deploys via GitHub Actions on push to `master`, so committing to `master` and pushing is the intended flow.

```bash
git add content/<slug>.md static/images/<slug>-*.webp
git commit -m "feat: new blog - <short title>"
git push origin master
```

**Never** add `Co-Authored-By` or any "Generated with Claude" footer to the commit message.

## Conventions reference

- Posts: `content/<slug>.md` (flat, not `content/blog/`)
- Images: `static/images/<slug>-*.webp`, converted with `cwebp` (q 88–90)
- Tags: Title Case
- `[extra] toc = true` is standard
- English only; no AI signature in commits
- Don't use the `full_width_image` shortcode for screenshots — plain markdown `![]()` looks better
