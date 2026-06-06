# AGENTS.md

## Project Overview

Personal technical blog built with **Hugo** + **PaperMod** theme (via Hugo Modules), deployed to GitHub Pages via GitHub Actions.

- **Stack**: Hugo v0.147+, PaperMod theme, Mermaid diagrams, KaTeX math
- **Deploy**: `main` branch push → GitHub Actions → `hugo --minify` → GitHub Pages
- **URL**: https://triumphpc.github.io/
- **Languages**: EN (default) + RU (bilingual)

## Bilingual Content Workflow

All new articles are published in **both languages simultaneously**:

1. Create `<slug>.ru.md` (Russian) and `<slug>.md` (English) in the same directory
2. Both files have `draft: true` while in progress
3. When ready: set `draft: false` in both, commit + push
4. CI deploys RU to `/ru/posts/<slug>/` and EN to `/posts/<slug>/`

Author prepares both versions independently — no automated translation step.

## File Naming Convention

| File | Language | URL |
|------|----------|-----|
| `<slug>.md` | English (default) | `/posts/<slug>/` |
| `<slug>.ru.md` | Russian | `/ru/posts/<slug>/` |

- Slug must be identical in both files (kebab-case)
- One file = one language (no bilingual single files)

## Front Matter Synchronization Rules

**Must be identical in both files:**
- `date` — same publication date
- `tags` — always Latin script (`"ai"`, `"agents"`, `"go"`, `"llm"`)
- `categories` — same list
- `series` — always English identifier (e.g., `"AI Agent Design Patterns"`)
- `cover` — same image
- `ShowToc` — same value

**Must be translated:**
- `title` — translated to respective language
- `summary` — translated to respective language
- `description` — translated (if present)

**Series rule**: `series` field always contains English name even in RU files. This enables cross-language series navigation in Hugo/PaperMod.

## Common Commands

```bash
# Local development with drafts
hugo server -D

# Production build
hugo --minify
```

## Re-translation Rule

If a RU article is significantly updated after publication, update the EN version accordingly. No automatic synchronization between files.
