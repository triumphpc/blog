---
title: "Шаблоны проектирования AI-агентов. Часть 2: Plan-and-Execute Pattern"
date: 2026-06-07
draft: false
tags: ["ai", "agents", "llm", "plan-execute", "eino", "go"]
categories: ["ai", "engineering"]
summary: "Plan-and-Execute — паттерн, который разделяет стратегическое планирование и тактическое выполнение. Разбираю архитектуру Planner+Executor+Reviser, варианты ReWOO и LLMCompiler с цифрами из оригинальных работ, защиту от prompt injection через control-flow integrity, и минимальный рабочий код на Go через Eino."
ShowToc: true
series: ["AI Agent Design Patterns"]
---

## 1. Введение

В [Части 1]({{< ref "ai-agent-design-patterns-1-react" >}}) я разобрал ReAct — паттерн, где агент чередует рассуждение и действие. Проблема: ReAct планирует на один шаг вперёд. Для задач, где нужен глобальный план — анализ инцидента, многошаговый debug, оркестрация сервисов — myopic-подход ломается. Plan-and-Execute решает эту проблему разделением планирования и выполнения.

## 2. Что такое Plan-and-Execute

### История

В мае 2023 года Wang et al. опубликовали [Plan-and-Solve Prompting](https://arxiv.org/abs/2305.04091) — zero-shot стратегию, которая сначала составляет план разбиения задачи на подзадачи, а затем выполняет их по порядку. Это была прямая реакция на три ошибки Zero-shot-CoT: пропущенные шаги, арифметические ошибки, семантические недопонимания.

В марте 2023 года Shen et al. представили [HuggingGPT](https://arxiv.org/abs/2303.17580) — LLM как контроллер, который планирует задачу, выбирает модели из Hugging Face и выполняет каждую подзадачу. Это первая реализация паттерна в production-масштабе: ChatGPT декомпозирует запрос, подбирает AI-модели по описаниям, выполняет и суммаризирует результат.

Итак. Паттерн оформился: отдельный **Planner** (стратег), отдельный **Executor** (исполнитель), опциональный **Reviser** (ревьюер). [LangChain](https://www.langchain.com/) — Python-фреймворк для построения LLM-приложений, включающий агентные примитивы, RAG-пайплайны и инструменты оркестрации — популяризировал этот паттерн как `Plan-and-Execute Agent` в 2024 году, но идея — из 2023-го.

### Аналогия с командой

Представьте: техлид (Planner) декомпозирует задачу на тикеты, разработчик (Executor) берёт тикет и делает, QA-инженер (Reviser) проверяет — и если найдёт проблему, тикет возвращается на доработку. ReAct — это разработчик-одиночка, который сам ставит себе задачу на один шаг, делает, проверяет, ставит следующую. Для простых задач — ок. Для сложных — нужен менеджер.

### Формальное определение

Plan-and-Execute работает в три фазы:

1. **Plan**: Planner генерирует список шагов `[S₁, S₂, ..., Sₙ]` для решения задачи.
2. **Execute**: Executor выполняет каждый шаг Sᵢ, вызывая инструменты и получая наблюдения.
3. **Revise**: Reviser оценивает прогресс и при необходимости модифицирует оставшийся план.

Ключевое отличие от ReAct: планирование **глобальное**, а не пошаговое. Planner видит задачу целиком и строит план на N шагов вперёд. Executor получает один шаг за раз — не весь план, не пользовательский ввод. Это создаёт естественную границу безопасности.

## 3. Какую задачу решает

### Проблема ReAct: myopic планирование

ReAct, как я показал в Части 1, планирует на один шаг. Каждый Thought — реакция на предыдущее Observation, а не на общую стратегию. Для задач с известной структурой (анализ логов → фильтрация → корреляция → диагноз) это неэффективно: агент «бродит» вместо того, чтобы следовать плану.

### Проблема ReAct: стоимость

Каждый шаг ReAct — полноценный вызов LLM с полным контекстом (все предыдущие Thoughts + Actions + Observations). При 10 шагах — 10 вызовов дорогой модели. ReWOO (Xu et al., 2023) показал, что это избыточно: до **5x** сокращение потребления токенов на HotpotQA при росте точности на **4.4%** ([Xu et al., 2023](https://arxiv.org/abs/2305.18323)).

### Проблема ReAct: нет явного перепланирования

ReAct не пересматривает план — его просто нет. Если шаг привёл в тупик, модель «имплицитно» корректируется через следующий Thought. Но это не перепланирование — это реактивное блуждание. Plan-and-Execute делает перепланирование **явным** через Reviser.

### Что обеспечивает Plan-and-Execute

| Проблема | Решение Plan-and-Execute |
|----------|--------------------------|
| Myopic планирование | Planner видит задачу целиком, строит N-шаговый план |
| Высокая стоимость | Executor может использовать дешёвую модель — ему нужен только один шаг |
| Нет перепланирования | Reviser явно оценивает прогресс и модифицирует план |
| Prompt injection | Executor не видит пользовательский ввод и полный план |
| Непрозрачность | План — читаемый артефакт, который можно аудировать |

## 4. Архитектура

### Plan → Execute → Revise

{{< mermaid align="center" >}}
sequenceDiagram
    participant U as User
    participant P as Planner
    participant E as Executor
    participant T as Tools
    participant R as Reviser

    U->>P: Task
    P->>P: Generate plan [S₁, S₂, ..., Sₙ]
    loop For each step
        P->>E: Step Sᵢ
        E->>T: Call tool
        T-->>E: Observation
        E-->>R: Step result
        R->>R: Evaluate progress
        alt Plan needs revision
            R->>P: Revised plan
        else Step OK
            R->>E: Next step Sᵢ₊₁
        end
    end
    R-->>U: Final answer
{{< /mermaid >}}

Planner получает задачу и генерирует план. Executor выполняет шаг за шагом, вызывая инструменты. Reviser проверяет результат каждого шага и решает: продолжать план, пересмотреть или завершить.

### Eino State Graph

В Eino паттерн реализуется через `compose.Graph` — направленный граф с тремя узлами-агентами:

{{< mermaid align="center" >}}
graph TB
    START((START)) --> PL[Planner<br/>BaseChatModel]
    PL --> EX[Executor<br/>ToolCallingChatModel<br/>+ ToolsNode]
    EX --> RV{Reviser<br/>BaseChatModel}
    RV -->|needs revision| PL
    RV -->|step OK, more steps| EX
    RV -->|all done| END((END))
{{< /mermaid >}}

State-структура хранит: сообщения (history), текущий план (steps), номер шага. Параметр `maxStep` ограничивает число итераций — защита от бесконечных циклов.

### Эволюция паттернов

{{< mermaid align="center" >}}
graph LR
    C["CoT<br/>Wei 2022"] --> R["ReAct<br/>Yao 2022"]
    R --> PS["Plan-and-Solve<br/>Wang 2023"]
    PS --> PA["Plan-and-Execute<br/>LangChain 2024"]
    PA --> RW["ReWOO<br/>Xu 2023"]
    PA --> LC["LLMCompiler<br/>Kim 2023"]
{{< /mermaid >}}

CoT (Chain-of-Thought, [Wei et al., 2022](https://arxiv.org/abs/2201.11903)) дал пошаговое рассуждение без действий. ReAct ([Yao et al., 2022](https://arxiv.org/abs/2210.03629)) добавил действия. Plan-and-Solve ([Wang et al., 2023](https://arxiv.org/abs/2305.04091)) ввёл явное планирование. Plan-and-Execute оформил это как архитектурный паттерн с отдельными агентами. ReWOO и LLMCompiler — оптимизации базового паттерна.

## 5. Варианты

### ReWOO: Reasoning Without Observation

ReWOO ([Xu et al., 2023](https://arxiv.org/abs/2305.18323)) убирает interleaved-вызовы LLM. Planner один раз генерирует план с переменными, Executor подставляет результаты — **без вызова LLM на каждом шаге**. Результат: **5x сокращение токенов** и **+4.4% точности** на HotpotQA. Дополнительный бонус: можно дистиллировать Planner с 175B (GPT-3.5) на 7B (LLaMA) — и это работает.

Как это выглядит на нашем сценарии анализа логов:

{{< mermaid align="center" >}}
graph LR
    subgraph "1. Planner (LLM, 1 вызов)"
        P["План:<br/>#E1 = query_logs('error spike')<br/>#E2 = filter_by_time(#E1, 'last hour')<br/>#E3 = correlate_deployments(#E1)<br/>#E4 = suggest_fix(#E2, #E3)"]
    end
    subgraph "2. Executor (без LLM)"
        E1["#E1 → query_logs<br/>→ результат в #E1"]
        E2["#E2 → filter_by_time(#E1)<br/>→ результат в #E2"]
        E3["#E3 → correlate(#E1)<br/>→ результат в #E3"]
        E4["#E4 → suggest_fix(#E2,#E3)<br/>→ итоговый ответ"]
    end
    P --> E1
    E1 --> E2
    E1 --> E3
    E2 --> E4
    E3 --> E4

    style P fill:#4a9eff,color:#fff
    style E1 fill:#2d8659,color:#fff
    style E2 fill:#2d8659,color:#fff
    style E3 fill:#2d8659,color:#fff
    style E4 fill:#2d8659,color:#fff
{{< /mermaid >}}

Ключевой инсайт: Planner использует LLM **один раз** для генерации плана. Дальше Executor просто вызывает инструменты и подставляет результаты в переменные — как в шаблонизаторе. LLM нужна только для следующего Planner-вызова, если Reviser решил перепланировать.

### LLMCompiler: параллельное выполнение

LLMCompiler ([Kim et al., 2023](https://arxiv.org/abs/2312.04511)) берёт идею классических компиляторов: Planner генерирует DAG (Directed Acyclic Graph, направленный ациклический граф) зависимостей между шагами, Task Fetching Unit диспетчеризирует независимые шаги параллельно, Executor выполняет их concurrently. Результат: **3.7x ускорение latency**, **6.7x экономия стоимости**, **~9% рост точности** по сравнению с ReAct.

На нашем сценарии это выглядит так:

{{< mermaid align="center" >}}
graph LR
    subgraph "1. Planner → DAG"
        P["План с зависимостями:<br/>#E1 = query_logs('error')<br/>#E2 = filter_by_time(#E1)<br/>#E3 = correlate_deployments(#E1)<br/>#E4 = query_metrics('cpu')<br/>#E5 = suggest_fix(#E2, #E3, #E4)"]
    end
    subgraph "2. Параллельное выполнение"
        E1["#E1 query_logs<br/>⏱ 200ms"]
        E2["#E2 filter_by_time<br/>⏱ 100ms"]
        E3["#E3 correlate<br/>⏱ 150ms"]
        E4["#E4 query_metrics<br/>⏱ 120ms"]
        E5["#E5 suggest_fix<br/>⏱ 300ms"]
    end
    P --> E1
    E1 --> E2
    E1 --> E3
    E1 --> E4
    E2 --> E5
    E3 --> E5
    E4 --> E5

    style P fill:#4a9eff,color:#fff
    style E1 fill:#2d8659,color:#fff
    style E2 fill:#e6a817,color:#333
    style E3 fill:#e6a817,color:#333
    style E4 fill:#e6a817,color:#333
    style E5 fill:#2d8659,color:#fff
{{< /mermaid >}}

Жёлтым выделены шаги, которые выполняются **параллельно**: после получения #E1 (query_logs) шаги #E2, #E3, #E4 запускаются одновременно — каждый зависит только от #E1, но не друг от друга. Итоговое время: 200ms + max(100, 150, 120) + 300ms = **650ms** вместо секвенциальных 870ms. На реальных задачах с десятками шагов выигрыш растёт до 3.7x.

### HuggingGPT: LLM как контроллер

HuggingGPT ([Shen et al., 2023](https://arxiv.org/abs/2303.17580)) — конкретная реализация паттерна: ChatGPT планирует задачу, выбирает AI-модели из Hugging Face по описаниям, выполняет каждую подзадачу выбранной моделью, суммаризирует результат. Валидация — через ручной контроль формата вывода. Не отдельный вариант, а иллюстрация применимости паттерна к реальным системам.

{{< mermaid align="center" >}}
graph LR
    U["Пользователь:<br/>Сгенерируй изображение<br/>и опиши его голосом"]
    P["ChatGPT<br/>(Planner)"]
    M1["Stable Diffusion<br/>(text-to-image)"]
    M2["Whisper<br/>(speech-to-text)"]
    M3["Bark<br/>(text-to-speech)"]
    S["ChatGPT<br/>(Summarizer)"]

    U --> P
    P -->|"1. image generation"| M1
    P -->|"2. describe image"| M2
    P -->|"3. voice output"| M3
    M1 --> S
    M2 --> S
    M3 --> S
    S --> U

    style P fill:#4a9eff,color:#fff
    style M1 fill:#9b59b6,color:#fff
    style M2 fill:#9b59b6,color:#fff
    style M3 fill:#9b59b6,color:#fff
    style S fill:#4a9eff,color:#fff
    style U fill:#555,color:#fff
{{< /mermaid >}}

Фиолетовым — специализированные AI-модели из Hugging Face. ChatGPT выступает как Planner: разбирает запрос, подбирает модели по описаниям их возможностей, передаёт результаты между ними и формирует финальный ответ. В отличие от ReWOO и LLMCompiler, Executor здесь — не LLM, а внешние модели. Паттерн тот же: планирование → выполнение → суммаризация.

Полный разбор ReWOO и LLMCompiler — в Части 4 этого цикла.

## 6. Безопасность: Control-Flow Integrity

И тут начинается самое интересное. Разделение Planner и Executor — не просто архитектурный изыск. Это защита от prompt injection (атака, при которой злоумышленник внедряет вредоносную инструкцию в данные, которые LLM обрабатывает, — например, в вывод инструмента или пользовательский контент).

Del Rosario et al. (2025) в [«Architecting Resilient LLM Agents»](https://arxiv.org/abs/2509.08646) формулируют принцип **control-flow integrity**: если Executor видит только один шаг плана (не весь план, не пользовательский ввод), то атакующий, контролирующий вывод инструмента, не может перенаправить весь workflow. Blast radius ограничен одним шагом.

Контраст с ReAct: в ReAct-агенте полный контекст (включая пользовательский ввод) доступен на каждом шаге. Если инструмент вернул вредоносный контент, модель может интерпретировать его как новую инструкцию — и изменить поведение. В Plan-and-Execute Executor получает изолированную задачу: «вызови инструмент X с параметрами Y».

Дополнительные меры защиты из Del Rosario 2025:

- **Principle of Least Privilege**: каждому шагу — минимальный набор инструментов
- **Task-scoped tool access**: Executor для шага «фильтрация логов» не должен иметь доступ к инструменту «удаление записей»
- **Sandboxed execution**: код, генерируемый Executor, выполняется в песочнице
- **HITL (Human-in-the-Loop)**: критические шаги требуют подтверждения человеком

Это не теоретические рекомендации. Del Rosario et al. приводят имплементационные чертежи для LangGraph, CrewAI и AutoGen — с рабочим кодом.

## 7. Практика: Plan-and-Execute на Eino, Go

### Сценарий: анализ production-инцидента

Дежурный инженер получает алерт: latency spike на сервисе orders. Нужно: запросить логи → отфильтровать по времени → скоррелировать с деплоями → запросить метрики → предложить фикс. Пять шагов, известная структура — идеальный кейс для Plan-and-Execute.

### Установка

```bash
go get github.com/cloudwego/eino@latest
go get github.com/cloudwego/eino-ext/components/model/openai@latest
```

### Минимальный Plan-and-Execute агент

```go
package main

import (
    "context"
    "fmt"

    "github.com/cloudwego/eino-ext/components/model/openai"
    "github.com/cloudwego/eino/components/tool"
    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/flow/agent/multiagent/planexecute"
    "github.com/cloudwego/eino/schema"
)

func main() {
    ctx := context.Background()

    plannerModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        Model: "gpt-4o",
    })
    executorModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        Model: "gpt-4o-mini",
    })
    reviserModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        Model: "gpt-4o",
    })

    agent, _ := planexecute.NewAgent(ctx, &planexecute.AgentConfig{
        Planner:   plannerModel,
        Reviser:   reviserModel,
        Executor:  executorModel,
        ToolsConfig: compose.ToolsNodeConfig{
            InvokableTools: []tool.InvokableTool{
                queryLogsTool(),
                filterByTimeTool(),
                correlateDeploymentsTool(),
                queryMetricsTool(),
                suggestFixTool(),
            },
        },
        MaxStep: 10,
    })

    msg, _ := agent.Generate(ctx, []*schema.Message{
        schema.UserMessage(
            "Latency spike на сервисе orders. " +
            "Найди root cause и предложи фикс.",
        ),
    })
    fmt.Println(msg.Content)
}
```

Обратите внимание: **Planner и Reviser используют `gpt-4o`** (стратегические решения), а **Executor — `gpt-4o-mini`** (вызов одного инструмента). Это экономия: дешёвая модель для тактической работы, дорогая — для стратегической. В ReAct такой оптимизации нет — каждый шаг требует полноценной модели.

Инструменты определяются через `utils.NewTool` — так же, как в Части 1 для ReAct. Полный пример с инструментами и промптами: [eino-examples/flow/agent/multiagent/plan_execute](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/multiagent/plan_execute).

### Трассировка выполнения

Раскройте блоки, чтобы увидеть промежуточные выходы агентов:

<details>
<summary><strong>Шаг 1: Planner — генерация плана</strong></summary>

```
Задача: Latency spike на сервисе orders. Найди root cause и предложи фикс.

План:
1. query_logs — запросить логи сервиса orders за последний час
2. filter_by_time — отфильтровать логи по временному окну spike (14:00-14:30)
3. correlate_deployments — проверить деплои за то же окно
4. query_metrics — запросить CPU/memory метрики сервиса orders
5. suggest_fix — на основе данных предложить исправление
```

</details>

<details>
<summary><strong>Шаг 2: Executor — query_logs</strong></summary>

```
Вызов: query_logs(service="orders", level="ERROR", limit=100)
Результат: 47 ERROR записей за последний час. Основные ошибки:
- "timeout connecting to payments service" (23 occurrences)
- "connection pool exhausted" (18 occurrences)
- "slow query: SELECT * FROM orders WHERE..." (6 occurrences)
```

</details>

<details>
<summary><strong>Шаг 3: Executor — filter_by_time + correlate_deployments</strong></summary>

```
Вызов: filter_by_time(logs=<step2_result>, start="14:00", end="14:30")
Результат: Spike начался в 14:12. До 14:12 — нормальный уровень ошибок.

Вызов: correlate_deployments(service="orders", start="14:00", end="14:30")
Результат: Деплой payments-service v2.3.1 в 14:10 (за 2 минуты до spike).
```

</details>

<details>
<summary><strong>Шаг 4: Executor — query_metrics</strong></summary>

```
Вызов: query_metrics(service="orders", metric="cpu,memory", start="14:00", end="14:30")
Результат: CPU orders — норма (45%). Memory — норма (62%).
Но: latency payments-service вырос с 50ms до 2000ms начиная с 14:10.
```

</details>

<details>
<summary><strong>Шаг 5: Reviser — валидация + suggest_fix</strong></summary>

```
Reviser: Root cause найден — деплой payments-service v2.3.1 в 14:10 вызвал
рост latency в payments, что привело к timeout в orders. План выполнен,
перепланирование не требуется.

Вызов: suggest_fix(root_cause="payments-service v2.3.1 latency regression",
                   affected_service="orders")
Результат:
1. Немедленно: откатить payments-service до v2.3.0
2. Краткосрочно: увеличить timeout и connection pool для orders→payments
3. Долгосрочно: добавить circuit breaker между orders и payments
```

</details>

### Ключевые параметры

- **`Planner`** — модель для генерации плана (интерфейс `BaseChatModel`).
- **`Reviser`** — модель для оценки прогресса и перепланирования (`BaseChatModel`).
- **`Executor`** — модель для выполнения шагов (`ToolCallingChatModel` + `ToolsNode`).
- **`MaxStep`** — лимит итераций графа (default 12). Защита от бесконечных циклов.
- **`ToolsConfig`** — инструменты, доступные Executor.

### Ссылки для углубления

- [Eino Plan-and-Execute Agent Manual](https://www.cloudwego.io/docs/eino/core_modules/eino_adk/agent_implementation/plan_execute/) — полное описание параметров
- [eino-examples/flow/agent/multiagent/plan_execute](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/multiagent/plan_execute) — полный рабочий пример
- [GitHub: cloudwego/eino](https://github.com/cloudwego/eino) — исходники

## 8. Сравнительные таблицы

### Архитектура: ReAct vs Plan-and-Execute

| Измерение | ReAct | Plan-and-Execute |
|-----------|-------|-------------------|
| Горизонт планирования | Myopic: 1 шаг | Global: N шагов |
| LLM-вызовы на шаг | Полная модель каждый шаг | Дешёвый Executor для шага, дорогой Planner один раз |
| Управление контекстом | Scratchpad: все Thoughts/Actions/Observations накапливаются | State: план + результаты шагов |
| Перепланирование | Имплицитное: через следующий Thought | Явное: Reviser модифицирует план |
| Восстановление после ошибок | Retry in-place: модель корректирует на следующем шаге | Re-plan from checkpoint: Reviser перестраивает оставшийся план |
| Архитектурная сложность | Один цикл (ChatModel → ToolsNode) | Три агента (Planner → Executor → Reviser) |
| Устойчивость к prompt injection | Низкая: полный контекст доступен на каждом шаге | Выше: Executor видит только один шаг ([Del Rosario et al., 2025](https://arxiv.org/abs/2509.08646)) |

### Производительность и стоимость

| Метрика | ReAct | Plan-and-Execute (naive) | ReWOO | LLMCompiler |
|---------|-------|--------------------------|-------|-------------|
| Потребление токенов (относительно ReAct) | 1x | ~1.2x (overhead от Planner/Reviser) | **0.2x** ([Xu 2023](https://arxiv.org/abs/2305.18323)) | ~0.5x (параллельное выполнение) |
| Число LLM-вызовов | N шагов × полная модель | 1 Planner + N × Executor + k × Reviser | 1 Planner + N × tool calls (без LLM) | 1 Planner + параллельные Executor |
| Latency | Секвенциальная: N × T_step | Секвенциальная: T_plan + N × T_exec | **Секвенциальная, но без LLM на шаг** | **Параллельная: ~T_plan + T_slowest_step** ([Kim 2023](https://arxiv.org/abs/2312.04511)) |
| Точность (HotpotQA) | Baseline | Сопоставимо | **+4.4%** ([Xu 2023](https://arxiv.org/abs/2305.18323)) | **+~9%** vs ReAct ([Kim 2023](https://arxiv.org/abs/2312.04511)) |
| Ускорение latency | 1x | ~1x | ~1x | **3.7x** ([Kim 2023](https://arxiv.org/abs/2312.04511)) |
| Экономия стоимости | 1x | ~0.8x (дешёвый Executor) | **~5x** ([Xu 2023](https://arxiv.org/abs/2305.18323)) | **6.7x** ([Kim 2023](https://arxiv.org/abs/2312.04511)) |

### Варианты Plan-and-Execute

| Вариант | Токены vs ReAct | Ускорение | Точность (delta) | Сложность | Ключевая инновация |
|---------|-----------------|-----------|-------------------|-----------|---------------------|
| **Naive** (LangChain-style) | ~1.2x | ~1x | ~0% | Низкая | Планирование + секвенциальное выполнение |
| **ReWOO** ([Xu 2023](https://arxiv.org/abs/2305.18323)) | **0.2x** (-5x) | ~1x | **+4.4%** | Средняя | Подстановка переменных, LLM только для Planner |
| **LLMCompiler** ([Kim 2023](https://arxiv.org/abs/2312.04511)) | ~0.5x | **3.7x** | **+~9%** | Высокая | DAG зависимостей, параллельное выполнение |

### Руководство по выбору паттерна

| Характеристика задачи | Рекомендуемый паттерн | Пример |
|----------------------|-----------------------|--------|
| Простая, один API-вызов с парсингом | **ReAct** | «Какая погода в Париже?» → один вызов weather API |
| Сложная, многошаговая, структура известна | **Plan-and-Execute** | Анализ production-инцидента: логи → фильтр → корреляция → диагноз |
| Параллелизуемые подзадачи | **LLMCompiler** | «Сравни цены на iPhone в 5 магазинах» → 5 параллельных вызовов |
| Чувствительность к стоимости, фиксированный набор инструментов | **ReWOO** | Регулярный мониторинг: тот же набор шагов, тот же набор инструментов |
| Требуется учёсть прошлые ошибки | **Reflexion** | Код-ревью: агент проверяет, находит баг, перепроверяет исправление |

## 9. Когда НЕ использовать

Microsoft Azure Architecture Center в [руководстве по оркестрации AI-агентов](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) формулирует принцип: **используй минимальный уровень сложности, который решает задачу**.

| Если ваша задача... | ...рассмотрите альтернативу |
|---------------------|----------------------------|
| Решается прямым вызовом модели | **Direct model call** — классификация, суммаризация |
| Нужен один-два инструмента, шаги неизвестны заранее | **ReAct** — как в Части 1 |
| Фиксированный workflow без перепланирования | **ReWOO** — дешевле, без Reviser-overhead |
| Требуется деревья гипотез | **Tree of Thoughts** ([Yao et al., 2023](https://arxiv.org/abs/2305.10601)) |
| Несколько доменов, разные security boundaries | **Multi-agent** — Part 5 этого цикла |

Plan-and-Execute — это средний уровень сложности между ReAct и multi-agent. Не прыгайте на него, если задача решается ReAct. Не оставайтесь на нём, если нужна координация нескольких агентов.

## 10. Что дальше

- **Часть 3: Reflexion Pattern** — ReAct + самооценка: агент учится на собственных ошибках через вербальное подкрепление.
- **Часть 4: ReWOO и LLMCompiler** — глубокий разбор двух оптимизаций Plan-and-Execute: variable substitution и parallel DAG execution.
- **Часть 5: Сравнение паттернов + Multi-Agent Orchestration** — когда какой паттерн выбирать, и как несколько агентов координируются для решения сложных задач.

Обсуждение приветствуется — комментарии на сайте или [GitHub Issues](https://github.com/triumphpc/blog/issues).

## 11. Ссылки

### Исследовательские статьи

1. **Wang et al., 2023 — Plan-and-Solve Prompting**: Zero-shot декомпозиция задачи на подзадачи. Решение проблемы пропущенных шагов Zero-shot-CoT. [arXiv:2305.04091](https://arxiv.org/abs/2305.04091)
2. **Shen et al., 2023 — HuggingGPT**: LLM как контроллер для оркестрации AI-моделей из Hugging Face. Первая production-реализация паттерна. [arXiv:2303.17580](https://arxiv.org/abs/2303.17580)
3. **Xu et al., 2023 — ReWOO**: Отделение рассуждения от наблюдений. Variable substitution, 5x token efficiency, +4.4% точности на HotpotQA. Дистилляция 175B→7B. [arXiv:2305.18323](https://arxiv.org/abs/2305.18323)
4. **Kim et al., 2023 — LLMCompiler**: Параллельное выполнение function calls через DAG. 3.7x latency speedup, 6.7x cost savings, +~9% точности. [arXiv:2312.04511](https://arxiv.org/abs/2312.04511)
5. **Del Rosario et al., 2025 — Plan-then-Execute Security**: Control-flow integrity как защита от prompt injection. Least privilege, task-scoped tool access, sandboxed execution. [arXiv:2509.08646](https://arxiv.org/abs/2509.08646)
6. **Yao et al., 2022 — ReAct**: Базовый паттерн Reasoning + Acting, от которого отталкивается Plan-and-Execute. [arXiv:2210.03629](https://arxiv.org/abs/2210.03629)
7. **Wei et al., 2022 — Chain-of-Thought**: Пошаговое рассуждение — предшественник всех агентных паттернов. [arXiv:2201.11903](https://arxiv.org/abs/2201.11903)

### Документация и примеры

- [Eino Plan-and-Execute Agent Manual](https://www.cloudwego.io/docs/eino/core_modules/eino_adk/agent_implementation/plan_execute/) — все параметры и конфигурация
- [Eino Open Source Announcement](https://www.cloudwego.io/docs/eino/overview/eino_open_source/) — обзор фреймворка
- [GitHub: cloudwego/eino](https://github.com/cloudwego/eino) — исходники
- [GitHub: eino-examples/flow/agent/multiagent/plan_execute](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/multiagent/plan_execute) — полный рабочий пример

### Архитектурные руководства

- [AI agent design patterns — Microsoft Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) — иерархия сложности, паттерны оркестрации, production-рекомендации

---

**Следующая статья:** [Шаблоны проектирования AI-агентов. Часть 3: Reflexion Pattern]({{< ref "ai-agent-design-patterns-3-reflexion" >}}) — ReAct + самооценка: агент учится на собственных ошибках через вербальное подкрепление и эпизодическую память.
