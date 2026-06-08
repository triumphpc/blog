---
title: "Шаблоны проектирования AI-агентов. Часть 4: Multi-Agent Patterns"
date: 2026-06-08
draft: false
tags: ["ai", "agents", "llm", "multi-agent", "eino", "go"]
categories: ["ai", "engineering"]
summary: "Пять архитектур multi-agent оркестрации — Supervisor, Swarm, Debate, Assembly Line, Hierarchical — с Mermaid-схемами, код-примерами на Eino (Go) и честным разбором ограничений. Когда один агент не справляется, а когда лучше им и остаться."
ShowToc: true
series: ["AI Agent Design Patterns"]
---

## 1. Введение

В [Части 1]({{< ref "ai-agent-design-patterns-1-react" >}}) я разобрал <abbr title="Reasoning + Acting — паттерн, где LLM чередует рассуждение и действие, получая наблюдения из среды">ReAct</abbr> — агента, который думает и действует. В [Части 2]({{< ref "ai-agent-design-patterns-2-plan-execute" >}}) — Plan-and-Execute, где планировщик строит N-шаговый план. В [Части 3]({{< ref "ai-agent-design-patterns-3-reflexion" >}}) — Reflexion, где агент учится на ошибках. Три паттерна, три одиночных агента.

И тут мы упираемся в потолок. Один <abbr title="Large Language Model — большая языковая модель; нейросеть, обученная на текстовых корпусах">LLM</abbr>-агент не может быть экспертом во всём. Точнее, может — но плохо. Один system prompt — одна «роль». Writer ≠ Reviewer ≠ Researcher в одной голове. Конфликт интересов, галлюцинации, потеря фокуса. И даже Reflexion не поможет, если задача требует принципиально разных экспертиз — агент не «отрефлексирует» себе навык, которого у него нет.

Значит, нужна команда. Несколько агентов, каждый со своей ролью, координирующихся для решения общей задачи. Звучит просто, но тут начинается самое интересное: **как именно они координируются?** Кто принимает решения? Как передавать контекст? Как избежать бесконечных циклов handoff?

Я разберу пять архитектур multi-agent оркестрации — от централизованного супервизора до параллельного дебата — с Mermaid-схемами, кодом на Eino (Go) и честными ограничениями. Все примеры на одном сценарии: code review pipeline с Researcher, Coder и Reviewer — так будет проще сравнивать.

## 2. Supervisor Pattern

### Мотивация

Самый частый сценарий: у вас есть задача, требующая разных экспертиз, но нужен кто-то, кто решит, **какому агенту что делегировать**. Без централизованной координации агенты будут говорить одновременно, перебивать друг друга или, что ещё хуже, передавать задачу по кругу.

Supervisor — это <abbr title="Централизованная координация — один агент-маршрутизатор принимает все решения о делегировании; исключает конфликты маршрутизации">централизованная координация</abbr>: один агент-маршрутизатор получает задачу, решает, кому делегировать, собирает результаты и принимает решение о следующем шаге.

### Архитектура

{{< mermaid >}}
graph TB
    USER["👤 User"] -->|"задача"| SUP["🧑‍💼 Supervisor<br/>делегирует, агрегирует"]
    SUP -->|"delegate: research"| RES["🔍 Researcher<br/>собирает контекст"]
    SUP -->|"delegate: code"| COD["💻 Coder<br/>пишет код"]
    SUP -->|"delegate: review"| REV["🔎 Reviewer<br/>проверяет код"]
    RES -->|"результат"| SUP
    COD -->|"результат"| SUP
    REV -->|"результат"| SUP
    SUP -->|"финальный ответ"| USER

    style SUP fill:#2e7d32,color:#fff
    style RES fill:#1565c0,color:#fff
    style COD fill:#e65100,color:#fff
    style REV fill:#6a1b9a,color:#fff
{{< /mermaid >}}

Supervisor видит полную картину: каждый sub-agent возвращает результат супервизору, а не следующему агенту. Это исключает хаос, но создаёт <abbr title="Узкое место — единственная точка, через которую проходят все данные; при росте числа агентов supervisor становится bottleneck">bottleneck</abbr>.

### Реализация на Eino

Eino ADK предоставляет готовый `supervisor.New`:

```go
package main

import (
    "context"
    "log"

    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/adk/prebuilt/supervisor"
    "github.com/cloudwego/eino-ext/components/model/openai"
)

func main() {
    ctx := context.Background()

    // Инициализируем ChatModel для всех агентов
    chatModel, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        APIKey:  os.Getenv("OPENAI_API_KEY"),
        Model:   os.Getenv("OPENAI_MODEL"),
        BaseURL: os.Getenv("OPENAI_BASE_URL"),
    })
    if err != nil {
        log.Fatal(err)
    }

    // Создаём sub-агентов
    researcher, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "researcher",
        Description: "Researches best practices and gathers context for coding tasks",
        Instruction: "You are a research specialist. Find relevant information, API docs, and best practices.",
        Model:       chatModel,
    })

    coder, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "coder",
        Description: "Writes Go code based on research findings",
        Instruction: "You are a Go developer. Write clean, idiomatic Go code.",
        Model:       chatModel,
    })

    reviewer, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "reviewer",
        Description: "Reviews code for bugs, style issues, and correctness",
        Instruction: "You are a code reviewer. Check for bugs, edge cases, and style issues.",
        Model:       chatModel,
    })

    // Supervisor-агент (координатор)
    coordinator, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "coordinator",
        Description: "Coordinates research, coding, and review sub-agents",
        Instruction: "You are a project coordinator. Delegate tasks to the right specialist.",
        Model:       chatModel,
    })

    // Собираем Supervisor pattern
    supervisorAgent, err := supervisor.New(ctx, &supervisor.Config{
        Supervisor: coordinator,
        SubAgents:  []adk.Agent{researcher, coder, reviewer},
    })
    if err != nil {
        log.Fatal(err)
    }

    // Запускаем
    runner := adk.NewRunner(ctx, adk.RunnerConfig{
        Agent: supervisorAgent,
    })

    result, _ := runner.Query(ctx, "Implement a thread-safe LRU cache in Go")
    _ = result
}
```

Ключевой момент: `supervisor.Config` принимает `Supervisor` (координатор) и `SubAgents` (массив специалистов). Supervisor сам решает, кого вызвать и когда.

### Ограничения

1. **Bottleneck**: все результаты проходят через supervisor → при росте числа агентов задержка растёт линейно
2. **Контекстный взрыв**: supervisor должен держать в контексте результаты всех sub-agents → токены расходуются быстро
3. **Single point of failure**: если supervisor делегирует неправильно, весь процесс идёт не туда

## 3. Swarm / Handoff Pattern

### Мотивация

А что если supervisor не нужен? Что если агенты сами знают, кому передать задачу? Это <abbr title="Децентрализованная маршрутизация — агенты сами решают, кому передать управление; нет единого координатора">децентрализованная координация</abbr>: агент выполнил свою часть и передал (handoff) следующему.

Звучит заманчиво — нет bottleneck, нет single point of failure. Но есть цена: кто решает, когда передавать? Агент должен понимать, когда его работа закончена и кому передавать эстафету.

### Архитектура

{{< mermaid >}}
graph LR
    RES["🔍 Researcher"] -->|"handoff:<br/>контекст готов"| COD["💻 Coder"]
    COD -->|"handoff:<br/>код готов"| REV["🔎 Reviewer"]
    REV -->|"handoff:<br/>нужен фикс"| COD
    REV -->|"done"| USER["👤 User"]

    style RES fill:#1565c0,color:#fff
    style COD fill:#e65100,color:#fff
    style REV fill:#6a1b9a,color:#fff
{{< /mermaid >}}

Обратите внимание: Reviewer может передать задачу обратно Coder (нашёл баг → исправь). Это цикл — в отличие от Assembly Line, где поток строго однонаправленный.

### Реализация на Eino

Eino предоставляет `host.NewMultiAgent` — Host-паттерн, где один агент-хост маршрутизирует к специалистам:

```go
package main

import (
    "context"
    "log"

    "github.com/cloudwego/eino/flow/agent/multiagent/host"
    "github.com/cloudwego/eino-ext/components/model/openai"
)

func main() {
    ctx := context.Background()

    chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        APIKey:  os.Getenv("OPENAI_API_KEY"),
        Model:   os.Getenv("OPENAI_MODEL"),
        BaseURL: os.Getenv("OPENAI_BASE_URL"),
    })

    // Host-агент: маршрутизатор
    hostAgent := &host.Host{
        ChatModel:    chatModel,
        SystemPrompt: "You manage a code review pipeline. Route requests to the right specialist.",
    }

    // Специалисты (реализуют интерфейс host.Specialist)
    researcher := NewResearchSpecialist(chatModel)
    coder := NewCodeSpecialist(chatModel)
    reviewer := NewReviewSpecialist(chatModel)

    // Собираем multi-agent
    multiAgent, err := host.NewMultiAgent(ctx, &host.Config{
        Host:        hostAgent,
        Specialists: []host.Specialist{researcher, coder, reviewer},
    })
    if err != nil {
        log.Fatal(err)
    }

    // Запускаем
    result, _ := multiAgent.Run(ctx, "Implement a thread-safe LRU cache in Go")
    _ = result
}
```

Host-паттерн в Eino — это гибрид: хост-агент маршрутизирует задачи, но специалисты могут быть вызваны по контексту, а не только по прямому приказу. Это ближе к swarm, чем к чистому supervisor.

### Ограничения

1. **Deadlock**: Agent A → B → A → B → ... бесконечный цикл handoff. Нет supervisor, который разорвёт петлю
2. **Качество маршрутизации**: если агент неправильно оценивает, кому передавать, вся цепочка ломается
3. **Контекстная изоляция**: каждый агент видит только свой контекст + handoff-сообщение. Нет глобальной картины

## 4. Debate Pattern

### Мотивация

А если задача не имеет единственного правильного ответа? Или если критично проверить несколько гипотез параллельно? Du et al. показали, что мультиагентный дебат улучшает <abbr title="Фактологическая точность — соответствие генерируемого текста реальным фактам; снижает галлюцинации">фактологическую точность</abbr> на +8% на GSM8K через debate между N агентами ([Du et al., 2023](https://arxiv.org/abs/2305.14325)).

Идея: несколько агентов независимо решают задачу, затем критикуют решения друг друга — и приходят к консенсусу. Как <abbr title="Общество разумов — концепция Марвина Мински: интеллект возникает из взаимодействия множества простых агентов, а не из единого центра">society of minds</abbr> Мински, только на практике.

### Архитектура

{{< mermaid >}}
graph TB
    TASK["📋 Task"] --> A1["🤖 Agent A"]
    TASK --> A2["🤖 Agent B"]
    TASK --> A3["🤖 Agent C"]
    A1 <-->|"critique"| A2
    A2 <-->|"critique"| A3
    A1 <-->|"critique"| A3
    A1 --> J["⚖️ Judge / Consensus"]
    A2 --> J
    A3 --> J
    J --> RESULT["✅ Final Answer"]

    style A1 fill:#2e7d32,color:#fff
    style A2 fill:#1565c0,color:#fff
    style A3 fill:#e65100,color:#fff
    style J fill:#c62828,color:#fff
{{< /mermaid >}}

Три агента решают задачу параллельно, критикуют друг друга, а <abbr title="Judge — отдельный агент-арбитр, который получает все решения и выбирает лучшее; альтернатива — majority vote, где побеждает вариант, за который проголосовало большинство">Judge (или majority vote)</abbr> выбирает финальный ответ. Это не конвейер — это **параллельный консенсус**.

### Реализация на Eino

Готового Debate-паттерна в Eino нет. Собираем через `compose.Graph`:

```go
package main

import (
    "context"
    "fmt"

    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/schema"
    "github.com/cloudwego/eino-ext/components/model/openai"
)

func main() {
    ctx := context.Background()

    chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        APIKey:  os.Getenv("OPENAI_API_KEY"),
        Model:   os.Getenv("OPENAI_MODEL"),
        BaseURL: os.Getenv("OPENAI_BASE_URL"),
    })

    g := compose.NewGraph[string, string]()

    // Три независимых агента — генерируют решения параллельно
    g.AddGraphNode("agent_a", func(ctx context.Context, task string) (string, error) {
        msgs := []*schema.Message{
            schema.SystemMessage("You are a senior Go developer. Solve the task."),
            schema.UserMessage(task),
        }
        resp, _ := chatModel.Generate(ctx, msgs)
        return resp.Content, nil
    })

    g.AddGraphNode("agent_b", func(ctx context.Context, task string) (string, error) {
        msgs := []*schema.Message{
            schema.SystemMessage("You are a careful code reviewer. Solve the task with emphasis on correctness."),
            schema.UserMessage(task),
        }
        resp, _ := chatModel.Generate(ctx, msgs)
        return resp.Content, nil
    })

    g.AddGraphNode("agent_c", func(ctx context.Context, task string) (string, error) {
        msgs := []*schema.Message{
            schema.SystemMessage("You are a performance expert. Solve the task with emphasis on efficiency."),
            schema.UserMessage(task),
        }
        resp, _ := chatModel.Generate(ctx, msgs)
        return resp.Content, nil
    })

    // Judge: агрегирует и выбирает лучший ответ
    g.AddGraphNode("judge", func(ctx context.Context, input string) (string, error) {
        // input содержит результаты всех трёх агентов
        msgs := []*schema.Message{
            schema.SystemMessage("You are a judge. Compare three solutions and pick the best one. Explain your choice."),
            schema.UserMessage(input),
        }
        resp, _ := chatModel.Generate(ctx, msgs)
        return resp.Content, nil
    })

    // Параллельная генерация → Judge
    g.AddEdge(compose.START, "agent_a")
    g.AddEdge(compose.START, "agent_b")
    g.AddEdge(compose.START, "agent_c")
    g.AddEdge("agent_a", "judge")
    g.AddEdge("agent_b", "judge")
    g.AddEdge("agent_c", "judge")
    g.AddEdge("judge", compose.END)

    r, _ := g.Compile(ctx)
    result, _ := r.Invoke(ctx, "Implement a thread-safe LRU cache in Go")
    fmt.Println(result)
}
```

Ключевой момент: `compose.START` → три ноды параллельно → Judge агрегирует. Graph автоматически параллелит ноды, у которых все входы готовы.

### Ограничения

1. **3x cost**: три LLM-вызова вместо одного (минимум). Judge — четвёртый
2. **Не работает без Judge**: простое majority vote может совпасть на ошибке (коллективная галлюцинация)
3. **Задержка**: параллельно — быстрее, чем последовательно, но всё равно минимум max(agent_a, agent_b, agent_c) + judge

## 5. Assembly Line Pattern

### Мотивация

А что если задача строго <abbr title="Последовательная обработка — каждый этап получает результат предыдущего и передаёт следующему; конвейер">последовательна</abbr>? Research → Code → Review — каждый этап зависит от предыдущего. Никакой параллелизм, никакого дебата. Просто конвейер.

Это именно то, что реализовали ChatDev ([Qian et al., 2023](https://arxiv.org/abs/2307.07924)) и MetaGPT ([Hong et al., 2023](https://arxiv.org/abs/2308.00352)). MetaGPT добавил <abbr title="Standard Operating Procedures — стандартизированные операционные процедуры; набор правил, определяющих последовательность и формат взаимодействия агентов">SOP</abbr>: каждый агент получает чёткий формат входа/выхода, что снижает галлюцинации на 1.8x по сравнению с ChatDev.

### Архитектура

{{< mermaid >}}
graph LR
    TASK["📋 Task"] --> RES["🔍 Researcher<br/>собирает контекст"]
    RES -->|"research<br/>findings"| COD["💻 Coder<br/>пишет код"]
    COD -->|"code<br/>draft"| REV["🔎 Reviewer<br/>проверяет"]
    REV -->|"approved<br/>code"| DONE["✅ Result"]

    style RES fill:#1565c0,color:#fff
    style COD fill:#e65100,color:#fff
    style REV fill:#6a1b9a,color:#fff
{{< /mermaid >}}

Строгая последовательность. Никаких циклов, никаких параллельных веток. Каждый агент получает выход предыдущего — и только его.

### Реализация на Eino

Assembly Line — это классический `compose.Workflow` (или Chain):

```go
package main

import (
    "context"
    "fmt"

    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/schema"
    "github.com/cloudwego/eino-ext/components/model/openai"
)

func main() {
    ctx := context.Background()

    chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        APIKey:  os.Getenv("OPENAI_API_KEY"),
        Model:   os.Getenv("OPENAI_MODEL"),
        BaseURL: os.Getenv("OPENAI_BASE_URL"),
    })

    // Assembly Line через Chain
    chain := compose.NewChain[string, string]()

    // Stage 1: Research
    chain.AppendChatTemplate(
        prompt.FromMessages(schema.FString,
            schema.SystemMessage("You are a research specialist. Gather context and best practices for the task. Output: structured research findings."),
            schema.UserMessage("{input}"),
        ),
    ).AppendChatModel(chatModel)

    // Stage 2: Code
    chain.AppendChatTemplate(
        prompt.FromMessages(schema.FString,
            schema.SystemMessage("You are a Go developer. Write code based on the research findings. Output: complete Go source code."),
            schema.UserMessage("{input}"),
        ),
    ).AppendChatModel(chatModel)

    // Stage 3: Review
    chain.AppendChatTemplate(
        prompt.FromMessages(schema.FString,
            schema.SystemMessage("You are a code reviewer. Review the code for bugs, edge cases, and style issues. Output: approved code or list of fixes."),
            schema.UserMessage("{input}"),
        ),
    ).AppendChatModel(chatModel)

    r, _ := chain.Compile(ctx)
    result, _ := r.Invoke(ctx, "Implement a thread-safe LRU cache in Go")
    fmt.Println(result)
}
```

`compose.Chain` — это `Workflow` под капотом. Каждая пара `ChatTemplate + ChatModel` — один этап конвейера. Выход предыдущего автоматически подаётся на вход следующего.

### Ограничения

1. **Последовательность = медленно**: нет параллелизма, каждый этап ждёт предыдущий
2. **Каскадные ошибки**: если Researcher дал плохой контекст, Coder напишет плохой код, а Reviewer может не поймать корневую проблему
3. **Нет цикла**: если Reviewer нашёл баг — весь конвейер нужно перезапускать (в отличие от Swarm, где возможен handoff назад)

## 6. Hierarchical Pattern

### Мотивация

А если задача настолько сложная, что один уровень делегирования не справляется? Проект из 10 подзадач, каждая из которых сама разбивается на 3-4 микрозадачи. Один supervisor утонет в контексте.

Hierarchical — это <abbr title="Иерархическая декомпозиция — многоуровневое разбиение задачи: верхний уровень делегирует подзадачи, каждая из которых может иметь свои суб-задачи и координаторов">многоуровневая декомпозиция</abbr>. CEO делегирует Team Lead'ам, те — специалистам. Каждый уровень управляет только своим scope.

### Архитектура

{{< mermaid >}}
graph TB
    USER["👤 User"] --> CEO["🎯 CEO Agent<br/>стратегия"]
    CEO -->|"design<br/>research"| TL1["📋 Team Lead: Research"]
    CEO -->|"implement"| TL2["📋 Team Lead: Dev"]
    TL1 --> R1["🔍 Researcher 1"]
    TL1 --> R2["🔍 Researcher 2"]
    TL2 --> D1["💻 Coder 1"]
    TL2 --> D2["💻 Coder 2"]
    TL1 --> CEO
    TL2 --> CEO
    CEO --> USER

    style CEO fill:#c62828,color:#fff
    style TL1 fill:#2e7d32,color:#fff
    style TL2 fill:#2e7d32,color:#fff
    style R1 fill:#1565c0,color:#fff
    style R2 fill:#1565c0,color:#fff
    style D1 fill:#e65100,color:#fff
    style D2 fill:#e65100,color:#fff
{{< /mermaid >}}

Ключевое отличие от Supervisor: здесь **два уровня делегирования**. CEO не знает про Researcher 1 и Coder 2 — он работает только с Team Lead'ами. Это снижает контекстную нагрузку на каждом уровне.

### Реализация на Eino

Eino ADK предоставляет `deep.New` — DeepAgent с встроенным <abbr title="Управление задачами — автоматическое разбиение целей на подзадачи, трекинг прогресса и делегирование sub-agent'ам">task management</abbr>:

```go
package main

import (
    "context"
    "log"

    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/adk/prebuilt/deep"
    "github.com/cloudwego/eino-ext/components/model/openai"
)

func main() {
    ctx := context.Background()

    chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        APIKey:  os.Getenv("OPENAI_API_KEY"),
        Model:   os.Getenv("OPENAI_MODEL"),
        BaseURL: os.Getenv("OPENAI_BASE_URL"),
    })

    // Создаём sub-агентов
    researcher, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "researcher",
        Description: "Researches best practices and gathers context",
        Instruction: "You are a research specialist. Find relevant information and API docs.",
        Model:       chatModel,
    })

    coder, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "coder",
        Description: "Writes Go code based on requirements",
        Instruction: "You are a Go developer. Write clean, idiomatic Go code.",
        Model:       chatModel,
    })

    reviewer, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "reviewer",
        Description: "Reviews code for correctness and quality",
        Instruction: "You are a code reviewer. Check for bugs and style issues.",
        Model:       chatModel,
    })

    // DeepAgent: автоматически разбивает задачу на подзадачи
    // и делегирует их sub-agent'ам
    deepAgent, err := deep.New(ctx, &deep.Config{
        Name:        "project_manager",
        Description: "Manages complex coding projects by breaking them into subtasks",
        ChatModel:   chatModel,
        Instruction: "You are a project manager. Break down tasks and delegate to specialists.",
        SubAgents:   []adk.Agent{researcher, coder, reviewer},
    })
    if err != nil {
        log.Fatal(err)
    }

    runner := adk.NewRunner(ctx, adk.RunnerConfig{
        Agent: deepAgent,
    })

    result, _ := runner.Query(ctx, "Implement a thread-safe LRU cache in Go with tests")
    _ = result
}
```

DeepAgent использует встроенный `write_todos` tool для планирования и `task` tool для делегирования sub-agent'ам. Контекст изолирован между основным агентом и sub-agent'ами — это предотвращает <abbr title="Загрязнение контекста — когда нерелевантная информация из одного агента попадает в контекст другого, ухудшая качество ответов">контекстное загрязнение</abbr>.

### Ограничения

1. **Overhead**: два уровня координации = два уровня LLM-вызовов. CEO + Team Lead перед тем, как задача дойдёт до исполнителя
2. **Сложность отладки**: ошибка на нижнем уровне может незаметно просочиться наверх
3. **Over-engineering для простых задач**: если задача укладывается в одного supervisor — иерархия избыточна

## 7. Сравнение паттернов

| Паттерн | Централизация | Параллелизм | Сложность координации | Token overhead | Когда использовать |
|---------|:---:|:---:|:---:|:---:|---|
| **Supervisor** | Высокая | Низкий | Низкая | Средний | Чёткие роли, нужен контроль над порядком выполнения |
| **Swarm / Handoff** | Низкая | Средний | Высокая | Средний | Агенты знают, кому передавать; гибкие маршруты |
| **Debate** | Низкая (или Judge) | Высокий | Средняя | Высокий | Неоднозначные задачи, нужна проверка гипотез |
| **Assembly Line** | Низкая | Нет | Низкая | Низкий | Строгая последовательность этапов, SOP |
| **Hierarchical** | Высокая (многоуровневая) | Средний | Высокая | Высокий | Сложные проекты с декомпозицией на подзадачи |

Как читать таблицу: **Централизация** — сколько точек принятия решений. **Параллелизм** — могут ли агенты работать одновременно. **Сложность координации** — сколько усилий нужно на маршрутизацию. **Token overhead** — сколько дополнительных токенов тратится на координацию.

## 8. Честные ограничения

### Коммуникационный overhead

Каждый обмен между агентами = полный контекст в промпте. При 3 агентах и 2 раундах дебата — 3×2×context_length токенов. При длинном контексте это может стоить $0.50+ на один запрос. <abbr title="Способ снижения негативного эффекта; меры для уменьшения стоимости или риска">Mitigation</abbr>: суммаризация промежуточных результатов перед передачей.

### Координационные ошибки

Supervisor может делегировать неправильно. Agent может передать задачу не тому специалисту. Judge может выбрать худший ответ. В исследовании Li et al. показано, что <abbr title="Sampling-and-voting — метод: запускаем N агентов независимо, каждый генерирует ответ, затем голосованием выбираем наиболее популярный; аналог self-consistency, но с разными system prompt'ами">«more agents» улучшает результат через sampling-and-voting</abbr>, но это работает **только когда базовая модель уже достаточно хороша** ([Li et al., 2024](https://arxiv.org/abs/2402.05120)). Если модель даёт <50% accuracy, больше агентов не помогут.

### Diminishing returns

Больше агентов ≠ лучше результат. Li et al. показали, что <abbr title="Sampling-and-voting — метод: запускаем N агентов независимо, каждый генерирует ответ, затем голосованием выбираем наиболее популярный; аналог self-consistency, но с разными system prompt'ами">sampling-and-voting</abbr> масштабируется с числом агентов, но с **убывающей отдачей**: каждый следующий агент приносит меньше пользы, чем предыдущий. На практике 3-5 агентов — оптимум. После 5-7 стоимость растёт, а прирост минимален.

### Deadlock / Infinite loop

В Swarm/Handoff агент A передаёт B, B — A, и так бесконечно. В Debate агенты не могут договориться. В Hierarchical Team Lead'ы пересылают задачу друг другу. Это не теоретическая проблема — исследование MAST проанализировало 1642 трассы в 7 фреймворках и показало, что координационные сбои составляют **36.9% всех отказов** в multi-agent системах ([Tran et al., 2025](https://arxiv.org/abs/2503.13657)).

Как это решают на практике?

**1. Hard turn limit** — грубый, но надёжный потолок. LangGraph: `recursion_limit` (по умолчанию 25). CrewAI: `max_iter` + `max_execution_time`. AutoGen: `max_consecutive_auto_reply` + `max_rounds`. Eino: `WithMaxRunSteps` в compose.Graph. Достигли лимита — выполнение останавливается, даже если задача не решена. Это спасает от бесконечного расхода токенов, но не от потери качества.

**2. Termination signals** — агент явно сигнализирует о завершении. Каждый агент получает в инструкцию: «Если задача выполнена — верни `TASK_COMPLETED` + результат. Если не можешь завершить после 3 попыток — верни `TASK_FAILED`. Если нужен человек — верни `NEEDS_HUMAN`». Оркестратор проверяет сигнал после каждого шага и разрывает цикл при `COMPLETED` или `FAILED`.

**3. Handoff history** — отслеживание маршрута. Оркестратор хранит список `(agent_name, task_hash)`. Если агент получает задачу, которую уже обрабатывал с тем же хэшем → цикл обнаружен, передать следующему агенту или завершить. Это Circuit breaker из распределённых систем, применённый к агентам.

Ни один метод не работает отдельно. Рабочая комбинация: **termination signals** (осознанная остановка) + **handoff history** (обнаружение циклов) + **hard turn limit** (страховка от зависания).

### Когда один агент лучше

Если задача:
- Не требует разных экспертиз
- Имеет чёткий критерий успеха
- Укладывается в один system prompt

...то multi-agent — это лишняя сложность. ReAct с правильным промптом часто работает лучше, чем три агента, которые координируются через supervisor. Не добавляйте агентов ради агентов.

## 9. Резюме

Пять паттернов multi-agent оркестрации — не конкурирующие, а дополняющие:

- **Supervisor** — когда нужен контроль и предсказуемость
- **Swarm** — когда агенты знают свои границы и могут самоорганизоваться
- **Debate** — когда важна проверка гипотез и фактологическая точность
- **Assembly Line** — когда этапы строго последовательны и SOP важнее гибкости
- **Hierarchical** — когда сложность задачи требует многоуровневой декомпозиции

На Eino: три паттерна из коробки (`supervisor.New`, `host.NewMultiAgent`, `deep.New`), два через `compose.Graph`. Выбор паттерна — это выбор trade-off между контролем и гибкостью, параллелизмом и стоимостью, простотой и масштабируемостью.

И главное: **добавление агентов не решает проблему плохого промпта**. Multi-agent — это инструмент для координации экспертиз, а не волшебная таблетка от галлюцинаций.

---

**Следующая статья серии:** Часть 5: Tool Use Pattern _(скоро)_
