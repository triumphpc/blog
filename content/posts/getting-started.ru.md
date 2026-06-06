---
title: "Начало работы с Hugo + PaperMod"
date: 2026-06-05
draft: false
tags: ["hugo", "blog", "papermod"]
categories: ["engineering"]
summary: "Как родился этот блог — Hugo, тема PaperMod, диаграммы Mermaid, математика KaTeX, кастомная тёмная тема и полностью автоматизированный CI/CD через GitHub Actions."
ShowToc: true
cover:
  image: "/images/hugo-papermod-cover.svg"
  alt: "Hugo + PaperMod blog setup"
  caption: "From zero to deployed blog in one session"
  hidden: false
  hiddenInList: false
  hiddenInSingle: false
---

## Почему Hugo?

После оценки кастомных SSG-вариантов и Go-генераторов, я выбрал Hugo по трём причинам:

| Критерий | Hugo | Кастомный SSG | Likho |
|----------|------|---------------|-------|
| Скорость сборки | ~40ms | Н/Д | ~200ms |
| Экосистема тем | 400+ | Нет | Минимальная |
| Возможности Markdown | Полные | DIY | Базовые |
| Поддержка | Сообщество | Вы | Вы |

Hugo даёт **скорость**, **зрелую экосистему** и тему PaperMod, которая предоставляет всё необходимое из коробки.

## Обзор архитектуры

Блог работает по простому конвейеру — пишешь Markdown, пушешь в GitHub, и всё остальное автоматизировано:

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

## Основные моменты настройки

### Hugo Modules вместо Git Submodules

```bash
hugo mod init github.com/triumumphc/blog
```

В `hugo.yaml`:

```yaml
module:
  imports:
    - path: github.com/adityatelange/hugo-PaperMod
```

Никаких проблем с `git submodule` — только Go-модули, версионированные и воспроизводимые.

### Тёмная тема PaperMod

Тёмная тема по умолчанию с переключателем на светлую. Цветовая палитра вдохновлена [Tokyo Night](https://github.com/enkia/tokyo-night-vscode-theme):

{{< mermaid align="center" >}}
pie title Color Palette Distribution
    "Primary (#7aa2f7)" : 35
    "Accent (#bb9af7)" : 25
    "Success (#9ece6a)" : 20
    "Warning (#e0af68)" : 10
    "Error (#f7768e)" : 10
{{< /mermaid >}}

### Кнопки копирования кода

Каждый блок кода автоматически получает кнопку копирования. Go-код выглядит так:

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

### Диаграммы Mermaid

Диаграммы рендерятся на клиенте через CDN, загружаются только при использовании шорткода `mermaid`:

{{< mermaid align="center" >}}
sequenceDiagram
    participant W as Автор
    participant G as Git
    participant CI as GitHub Actions
    participant P as GitHub Pages
    
    W->>G: git push
    G->>CI: webhook trigger
    CI->>CI: hugo --minify
    CI->>P: deploy artifact
    P-->>W: сайт доступен ✅
{{< /mermaid >}}

Архитектурный стиль C4:

{{< mermaid align="center" >}}
graph TB
    subgraph "Написание"
        MD[Markdown-файлы]
        IMG[Изображения и ассеты]
    end
    
    subgraph "Сборочный конвейер"
        HUGO[Hugo v0.162]
        CSS[Кастомный CSS]
        SC[Шорткоды]
    end
    
    subgraph "Хостинг"
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

## Математика с KaTeX

KaTeX рендерит математику на клиенте, загружается только при наличии шорткода `katex`.

### Тождество Эйлера

Самое красивое уравнение в математике:

$$
e^{i\pi} + 1 = 0
$$

### Интеграл Гаусса

$$
\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}
$$

### Нормальное распределение

Функция плотности вероятности нормального распределения:

$$
f(x) = \frac{1}{\sigma\sqrt{2\pi}} e^{-\frac{1}{2}\left(\frac{x-\mu}{\sigma}\right)^2}
$$

Строчная математика тоже работает: среднее — $\mu$, стандартное отклонение — $\sigma$.

### Операции с матрицами

$$
A = \begin{pmatrix} a_{11} & a_{12} \\ a_{21} & a_{22} \end{pmatrix}, \quad \det(A) = a_{11}a_{22} - a_{12}a_{21}
$$

## Справочник шорткодов

| Шорткод | Назначение | Пример |
|---------|-----------|--------|
| `{{</* mermaid */>}}` | Диаграммы Mermaid | Блок-схемы, последовательности, C4 |
| `$$ ... $$` | Блочная математика | Центрированные уравнения |
| `$ ... $` | Строчная математика | $\mu$ отображается как μ |

## Кастомные стили

Тёмная тема расширяет PaperMod следующими элементами:

- **Блоки кода** — кастомный фон + рамка с закруглёнными углами
- **Ссылки** — тонкое подчёркивание при наведении
- **Цитаты** — акцентная левая рамка с фоном surface
- **Таблицы** — с рамками и строкой заголовка surface

## CI/CD-конвейер

GitHub Actions обрабатывает всё автоматически:

```yaml
name: Deploy Hugo Blog
on:
  push:
    branches: [main]
```

Конвейер:
1. Устанавливает Hugo Extended
2. Клонирует репозиторий
3. Кэширует Hugo Modules
4. Собирает с `hugo --minify`
5. Деплоит на GitHub Pages

## Что дальше

- **Паттерны конкурентности Go** — горутины, каналы, errgroup
- **Интеграция с LLM** — RAG-пайплайны, промпт-инжиниринг
- **Архитектура AI-агентов** — агентная оркестрация, использование инструментов

Следите за обновлениями!
