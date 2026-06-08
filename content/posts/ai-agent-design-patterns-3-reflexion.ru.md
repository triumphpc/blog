---
title: "Шаблоны проектирования AI-агентов. Часть 3: Reflexion Pattern"
date: 2026-06-07
draft: false
tags: ["ai", "agents", "llm", "reflexion", "self-reflection", "eino", "go"]
categories: ["ai", "engineering"]
summary: "Reflexion — паттерн, в котором агент учится на собственных ошибках через вербальное подкрепление и эпизодическую память. Разбираю эволюцию self-critique от Constitutional AI до Reflexion, честно показываю ограничения (без внешнего верификатора не работает), и реализую рабочий пример на Go через Eino compose.Graph с циклом Actor → Evaluator → Reflector → retry."
ShowToc: true
series: ["AI Agent Design Patterns"]
---

## 1. Введение

В [Части 1]({{< ref "ai-agent-design-patterns-1-react" >}}) я разобрал <abbr title="Reasoning + Acting — паттерн, где LLM чередует рассуждение (Thought) и действие (Action), получая наблюдения (Observation) из среды; основа для Reflexion Actor">ReAct</abbr> — паттерн, где агент чередует рассуждение и действие. В [Части 2]({{< ref "ai-agent-design-patterns-2-plan-execute" >}}) — Plan-and-Execute, где отдельный Planner строит N-шаговый план. Оба паттерна объединяет один недостаток: **агент не учится на своих ошибках**. Каждый запуск — с чистого листа. Reflexion-паттерн решает эту проблему: агент совершает попытку, оценивает результат, рефлексирует — и при следующей попытке использует накопленный опыт.

## 2. Что такое Reflexion

### История

В декабре 2022 года Anthropic опубликовала [Constitutional AI](https://arxiv.org/abs/2212.08073) — подход, где языковая модель критикует собственные ответы и переписывает их для снижения вредности. Это был первый масштабный пример <abbr title="Паттерн, при котором LLM оценивает собственный вывод и предлагает улучшения">self-critique</abbr>-паттерна: generate → self-critique → revise. Но Constitutional AI использовала критику для **обучения** (<abbr title="Дообучение предобученной модели на специфичных данных; меняет веса модели">finetuning</abbr> через <abbr title="Reinforcement Learning from AI Feedback — метод обучения модели с помощью обратной связи от другой LLM вместо людей-аннотаторов">RLAIF</abbr>), а не для <abbr title="Использование обученной модели для генерации ответа; веса модели не меняются">инференса</abbr>.

В марте 2023 года Madaan et al. опубликовали [Self-Refine](https://arxiv.org/abs/2303.17651) — итеративное улучшение через self-feedback. Та же <abbr title="Large Language Model — большая языковая модель; нейросеть, обученная на больших текстовых корпусах (GPT-4, Claude, Llama)">LLM</abbr> выступает в трёх ролях: генератор, критик и рефайнер. Результат: ~20% average improvement на 7 задачах ([Madaan et al., 2023](https://arxiv.org/abs/2303.17651)). Но есть нюанс — на задачах рассуждения (Math Reasoning) улучшение **0%**: модель не способна надёжно определить, правильное у неё рассуждение или нет.

И тут начинается самое интересное. В том же марте 2023 года Shinn et al. опубликовали [Reflexion](https://arxiv.org/abs/2303.11366) — и решили главную проблему Self-Refine: добавили **<abbr title="Эпизодическая память — хранилище текстовых «уроков» из прошлых попыток; рефлексии инжектируются в контекст Actor">эпизодическую память</abbr>** и **внешнюю оценку**. Вместо одного цикла <abbr title="Цикл «критикуй-улучшай»: LLM оценивает свой вывод и перегенерирует его с учётом замечаний; базовый паттерн Self-Refine">critique-refine</abbr> — <abbr title="Мульти-триальный подход — многократные попытки решения с накоплением опыта между ними">мульти-триальный</abbr> процесс, где агент накапливает вербальные «уроки» и использует их при следующих попытках. Результат: 91% <abbr title="Метрика: доля правильных ответов с первой попытки">pass@1</abbr> на HumanEval (против 80% у GPT-4) и +22% absolute на AlfWorld ([Shinn et al., 2023](https://arxiv.org/abs/2303.11366)).

### Аналогия с разработчиком

Представьте: джун пишет код, запускает тесты — падают. Что он делает? Не переписывает с нуля — он читает ошибку, понимает причину, записывает «в следующий раз проверю <abbr title="Краевой случай — входные данные на границе допустимых значений (пустой список, nil, нулевая длина); частый источник багов">edge-case</abbr> с nil». Это и есть рефлексия: не просто исправить, а **извлечь урок на будущее**. ReAct — джун без памяти: каждый раз наступает на те же грабли. Plan-and-Execute — джун с планом, но без ретроспективы. Reflexion — джун, который ведёт дневник ошибок.

### Формальное определение

Reflexion работает в три фазы, повторяющиеся циклически:

1. **Act**: Actor (LLM) генерирует действия и получает наблюдения из среды.
2. **Evaluate**: <abbr title="Детерминированная система оценки результата (unit-тесты, компилятор, game environment); даёт объективный PASS/FAIL сигнал">Evaluator</abbr> оценивает результат — скалярной оценкой или свободным текстом.
3. **Reflect**: Self-Reflection модель генерирует <abbr title="Вербальное подкрепление — текстовая обратная связь, описывающая, что пошло не так и как исправить; в отличие от числового reward в классическом RL">вербальную обратную связь</abbr> — что пошло не так и как исправить. Рефлексия сохраняется в **<abbr title="Эпизодическая память — хранилище текстовых «уроков» из прошлых попыток">episodic memory</abbr>**.

При следующей попытке Actor получает содержимое эпизодической памяти в контексте — и может избежать повторения ошибок.

## 3. Какую задачу решает

### Проблема ReAct: нет обучения на ошибках

ReAct, как я показал в Части 1, чередует Thought-Action-Observation в одном цикле. Если задача не решена — агент просто начинает заново. Прошлый опыт? Утрачен. Каждый запуск — первый и последний.

### Проблема Plan-and-Execute: нет ретроспективы

Plan-and-Execute строит план и выполняет его. Reviser корректирует план при необходимости — но **внутри одного запуска**. Между запусками — чистый лист. На прошлых ошибках не учимся.

### Проблема Self-Refine: нет памяти между попытками

Self-Refine делает <abbr title="Цикл «критикуй-улучшай»: LLM оценивает свой вывод и перегенерирует его с учётом замечаний; базовый паттерн Self-Refine">critique-refine</abbr> в одном вызове LLM. Улучшение есть на задачах генерации (стиль, формат) — но на задачах рассуждения **0%**, потому что модель не может надёжно оценить правильность собственного рассуждения без внешнего арбитра ([Madaan et al., 2023](https://arxiv.org/abs/2303.17651), Table 1: Math Reasoning).

### Что обеспечивает Reflexion

| Проблема | Решение Reflexion |
|----------|-------------------|
| Нет обучения на ошибках | Episodic memory хранит вербальные уроки между попытками |
| Ненадёжная самооценка | Evaluator — внешний арбитр (тесты, компилятор, среда) |
| Нет ретроспективы | Каждая попытка обогащается рефлексиями из прошлых |
| <abbr title="Близорукий — исправляющий симптом, а не причину; в контексте агентов: локальное исправление без понимания корня проблемы">Myopic</abbr> исправления | Рефлексия фокусируется на причине ошибки, а не на симптоме |

## 4. Архитектура

### Actor → Evaluator → Reflector → Memory → retry

{{< mermaid align="center" >}}
sequenceDiagram
    participant U as User
    participant A as Actor
    participant E as Evaluator
    participant R as Reflector
    participant M as Episodic Memory

    U->>A: Task + reflections from memory
    A->>A: Generate actions (ReAct loop)
    A->>E: Trajectory (actions + observations)
    
    alt Result is correct
        E-->>U: ✅ Success
    else Result has errors
        E->>R: Trajectory + evaluation
        R->>R: Generate verbal reflection<br/>"I failed because..."
        R->>M: Store reflection
        M-->>A: Enriched context for next attempt
        Note over A,M: Retry with accumulated experience
    end
{{< /mermaid >}}

### Компоненты

**Actor** — LLM, генерирующая действия. Может быть ReAct-агентом или простой ChatModel. Ключевое отличие от обычного ReAct: Actor получает содержимое эпизодической памяти в <abbr title="Системный промпт — скрытая инструкция, задающая поведение и роль LLM; не видна пользователю, определяет контекст и ограничения модели">system prompt</abbr>, что позволяет учитывать прошлые ошибки.

**Evaluator** — оценивает результат Actor. Может быть:
- *Детерминированным*: unit tests, компилятор, game environment — даёт объективную оценку
- *LLM-based*: другая языковая модель оценивает качество — менее надёжно, но применимо для <abbr title="Открытые задачи — задачи без единственного правильного ответа; критерий успеха субъективен или не определён формально">open-ended</abbr> задач

Почему это важно? Именно внешний верификатор решает проблему Self-Refine, где модель не способна оценить собственную правильность.

**Self-Reflection model** — LLM, генерирующая вербальную рефлексию. Получает: <abbr title="Траектория — последовательность действий и наблюдений агента за одну попытку; передаётся в Reflector для анализа">траекторию</abbr> Actor (actions + observations), оценку Evaluator, прошлые рефлексии из памяти. Генерирует текст вида: «Я допустил ошибку в обработке <abbr title="Краевой случай — входные данные на границе допустимых значений (пустой список, nil, нулевая длина); частый источник багов">edge-case</abbr> с пустым списком. В следующий раз нужно проверить len() > 0 перед доступом к элементу».

**Episodic Memory** — хранилище рефлексий. Простая структура: список текстовых строк, которые инжектируются в контекст Actor при следующей попытке. Чем больше попыток — тем богаче память.

## 5. Эволюция self-critique

Три паттерна self-critique появились за 3 месяца — и каждый решал проблему предыдущего:

{{< mermaid align="center" >}}
flowchart TD
    CAI["🛡️ Constitutional AI<br/><b>Dec 2022</b> • Bai/Anthropic<br/>Self-critique → revise<br/>для <b>обучения</b> (finetuning)<br/>Цель: harmlessness"]
    SRF["🔄 Self-Refine<br/><b>Mar 2023</b> • Madaan/CMU<br/>Same LLM: generate → critique → refine<br/>для <b>инференса</b> (1 вызов)<br/>~20% improvement на генерации"]
    REF["🧠 Reflexion<br/><b>Mar 2023</b> • Shinn/Princeton+NEU<br/>Actor → Evaluator → Reflector<br/>для <b>инференса</b> (N попыток)<br/>+ Episodic Memory"]
    HUANG["⚠️ Huang et al.<br/><b>Oct 2023</b> • ICLR 2024<br/><i>LLMs Cannot Self-Correct<br/>Reasoning Yet</i><br/>Без external feedback — не работает"]
    RRR["🚀 Reflect, Retry, Reward<br/><b>May 2025</b> • Bensal/Writer<br/>RL-тренировка рефлексий<br/>1.5B-7B бьёт 10x модели"]

    CAI -->|"добавил<br/>inference-time<br/>critique"| SRF
    SRF -->|"добавил<br/>episodic memory<br/>+ external eval"| REF
    REF -->|"показал<br/>ограничения<br/>без верификатора"| HUANG
    HUANG -->|"RL-тренировка<br/>лучших рефлексий"| RRR

    style CAI fill:#2e7d32,color:#fff
    style SRF fill:#1565c0,color:#fff
    style REF fill:#e65100,color:#fff
    style HUANG fill:#c62828,color:#fff
    style RRR fill:#6a1b9a,color:#fff
{{< /mermaid >}}

### Сравнение трёх паттернов

| Аспект | Constitutional AI | Self-Refine | Reflexion |
|--------|-------------------|-------------|-----------|
| **Дата** | Dec 2022 | Mar 2023 | Mar 2023 |
| **Цель** | Harmlessness (safety) | Качество вывода | Качество вывода + обучение |
| **Критика** | Self-critique | Self-feedback | <abbr title="Самокоррекция с внешним сигналом (тесты, компилятор, среда); именно этот вариант использует Reflexion">External evaluator</abbr> + self-reflection |
| **Память** | Нет | Нет | Episodic memory |
| **Попытки** | 1 | 1 (итерации внутри) | N (мульти-триальный) |
| **Применение** | Finetuning (offline) | Инференс (online) | Инференс (online) |
| **Результат** | Улучшение harmlessness | +20% на генерации, 0% на рассуждении | +11% HumanEval (91% vs 80% GPT-4), +22% AlfWorld |
| **Тип оценки** | <abbr title="RLAIF (RL от AI-судьи) — уже раскрыто выше">RLAIF</abbr> (RL от AI-судьи) | Self-judge | External verifier |

Закономерность очевидна: каждый следующий паттерн добавляет то, чего не хватало предыдущему. Constitutional AI не было памяти и мульти-триальности. Self-Refine добавил inference-time critique, но без памяти и без внешнего верификатора. Reflexion замкнул петлю: внешний верификатор решает проблему ненадёжной самооценки, а эпизодическая память позволяет учиться между попытками.

## 6. Когда работает / когда нет

Зачем отдельная секция про ограничения? Потому что в октябре 2023 года Huang et al. опубликовали [«Large Language Models Cannot Self-Correct Reasoning Yet»](https://arxiv.org/abs/2310.01798) — и доказали, что self-correction без внешней обратной связи **ухудшает** результат.

{{< mermaid align="center" >}}
flowchart TD
    START[Задача агента] --> Q{Есть внешний<br/>верификатор?}
    Q -->|Да| Q2{Низкая начальная<br/>точность?}
    Q -->|Нет| FAIL["❌ Reflexion не поможет<br/>Self-correction ухудшит результат<br/>(Huang et al., 2023)"]
    
    Q2 -->|Да| WORKS["✅ Reflexion работает<br/>+11-22% improvement<br/>(HumanEval, AlfWorld)"]
    Q2 -->|Нет| WARN["⚠️ Может не окупиться<br/>Риск ухудшения на простых задачах<br/><abbr title="Убывающая отдача — каждая следующая попытка приносит меньше улучшений, чем предыдущая; после 3-5 попыток стоимость растёт, а прирост минимален">Diminishing returns</abbr>"]
    
    WORKS --> REC1["Рекомендация:<br/>max 3-5 попыток,<br/>оценивай cost/benefit"]
    WARN --> REC2["Рекомендация:<br/>не более 2 попыток,<br/>мониторь качество"]

    style FAIL fill:#c62828,color:#fff
    style WORKS fill:#2e7d32,color:#fff
    style WARN fill:#e65100,color:#fff
{{< /mermaid >}}

### Цифры Huang et al.

Без внешнего верификатора (<abbr title="Самокоррекция без внешней обратной связи — модель сама оценивает и исправляет себя; ухудшает результат на задачах рассуждения (Huang et al., 2023)">intrinsic self-correction</abbr>), качество **падает** на всех моделях:

| Модель | GSM8K (было → стало) | CommonSenseQA (было → стало) |
|--------|---------------------|------------------------------|
| GPT-3.5 | 75.9 → 74.7 | 75.8 → 41.8 |
| GPT-4 | 95.5 → 89.0 | 82.0 → 80.0 |
| Llama-2-70b | 62.0 → 36.5 | 64.0 → 36.5 |

Причины: LLM чаще меняет **правильный** ответ на **неправильный**, чем наоборот. Фундаментальная проблема — модель не способна надёжно оценить правильность собственного рассуждения ([Huang et al., 2023](https://arxiv.org/abs/2310.01798), Figure 1).

### Когда Reflexion работает

| Условие | Почему |
|----------|--------|
| Внешний верификатор (тесты, компилятор, env) | Объективная оценка → точная рефлексия |
| Низкая начальная точность | Есть пространство для улучшения |
| Мульти-триальный сценарий | Память накапливает уроки |
| Задачи с объективным критерием успеха | Чёткий сигнал об ошибке |

### Когда Reflexion НЕ работает

| Условие | Почему |
|----------|--------|
| Нет внешнего верификатора | Модель не может оценить свою правильность |
| Высокая начальная точность | Риск ухудшения (correct → incorrect) |
| <abbr title="Открытые задачи — задачи без единственного правильного ответа; критерий успеха субъективен или не определён формально">Open-ended</abbr> задачи без критерия | Нечего оценивать → неточная рефлексия |
| Простые задачи | Cost рефлексии не окупается |

## 7. Варианты реализации на Eino

Eino не предоставляет готового `reflection.NewAgent()` — в отличие от [`react.NewAgent()`]({{< ref "ai-agent-design-patterns-1-react" >}}) из Части 1 или [`planexecute.NewAgent()`]({{< ref "ai-agent-design-patterns-2-plan-execute" >}}) из Части 2. Но это и к лучшему: Reflexion — это не отдельный тип агента, а **паттерн компоновки**, который можно реализовать несколькими способами.

### Вариант A: compose.Graph с циклом

Eino Graph поддерживает циклы через `AddEdge` из узла в себя же + `WithMaxRunSteps` для ограничения итераций. Это наиболее естественная реализация Reflexion:

{{< mermaid align="center" >}}
flowchart LR
    START(("START")) --> Actor["🎬 Actor<br/>(react.Agent)"]
    Actor --> Evaluator["🔍 Evaluator<br/>(Lambda: run tests)"]
    Evaluator --> Branch{"Pass?"}
    Branch -->|Yes| END(("END"))
    Branch -->|No| Reflector["🪞 Reflector<br/>(ChatModel)"]
    Reflector --> Memory["💾 Memory<br/>(State: []string)"]
    Memory --> Actor
    
    style START fill:#2e7d32,color:#fff
    style END fill:#2e7d32,color:#fff
    style Branch fill:#e65100,color:#fff
    style Actor fill:#1565c0,color:#fff
    style Evaluator fill:#1565c0,color:#fff
    style Reflector fill:#6a1b9a,color:#fff
    style Memory fill:#ad1457,color:#fff
{{< /mermaid >}}

**Плюсы**: явный цикл retry, поддержка ветвления, <abbr title="Сохранение состояния графа выполнения между шагами; в Eino: WithCheckPointStore позволяет прервать и возобновить выполнение">checkpoint</abbr> через `WithCheckPointStore`, можно встроить ReAct-агента через `ExportGraph()`.

**Минусы**: Graph API использует неявную передачу данных (весь output → весь input), нужно аккуратно управлять state.

### Вариант B: compose.Workflow (линейный, без цикла)

Workflow — декларативный граф с явным маппингом полей. Проблема: **Workflow не поддерживает циклы** — всегда `AllPredecessor`. Для Reflexion это фатально: нет retry-loop.

Но если цикл реализовать **снаружи** (в Go-коде), а Workflow использовать для одной итерации Actor → Evaluator → Reflector — это работает:

```go
// Внешний retry-loop
for attempt := 0; attempt < maxAttempts; attempt++ {
    result, err := workflow.Invoke(ctx, input)
    if result.Passed { break }
    memory = append(memory, result.Reflection)
    // Инжектируем память в следующий вызов
    input.Reflections = memory
}
```

**Плюсы**: явный field mapping, типобезопасность, проще тестировать одну итерацию.

**Минусы**: нет встроенного цикла — пришлось реализовывать вручную, нет checkpoint между итерациями.

### Вариант C: deer-go паттерн (State Graph)

[deer-go](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/deer-go) — Go-реализация ByteDance DEER-flow на Eino Graph. Использует `Goto`-маршрутизацию: каждый узел записывает следующий целевой узел в `state.Goto`, а `agentHandOff`-функция направляет выполнение.

Для Reflexion можно добавить Critic-узел и ребро `Critic → Actor` (retry). Это расширяет существующую архитектуру, но deer-go не содержит готовых паттернов рефлексии — только re-planning loop.

**Плюсы**: готовая архитектура state graph, checkpoint, <abbr title="Человек-в-цикле — паттерн, где система приостанавливает выполнение и ждёт решения человека перед продолжением; в Eino: InterruptAndRerun">human-in-the-loop</abbr> через `InterruptAndRerun`.

**Минусы**: сложнее, требует HTTP-сервер и MCP tools, избыточно для простых сценариев.

### Сравнение вариантов

| Аспект | Graph + цикл | Workflow + внешний loop | deer-go |
|--------|-------------|------------------------|---------|
| **Цикл retry** | Встроенный | Внешний (Go-код) | Через Goto |
| **Checkpoint** | ✅ `WithCheckPointStore` | ❌ Вручную | ✅ Встроенный |
| **Сложность** | Средняя | Низкая | Высокая |
| **Field mapping** | Неявный | Явный | Через State |
| **Готовность к prod** | Высокая | Средняя | Высокая |

Мой выбор для примера: **compose.Graph** — нативная поддержка циклов, checkpoint, и прямое встраивание react.Agent через `ExportGraph()`.

## 8. Код-пример

Реализую Reflexion на compose.Graph: Actor (ReAct-агент) генерирует Go-код, Evaluator запускает тесты, Reflector анализирует ошибки, Memory накапливает рефлексии.

{{< mermaid align="center" >}}
flowchart TD
    START(("START")) --> Actor["🎬 Actor<br/>react.Agent<br/>+ write_code tool"]
    Actor --> Evaluator["🔍 Evaluator<br/>Lambda: go test"]
    Evaluator --> Branch{"Tests pass?"}
    Branch -->|Yes| END(("END ✅"))
    Branch -->|No| Reflector["🪞 Reflector<br/>ChatModel: analyze<br/>test failures"]
    Reflector --> Memory["💾 Append reflection<br/>to episodic memory"]
    Memory --> Actor
    
    style START fill:#2e7d32,color:#fff
    style END fill:#2e7d32,color:#fff
    style Branch fill:#e65100,color:#fff
{{< /mermaid >}}

```go
package main

import (
	"context"
	"fmt"

	"github.com/cloudwego/eino/compose"
	"github.com/cloudwego/eino/components/tool"
	"github.com/cloudwego/eino/components/tool/utils"
	"github.com/cloudwego/eino-ext/components/model/openai"
	"github.com/cloudwego/eino/flow/agent/react"
	"github.com/cloudwego/eino/schema"
)

// reflexionState — разделяемое состояние для одного выполнения Reflexion.
// Создаётся заново при каждом Invoke.
type reflexionState struct {
	// Task — исходная задача (например, "write a function that sorts a list")
	Task string
	// Reflections — накопленные вербальные рефлексии из прошлых попыток
	Reflections []string
	// Attempt — текущий номер попытки (1-based)
	Attempt int
	// MaxAttempts — максимальное число попыток
	MaxAttempts int
	// Code — сгенерированный код (выход Actor)
	Code string
	// TestResult — результат запуска тестов (выход Evaluator)
	TestResult string
	// Passed — флаг: тесты пройдены?
	Passed bool
}

// writeCodeTool — инструмент для Actor: "записывает" код в файл.
// В реальном приложении здесь будет запись на диск.
type writeCodeTool struct{}

func (t *writeCodeTool) Info(ctx context.Context) (*schema.ToolInfo, error) {
	return &schema.ToolInfo{
		Name: "write_code",
		Desc: "Write Go code to solve the task. The code will be tested automatically.",
	}, nil
}

func (t *writeCodeTool) InvokableRun(ctx context.Context, args string, opts ...tool.Option) (string, error) {
	// В реальном приложении: записать args в .go файл
	return fmt.Sprintf("Code written (%d bytes)", len(args)), nil
}

func main() {
	ctx := context.Background()

	// 1. Создаём модель для Actor и Reflector
	chatModel, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		Model: "gpt-4o",
	})
	if err != nil {
		panic(err)
	}

	// 2. Создаём ReAct-агента как Actor
	//    ExportGraph() позволяет встроить его в compose.Graph
	codeTool := utils.NewTool(&writeCodeTool{}, nil)
	actor, err := react.NewAgent(ctx, &react.AgentConfig{
		ToolCallingModel: chatModel,
		ToolsConfig: compose.ToolsNodeConfig{
			Tools: []tool.BaseTool{codeTool},
		},
		MaxStep: 5,
	})
	if err != nil {
		panic(err)
	}

	// 3. Строим Reflexion Graph
	g := compose.NewGraph[string, string](
		compose.WithGenLocalState(func(ctx context.Context) *reflexionState {
			return &reflexionState{
				MaxAttempts: 3,
			}
		}),
		compose.WithMaxRunSteps(20), // ограничение на циклы
	)

	// Actor: встроим ReAct-агента через ExportGraph
	actorGraph, actorOpts := actor.ExportGraph()
	g.AddGraphNode("actor", actorGraph, actorOpts...)
	g.AddEdge(compose.START, "actor")

	// Evaluator: Lambda-узел, который "запускает тесты"
	g.AddLambdaNode("evaluator",
		compose.InvokableLambda(func(ctx context.Context, code string) (string, error) {
			// В реальном приложении: exec.Command("go", "test", "./...")
			// Здесь: симуляция
			if len(code) > 10 {
				return "PASS: all tests passed", nil
			}
			return "FAIL: TestSortEmpty - expected [], got nil", nil
		}),
	)
	g.AddEdge("actor", "evaluator")

	// Reflector: ChatModel анализирует ошибки
	g.AddChatModelNode("reflector", chatModel)
	g.AddEdge("evaluator", "reflector")

	// Conditional Branch: pass → END, fail → back to Actor
	g.AddBranch("evaluator", compose.NewGraphBranch(
		func(ctx context.Context, testResult string) (string, error) {
			// Читаем state для проверки числа попыток
			if err := compose.ProcessState[*reflexionState](ctx,
				func(ctx context.Context, s *reflexionState) error {
					s.Attempt++
					s.TestResult = testResult
					// Простой эвристический критерий
					s.Passed = len(testResult) > 4 && testResult[:4] == "PASS"
				},
			); err != nil {
				return "", err
			}

			var target string
			_ = compose.ProcessState[*reflexionState](ctx,
				func(ctx context.Context, s *reflexionState) error {
					if s.Passed || s.Attempt >= s.MaxAttempts {
						target = compose.END
					} else {
						target = "reflector" // сначала рефлексируем, потом retry
					}
					return nil
				},
			)
			return target, nil
		},
		map[string]bool{compose.END: true, "reflector": true},
	))

	// Reflector → Actor: retry с рефлексией в контексте
	g.AddEdge("reflector", "actor")

	// END
	g.AddEdge(compose.END, compose.END)

	// 4. Компилируем и запускаем
	runnable, err := g.Compile(ctx)
	if err != nil {
		panic(fmt.Sprintf("compile error: %v", err))
	}

	result, err := runnable.Invoke(ctx, "Write a function that sorts a slice of integers")
	if err != nil {
		panic(fmt.Sprintf("invoke error: %v", err))
	}

	fmt.Println("Result:", result)
}
```

### Что здесь происходит

1. **Состояние** — `reflexionState` хранит текущую попытку, рефлексии и результат оценки. Создаётся через `WithGenLocalState` при каждом запуске графа.

2. **Actor** — встроен через `ExportGraph()`. ReAct-агент с инструментом `write_code`. При повторном заходе (после рефлексии) получает обновлённый контекст.

3. **Evaluator** — Lambda-узел, который «запускает тесты». В реальном приложении — `exec.Command("go", "test")`. В примере — симуляция: если код длиннее 10 байт — PASS.

4. **Reflector** — ChatModel, анализирующая ошибки. Получает результат тестов и генерирует вербальную рефлексию.

5. **Branch** — условное ветвление после Evaluator: PASS → END, FAIL → Reflector → Actor (retry). Проверяет `Attempt < MaxAttempts`.

6. **Цикл** — `g.AddEdge("reflector", "actor")` замыкает петлю. `WithMaxRunSteps(20)` ограничивает общее число шагов (защита от бесконечного цикла).

## 9. Инженерный сценарий

Кодогенерация с тестами — идеальный сценарий для Reflexion. Почему? Потому что есть **объективный внешний верификатор**: компилятор и unit-тесты. Это не субъективная LLM-оценка, а бинарный PASS/FAIL.

### <abbr title="Test-Driven Development — разработка через тестирование: сначала пишутся тесты, затем код, который их проходит; в контексте агента: тесты выступают как Evaluator">TDD</abbr> для агента

{{< mermaid align="center" >}}
sequenceDiagram
    participant Dev as Developer
    participant A as Actor Agent
    participant T as go test
    participant R as Reflector
    participant M as Memory

    Dev->>A: "Write sort([]int)"
    
    Note over A: Attempt 1
    A->>T: sort.go
    T-->>A: ❌ FAIL: TestSortEmpty<br/>expected [], got nil
    
    A->>R: Trajectory + test failure
    R->>R: "Forgot to handle<br/>empty slice"
    R->>M: Store reflection #1
    
    Note over A: Attempt 2 (with reflection)
    M-->>A: "Check empty slice first"
    A->>T: sort_v2.go
    T-->>A: ❌ FAIL: TestSortStable<br/>unstable sort on equal elements
    
    A->>R: Trajectory + test failure
    R->>R: "Used unstable sort,<br/>need stable"
    R->>M: Store reflection #2
    
    Note over A: Attempt 3 (with 2 reflections)
    M-->>A: "Check empty slice first<br/>Use stable sort"
    A->>T: sort_v3.go
    T-->>A: ✅ PASS: all tests passed
    
    A-->>Dev: sort_v3.go ✅
{{< /mermaid >}}

### Почему это работает лучше Self-Refine

Self-Refine в том же сценарии дал бы **0%** improvement на Math Reasoning ([Madaan et al., 2023](https://arxiv.org/abs/2303.17651)). Почему? Без тестов модель не отличает `sort([]int{})` от `sort([]int{1})` — ей не хватает объективного сигнала. Reflexion с `go test` как Evaluator решает эту проблему: тесты дают точную диагностику ошибки → рефлексия фокусируется на реальной проблеме → Actor исправляет именно то, что нужно.

### Production-реализация

В production сценарий расширяется:

| Компонент | Пример | В production |
|-----------|--------|--------------|
| Actor | `react.NewAgent` с `write_code` | + `read_file`, `search_docs`, `lint_code` |
| Evaluator | `go test ./...` | + `go vet`, `golangci-lint`, coverage ≥ 80% |
| Reflector | ChatModel «analyze failures» | Промпт с конкретными паттернами ошибок |
| Memory | `[]string` в state | Redis / файл с reflection history |
| Max attempts | 3 | 5 (HumanEval: 91% достигнуто за 12 попыток, но 3-5 обычно достаточно) |

## 10. Практические рекомендации

### Когда применять Reflexion

| Сценарий | Применимость | Обоснование |
|----------|-------------|-------------|
| Кодогенерация + тесты | ✅ Отлично | Объективный верификатор (компилятор/тесты) |
| Game-агенты | ✅ Отлично | Среда даёт чёткий <abbr title="Сигнал подкрепления — числовая оценка действия агента средой; в RL: reward за успешное/неуспешное действие">reward signal</abbr> |
| Data pipeline с валидацией | ✅ Хорошо | Schema validation как верификатор |
| Code review автоматизация | ⚠️ Осторожно | LLM-оценка менее надёжна, чем тесты |
| Креативное письмо | ⚠️ Осторожно | Нет объективного критерия успеха |
| Математика / рассуждение | ❌ Не рекомендуется | Без внешнего верификатора — ухудшает результат |

### Настройка гиперпараметров

**Max attempts**: 3-5 для задач с быстрым верификатором (тесты). HumanEval достиг 91% за 12 попыток, но <abbr title="Убывающая отдача — каждая следующая попытка приносит меньше улучшений, чем предыдущая; после 3-5 попыток стоимость растёт, а прирост минимален">diminishing returns</abbr> начинается после 3-5. Больше — дороже, но не лучше.

**Episodic memory size**: храните последние 5-10 рефлексий. Слишком много — контекст разрастается, модель теряет фокус. Слишком мало — не учитывает давние ошибки.

**Evaluator choice**: детерминированный верификатор (тесты, компилятор) всегда лучше LLM-based. Если верификатор ненадёжный — Reflexion превращается в Self-Refine с его проблемами.

**Reflector prompt**: конкретный > абстрактный. Не «проанализируй ошибку», а «идентифицируй: (1) какой тест упал, (2) какой вход вызвал падение, (3) какое предположение было неверным, (4) что изменить в коде».

### Cost management

Каждая попытка Reflexion = полный цикл Actor + Evaluator + Reflector. При 3 попытках — 3x cost. <abbr title="Способ снижения негативного эффекта; меры для уменьшения стоимости или риска">Mitigation</abbr>:
- Использовать дешёвую модель для Evaluator (детерминированная проверка не требует GPT-4)
- Рано останавливаться: если 2 попытки не помогли — третья вряд ли поможет
- Кешировать рефлексии для похожих задач

## 11. Итоги

Reflexion-паттерн — это ReAct + самооценка + эпизодическая память. Ключевые выводы:

1. **Внешний верификатор — обязателен**. Без него self-correction ухудшает результат ([Huang et al., 2023](https://arxiv.org/abs/2310.01798)). С ним — Reflexion даёт +11% на HumanEval и +22% на AlfWorld.

2. **Эпизодическая память — ключевое отличие от Self-Refine**. Не просто <abbr title="Цикл «критикуй-улучшай»: LLM оценивает свой вывод и перегенерирует его с учётом замечаний">critique-refine</abbr>, а накопление вербальных уроков между попытками. Это превращает одноразового агента в обучающегося.

3. **Не панацея**. Reflexion не работает на задачах без объективного критерия успеха и может ухудшить результат при высокой начальной точности.

4. **В Eino — compose.Graph с циклом**. Workflow не подходит (нет циклов). Graph + `AddEdge("reflector", "actor")` + `WithMaxRunSteps` — естественная реализация.

Что дальше? В [Части 4]({{< ref "ai-agent-design-patterns-4-multi-agent" >}}) — Multi-Agent паттерны: когда один агент не справляется, и нужна команда. А тему памяти агентов — долгосрочной, эпизодической, семантической — я подробно разберу в Части 6.

{{< collapse title="📚 Источники" >}}

- **Reflexion**: Shinn, N., Cassano, F., Labash, A., Gopinath, A., Narasimhan, K., & Yao, S. (2023). [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366). NeurIPS 2023.
- **Self-Refine**: Madaan, A., Tandon, N., Gupta, P., Hallinan, S., Gao, L., Wiegreffe, S., ... & Yang, D. (2023). [Self-Refine: Iterative Refinement with Self-Feedback](https://arxiv.org/abs/2303.17651). NeurIPS 2023.
- **Constitutional AI**: Bai, Y., Kadavath, S., Kundu, S., Askell, A., Kernion, J., Jones, A., ... & Kaplan, J. (2022). [Constitutional AI: Harmlessness from AI Feedback](https://arxiv.org/abs/2212.08073). Anthropic.
- **Huang et al.**: Huang, J., Chen, X., Mishra, S., Zheng, H. S., Yu, A. W., Song, Q., & Zhou, D. (2023). [Large Language Models Cannot Self-Correct Reasoning Yet](https://arxiv.org/abs/2310.01798). ICLR 2024.
- **Reflect, Retry, Reward**: Bensal, Y., Kim, S., Bosselut, A., & Guha, N. (2025). [Reflect, Retry, Reward: Training LLM Agents to Reflect and Retry with Reward-Guided Self-Reflection](https://arxiv.org/abs/2505.24726).
- **Eino**: CloudWeGo. [Eino: The ultimate LLM/AI application development framework in Go](https://github.com/cloudwego/eino).
- **deer-go**: CloudWeGo. [DEER-flow Go implementation](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/deer-go).

{{< /collapse >}}
