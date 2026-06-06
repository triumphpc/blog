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

## Article Writing Rules

1. **Пишем по существу.** Никакой воды, долгих вступлений и философствований. Инженер открывает статью — хочет увидеть факты, архитектуру, код, цифры. Введение — 2-3 предложения: о чём статья и зачем она нужна. Всё остальное — технический контент.

2. **Пишем от первого лица.** «Я покажу», «я реализовал», «разберём» — а не «мы рассмотрим», «статья описывает», «читатель узнает».

3. **Цифры с ссылками.** Любой численный результат — с inline-ссылкой на источник. Не «исследование показало улучшение», а «+34% success rate ([Yao et al., 2022](https://arxiv.org/abs/2210.03629))».

4. **Аудория — senior-инженеры.** Не объяснять основы: что такое LLM, tool calling, prompt engineering. Допустимо напоминание в 1 предложении со ссылкой.

5. **Конкретные примеры.** Не абстрактные «task» / «tool» / «observation», а «погода в Париже», «президент Непала», «Food Recommender demo».

6. **Схемы — только Mermaid.** Любая схема,流程, архитектура, последовательность — через `{{</* mermaid */>}}` shortcode. Никаких текстовых псевдосхем вроде `A → B → C`.

7. **Аббревиатуры — с расшифровкой и ссылкой.** При первом упоминании каждой аббревиатуры — полное название и inline-ссылка на источник. Пример: `CoT (Chain-of-Thought, [Wei et al., 2022](https://arxiv.org/abs/2201.11903))`. Дальше в тексте — только аббревиатура.

## Re-translation Rule

If a RU article is significantly updated after publication, update the EN version accordingly. No automatic synchronization between files.
