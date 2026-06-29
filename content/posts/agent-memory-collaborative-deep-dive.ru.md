---
title: "Память агентов: от изолированного контекста к коллаборативной памяти"
date: 2026-06-29
draft: false
tags: ["ai", "agents", "llm", "memory"]
categories: ["ai", "engineering"]
summary: "Глубокий разбор архитектуры памяти LLM-агентов на основе статьи MemRec (ACL 2026): collaborative memory, decoupling управления памятью от reasoning, Information Bottleneck. Сравнение TOP-3 production-инструментов — Mem0, Letta (MemGPT) и Zep — по девяти осям. Подсветка gap'а: research опережает tooling."
ShowToc: true
---

Память — фундаментальный компонент LLM-агентов, но индустрия застряла на изолированной парадигме: агент помнит только свой собственный опыт. Статья MemRec (ACL 2026) вводит следующий milestone — collaborative memory, где память агентов связана в граф и обменивается реляционными сигналами. Я разберу MemRec до формул и промптов, сравню три production-инструмента (Mem0, Letta, Zep) и покажу, где research опережает tooling.

## 1. Почему изолированная память — это тупик

Зачем агенту память вообще? Казалось бы, контекстное окно LLM растёт — GPT-4o держит 128K токенов, Gemini — миллион. Но контекстное окно — это не память. Это рабочий стол. Положить на рабочий стол все когда-либо увиденные диалоги, все предпочтения пользователя, все выводы инструментов — рабочий стол сломается. LLM начнёт терять релевантные факты в шуме, как показало исследование [Lost in the Middle](https://arxiv.org/abs/2307.03172) (Liu et al., 2024): производительность падает, когда нужная информация находится в середине длинного контекста.

Память решает другую задачу — она делает агента stateful (состояние сохраняется между запусками), позволяет накапливать опыт и адаптироваться. Без памяти агент — чистая функция: одни и те же входы дают одни и те же выходы. С памятью агент эволюционирует. Этот принцип хорошо описан в работе [Generative Agents](https://arxiv.org/abs/2304.03442) (Park et al., 2023), где симуляция жителей виртуального города показала, что именно memory stream делает поведение агентов правдоподобным.

Эволюция памяти агентов прошла три milestone'а. Покажу на схеме:

{{< mermaid >}}
timeline
    title Эволюция парадигм памяти LLM-агентов
    section No Memory
        2023 : Vanilla LLM : Полагается на inherent knowledge<br/>каждый запуск с чистого листа
    section Static Memory
        2024 : iAgent, Chat-Rec : Retrieval из фиксированного<br/>хранилища, без обновления
    section Dynamic Memory
        2025 : AgentCF, i2Agent : Агент итеративно обновляет<br/>своё понимание
    section Collaborative Memory
        2026 : MemRec (ACL 2026) : Память связана в граф<br/>коллаборативные сигналы
{{< /mermaid >}}

Каждый milestone решал конкретную проблему, но все три держали память **в изоляции**. Что это значит на практике? Рассмотрим на примере рекомендательных систем — именно там эта проблема выражена наиболее чётко, и именно там MemRec показывает решение, которое обобщается на произвольных агентов.

В agentic recommender systems (RS) агент хранит память о пользователе {{< katex >}}M_u{{< /katex >}} — нарратив о прошлых взаимодействиях, предпочтениях. И память об айтеме {{< katex >}}M_i{{< /katex >}} — семантическое описание. Когда агент рекомендует книгу пользователю, он видит только {{< katex >}}M_u{{< /katex >}} этого пользователя. Но что если другой пользователь с похожими вкусами только что оценил книгу, которую наш пользователь ещё не видел? Изолированный агент об этом не знает — collaborative signal (коллаборативный сигнал, реляционная информация от сообщества пользователей) отрезан.

Это не специфика рекомендательных систем. Представьте команду агентов, работающих над кодом: один агент нашёл решение проблемы с зависимостями, другой сталкивается с той же проблемой — но не знает, потому что память изолирована. Или агент поддержки: пользователь рассказал о проблеме агенту A, а потом обращается к агенту B — и B начинает с нуля. Изоляция памяти — это потеря коллективного интеллекта.

## 2. MemRec: два архитектурных сдвига

Статья [MemRec](https://arxiv.org/abs/2601.08816) (Chen et al., ACL 2026) решает проблему изоляции через два архитектурных сдвига. Но прежде чем их разбирать, нужно понять, почему naive решение не работает.

### 2.1 Наивный brute-force и почему он падает

Идея «просто дай агенту доступ к памяти всех соседей» приходит в голову первой. Если пользователь смотрел книги вместе с тысячей других пользователей — скормим агенту память всех тысячи. Проблема в том, что это вызывает две катастрофы.

**Cognitive overload (когнитивная перегрузка).** LLM не может дистиллировать сигнал из изобилия. [Lost in the Middle](https://arxiv.org/abs/2307.03172) показал, что при росте контекста LLM теряет способность находить релевантные факты. Загрузить в контекст память десятков соседей — и агент начнёт галлюцинировать, терять instruction adherence, путать чьё предпочтение чьё.

**Prohibitive collaborative updates (неподъёмные обновления).** Коллаборативная память должна эволюционировать. Когда пользователь взаимодействует с айтемом, знание должно распространиться на всех связанных соседей. Naive подход требует отдельного LLM-вызова для каждого соседа — обновить память одного пользователя значит вызвать LLM десятки раз. При real-time serving это непозволительно.

Покажу, как naive и MemRec подходят к одной задаче:

{{< mermaid >}}
graph LR
    subgraph "Naive: один проход, monolithic"
        RAW["Raw graph context<br/>1000 neighbors<br/>verbose memories"]
        MONO["Single LLM<br/>(does everything)"]
        RES1["Result:<br/>cognitive overload,<br/>hallucinations"]
        RAW --> MONO --> RES1
    end

    subgraph "MemRec: два прохода, decoupled"
        GRAPH["Raw graph<br/>G=(V,E)"]
        CUR["Curate<br/>(LLM rules filter)"]
        SYN["Synthesize<br/>(LM_Mem distill)"]
        FACETS["M_collab<br/>7 structured facets"]
        REC["LLM_Rec<br/>(reasoning only)"]
        RES2["Result:<br/>high-signal context,<br/>grounded rationales"]
        GRAPH --> CUR --> SYN --> FACETS --> REC --> RES2
    end

    style MONO fill:#c62828,color:#fff
    style RES1 fill:#c62828,color:#fff
    style CUR fill:#2e7d32,color:#fff
    style SYN fill:#2e7d32,color:#fff
    style REC fill:#1565c0,color:#fff
    style RES2 fill:#2e7d32,color:#fff
{{< /mermaid >}}

Naive подход скормит весь сырой контекст одной модели — и получит information bottleneck: модель не может одновременно поглощать и reasoning. MemRec делит задачу: curate (отфильтровать шум через domain-adaptive rules), synthesize (дистиллировать в structured facets), потом reasoning на чистом контексте. Это и есть «Curate-then-Synthesize» — два прохода сжатия по принципу Information Bottleneck.

MemRec решает обе проблемы через два архитектурных сдвига.

### 2.2 Сдвиг #1: Разделение управления памятью и вывода

MemRec разделяет две функции, которые naive агенты выполняют в одной модели:

- **LM_Mem** (lightweight language model) — управляет памятью в фоне: курирует граф, синтезирует контекст, обновляет соседей.
- **LLM_Rec** (heavyweight large language model) — выполняет финальный reasoning: ранжирует кандидатов, генерирует обоснования.

Почему нельзя мешать это в одной тяжёлой модели? Тут работает принцип Information Bottleneck (IB, Information Bottleneck — принцип сжатия, сохраняющий максимум информации, релевантной задаче, при минимизации избыточных сигналов; [Tishby et al.](https://arxiv.org/abs/physics/0004057)). Задача LM_Mem — сжать сырой граф-контекст в компактное представление, которое сохраняет максимум полезного сигнала для reasoning и отбрасывает шум. Если та же модель должна и сжимать, и reasoning — возникает information bottleneck: она не может одновременно поглощать verbose context и выполнять сложное ранжирование. Эксперимент это подтверждает: naive monolithic agent (одна модель делает всё) упирается в плато, а MemRec с decoupling даёт +34% относительного прироста H@1 на датасете Books (см. раздел 5, RQ2).

Это инженерный принцип, выходящий за рамки рекомендательных систем. Не заставляйте дорогую reasoning-модель заниматься библиотечным делом. Разделите: лёгкая модель копит контекст и дистиллирует, тяжёлая — рассуждает. Cost-эффект тоже значительный — на expensive output tokens (которые в 3-4 раза дороже input) MemRec тратит минимум: Stage-R и Stage-W heavily input-biased (input составляет ~80% от общего usage), что радикально снижает effective cost.

### 2.3 Сдвиг #2: Граф коллаборативной памяти

MemRec не просто даёт агенту доступ к памяти соседей. Он строит **memory graph** {{< katex >}}G = (\mathcal{V}, E){{< /katex >}}. Узлы {{< katex >}}\mathcal{V} = \mathcal{U} \cup \mathcal{I}{{< /katex >}} — это пользователи и айтемы, каждый узел {{< katex >}}v{{< /katex >}} хранит evolving semantic memory {{< katex >}}M_v{{< /katex >}}. Рёбра {{< katex >}}E{{< /katex >}} кодируют взаимодействия и производные отношения.

{{< mermaid >}}
graph LR
    subgraph "Изолированная память"
        U1["User A<br/>M_A"]
        U2["User B<br/>M_B"]
        I1["Item X<br/>M_X"]
        I2["Item Y<br/>M_Y"]
    end

    subgraph "Collaborative memory graph G=(V,E)"
        UA["User A<br/>M_A"]
        UB["User B<br/>M_B"]
        UC["User C<br/>M_C"]
        IX["Item X<br/>M_X"]
        IY["Item Y<br/>M_Y"]
        IZ["Item Z<br/>M_Z"]

        UA -.->|"co-engagement"| IX
        UB -.->|"co-engagement"| IX
        UA -.->|"peer similarity"| UB
        IX -.->|"related"| IY
        UC -.->|"co-engagement"| IY
        UB -.->|"co-engagement"| IZ
    end

    style UA fill:#1565c0,color:#fff
    style UB fill:#1565c0,color:#fff
    style UC fill:#1565c0,color:#fff
    style IX fill:#c62828,color:#fff
    style IY fill:#c62828,color:#fff
    style IZ fill:#c62828,color:#fff
{{< /mermaid >}}

Разница фундаментальная. Изолированная парадигма — это набор disconnected narratives {{< katex >}}\{M_u\} \cup \{M_i\}{{< /katex >}}. Collaborative парадигма — граф с high-order connectivity: сигнал может переходить от пользователя к пользователю через общие айтемы, от айтема к айтему через co-engagement. Для data-sparse пользователей (мало истории) это критично — коллаборативный граф компенсирует дефицит личной истории сигналом от похожих peers. И MemRec подтверждает: +91.4% относительного прироста H@1 для low-activity пользователей над Vanilla LLM.

## 3. Глубокий разбор MemRec: трёхстадийный конвейер

Теперь разберём архитектуру MemRec до деталей — уравнений, промптов, engineering trade-offs. Это «раскрытие» статьи, как есть.

MemRec работает в три стадии. Покажу pipeline целиком, потом каждую стадию отдельно.

{{< mermaid >}}
graph TB
    subgraph "Background (LM_Mem, lightweight)"
        G["Memory Graph<br/>G = (V, E)"]
        C["Stage 1:<br/>Collaborative Memory<br/>Retrieval"]
        P["Stage 3:<br/>Async Collaborative<br/>Propagation"]
    end

    subgraph "Foreground (LLM_Rec, heavyweight)"
        R["Stage 2:<br/>Grounded Reasoning"]
    end

    G -->|"raw neighbor<br/>memories"| C
    C -->|"M_collab<br/>distilled facets"| R
    R -->|"scores s_i,<br/>rationales r_i"| OUT["Ranked<br/>recommendations"]
    R -.->|"new interaction"| P
    P -.->|"ΔM updates<br/>to neighbors"| G

    style C fill:#2e7d32,color:#fff
    style P fill:#2e7d32,color:#fff
    style R fill:#c62828,color:#fff
    style G fill:#1565c0,color:#fff
{{< /mermaid >}}

Зелёное — LM_Mem работает в фоне, лёгкая модель. Красное — LLM_Rec работает в foreground, тяжёлая модель, только когда нужен reasoning. Синее — memory graph, персистентное состояние. Обратите внимание: propagation (Stage 3) — async, не блокирует foreground reasoning.

### 3.1 Стадия 1: Извлечение коллаборативной памяти

Цель первой стадии — взять expansive memory graph и извлечь сжатую коллаборативную память {{< katex >}}M_\text{collab}{{< /katex >}} для текущей задачи. Задача: не перегрузить reasoning-агента. Стратегия — «Отфильтровать-затем-Синтезировать», два прохода сжатия по принципу IB.

#### 3.1.1 LLM-генератор правил: офлайн-фильтрация

Первый проход — фильтрация. Проблема: как выбрать top-{{< katex >}}k{{< /katex >}} наиболее релевантных соседей из графа? Традиционные подходы — эвристики на правилах (random walk, [DeepWalk](https://arxiv.org/abs/1403.6652)) или обучаемые нейросетевые скореры (GNN attention). Оба плохи для LLM-агентов: эвристики не адаптируются к семантике домена, обучаемые скореры требуют дорогого обучения.

MemRec предлагает zero-shot парадигму: **LLM-генератор правил**. В офлайн-фазе LM_Mem анализирует статистику домена {{< katex >}}\mathcal{D}_\text{domain}{{< /katex >}} и генерирует интерпретируемые эвристические правила {{< katex >}}R_\text{domain}{{< /katex >}}:

{{< katex-block >}}
R_\text{domain} \leftarrow \text{LM}_\text{Mem}(\mathcal{D}_\text{domain} \| P_\text{meta}) \quad \text{(offline)}
{{< /katex-block >}}
{{< katex >}}P_\text{meta}{{< /katex >}} — meta-prompt, который направляет генерацию правил. Вот он (источник: MemRec, [Appendix F.1](https://arxiv.org/abs/2601.08816)):

```
Meta-Prompt Template for Rule Generation

You are an expert AI engineer specializing in recommender systems
and graph-based memory networks. Your task is to generate a set of
domain-specific heuristic rules for a collaborative neighbor pruning
algorithm. The goal is to select the top-k most relevant neighbors
(users or items) from a candidate graph to build a compact, high-signal
context for a downstream LLM recommender (MemRec).

DOMAIN CONTEXT
• Domain Name: {Domain Name}
• Primary Interaction: {Primary Interaction with example}
• Key Metadata: {Key Metadata}
• Available Features:
  – edge_weight: {Domain-specific explanation}
  – recency_days: {Domain-specific explanation}
  – co_interaction_count: {Domain-specific explanation}
  – metadata_overlap_score: {Domain-specific explanation}
  – memory_similarity_score: {Domain-specific explanation}

INSTRUCTIONS:
1. Based only on the domain context provided, generate 3-5 high-priority,
   interpretable ranking rules.
2. The rules should explain how to combine or prioritize the available
   features to find the best neighbors for this specific domain.
3. Be specific about thresholds and weights.
4. Consider domain-specific characteristics (e.g., books are content-driven
   with long-term preferences; movies are recency-critical).

OUTPUT FORMAT:
Rule 1: [Your rule here]
Rule 2: [Your rule here]
...
```

Что важно: правила адаптивны к домену. Для Books (контент-driven) LM_Mem генерирует boost для metadata_overlap (>0.6 → множитель 2.5x) — книги читают по жанру/автору. Для MovieTV (зависит от давности) — сильное затухание по давности (exp(-0.018 × recency_days)). Для Yelp (категорийный) — доминирование категорий (metadata_overlap > 0.7 → 3.5x). Это zero-shot: никакого обучения, только семантическое понимание LLM.

Во время вывода правила работают как высокоскоростной фильтр — выбирают top-{{< katex >}}k{{< /katex >}} соседей за миллисекунды:

{{< katex-block >}}
N'_k(u) = \text{Curate}(N(u), R_\text{domain}, k)
{{< /katex-block >}}
Количественная валидация: LLM-фильтрация **сокращает иррелевантных item-соседей на 73.8%** против generic-эвристики, при этом сохраняет на 6.4% больше user-соседей с низким ID-overlap — потому что LLM находит семантическое сходство в нарративах памяти (два пользователя, любящих «dystopian YA novels», без общих кликов).

#### 3.1.2 Синтез коллаборативной памяти: дистилляция в грани

Второй проход — синтез. Отобранные соседи {{< katex >}}N'_k{{< /katex >}} всё ещё слишком подробные. LM_Mem дистиллирует их в структурированные грани предпочтений {{< katex >}}\{F\}{{< /katex >}} — это и есть {{< katex >}}M_\text{collab}{{< /katex >}}:

{{< katex-block >}}
M_\text{collab} = \{F\} \leftarrow \text{LM}_\text{Mem}(\text{Rep}(N'_k) \| M_u^{t-1} \| P_\text{synth})
{{< /katex-block >}}
{{< katex >}}\text{Rep}(N'_k){{< /katex >}} — многоуровневое представление: целевой пользователь {{< katex >}}u{{< /katex >}} представлен полной памятью {{< katex >}}M_u^{t-1}{{< /katex >}}, а соседи — компактными контекстными представлениями (сжатые сигналы, а не развёрнутые истории). {{< katex >}}P_\text{synth}{{< /katex >}} — промпт синтеза (источник: MemRec, [Appendix F.3](https://arxiv.org/abs/2601.08816)):

```
Stage-R Prompt: Collaborative Memory Synthesis

You are an intelligent memory retrieval system for personalized
recommendation. Your task is to analyze the user's personal memory
and collaborative memories from their neighbors to extract preference
facets.

Target User: User {user_id}

User's Personal Memory:
User Memory Summary: {user_memory_summary}

Collaborative Neighbor Memories:
The following neighboring users and items provide collaborative signals:
Collaborative Neighbors: {formatted_neighbor_list}

Your Task:
Analyze the user's personal memory and the collaborative memories
to identify {n_facets} distinct preference facets.

For each preference facet, provide:
1. A concise natural language description of the preference
2. A confidence score between 0 and 1
3. A list of supporting neighbors (user IDs or item IDs)

Output: JSON with "facets" array and "support_edges" array.
```

Результат — вместо «Пользователь предпочитает антиутопии» (изолированная память) получаем:

> **Тема:** Киберпанк и корпоративная антиутопия (уверенность: 0.9)  
> Основание: Сосед-пользователь (ID: 2057) проявляет глубокий интерес к корпоративному контролю; сосед-айтем («1984») разделяет фундаментальные антиутопические темы.  
> **Тема:** Выживание с высокими ставками (уверенность: 0.75)  
> Основание: Сосед-айтем («Battle Royale») обладает сильными элементами выживания, соответствующими недавним взаимодействиям.

Каждая грань обоснована конкретными соседями. Это и есть дистиллированный высокоинформативный контекст, который пойдёт в LLM_Rec.

### 3.2 Стадия 2: Обоснованный вывод

Вторая стадия — финальное рассуждение. LLM_Rec получает синтезированную память {{< katex >}}M_\text{collab}{{< /katex >}}, инструкцию пользователя {{< katex >}}\mathcal{I}_u{{< /katex >}} и память кандидатов {{< katex >}}C_\text{info}{{< /katex >}}:

{{< katex-block >}}
\{s_i, r_i\}_{i=1}^{N} \leftarrow \text{LLM}_\text{Rec}(\mathcal{I}_u \| M_\text{collab} \| C_\text{info} \| P_\text{rerank})
{{< /katex-block >}}
Для каждого кандидата LLM_Rec генерирует оценку релевантности {{< katex >}}s_i{{< /katex >}} (0-1) и обоснование на естественном языке {{< katex >}}r_i{{< /katex >}}. Обоснование в {{< katex >}}M_\text{collab}{{< /katex >}} гарантирует, что rationale подкреплён свидетельствами сообщества, а не выдуман. Промпт ранжирования (источник: MemRec, [Appendix F.4](https://arxiv.org/abs/2601.08816)):

```
Stage-ReRank Prompt (MemRec Mode)

You are an intelligent recommendation scoring system. Your task is
to evaluate how well each candidate item matches the target user's
preferences based on their personal memory and collaborative signals.

Target User: User {user_id}
User's Current Request: {instruction}

User Preferences (Extracted from Collaborative Memories):
{formatted_facets}

Candidate Item Memories:
{formatted_item_memories}

Your Task:
For each candidate item, provide a relevance score between 0 and 1:
• 1.0 = Excellent match, highly aligned with user's facets
• 0.5 = Moderate match, partially relevant
• 0.0 = Poor match, not aligned

For each item, provide a brief rationale explaining your scoring.

Output: JSON with "scores" array of {item_id, score, rationale}.
```

И тут начинается самое интересное. LLM_Rec не видит сырой граф. Он видит только дистиллированные грани с оценками уверенности и свидетельствами. Когнитивная перегрузка невозможна по построению — контекст уже отфильтрован и синтезирован на стадии 1.

### 3.3 Стадия 3: Асинхронное коллаборативное распространение

Третья стадия — эволюция графа. Когда пользователь {{< katex >}}u{{< /katex >}} взаимодействует с айтемом {{< katex >}}i_c{{< /katex >}}, память должна обновиться: пользователь узнал что-то новое, айтем получил новый сигнал, и — ключевое — связанные соседи должны получить распространение знаний.

Проблема: наивный синхронный подход вызывает LLM для каждого соседа отдельно — {{< katex >}}O(|N'_k|){{< /katex >}} вызовов на одно взаимодействие, с повторением контекста пользователя в каждом. MemRec добивается сложности {{< katex >}}O(1){{< /katex >}}.

Как? Вдохновлено [Label Propagation](https://www.semanticscholar.org/paper/Learning-from-labeled-and-unlabeled-data-with-label-Zhu-Ghahramani) (Zhu & Ghahramani, 2002) — алгоритмом, где метки распространяются по графу от узла к узлу. MemRec распространяет «семантические метки» (инсайты, выводы) по графу памяти. Но вместо отдельных вызовов для каждого узла, MemRec концептуально декомпозирует обновление:

{{< katex-block >}}
M_u^t, M_{i_c}^t \leftarrow \text{LM}_\text{Mem}(M_\text{collab} \| M_u^{t-1} \| M_{i_c}^{t-1} \| P_\text{update}) \quad \text{(self-reflection)}
{{< /katex-block >}}
{{< katex-block >}}
\{\Delta M_\text{neigh}\} \leftarrow \text{LM}_\text{Mem}(M_\text{collab} \| M_u^{t-1} \| M_{i_c}^{t-1} \| N'_k(u) \| P_\text{update}) \quad \text{(neighbor propagation)}
{{< /katex-block >}}
И выполняет оба шага в **одном пакетном асинхронном вызове** с единым промптом {{< katex >}}P_\text{update}{{< /katex >}} (источник: MemRec, [Appendix F.5](https://arxiv.org/abs/2601.08816)):

```
Stage-W Prompt: Asynchronous Collaborative Propagation

You are an intelligent memory management system for collaborative
recommendation. Your task is to update the personal memories of the
user, the clicked item, and relevant collaborative neighbors based
on this new interaction.

Interaction Context:
User {user_id} has just interacted with (clicked) Item {item_id}.

User Preferences (Extracted from Collaborative Memories):
{formatted_facets}

Current Personal Memory of User {user_id}: {current_user_memory}
Current Memory of Item {item_id}: {current_item_memory}

Collaborative Neighbors Available for Memory Propagation:
{n_neighbors} neighbors: {formatted_neighbors}

Your Task — Generate UPDATED memories for:
1. The current user (synthesize current memory + facets + clicked item)
2. The clicked item (describe what it is and who might enjoy it)
3. Selected neighbors (collaborative propagation is key!)
   * Select neighbors RELEVANT to this interaction
   * Update their memories to reflect new insights
   * This helps the system learn collaboratively!

Output: JSON with "user_memory", "item_memory", "neighbor_updates".
```

Один вызов LM_Mem обновляет пользователя, айтем и выбранных соседей одновременно. Асинхронно — не блокирует основной вывод. Сложность {{< katex >}}O(1){{< /katex >}}. Это разрешает проблему неподъёмных обновлений.

{{< mermaid >}}
graph TB
    subgraph "Naive: O(|N_k'|) calls"
        N1["User update<br/>call 1"]
        N2["Item update<br/>call 2"]
        N3["Neighbor 1<br/>call 3"]
        N4["Neighbor 2<br/>call 4"]
        N5["Neighbor k<br/>call k+2"]
        N1 --> N2 --> N3 --> N4 --> N5
    end

    subgraph "MemRec: O(1) batched async"
        B["Single LM_Mem call<br/>P_update"]
        B -->|"user_memory"| UU["User M_u^t"]
        B -->|"item_memory"| II["Item M_i^t"]
        B -->|"neighbor_updates"| NN["ΔM_neigh<br/>(selected neighbors)"]
    end

    style B fill:#2e7d32,color:#fff
{{< /mermaid >}}

Наивный подход: линейное число вызовов, каждый повторяет контекст пользователя. MemRec: один пакетный вызов, всё в одном промпте. Разница в задержке и стоимости — на порядки.

## 4. Эксперименты MemRec: что показали цифры

MemRec оценивается на четырёх benchmark-датасетах: Amazon Books, Amazon Goodreads, MovieTV, Yelp. Разная плотность взаимодействия, разные domain characteristics. Метрики — Hit Rate (H@K) и NDCG (N@K) для K ∈ {1, 3, 5}. Реализация: gpt-4o-mini для обеих LM_Mem и LLM_Rec, {{< katex >}}k=16{{< /katex >}} соседей, {{< katex >}}N_f=7{{< /katex >}} facets.

### 4.1 Главный результат (RQ1)

Полная таблица результатов для всех четырёх датасетов:

| Model | Books H@1 | Books H@5 | Goodreads H@1 | Goodreads H@5 | MovieTV H@1 | MovieTV H@5 | Yelp H@1 | Yelp H@5 |
|-------|-----------|-----------|---------------|---------------|-------------|-------------|----------|----------|
| LightGCN | 0.1753 | 0.5703 | 0.2499 | 0.7903 | 0.3482 | 0.6883 | 0.3444 | 0.7546 |
| SASRec | 0.0914 | 0.4845 | 0.1324 | 0.5407 | 0.3399 | 0.6382 | 0.2305 | 0.5597 |
| P5 | 0.2192 | 0.5273 | 0.1569 | 0.5060 | 0.1696 | 0.5008 | 0.1444 | 0.5220 |
| Vanilla LLM | 0.3138 | 0.7270 | 0.2864 | 0.7390 | 0.4050 | 0.8603 | 0.1692 | 0.6861 |
| iAgent (static) | 0.3925 | 0.6905 | 0.2617 | 0.6591 | 0.4253 | 0.7420 | 0.3995 | 0.7300 |
| RecBot (dynamic) | 0.3984 | 0.6786 | 0.2705 | 0.6495 | 0.4367 | 0.7309 | 0.4007 | 0.7169 |
| AgentCF (dynamic) | 0.3457 | 0.7403 | 0.2951 | 0.7726 | 0.3906 | 0.7864 | 0.1925 | 0.6374 |
| i2Agent (dynamic) | 0.4453 | 0.7708 | 0.3099 | 0.7675 | 0.4912 | 0.8221 | 0.4205 | 0.7648 |
| **MemRec** | **0.5117** | **0.8007** | **0.3997** | **0.8052** | **0.5882** | **0.8817** | **0.4868** | **0.7908** |

*Источник: [MemRec, Tables 2-3](https://arxiv.org/abs/2601.08816)*

Ключевой takeaway — **иерархия парадигм**: Collaborative > Dynamic > Static > No Memory. Динамические агенты (AgentCF, i2Agent) стабильно бьют статические (iAgent), те — Vanilla LLM. Но даже SOTA dynamic agent (i2Agent) сильно уступает MemRec: на Goodreads H@1 **+28.98%** относительного прироста, на MovieTV **+19.75%**, на Yelp **+15.77%**.

### 4.2 Валидация когнитивной перегрузки (RQ2)

Это самый инженерно интересный эксперимент. MemRec сравнивается с:
- **Vanilla LLM** — без памяти
- **Naive Collaborative Agent** — monolithic, обрабатывает uncurated context в одной модели
- **MemRec** — decoupled, curate-then-synthesize

Naive Agent упирается в плато: одна модель не может одновременно поглощать verbose context и выполнять сложное ранжирование. MemRec ломает плато через decoupling — LLM_Rec получает только high-signal distilled context. Результат: **+34% относительного прироста H@1 на Books** над Naive Agent. Это прямое подтверждение, что architectural decoupling — не оптимизация, а необходимость.

### 4.3 Абляции (RQ4)

Что произойдёт, если убрать компоненты MemRec? Ablation на Books:

| Ablation | H@1 | Drop | Что значит |
|----------|-----|------|------------|
| MemRec (Full) | 0.527 | — | baseline |
| w/o Collab. Read | 0.475 | **−9.9%** | Без collaborative retrieval — агент видит только личную историю. Самый большой drop |
| w/o LLM Curation | 0.498 | −5.5% | Generic heuristics вместо domain-adaptive LLM rules. Больше шума |
| w/o Collab. Write | 0.505 | −4.2% | Без async propagation. Static graph всё ещё работает, но теряется evolving precision |

Collaborative retrieval — самый важный компонент. LLM curation бьёт generic heuristics. Async propagation критичен для top-1 precision, даже без него broad recall (H@5) остаётся высоким (0.814).

### 4.4 Устойчивость (RQ5)

MemRec не просто устойчив к data sparsity — он **наиболее полезен** именно для data-sparse пользователей. Low-activity группа получает **+91.4%** относительного прироста H@1 над Vanilla LLM. Коллаборативный граф компенсирует дефицит личной истории.

При 30% noise injection (фейковые айтемы в истории) MemRec держит H@1 = 0.491 — resilience за счёт «Curate-then-Synthesize», LLM curation фильтрует иррелевантных peers до того, как они достигнут reasoning-агента.

### 4.5 Граница Парето (RQ3)

MemRec не просто лучше — он устанавливает новый Pareto frontier между reasoning quality, online inference cost и deployment flexibility. Ключевое: тяжёлая cognitive load обработки collaborative graph offloaded в async offline batches. Online inference видит только distilled {{< katex >}}M_\text{collab}{{< /katex >}}.

Конфигурации:

| Configuration | LLM_Rec / LM_Mem | H@1 | Latency | Cost | Примечание |
|---------------|------------------|-----|---------|------|------------|
| Vanilla LLM | 4o-mini / — | 0.330 | ~5.1s | lowest | Без памяти |
| Standard | 4o-mini / 4o-mini | 0.524 | ~16.5s | low | Базовый MemRec |
| Cloud-OSS | 4o-mini / OSS-120B | 0.561 | ~11.8s | low | Near-ceiling, open-weights LM_Mem |
| Local-Qwen | 4o-mini / Qwen-2.5-7B | 0.470 | ~34.0s | low* | Privacy-sensitive, on-premise |
| Ceiling | gpt-4o / 4o-mini | 0.580 | ~10.4s | high | Peak performance |

*Локальное развёртывание — нулевая стоимость API для обслуживания памяти.

Cloud-OSS особенно интересен: открытые веса LM_Mem (gpt-oss-120b) даёт результаты, близкие к потолку, при низкой стоимости. Это значит, что управление памятью можно полностью держать on-premise без потери качества — развёртывание с сохранением приватности реально.

Распределение токенов даёт инженерный инсайт: входные токены составляют ~80% от общего использования (вход в 3-4 раза дешевле выхода). Стадии R и W сильно смещены в сторону входа — они поглощают подробный контекст (дёшево) и продуцируют сжатые инсайты (дорого, но мало). На пользователя: ~5,100 входных + ~1,300 выходных = ~6,400 всего токенов. Эффективная стоимость радикально ниже, чем наивная оценка по общему числу токенов.

## 5. От рекомендательных систем к произвольным агентам

MemRec — статья про рекомендательные системы. Но её архитектурные паттерны универсальны. Давайте перенесём.

Явное отображение концептов:

| Рекомендательные системы | Произвольные LLM-агенты |
|---------------------------|------------------------|
| Пользователь {{< katex >}}u{{< /katex >}} | Агент {{< katex >}}a{{< /katex >}} с личной памятью {{< katex >}}M_a{{< /katex >}} |
| Айтем {{< katex >}}i{{< /katex >}} | Задача, артефакт, документ {{< katex >}}t{{< /katex >}} с памятью {{< katex >}}M_t{{< /katex >}} |
| Взаимодействие пользователь-айтем | Выполнение агентом задачи, чтение документа агентом |
| Совместная вовлечённость (общие айтемы) | Общий контекст (агенты работали над одной задачей) |
| Похожие пользователи | Агенты-коллеги (члены команды) |
| Сигнал коллаборативной фильтрации | Передача организационного знания |

Два конкретных сценария, где collaborative memory применима к произвольным агентам.

**Общая память нескольких агентов.** Команда агентов работает над проектом. Агент A решает проблему с конфликтом зависимостей в Go-модулях. Агент B сталкивается с той же проблемой через неделю. Без коллаборативной памяти — B начинает с нуля, тратит те же часы. С графом коллаборативной памяти — взаимодействие A→«решение конфликта зависимостей» распространяется к агенту B через общее ребро (оба работают с Go-модулями). Это не общая файловая система, это семантическое распространение: B получает дистиллированный инсайт от A, обоснованный свидетельствами.

**Организационная память.** Агент поддержки обработал 100 тикетов по интеграции SSO. Граф памяти связывает эти тикеты через общие темы. Новый агент поддержки приходит — и вместо чтения 100 тикетов получает синтезированные грани: «Подводные камни интеграции SSO: цепочка сертификатов (уверенность 0.9), несоответствие redirect URI (0.85), гонка обновления токенов (0.7)». Это и есть {{< katex >}}M_\text{collab}{{< /katex >}}, дистиллированный из организационного графа.

Звучит как фантастика? MemRec показывает, что это работает на рекомендательных системах. Перенос на произвольных агентов — инженерная задача, не исследовательский прорыв. Архитектура та же: LM_Mem курирует граф, LLM_Rec рассуждает, асинхронное распространение обновляет.

## 6. Ландшафт: три парадигмы production-памяти

Пока research прокладывал путь к collaborative memory, индустрия строила инструменты. Три парадигмы, у каждой arxiv-статья и production-след. Я выбрал именно эти три, потому что они фундаментально разные — не feature-list конкуренты, а разные модели памяти.

{{< mermaid >}}
graph TB
    subgraph "Mem0: Managed Memory Layer"
        M0["Vector store<br/>+ hybrid retrieval"]
        M0 -->|"semantic + BM25<br/>+ entity"| M0R["Fused results"]
    end

    subgraph "Letta: OS-Tiered Memory"
        L0["Core memory<br/>(in-context, RAM)"]
        L1["Archival memory<br/>(external, disk)"]
        L0 <-->|"page via<br/>function calls"| L1
    end

    subgraph "Zep: Temporal Knowledge Graph"
        Z0["Graphiti engine<br/>time-aware facts"]
        Z0 -->|"graph traversal<br/>+ temporal"| Z0R["Context with<br/>history"]
    end

    style M0 fill:#e65100,color:#fff
    style L0 fill:#4a148c,color:#fff
    style L1 fill:#4a148c,color:#fff
    style Z0 fill:#004d40,color:#fff
{{< /mermaid >}}

### 6.1 Mem0: управляемый слой памяти

[Mem0](https://arxiv.org/abs/2504.19413) (Chhikara et al., 2025) — универсальный слой памяти для AI-агентов. 59.6k звёзд на GitHub, YC S24. Идеология: память как управляемый сервис — агент не думает о хранении, Mem0 извлекает, консолидирует и находит.

Архитектурно — multi-level memory с layer'ами:

| Layer | Lifetime | Что хранит |
|-------|----------|------------|
| Conversation | один turn | In-flight сообщения, tool outputs |
| Session | минуты-часы | Контекст текущей задачи |
| User | недели-навсегда | Персонализация, preferences |
| Organizational | глобально | Shared FAQs, policies |

 retrieval многосигнальный (после April 2026 update): semantic search + BM25 keyword + entity matching, всё fused в parallel. Это даёт benchmark'и: **LoCoMo 91.6** (+20 над предыдущим алгоритмом), **LongMemEval 94.8** (+27), **BEAM 1M 64.1**, **BEAM 10M 48.6**.

Новый алгоритм (April 2026, [migration guide](https://docs.mem0.ai/migration/oss-v2-to-v3)) — single-pass ADD-only: один LLM-вызов для extraction, без UPDATE/DELETE. Memories накапливаются, ничего не перезаписывается. Agent-generated facts — first-class citizen: если агент подтвердил действие, информация сохраняется с равным весом. Entity linking — сущности извлекаются, эмбеддятся, связываются между memories для retrieval boosting.

Минимальный SDK snippet:

```python
from mem0 import Memory

memory = Memory()

# Добавить память
memory.add(
    ["Я Алексей, предпочитаю YAML и тёмную тему."],
    user_id="alex"
)

# Retrieval — hybrid: semantic + BM25 + entity
results = memory.search(
    "Какие предпочтения у Алексея?",
    filters={"user_id": "alex"},
    top_k=3
)
```

### 6.2 Letta (MemGPT): многоуровневая память по образу ОС

[MemGPT](https://arxiv.org/abs/2310.08560) (Packer et al., UC Berkeley, 2023) — seminal paper, который представил LLM как операционную систему. Идея: virtual context management, вдохновлённый hierarchical memory в традиционных ОС. Core memory (in-context, как RAM) + archival memory (external storage, как disk). Агент сам управляет data movement между tier'ами через function calls.

[Letta](https://docs.letta.com/) — production framework на базе MemGPT. White-box, model-agnostic. Базовая абстракция — **memory blocks**: structured sections контекстного окна, которые персистят между взаимодействиями.

{{< mermaid >}}
graph TB
    subgraph "Context Window (Limited)"
        SYS["System Prompt"]
        BLOCKS["Memory Blocks<br/>(Core Memory = RAM)"]
        CONV["Conversation<br/>(working memory)"]
    end

    subgraph "External Storage"
        ARCH["Archival Memory<br/>(= Disk)<br/>unlimited"]
    end

    LLM["LLM"]

    SYS --> LLM
    BLOCKS --> LLM
    CONV --> LLM

    LLM -->|"core_memory_append<br/>core_memory_replace"| BLOCKS
    LLM <-->|"archival_memory_search<br/>archival_memory_insert"| ARCH

    style BLOCKS fill:#4a148c,color:#fff
    style ARCH fill:#311b92,color:#fff
    style LLM fill:#1565c0,color:#fff
{{< /mermaid >}}

Core memory (memory blocks) — всегда в контексте, как RAM. Архив — external storage, агент «пейджит» данные через function calls (`archival_memory_search`, `archival_memory_insert`). Если block переполняется (chars_limit), агент сам решает, что вытеснить в archival. Вот что видит LLM:

```xml
<memory_blocks>
<persona>
  <description>The persona block: Stores details about your current
  persona, guiding how you behave and respond.</description>
  <metadata>- chars_current=128 - chars_limit=5000</metadata>
  <value>I am a helpful assistant named Sam. I enjoy helping users
  solve problems.</value>
</persona>
<human>
  <description>The human block: Stores key details about the person
  you are conversing with.</description>
  <value>The user's name is Alice. She is a software engineer who
  prefers concise answers.</value>
</human>
</memory_blocks>
```

Ключевые свойства memory blocks:
- **Agent-managed** — агент автономно организует информацию по block labels
- **Always visible** — блоки в контексте всегда, retrieval не нужен
- **Shareable** — несколько агентов могут читать один block (shared memory, multi-agent coordination)
- **Read-only option** — политики, read-only блоки, агент не может их менять

Benchmark: **DMR (Deep Memory Retrieval) 93.4%** — baseline, который Zep превзошёл.

Минимальный snippet:

```python
from letta import Letta

client = Letta()

agent = client.agents.create(
    name="memory_agent",
    memory_blocks=[
        {"label": "human", "value": "User prefers Go and concise answers."},
        {"label": "persona", "value": "I am a senior engineering assistant."},
    ],
    # archival memory — external, agent pages via function calls
)
```

### 6.3 Zep: темпоральный граф знаний

[Zep](https://arxiv.org/abs/2501.13956) (Rasmussen et al., 2025) — memory layer service на базе Graphiti. Graphiti — open-source framework для **temporal knowledge graphs** (контекстных графов с временной осью). В отличие от статического RAG, Graphiti делает real-time incremental updates: отношения и факты эволюционируют вместе с данными, без batch recomputation.

Архитектура: dynamic synthesis — одновременно обрабатывает unstructured conversational data и structured business data, сохраняя historical relationships. Факт в графе имеет time validity: когда он стал истинным, когда перестал. Запрос может reasoning'ить над тем, **как факты менялись во времени**, а не только что истинно сейчас.

{{< mermaid >}}
graph LR
    subgraph "Input"
        CONV["Conversational data<br/>(unstructured)"]
        BIZ["Business data<br/>(structured)"]
    end

    subgraph "Graphiti Engine"
        ENT["Entity<br/>Extraction"]
        EDGE["Edge<br/>Creation"]
        TEMP["Temporal<br/>Annotation"]
        GRAPH["Temporal<br/>Knowledge Graph"]
    end

    subgraph "Output"
        FACTS["Facts with<br/>time validity"]
        Q["Query:<br/>'what changed<br/>since X?'"]
    end

    CONV --> ENT
    BIZ --> ENT
    ENT --> EDGE --> TEMP --> GRAPH
    GRAPH --> FACTS
    FACTS --> Q

    style GRAPH fill:#004d40,color:#fff
    style TEMP fill:#00695c,color:#fff
{{< /mermaid >}}

Каждый факт в графе annotated временными метками: когда стал истинным (valid_from), когда перестал (valid_to). Запрос «что пользователь делал в марте?» traversal'ит граф по temporal edges, а не просто semantic similarity. Это качественно другой retrieval — не «найди похожее», а «найди, что было истинно в момент X».

Benchmark'и:
- **DMR: 94.8%** vs MemGPT 93.4% — Zep превосходит на benchmark'е, который MemGPT team установила как primary metric
- **LongMemEval: +18.5% accuracy, −90% latency** vs baseline — особенно сильна в cross-session synthesis и long-term context maintenance

Минимальный snippet:

```python
from zep_cloud import Zep

client = Zep(api_key="...")

# Добавить episode — Graphiti синхронизирует граф
client.memory.add(
    session_id="session-1",
    messages=[{"role": "user", "content": "Я перешёл с PostgreSQL на SQLite для проекта X."}]
)

# Retrieval — graph traversal + temporal reasoning
context = client.memory.get(session_id="session-1")
# context содержит facts с temporal validity
```

### 6.4 Новые направления: A-MEM и LangMem

[Mem0](https://arxiv.org/abs/2504.19413), [Letta](https://arxiv.org/abs/2310.08560), [Zep](https://arxiv.org/abs/2501.13956) — три production-инструмента. Но research не стоит на месте.

**A-MEM** ([arXiv:2502.12110](https://arxiv.org/abs/2502.12110), NeurIPS 2025) — agentic memory system following Zettelkasten principles. Память организует себя: dynamic indexing и linking создают interconnected knowledge networks. Это research-направление, production-следа пока нет, но идея — memory как self-organizing систему — перекликается с collaborative propagation в MemRec.

**RecMem** ([arXiv:2605.16045](https://arxiv.org/abs/2605.16045), ACL 2026 Findings) — переосмысливает не ЧТО хранить, а КОГДА консолидировать. Существующие системы eagerly вызывают LLM для каждого взаимодействия — дорого. RecMem копит взаимодействия в subconscious-слое (дешёвые embeddings), и вызывает LLM для извлечения эпизодической и семантической памяти только при устойчивом повторении семантически похожих взаимодействий — аналог фазового перехода. Результат: **до 87% снижения стоимости токенов** при превышении точности трёх SOTA-систем. Это прямой ответ на тот же cost-вызов, что и MemRec, но через density-triggered consolidation вместо async propagation. Наглядный разбор архитектуры — на YouTube: [«Фазовые переходы в памяти ИИ-агентов: архитектура REC Memory»](https://www.youtube.com/watch?v=8MmxGVwFr_g).

**LangMem** ([langchain-ai/langmem](https://github.com/langchain-ai/langmem)) — SDK-level primitives для LangGraph. Semantic, episodic, procedural memory как functional building blocks. Не standalone система, а библиотека для тех, кто строит свою memory architecture поверх LangGraph storage layer.

## 7. Сравнение TOP-3: девять осей

Теперь сравним Mem0, Letta и Zep системно — по девяти осям. Каждая ось — архитектурное решение, не feature.

| Ось | Mem0 | Letta (MemGPT) | Zep |
|-----|------|----------------|-----|
| **1. Модель памяти** | Vector store + entity graph | OS-tiered (core=RAM, archival=disk) | Temporal knowledge graph (Graphiti) |
| **2. Архитектура** | Managed layer (service/SDK) | In-process agent (framework) | Service (Graphiti engine) |
| **3. Retrieval** | Hybrid: semantic + BM25 + entity (parallel fused) | Hierarchical paging (function calls) | Graph traversal + temporal reasoning |
| **4. Temporal awareness** | Added Apr 2026 (time-aware retrieval) | Нет | **First-class** (факты с time validity) |
| **5. Update cost** | Single-pass ADD-only (1 LLM call) | Agent self-edits via function calls | Real-time incremental (no batch) |
| **6. Collaborative memory** | **Нет** (entity linking — intra-agent) | Partial (shared blocks — multi-agent) | **Нет** (graph per-session) |
| **7. Benchmarks** | LoCoMo 91.6, LongMemEval 94.8, BEAM 1M 64.1 | DMR 93.4 | DMR 94.8, LongMemEval +18.5% |
| **8. Production** | Self-host (Docker) / Cloud / CLI | Self-host (framework) / Cloud | Self-host / Cloud |
| **9. Ergonomics** | SDK (Python/TS), CLI, agent skills | SDK (Python/TS), white-box, model-agnostic | SDK (Python/TS/Go) |

### 7.1 Сводная таблица бенчмарков

Benchmarks — болезненная тема. LoCoMo, LongMemEval, DMR, BEAM — разные benchmark'и, измеряющие разные вещи. Direct comparison валиден только на shared benchmark'ах.

| Benchmark | Что измеряет | Mem0 | Letta | Zep |
|-----------|--------------|------|-------|-----|
| **LoCoMo** | Long conversational memory (single/multi-hop, temporal, open-domain) | **91.6** | — | — |
| **LongMemEval** | Cross-session synthesis, long-term context | **94.8** | — | **+18.5%** acc, **−90%** latency |
| **DMR** | Deep Memory Retrieval | — | **93.4%** | **94.8%** |
| **BEAM** | Million-token scale memory | **64.1** (1M), **48.6** (10M) | — | — |

Shared benchmark'и: LongMemEval у Mem0 и Zep (прямое сравнение возможно); DMR у Letta и Zep (Zep превосходит 94.8 vs 93.4). LoCoMo и BEAM — Mem0-only. **Глобальное утверждение «X лучше Y» невозможно** — только per-benchmark.

### 7.2 Подсветка: коллаборативная память — разрыв

Ось 6 — самая важная для этой статьи. **Ни Mem0, ни Letta, ни Zep не реализуют collaborative memory** в том смысле, как её определяет MemRec (memory graph с cross-agent/cross-entity реляционным signal transfer).

Уточню нюансы:
- **Mem0** entity linking — intra-agent: сущности связываются внутри памяти одного пользователя, не между агентами. Это boosting retrieval, не collaborative signal transfer.
- **Letta** shared blocks — multi-agent coordination: несколько агентов читают один read-only block. Это shared state, но не collaborative propagation — нет эволюции графа.
- **Zep** temporal graph — per-session: граф строится для одной сессии/пользователя, нет cross-agent graph с propagation.

MemRec показывает, что collaborative memory даёт +14-29% H@1. Это не маржинальная оптимизация — это качественный скачок, и ни один production-инструмент его не делает. Подробнее в разделе 8.

## 8. Research vs Tools: где разрыв

Коллаборативная память — не единственный gap. Research frontier (MemRec) опережает tooling (TOP-3) по четырём направлениям. Разберу каждое: что говорит research, что делают инструменты, что внедрять сейчас.

{{< mermaid >}}
graph TB
    subgraph "Research Frontier (MemRec)"
        R1["Collaborative memory<br/>graph G=(V,E)"]
        R2["Architectural decoupling<br/>LM_Mem / LLM_Rec"]
        R3["Temporal-aware<br/>facts + history"]
        R4["Curate-then-Synthesize<br/>IB-grounded distillation"]
    end

    subgraph "Tooling Reality"
        T1["Mem0: intra-agent<br/>entity linking"]
        T2["Letta: in-process<br/>no decoupling"]
        T3["Zep: first-class<br/>Mem0: added 2026<br/>Letta: none"]
        T4["Mem0: consolidation<br/>Letta: agent-managed<br/>Zep: graph synthesis"]
    end

    R1 -.->|"GAP"| T1
    R2 -.->|"PARTIAL"| T2
    R3 -.->|"Zep: COVERED"| T3
    R4 -.->|"AD HOC"| T4

    style R1 fill:#c62828,color:#fff
    style T1 fill:#c62828,color:#fff
    style R2 fill:#f9a825,color:#000
    style T2 fill:#f9a825,color:#000
{{< /mermaid >}}

### Разрыв 1: Коллаборативная память — самый большой

**Что говорит исследование:** MemRec показывает +14-29% H@1 от collaborative signal. Memory graph {{< katex >}}G=(V,E){{< /katex >}} с high-order connectivity передаёт реляционные сигналы между агентами/айтемами ([Chen et al., 2026](https://arxiv.org/abs/2601.08816)).

**Статус инструментов:** Ни один из TOP-3 не делает коллаборативную память. Entity linking у Mem0 — внутри одного агента. Shared blocks у Letta — общее состояние без распространения. Темпоральный граф у Zep — в рамках сессии.

**Внедрять сейчас:** Если нужен коллаборативный сигнал — собирать самостоятельно. Архитектура MemRec переносима: LM_Mem курирует граф, LLM_Rec рассуждает, асинхронное распространение обновляет. Никакой магии, чистая инженерия. Либо ждать, пока исследования дойдут до инструментов — судя по темпам (MemRec ACL 2026, A-MEM NeurIPS 2025), 12-18 месяцев.

### Разрыв 2: Разделение архитектуры — частичное

**Что говорит исследование:** Разделение управления памятью (LM_Mem) и вывода (LLM_Rec) разрешает когнитивную перегрузку (+34% H@1 над наивной монолитной моделью). Основанный на IB «Отфильтровать-затем-Синтезировать» — два прохода сжатия.

**Статус инструментов:** Частично. У Mem0 извлечение/поиск отделены от reasoning-LLM, но не как архитектурный принцип первого класса с IB-обоснованием. Letta — в процессе, агент сам редактирует memory blocks, никакого разделения. Zep — движок Graphiti отделён от вывода, но без явных стадий дистилляции.

**Внедрять сейчас:** Mem0 ближе всех. Если важен trade-off стоимость/задержка — разделение Mem0 даёт часть выгоды. Полное разделение как в MemRec — собирать самостоятельно.

### Разрыв 3: Темпоральность — Zep first-class

**Что говорит исследование:** Факты эволюционируют. Запрос должен рассуждать над тем, **как** факты менялись, не только что истинно сейчас.

**Статус инструментов:** Zep — первого класса (темпоральный граф Graphiti, факты со временем жизни). Mem0 — добавлено в апреле 2026 (поиск с учётом времени, но не первого класса как у Zep). Letta — нет темпоральности.

**Внедрять сейчас:** Zep. Если темпоральный вывод критичен (enterprise-сценарии, синтез между сессиями) — Zep единственный вариант первого класса. LongMemEval +18.5% точности / −90% задержки — прямое подтверждение.

### Разрыв 4: Дистилляция — ad hoc

**Что говорит исследование:** «Отфильтровать-затем-Синтезировать» — основанный на IB, доменно-адаптивные LLM-правила фильтрации, структурированные грани с уверенностью и свидетельствами. Единый подход.

**Статус инструментов:** У каждого по-своему. Mem0 — консолидация (однослойная экстракция, но без структурированных граней). Letta — агент сам управляет (агент решает, что в core memory). Zep — синтез графа (Graphiti строит граф, но не в формате граней). Нет единого подхода к дистилляции.

**Внедрять сейчас:** Зависит от сценария. Консолидация Mem0 — для простой персонализации. Синтез графа Zep — для сложного реляционного вывода. Структурированные грани как в MemRec — собирать самостоятельно.

## 9. Инженерный фреймворк выбора

Какой инструмент выбрать? Зависит от четырёх характеристик use case.

{{< mermaid >}}
graph TB
    START["Use case needs<br/>agent memory?"]

    Q1{"Temporal reasoning<br/>critical?"}
    Q2{"Scale > 1M<br/>memories?"}
    Q3{"Multi-agent<br/>coordination?"}
    Q4{"Collaborative<br/>memory needed?"}

    ZEP["Zep<br/>(temporal graph)"]
    MEM0["Mem0<br/>(managed layer)"]
    LETTA["Letta<br/>(OS-tiered)"]
    CUSTOM["Custom build<br/>MemRec-style architecture<br/>or wait for tooling"]

    START --> Q1
    Q1 -->|"Да"| ZEP
    Q1 -->|"Нет"| Q2
    Q2 -->|"Да"| MEM0
    Q2 -->|"Нет"| Q3
    Q3 -->|"Да"| LETTA
    Q3 -->|"Нет"| Q4
    Q4 -->|"Да"| CUSTOM
    Q4 -->|"Нет"| MEM0

    style ZEP fill:#004d40,color:#fff
    style MEM0 fill:#e65100,color:#fff
    style LETTA fill:#4a148c,color:#fff
    style CUSTOM fill:#c62828,color:#fff
{{< /mermaid >}}

Краткая summary по веткам:

| Характеристика сценария | Рекомендация | Почему |
|------------------------|--------------|-------|
| Темпоральный вывод критичен | **Zep** | Темпоральный граф первого класса, DMR 94.8%, LongMemEval +18.5% |
| Масштаб > 1M памятей, персонализация | **Mem0** | BEAM 1M 64.1, управляемый слой, гибридный поиск, 59.6k звёзд, обкатан в продакшене |
| Координация нескольких агентов | **Letta** | Общие memory blocks, многоуровневая память, агент сам управляет |
| Нужна коллаборативная память | **Собирать самостоятельно** | Ни один из TOP-3 не делает коллаборативную память. Архитектура MemRec переносима. |
| Простая персонализация, быстрый старт | **Mem0** | Лучшая эргономика, CLI, agent skills, cloud + self-host |

Важная оговорка: decision tree упрощён. Реальный выбор зависит от deployment model (on-premise vs cloud), latency requirements, compliance. Mem0 Cloud-OSS config показывает, что open-weights LM_Mem даёт near-ceiling results — privacy-preserving deployment реален.

## 10. Открытые проблемы и куда движется research

Статья подходит к концу, но тема — нет. Несколько открытых проблем, которые определят следующие 12-18 месяцев.

**Collaborative memory across agents.** MemRec показал collaborative memory в рамках одной системы (рекомендательной). Перенос на multi-agent systems — открытая задача. Как propagate insights между агентами с разными specializations? Как избежать noise при propagation? A-MEM (NeurIPS 2025) указывает направление — self-organizing memory через Zettelkasten — но production-инструментов пока нет.

**Memory consolidation / dreaming.** Anthropic и OpenAI в 2026 shipped background memory consolidation — так называемое «Dreaming». Идея: агент «спит» между сессиями и консолидирует память — компрессия, deduplication, extracting patterns. Это перекликается с async propagation MemRec, но на другом масштабе: не propagation между узлами графа, а consolidation всей памяти агента. Подробно механика Dreaming (Anthropic Auto Dream, OpenAI Dreaming V3 с 82.8% factual recall) разобрана в серии AI Agent Design Patterns, Part 6 — не буду дублировать.

**Evaluation gaps.** Нет единого benchmark'а для collaborative memory. LoCoMo, LongMemEval, DMR — single-agent benchmarks. MemRec использует RS-specific metrics (H@K, N@K). Как оценить collaborative memory для arbitrary agents? Это research-задача — без benchmark'а нет сравнения, без сравнения нет прогресса.

**Privacy-preserving federated memory.** MemRec в limitations указывает: future work — federated memory updates. Если коллаборативный граф распределён между организациями (каждая со своим NDA) — как transfer insights без transfer raw data? Federated learning для memory graphs — открытая область.

## Заключение

Память LLM-агентов прошла путь от полного отсутствия к dynamic isolated memory. MemRec (ACL 2026) указывает следующий milestone — collaborative memory, где память агентов связана в граф и обменивается реляционными сигналами. Архитектурно это означает: decouple memory management от reasoning, build memory graph, curate-then-synthesize для distillation, async propagation для эволюции.

Три production-инструмента — Mem0, Letta, Zep — представляют три разные парадигмы изолированной памяти: управляемый слой, многоуровневая по образу ОС, темпоральный граф. Все три годятся для однозадачных сценариев. Но ни один не делает коллаборативную память, которую MemRec доказал как следующий шаг (+14-29% H@1).

Исследования опережают инструменты. Это нормально — исследовательский фронт двигается быстрее внедрения в продакшен. Для инженера сегодня: выбирайте из TOP-3 по характеристикам сценария, но помните, что коллаборативная память — следующая волна, и MemRec показывает, как её строить.

Ссылки по статье:
- [MemRec: Collaborative Memory-Augmented Agentic Recommender System](https://arxiv.org/abs/2601.08816) — Chen et al., ACL 2026
- [Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory](https://arxiv.org/abs/2504.19413) — Chhikara et al., 2025
- [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) — Packer et al., 2023
- [Zep: A Temporal Knowledge Graph Architecture for Agent Memory](https://arxiv.org/abs/2501.13956) — Rasmussen et al., 2025
- [A-MEM: Agentic Memory for LLM Agents](https://arxiv.org/abs/2502.12110) — Xu et al., NeurIPS 2025
- [RecMem: Recurrence-based Memory Consolidation for Efficient and Effective Long-Running LLM Agents](https://arxiv.org/abs/2605.16045) — Dai et al., ACL 2026 Findings
- [Lost in the Middle](https://arxiv.org/abs/2307.03172) — Liu et al., 2024
- [Generative Agents](https://arxiv.org/abs/2304.03442) — Park et al., 2023
