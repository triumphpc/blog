---
title: "Шаблоны проектирования AI-агентов. Часть 1: ReAct Pattern"
date: 2026-06-06
draft: false
tags: ["ai", "agents", "llm", "react-pattern", "eino", "go"]
categories: ["ai", "engineering"]
summary: "ReAct (Reasoning + Acting) — фундаментальный паттерн проектирования AI-агентов, объединяющий рассуждение и действие в едином цикле. Разбираем архитектуру, цифры из Yao et al. 2022, обзор фреймворков и минимальный рабочий код на Go через Eino."
ShowToc: true
series: ["AI Agent Design Patterns"]
---

## 1. Введение

Я начинаю цикл о шаблонах проектирования AI-агентов. Цель — референс с оригинальными research papers, рабочим кодом на Go и честным анализом ограничений. Каждое утверждение — с отсылкой к первоисточнику, никаких пересказов. Аудитория — опытные инженеры, которым не нужны объяснения основ.

ReAct Pattern (Yao et al., 2022) — фундамент, на котором строятся Reflexion, Plan-and-Execute, ReWOO и все агентные архитектуры. Без него остальные паттерны не складываются.

## 2. Что такое ReAct Pattern

### История

В октябре 2022 года команда из Princeton University и Google Brain — Shunyu Yao, Jeffrey Zhao, Dian Yu, Nan Du, Izhak Shafran, Karthik Narasimhan, Yuan Cao — опубликовала работу [«ReAct: Synergizing Reasoning and Acting in Language Models»](https://arxiv.org/abs/2210.03629). Проектная страница: [react-lm.github.io](https://react-lm.github.io).

Идея проста: LLM (Large Language Model) должен не только рассуждать (как в Chain-of-Thought), и не только действовать (как в tool-use агентах), но и **чередовать** рассуждение и действие, формируя замкнутый цикл обратной связи.

### Аналогия с человеческим мышлением

Когда вы решаете сложную задачу — скажем, планируете маршрут в незнакомом городе — вы не строите весь план в голове, а потом действуете. Вы: смотрите карту → думаете → идёте → видите знак → корректируете маршрут → идёте дальше. Именно этот паттерн — **interleaved Reasoning + Acting** — и формализует ReAct.

### Контраст с предшественниками

| Подход | Рассуждение | Действие | Grounding | Self-correction |
|--------|-------------|----------|-----------|-----------------|
| **CoT** (Chain-of-Thought, [Wei et al., 2022](https://arxiv.org/abs/2201.11903)) | Да | Нет | Нет | Нет |
| **Act-only** | Нет | Да | Да | Нет |
| **ReAct** | Да | Да | Да | Да |

CoT (Chain-of-Thought) генерирует цепочку рассуждений, но не может проверить факты. Act-only вызывает инструменты, но не рефлексирует над результатами. ReAct объединяет оба мира.

### Формальное определение

ReAct-агент работает в цикле:

{{< mermaid align="center" >}}
graph LR
    Q((Question)) --> T1[Thought]
    T1 --> A1[Action]
    A1 --> O1[Observation]
    O1 --> T2[Thought]
    T2 --> A2[Action]
    A2 --> O2[Observation]
    O2 --> TN[...]
    TN --> Ans((Answer))
{{< /mermaid >}}

Пример трассы (задача: «Какая погода в Париже?»):

```
Thought: Нужно узнать текущую погоду в Париже
Action:  get_weather(city="Paris")
Observation: 18°C, ясно, влажность 45%
Thought: Данные получены, могу ответить
Answer: В Париже сейчас 18°C, ясно, влажность 45%
```

Каждый **Thought** — рассуждение модели о том, что делать дальше. Каждый **Action** — вызов инструмента. Каждая **Observation** — результат, который модель учитывает в следующем шаге.

## 3. Какую задачу решает

### Проблема CoT: hallucination

Chain-of-Thought ([Wei et al., 2022](https://arxiv.org/abs/2201.11903)) впечатляет на задачах, где ответ можно вывести из контекста. Но как только нужны внешние факты — модель галлюцинирует. Классический пример: CoT уверенно генерирует «президент Непала — Хари Бахадур Баснет» — имя звучит правдоподобно, но фактически неверно. Нет grounding — нет гарантии.

### Проблема Act-only: нет рефлексии

Агент, который только действует, может вызвать инструмент, получить результат, но не способен синтезировать ответ. Он не «думает» над наблюдением — просто передаёт его дальше. Это работает для простых запросов, но ломается на многошаговых задачах.

### Проблема error propagation

В длинных цепочках CoT ошибка на раннем шаге не замечается и каскадно размывается, делая финальный ответ бессмысленным. Без возможности перепроверить себя через внешнюю среду модель не может корректировать курс.

### Что обеспечивает ReAct

| Проблема | Механизм ReAct |
|----------|----------------|
| Hallucination | Grounding через инструментальные вызовы (Observation = факт) |
| Отсутствие рефлексии | Thought после каждого Observation — модель анализирует результат |
| Error propagation | Self-correction: модель видит ошибочный результат и корректирует план |
| Непрозрачность | Interpretability: каждая Thought — читаемый след рассуждения |
| Жёсткий план | Dynamic planning: план пересматривается на каждом шаге |

### Цифры из Yao et al., 2022

Ключевые результаты из [оригинальной работы](https://arxiv.org/abs/2210.03629):

- **HotpotQA + FEVER**: ReAct преодолевает hallucination CoT через Wikipedia API — модель получает факты, а не выдумывает их ([Yao et al., 2022, §4.1](https://arxiv.org/abs/2210.03629)).
- **ALFWorld**: +34% success rate над imitation learning — в среде текстового взаимодействия с домом ReAct значительно превосходит базовый подход ([Yao et al., 2022, §4.2](https://arxiv.org/abs/2210.03629)).
- **WebShop**: +10% success rate над RL (Reinforcement Learning) — даже в сложной задаче онлайн-покупок ReAct демонстрирует преимущество ([Yao et al., 2022, §4.2](https://arxiv.org/abs/2210.03629)).

### Применимость паттерна

Важно: паттерн проверен на широком классе моделей и задач. Конкретные числа из Yao et al. получены на PaLM-540B, но сам принцип interleaved reasoning + acting не зависит от конкретной модели — он работает и с GPT-4, и с Claude, и с открытые моделями, поддерживающими tool calling.

## 4. Архитектура и диаграммы

### Цикл Thought → Action → Observation

{{< mermaid align="center" >}}
sequenceDiagram
    participant U as User
    participant A as ReAct Agent
    participant T as Tools
    
    U->>A: Question
    loop ReAct Loop
        A->>A: Thought (рассуждение)
        A->>T: Action (вызов инструмента)
        T-->>A: Observation (результат)
    end
    A->>A: Final Thought
    A-->>U: Answer
{{< /mermaid >}}

На каждой итерации агент формирует **Thought** — внутреннее рассуждение о текущем состоянии и следующем шаге. Затем выполняет **Action** — вызов одного из доступных инструментов. Полученная **Observation** добавляется в контекст, и цикл повторяется. State (история сообщений) накапливается: все предыдущие Thoughts, Actions и Observations доступны на следующем шаге.

### State-Graph представление

В фреймворках вроде Eino ReAct реализуется как направленный граф (State Graph):

{{< mermaid align="center" >}}
graph TB
    START((START)) --> CM[ChatModel]
    CM -->|tool_calls| B{Branch}
    CM -->|no tool_calls| END((END))
    B -->|has tool calls| TN[ToolsNode]
    B -->|no tool calls| END
    TN --> CM
{{< /mermaid >}}

Graph-представление важно по трём причинам:

- **State**: все сообщения хранятся в едином состоянии графа — не нужно вручную управлять контекстом.
- **Streaming**: каждый узел графа может стримить результат — пользователь видит рассуждение агента в реальном времени.
- **Callbacks**: на каждый узел можно повесить обработчики — логирование, метрики, трейсинг.

### Эволюция подходов 2022–2025

{{< mermaid align="center" >}}
graph LR
    A["Prompt-based<br/>ReAct (2022)"] --> B["Tool Calling<br/>API (2023)"]
    B --> C["Agent<br/>Frameworks (2024)"]
    C --> D["Multi-Agent<br/>Orchestration (2025)"]
{{< /mermaid >}}

- **Prompt-based ReAct (2022)**: оригинальная реализация — few-shot промпты, инструменты через текстовый интерфейс. Работало, но хрупко и не масштабировалось.
- **Tool Calling API (2023)**: модели получили нативную поддержку function calling — инструменты стали структурированными и надёжными. Schick et al., 2023 ([arXiv:2302.04761](https://arxiv.org/abs/2302.04761)) показали, что LLM могут самостоятельно учиться вызывать инструменты.
- **Agent Frameworks (2024)**: LangGraph, LlamaIndex, Eino — фреймворки, которые инкапсулируют ReAct в переиспользуемые компоненты с графовой архитектурой.
- **Multi-Agent Orchestration (2025)**: несколько ReAct-агентов координируются для решения сложных задач — каждый специализируется на своей области.

## 5. Состояние рынка: фреймворки

ReAct — универсальный паттерн, реализованный во всех major-фреймворках. Сводная таблица:

| Фреймворк | Язык | Реализация | Ключевая особенность |
|-----------|------|------------|----------------------|
| **LangGraph** | Python | `create_react_agent` | Де-факто стандарт Python-мира, графовая модель |
| **LlamaIndex** | Python | `ReActAgent` | Глубокая интеграция с RAG (Retrieval-Augmented Generation) и индексами |
| **OpenAI Agents SDK** (Software Development Kit) | Python | `Agent` + tools | Нативная интеграция с GPT-4/GPT-4o |
| **Anthropic Claude API** | Python/TS | Tool use + system prompt | Максимум рассуждения через extended thinking |
| **Google ADK** | Python | `Agent` + tools | Интеграция с Gemini и Google Cloud |
| **Eino** | Go | `react.NewAgent` | Go-нативный, production-tested в ByteDance |

### LangGraph — де-факто стандарт

[LangGraph](https://github.com/langchain-ai/langgraph) стал стандартом Python-мира для агентных систем. Функция `create_react_agent` создаёт готового ReAct-агента из модели и списка инструментов за несколько строк. Графовая архитектура позволяет добавлять условные переходы, циклы и человеко-в-контуре (human-in-the-loop). Если вы в Python — это первый кандидат.

### Eino — почему Go

[Eino](https://www.cloudwego.io/docs/eino/overview/eino_open_source/) — фреймворк от CloudWeGo (ByteDance), production-tested в Doubao, TikTok и Coze. Выбран для практической секции по трём причинам:

1. **Go-нативный**: типизированные инструменты, интерфейсы, отсутствие `interface{}`-ада.
2. **Production-tested**: обслуживает сотни миллионов запросов в день внутри ByteDance.
3. **Графовая архитектура**: `compose.Graph` под капотом ReAct-агента — та же модель, что в LangGraph, но на Go.

Python-примеры кода в этой статье не приводятся — это намеренно. Фокус блога — Go.

## 6. Практика: ReAct на Eino, Go

### Установка

```bash
go get github.com/cloudwego/eino@latest
go get github.com/cloudwego/eino-ext/components/model/openai@latest
```

Документация API: [pkg.go.dev/github.com/cloudwego/eino](https://pkg.go.dev/github.com/cloudwego/eino)

### Минимальный ReAct-агент

```go
package main

import (
    "context"
    "fmt"

    "github.com/cloudwego/eino-ext/components/model/openai"
    "github.com/cloudwego/eino/components/tool"
    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/flow/agent/react"
    "github.com/cloudwego/eino/schema"
)

func main() {
    ctx := context.Background()

    // Модель с поддержкой tool calling
    chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        Model: "gpt-4o",
    })

    // Создаём агента с одним инструментом
    agent, _ := react.NewAgent(ctx, &react.AgentConfig{
        ToolCallingModel: chatModel,
        ToolsConfig: compose.ToolsNodeConfig{
            InvokableTools: []tool.InvokableTool{weatherTool()},
        },
    })

    // Вызов агента
    msg, _ := agent.Generate(ctx, []*schema.Message{
        schema.UserMessage("Какая погода в Париже?"),
    })
    fmt.Println(msg.Content)
}
```

### Типизированный инструмент через utils.NewTool

```go
type WeatherRequest struct {
    City string `json:"city" jsonschema:"description=Город для запроса погоды"`
}

type WeatherResponse struct {
    City        string `json:"city"`
    Temperature int    `json:"temperature"`
    Condition   string `json:"condition"`
}

func weatherTool() tool.InvokableTool {
    return utils.NewTool(
        &schema.ToolInfo{
            Name: "get_weather",
            Desc: "Получить текущую погоду в указанном городе",
            ParamsOneOf: schema.NewParamsOneOfByParams(map[string]*schema.ParameterInfo{
                "city": {Type: "string", Desc: "Город", Required: true},
            }),
        },
        func(ctx context.Context, input *WeatherRequest) (*WeatherResponse, error) {
            // Здесь — реальный вызов API погоды
            return &WeatherResponse{
                City: input.City, Temperature: 18, Condition: "ясно",
            }, nil
        },
    )
}
```

### Ключевые параметры (без кода)

- **`ToolCallingModel`** — модель обязана поддерживать tool calling (интерфейс `ToolCallingChatModel`).
- **`ToolsConfig`** — конфигурация узла инструментов: `InvokableTools` и `StreamableTools`.
- **`MaxStep`** — лимит шагов графа (default 12 = до 6 полных циклов ChatModel + Tools).
- **`MessageModifier`** — функция для модификации сообщений перед вызовом модели (например, добавление system prompt).
- **`ToolReturnDirectly`** — инструменты, результат которых возвращается напрямую пользователю, минуя следующий вызов модели.

### Ссылки для углубления

- [ReAct Agent Manual](https://www.cloudwego.io/docs/eino/core_modules/flow_integration_components/react_agent_manual/) — полное описание всех параметров
- [eino-examples/flow/agent/react](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/react) — полный рабочий пример (Food Recommender demo)
- [GitHub: cloudwego/eino](https://github.com/cloudwego/eino) — исходники

## 7. Когда ReAct НЕ подходит

ReAct — не серебряная пуля. Каждый цикл Thought→Action→Observation — это отдельный LLM-вызов, а значит: стоимость растёт линейно с числом шагов, latency накапливается, а горизонт планирования ограничен контекстным окном модели.

### Иерархия сложности

Microsoft Azure Architecture Center в [руководстве по оркестрации AI-агентов](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) формулирует принцип: **используй минимальный уровень сложности, который решает задачу**.

| Уровень | Описание | Когда достаточно |
|---------|----------|------------------|
| **Direct model call** | Один вызов LLM, без инструментов | Классификация, суммаризация, перевод |
| **Single agent + tools (ReAct)** | Агент с рассуждением и инструментами | Динамический выбор инструментов в рамках одного домена |
| **Multi-agent orchestration** | Несколько специализированных агентов | Кросс-доменные задачи, разные security boundaries |

ReAct занимает средний уровень. Если задача решается прямым вызовом модели — не нужен агент. Если один агент не справляется из-за сложности промпта, перегрузки инструментами или требований безопасности — переходи к multi-agent. Но не раньше.

| Если ваша задача... | ...рассмотрите альтернативу |
|---------------------|----------------------------|
| Фиксированный workflow с известными шагами | **Plan-and-Solve** — планирование без итеративного поиска |
| Чувствительна к cost (много LLM-вызовов) | **ReWOO** — план без interleaved вызовов модели ([Xu et al., 2023](https://arxiv.org/abs/2305.18323)) |
| Требует учёта прошлых ошибок | **Reflexion** (он же maker-checker, evaluator-optimizer) — ReAct + самооценка ([Shinn et al., 2023](https://arxiv.org/abs/2303.11366)) |
| Нуждается в дереве гипотез | **Tree of Thoughts** — ветвящееся рассуждение ([Yao et al., 2023](https://arxiv.org/abs/2305.10601)) |
| Длинный горизонт планирования | **Plan-and-Execute** — декомпозиция на подзадачи |

Подробное сравнение паттернов — в Части 5 этого цикла.

## 8. Что дальше в цикле

- **Часть 2: Plan-and-Execute Pattern** — декомпозиция задачи на подзадачи с отдельным планировщиком и исполнителем.
- **Часть 3: Reflexion Pattern** — ReAct + самооценка: агент учится на собственных ошибках.
- **Часть 4: ReWOO Pattern** — планирование без interleaved вызовов модели: дешевле, быстрее, но без динамической корректировки.
- **Часть 5: Сравнение паттернов + Multi-Agent Orchestration** — когда какой паттерн выбирать, и как несколько агентов координируются для решения сложных задач.

Обсуждение приветствуется — комментарии на сайте или [GitHub Issues](https://github.com/triumphpc/blog/issues).

## 9. Ссылки

### Исследовательские статьи

1. **Yao et al., 2022 — ReAct**: Синергия рассуждения и действия в языковых моделях. Формализация цикла Thought→Action→Observation. [arXiv:2210.03629](https://arxiv.org/abs/2210.03629)
2. **Wei et al., 2022 — Chain-of-Thought**: Пошаговое рассуждение без взаимодействия с внешней средой. Предшественник ReAct. [arXiv:2201.11903](https://arxiv.org/abs/2201.11903)
3. **Schick et al., 2023 — Toolformer**: LLM обучаются самостоятельно вызывать инструменты. Мост между prompt-based и API-based подходами. [arXiv:2302.04761](https://arxiv.org/abs/2302.04761)
4. **Shinn et al., 2023 — Reflexion**: Расширение ReAct с самооценкой и вербальным подкреплением. [arXiv:2303.11366](https://arxiv.org/abs/2303.11366)
5. **Yao et al., 2023 — Tree of Thoughts**: Обобщение CoT на дерево гипотез с поиском. [arXiv:2305.10601](https://arxiv.org/abs/2305.10601)
6. **Xu et al., 2023 — ReWOO**: Планирование без interleaved вызовов модели — показывает ограничения ReAct по cost/latency. [arXiv:2305.18323](https://arxiv.org/abs/2305.18323)

### Документация и примеры

- [Eino ReAct Agent Manual](https://www.cloudwego.io/docs/eino/core_modules/flow_integration_components/react_agent_manual/) — все параметры и конфигурация
- [Eino Open Source Announcement](https://www.cloudwego.io/docs/eino/overview/eino_open_source/) — обзор фреймворка
- [GitHub: cloudwego/eino](https://github.com/cloudwego/eino) — исходники
- [GitHub: eino-examples/flow/agent/react](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/react) — полный рабочий пример (Food Recommender)
- [LangGraph ReAct Agent template](https://github.com/langchain-ai/react-agent) — реализация на Python
- [React Project Page](https://react-lm.github.io) — проектная страница оригинальной работы

### Архитектурные руководства

- [AI agent design patterns — Microsoft Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) — иерархия сложности (direct call → single agent → multi-agent), паттерны оркестрации, production-рекомендации по reliability, security, cost optimization

