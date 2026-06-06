---
title: "Getting Started with Hugo + PaperMod"
date: 2026-06-05
draft: false
tags: ["hugo", "blog", "papermod"]
categories: ["engineering"]
summary: "How this blog was born — Hugo, PaperMod theme, Mermaid diagrams, KaTeX math, custom dark styling, and fully automated CI/CD via GitHub Actions."
ShowToc: true
cover:
  image: "/images/hugo-papermod-cover.svg"
  alt: "Hugo + PaperMod blog setup"
  caption: "From zero to deployed blog in one session"
  hidden: false
  hiddenInList: false
  hiddenInSingle: false
---

## Why Hugo?

After evaluating custom SSG options and Go-based generators, I chose Hugo for three reasons:

| Criteria | Hugo | Custom SSG | Likho |
|----------|------|-----------|-------|
| Build speed | ~40ms | N/A | ~200ms |
| Theme ecosystem | 400+ | None | Minimal |
| Markdown features | Full | DIY | Basic |
| Maintenance | Community | You | You |

Hugo gives me **speed**, a **mature ecosystem**, and the PaperMod theme that provides everything I need out of the box.

## Architecture Overview

The blog follows a simple pipeline — write Markdown, push to GitHub, and everything else is automated:

{{< mermaid align="center" >}}
graph LR
    A["✏️ Markdown"] --> B["🔨 Hugo Build"]
    B --> C["⏱️ GitHub Actions"]
    C --> D["🚀 GitHub Pages"]
    
    style A fill:#24283b,stroke:#7aa2f7,color:#c0caf5
    style B fill:#24283b,stroke:#bb9af7,color:#c0caf5
    style C fill:#24283b,stroke:#7dcfff,color:#c0caf5
    style D fill:#24283b,stroke:#9ece6a,color:#c0caf5
{{< /mermaid >}}

## Setup Highlights

### Hugo Modules over Git Submodules

```bash
hugo mod init github.com/triumumphc/blog
```

In `hugo.yaml`:

```yaml
module:
  imports:
    - path: github.com/adityatelange/hugo-PaperMod
```

No `git submodule` headaches — just Go modules, versioned and reproducible.

### PaperMod Dark Mode

Dark theme by default, with a toggle for light mode. The color palette is inspired by [Tokyo Night](https://github.com/enkia/tokyo-night-vscode-theme):

{{< mermaid align="center" >}}
pie title Color Palette Distribution
    "Primary (#7aa2f7)" : 35
    "Accent (#bb9af7)" : 25
    "Success (#9ece6a)" : 20
    "Warning (#e0af68)" : 10
    "Error (#f7768e)" : 10
{{< /mermaid >}}

### Code Copy Buttons

Every code block gets a copy button automatically. Go code looks like this:

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", handler)

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
    }

    log.Fatal(srv.ListenAndServe())
}
```

### Mermaid Diagrams

Diagrams are rendered client-side via CDN, only loaded when the `mermaid` shortcode is used:

{{< mermaid align="center" >}}
sequenceDiagram
    participant W as Writer
    participant G as Git
    participant CI as GitHub Actions
    participant P as GitHub Pages
    
    W->>G: git push
    G->>CI: webhook trigger
    CI->>CI: hugo --minify
    CI->>P: deploy artifact
    P-->>W: site live ✅
{{< /mermaid >}}

C4 architecture style:

{{< mermaid align="center" >}}
graph TB
    subgraph "Authoring"
        MD[Markdown Files]
        IMG[Images & Assets]
    end
    
    subgraph "Build Pipeline"
        HUGO[Hugo v0.162]
        CSS[Custom CSS]
        SC[Shortcodes]
    end
    
    subgraph "Hosting"
        GHA[GitHub Actions]
        GHP[GitHub Pages]
    end
    
    MD --> HUGO
    IMG --> HUGO
    CSS --> HUGO
    SC --> HUGO
    HUGO --> GHA
    GHA --> GHP
{{< /mermaid >}}

## Math with KaTeX

KaTeX renders math client-side, loaded only when the `katex` shortcode is present.

### Euler's Identity

The most beautiful equation in mathematics:

$$
e^{i\pi} + 1 = 0
$$

### Gaussian Integral

$$
\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}
$$

### Normal Distribution

The probability density function of a normal distribution:

$$
f(x) = \frac{1}{\sigma\sqrt{2\pi}} e^{-\frac{1}{2}\left(\frac{x-\mu}{\sigma}\right)^2}
$$

Inline math works too: the mean is $\mu$ and standard deviation is $\sigma$.

### Matrix Operations

$$
A = \begin{pmatrix} a_{11} & a_{12} \\ a_{21} & a_{22} \end{pmatrix}, \quad \det(A) = a_{11}a_{22} - a_{12}a_{21}
$$

## Shortcodes Reference

| Shortcode | Purpose | Example |
|-----------|---------|---------|
| `{{</* mermaid */>}}` | Mermaid diagrams | Flowcharts, sequences, C4 |
| `$$ ... $$` | Display math | Centered equations |
| `$ ... $` | Inline math | $\mu$ renders as μ |

## Custom Styling

The dark theme extends PaperMod with:

- **Code blocks** — custom background + border with rounded corners
- **Links** — subtle underline on hover
- **Blockquotes** — accent-colored left border with surface background
- **Tables** — bordered with surface header row

## CI/CD Pipeline

GitHub Actions workflow handles everything:

```yaml
name: Deploy Hugo Blog
on:
  push:
    branches: [main]
```

The pipeline:
1. Installs Hugo Extended
2. Checks out the repo
3. Caches Hugo Modules
4. Builds with `hugo --minify`
5. Deploys to GitHub Pages

## What's Next

- **Go concurrency patterns** — goroutines, channels, errgroup
- **LLM integration** — RAG pipelines, prompt engineering
- **AI Agent architecture** — agentic orchestration, tool use

Stay tuned!
