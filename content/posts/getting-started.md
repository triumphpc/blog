---
title: "Getting Started with Hugo + PaperMod"
date: 2026-06-01
draft: false
tags: ["hugo", "blog", "papermod"]
summary: "How I set up this blog with Hugo, PaperMod theme, Mermaid diagrams, and KaTeX math — all automated via GitHub Actions."
ShowToc: true
---

## Why Hugo?

After evaluating custom SSG options and Go-based generators, I chose Hugo for its speed, mature ecosystem, and PaperMod theme that gives me everything out of the box.

## Setup Highlights

- **Hugo Modules** instead of git submodules — cleaner dependency management
- **PaperMod** dark mode with code copy buttons, TOC, and Fuse.js search
- **Mermaid** diagrams via shortcodes + CDN
- **KaTeX** math rendering via shortcodes + CDN
- **GitHub Actions** for CI/CD — build and deploy on push

## Mermaid Example

```mermaid
graph LR
    A[Markdown] --> B[Hugo Build]
    B --> C[GitHub Actions]
    C --> D[Deploy]
```

Using the shortcode:

```
{{</* mermaid */>}}
graph LR
    A[Markdown] --> B[Hugo Build]
    B --> C[GitHub Actions]
    C --> D[Deploy]
{{</* /mermaid */>}}
```

## KaTeX Example

Inline math: `\(E = mc^2\)`

Block math:

$$
\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}
$$

## What's Next

- More posts about Go concurrency patterns
- LLM integration tutorials
- AI Agent architecture deep dives

Stay tuned!
