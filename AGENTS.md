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

0. **Технически грамотные инженерные исследовательские статьи.** Каждая статья — это исследовательская работа, а не пост для хабра. Техническая грамотность: корректная терминология, понимание механизмов «под капотом», инженерные практики (ADR, trade-off analysis, benchmarking, failure modes). Глубокое разбирательство проблемы: не «обзор 5 инструментов», а «почему мы выбрали X — какие варианты рассматривали, какие отбросили и почему, где подводные камни». Обязательные ссылки на научные статьи и исследования: arxiv, ACM, IEEE, блоги исследовательских команд — любой нетривиальный факт или число должен иметь источник. Стиль изложения — человеческий и понятный: пишем как инженер объясняет коллеге, не как академик пишет论文. Сложное — просто, но не упрощённо.

1. **R&D-формат, до 100K символов.** Каждая статья — это research & deep dive, а не обзорный пост. Без искусственного ограничения снизу: тема определяет объём, а не наоборот. Но и без бесконечного раздувания — жёсткий потолок 100 000 символов (без учёта front matter). Если материал не дотягивает до глубины — значит тема не раскрыта: не хватает технических решений, сравнения подходов, архитектурных диаграмм или анализа рисков. Лучше углубить один аспект, чем поверхностно пройти по десяти.

1. **Пишем по существу.** Никакой воды, долгих вступлений и философствований. Инженер открывает статью — хочет увидеть факты, архитектуру, код, цифры. Введение — 2-3 предложения: о чём статья и зачем она нужна. Всё остальное — технический контент.

2. **Пишем от первого лица.** «Я покажу», «я реализовал», «разберём» — а не «мы рассмотрим», «статья описывает», «читатель узнает».

3. **Цифры с ссылками.** Любой численный результат — с inline-ссылкой на источник. Не «исследование показало улучшение», а «+34% success rate ([Yao et al., 2022](https://arxiv.org/abs/2210.03629))».

4. **Аудория — senior-инженеры.** Не объяснять основы: что такое LLM, tool calling, prompt engineering. Допустимо напоминание в 1 предложении со ссылкой.

5. **Конкретные примеры.** Не абстрактные «task» / «tool» / «observation», а «погода в Париже», «президент Непала», «Food Recommender demo».

6. **Схемы — только Mermaid.** Любая схема,流程, архитектура, последовательность — через `{{</* mermaid */>}}` shortcode. Никаких текстовых псевдосхем вроде `A → B → C`.

7. **Аббревиатуры и технические термины — с расшифровкой при первом упоминании.** При первом упоминании каждой аббревиатуры — полное название и inline-ссылка на источник. Пример: `CoT (Chain-of-Thought, [Wei et al., 2022](https://arxiv.org/abs/2201.11903))`. Дальше в тексте — только аббревиатура. То же правило для нетривиальных технических терминов — при первом появлении дать краткое определение в 1-2 предложения. Пример: `prompt injection (атака, при которой злоумышленник внедряет вредоносную инструкцию в данные, которые LLM обрабатывает, — например, в вывод инструмента или пользовательский контент)`. Аудитория — senior-инженеры, но не все работают в security/NLP; термин без объяснения при первом появлении = потерянный читатель.

8. **Разговорный тон.** Пишем так, будто объясняем коллеге за кофе — живо, с интригой, но без клоунады. Используем приёмы: **риторический вопрос** (не «Рассмотрим реализацию мьютекса», а «Зачем нужен мьютекс? Ведь генерация уидов не требует глобального состояния»), **одноабзацная пауза** (короткий абзац-перебивка, который создаёт ритм: «Так вот.» или «И тут начинается самое интересное.»), **интрига перед объяснением** (сначала странность, потом разгадка: не «В пакете есть мьютекс для монотонности UUIDv7», а «Если вы посмотрите исходники, то можете заметить кое-что странное — мьютекс v7mu, объявленный на уровне пакета»), **восклицание** (умеренно: «Оказывается, команда Go хотела гарантировать строго возрастающие UUID!»). Языковые конвенции: EN — professional conversational (сдержаннее, без лишних восклицаний), RU — свободнее, как разговор с коллегой. **Разговорный тон НЕ распространяется на**: код, конфигурацию, технические определения, таблицы — там строго и точно. **Это НЕ**: сленг, мемы, стендап. Тон коллеги в техническом обсуждении, а не комика на сцене.

9. **Ссылки на рабочие задачи.** При упоминании реальных задач с работы — живой, неформальный тон: «а мы с командой как раз решаем это в проекте X» или «у нас на работе — изучаем, прототипируем, и вот к чему пришли». JIRA-ключи указываем как есть (VKTAI-1356), но без прямых ссылок (доступ только из корпсети). Упоминание рабочей задачи — нарративный приём: показывает, что материал из реальной практики, а не из головы. **Никакой конкретики**: ни бизнес-логики, ни внутренних API, ни названий сервисов, ни данных, ни деталей реализации. Только абстрактные архитектурные решения на уровне паттернов — «у нас слоистая память», «агент собирает контекст на лету» — но никогда «у нас сервис X дергает ручку /api/v2/... из сервиса Y». NDA — это не про «не говори секретное», это про «не говори конкретное».

## Re-translation Rule

If a RU article is significantly updated after publication, update the EN version accordingly. No automatic synchronization between files.
