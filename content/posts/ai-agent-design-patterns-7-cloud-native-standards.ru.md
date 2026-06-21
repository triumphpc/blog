---
title: "AI Agent Design Patterns. Part 7: Cloud-Native Standards"
date: 2026-06-20
draft: false
tags: ["ai", "agents", "kubernetes", "cloud-native", "mcp", "go"]
categories: ["ai", "engineering"]
summary: "Подробный разбор CNCF 'Cloud Native Agentic Standards' (март 2026): пять разделов документа — General, Control and Communication, Observability, Governance, Security — пункт за пунктом. Что значит каждая рекомендация, как реализована сегодня в Kubernetes-экосистеме, куда копать глубже. От контейнерных основ до OWASP Agent Name Service, от MCP до Trusted Execution Environments."
ShowToc: true
series: ["AI Agent Design Patterns"]
---

## 1. Введение

В [Part 1]({{< ref "ai-agent-design-patterns-1-react" >}}) я разбирал ReAct — агент, который думает и действует. В [Part 4]({{< ref "ai-agent-design-patterns-4-multi-agent" >}}) — пять архитектур оркестрации нескольких агентов. В [Part 6]({{< ref "ai-agent-design-patterns-6-memory-deep-dive" >}}) — глубокий разбор памяти. Шесть частей, десятки паттернов, сотни строк кода на Eino. Но всё это — код, который работает **внутри пода**. ReAct-цикл крутится в одном процессе. Supervisor делегирует под-агентам в том же контейнере. Память живёт в in-process map.

23 марта 2026 CNCF AI TCG (AI Technical Community Group — техническая рабочая группа по ИИ внутри Cloud Native Computing Foundation) выпустил документ [«Cloud Native Agentic Standards»](https://www.cncf.io/blog/2026/03/23/cloud-native-agentic-standards/) — попытку собрать в одном месте лучшие практики для agentic-систем в Kubernetes. Документ охватывает пять разделов: General (контейнерные основы), Control and Communication (протоколы и обнаружение сервисов), Observability (метрики, трейсы, логи), Governance (оценка и аудит), Security (идентичность и доступ к данным).

Документ хороший как карта территории. Но он написан плотно — каждый пункт это одна-две фразы, за которыми стоит много контекста. «Enforce the principle of least privilege» — что именно? «Use MCP for tool exposure» — а что такое MCP, какая версия, где спецификация? «Agent-as-a-Judge» — это что, тоже агент?

В этой статье я разбираю документ пункт за пунктом. По каждой рекомендации: что CNCF имеет в виду, как это реализовано сегодня в Kubernetes-экосистеме, какие технологии и спецификации существуют, куда читать глубже. Где уместно — примеры кода на Go и K8s-манифесты, Mermaid-диаграммы, ссылки на arxiv-статьи и спецификации.

Уровень абстракции меняется — мы поднимаемся от паттернов внутри агента к паттернам эксплуатации агентов в cloud-native среде. Но принцип серии сохраняется: deep dive, honest limitations, code examples.

## 2. General — контейнерные основы

Первый раздел CNCF-документа — не agent-специфичный. Это введение в лучшие практики работы с контейнерами: безопасность, наблюдаемость, доступность. Ничего, чего не было бы в обычном Kubernetes-чеклисте. Но именно эта основа нужна, чтобы дальше говорить об agent-специфичных вещах.

### 2.1 Security — безопасность контейнеров

CNCF рекомендует: принцип минимальных привилегий, сокрытие информации, multi-stage builds, безопасные образы из доверенных репозиториев, сканирование уязвимостей, подпись образов, управление секретами через Kubernetes Secrets или внешние менеджеры, запуск от non-root пользователя, distroless-образы, мониторинг runtime-поведения.

**Что это значит на практике.** Принцип минимальных привилегий (least privilege) — агент получает только те права, которые ему нужны для работы, ничего больше. Для K8s-пода это значит: `securityContext.runAsNonRoot: true`, `runAsUser` с конкретным UID, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`. NetworkPolicy, ограничивающая исходящие соединения только нужными хостами. ServiceAccount с RBAC-ролью, где перечислены конкретные ресурсы и глаголы — не `cluster-admin`, а `get pods`, `list services` в конкретном namespace.

**Distroless-образы** (от Google) — образы контейнеров без операционной системы: только приложение и его зависимости, без shell, package manager, утилит. Меньше атакуемая поверхность — если злоумышленник получит доступ к контейнеру, ему некуда развернуться. Для Go-агента это естественно: статически слинкованный бинарник + CA-сертификаты, всё.

**Multi-stage builds** — в Dockerfile несколько этапов: первый собирает бинарник (с компилятором, зависимостями), второй копирует только результат. Сборочные инструменты не попадают в финальный образ.

**Подпись образов** — OCI-аннотации (Open Container Initiative — спецификация формата образов) + криптографическая подпись через [Sigstore](https://www.sigstore.dev/) (cosign). Позволяет проверить, что образ не был подменён в registry.

**Как сегодня.** [Kubernetes security checklist](https://kubernetes.io/docs/concepts/security/security-checklist/), [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/), [CNCF cloud native security whitepaper v2](https://github.com/cncf/tag-security/blob/main/community/resources/security-whitepaper/v2/CNCF_cloud-native-security-whitepaper-May2022-v2.pdf). Для сканирования уязвимостей — Trivy, Grype. Для runtime-мониторинга — Falco, Tetragon.

**Что почитать.** [Linux kernel security constraints](https://kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/) — как K8s использует seccomp, AppArmor, SELinux для изоляции подов.

### 2.2 Observability — наблюдаемость контейнеров

CNCF рекомендует: использовать стандартный observability-стек MELT (Metrics, Events, Logs, Traces — метрики, события, логи, трейсы), сетевую наблюдаемость через flow logs, мониторинг CPU/GPU и диска, alerting на основе SLO/SLA, cost observability для GPU и LLM-бенчмаркинга.

**Что это значит.** MELT — четыре сигнала наблюдаемости. Метрики — числовые показатели (CPU, память, RPS, latency). События — дискретные факты (под перезапустился, deployment завершён). Логи — текстовые записи работы приложения. Трейсы — распределённая трассировка запроса через несколько сервисов. Для агентов это особенно важно: один запрос пользователя может пройти через 5-10 tool-call'ов в разных подах, и без трейсов понять где что сломалось — невозможно.

**Cost observability для GPU и LLM** — агент-специфичное расширение. GPU-инстансы дороги ($2-12/час для A100/H100), LLM API вызовы тарифицируются по токенам. Без cost observability один runaway-агент (агент, вышедший из-под контроля — например, зациклившийся в retry-loop) может потратить тысячи долларов за часы. Нужны метрики: tokens per request, cost per trajectory, GPU utilization per agent instance.

**Как сегодня.** Prometheus — метрики, Loki — логи, Jaeger/Tempo — трейсы, всё через OpenTelemetry Collector. Для GPU — [NVIDIA DCGM exporter](https://github.com/NVIDIA/dcgm-exporter). Для LLM-cost — кастомные метрики (подробнее в разделе 4).

**Что почитать.** [CNCF «What is Observability 2.0»](https://www.cncf.io/blog/2025/01/27/what-is-observability-2-0/) — почему MELT недостаточно и что дальше.

### 2.3 Availability and fault tolerance — доступность и отказоустойчивость

CNCF рекомендует: resource limits и requests, PodDisruptionBudgets, Pod Anti-Affinity и Topology Spread Constraints, Horizontal Pod Autoscaler.

**Что это значит.** `resources.requests` — сколько CPU/памяти под гарантированно получит. `resources.limits` — потолок. Без requests под может попасть на перегруженный узел (noisy neighbor — «шумный сосед», когда один под потребляет ресурсы и мешает другим). Для GPU — `nvidia.com/gpu: 1` в limits.

**PodDisruptionBudget** (PDB) — K8s-ресурс, гарантирующий минимум доступных подов при добровольных сбоях (voluntary disruptions — обновления, drain узлов, масштабирование). `minAvailable: 2` значит, что всегда останется минимум 2 пода агента.

**Pod Anti-Affinity / Topology Spread Constraints** — распределение подов по разным узлам и зонам доступности. Если все поды агента на одном узле — его отказ убивает всю систему. `topologySpreadConstraints` с `maxSkew: 1` гарантирует равномерное распределение.

**Horizontal Pod Autoscaler** (HPA) — автоматическое масштабирование. CNCF упоминает custom metrics — для агентов это может быть «количество активных сессий» или «длина очереди tool-call'ов», не только CPU.

### 2.4 Availability — inference-specific

CNCF рекомендует: Gateway API Inference Extensions для path-based правил маршрутизации к inference-моделям.

**Что это значит.** Inference (инференс — процесс выполнения модели, генерация ответа) — отдельный класс нагрузки. В отличие от обычных HTTP-сервисов, inference-запросы долгие (секунды, не миллисекунды), потребляют GPU (не CPU), и чувствительны к batch size и KV-cache состоянию. Обычный round-robin load balancer здесь не оптимален.

[Gateway API Inference Extensions](https://gateway-api-inference-extension.sigs.k8s.io/) — официальный Kubernetes-проект от WG-Serving (Working Group по обслуживанию моделей) и SIG-Network. Расширяет Gateway API двумя концепциями:

{{< mermaid >}}
flowchart LR
    CLIENT["Client<br/>(agent)"] --> GW["Gateway<br/>(Envoy/NGINX)"]
    GW --> EPP["Endpoint Picker<br/>(smart load balancing)"]
    EPP -->|"metrics:<br/>queue depth<br/>KV cache state"| IP["InferencePool<br/>vLLM / Triton"]
    IP --> EP1["Endpoint 1<br/>model server"]
    IP --> EP2["Endpoint 2<br/>model server"]
    IP --> EP3["Endpoint 3<br/>model server"]

    style EPP fill:#1565c0,color:#fff
    style IP fill:#e65100,color:#fff
{{< /mermaid >}}

**InferencePool** — набор endpoint'ов (подов с model server — vLLM, Triton), рассматриваемых как единый пул. **Endpoint Picker** (EPP) — расширяемый компонент, выбирает оптимальный endpoint на основе метрик от model server: глубина очереди, состояние KV-cache (кэш контекста диалога в памяти GPU), доступность LoRA-адаптеров (низкоранговая адаптация — техника дообучения модели без изменения всех весов). Это «умный» load balancing — не round-robin, а выбор endpoint'а, который быстрее всего ответит.

**Serving priority** — Gateway API IE поддерживает приоритеты: chat (чувствителен к задержкам) — высокий приоритет, summarization (пакетная обработка) — низкий. Model rollouts — канареечное выкатывание новых версий моделей через traffic splitting по имени модели.

**Как сегодня.** Проект активно развивается, интеграция с [vLLM](https://github.com/vllm-project/vllm) (высокопроизводительный inference-сервер) и [Triton](https://github.com/triton-inference-server/server) (NVIDIA inference server). Поддерживается Envoy Gateway, kgateway, GKE Gateway. Совместим с OpenAI API — можно интегрировать self-hosted модели с LiteLLM, Gloo AI Gateway, Apigee.

## 3. Control and Communication — управление и коммуникация

Самый насыщенный раздел CNCF-документа. Протоколы, identity-фреймворки, messaging, discovery — всё, что связано с тем, как агенты общаются с инструментами, моделями и друг с другом.

### 3.1 Orchestration flow — оркестрация

CNCF рекомендует: проектировать оркестрацию по принципам GitOps, учитывая архитектурные паттерны (centralized, decentralized, star, ring) и их влияние на безопасность и отказоустойчивость.

**Что это значит.** GitOps — подход, где конфигурация системы хранится в Git-репозитории как source of truth, а контроллер (Flux, ArgoCD) синхронизирует состояние кластера с репозиторием. Для агентов это значит: системные промпты, конфигурация инструментов, RBAC-политики, версии моделей — всё в Git, с историей изменений, ревью через pull requests, откатом через revert.

Архитектурные паттерны оркестрации (из [Part 4]({{< ref "ai-agent-design-patterns-4-multi-agent" >}})):

{{< mermaid >}}
flowchart TB
    subgraph PATTERNS["Паттерны оркестрации (из Part 4)"]
        direction LR
        CENT["Centralized<br/>(Supervisor)<br/>один координатор"]
        DEC["Decentralized<br/>(Swarm/Handoff)<br/>агенты передают друг другу"]
        STAR["Star<br/>(Hub-and-spoke)<br/>центральный хаб"]
        RING["Ring<br/>(каскадная<br/>передача по кругу)"]
    end

    style CENT fill:#2e7d32,color:#fff
    style DEC fill:#e65100,color:#fff
    style STAR fill:#1565c0,color:#fff
    style RING fill:#6a1b9a,color:#fff
{{< /mermaid >}}

Каждый паттерн имеет разные trade-offs для безопасности и отказоустойчивости. Centralized (Supervisor) — один координатор, легко аудировать, но single point of failure. Decentralized (Swarm) — нет единой точки отказа, но сложнее отладка (distributed traces). Star — хаб как bottleneck. Ring — задержка растёт с числом участников.

**Как сегодня.** [Flux](https://fluxcd.io/) и [ArgoCD](https://argo-cd.readthedocs.io/) — оба CNCF-проекты, оба поддерживают GitOps для K8s. Для agent-конфигурации — Helm-чарты или Kustomize с промптами как ConfigMaps/Secrets.

### 3.2 Tools and services — инструменты и сервисы

CNCF рекомендует: единый подход к доступу к инструментам (MCP, A2A, ACP), избегать избыточной вариативности («tool sprawl» — «разрастание инструментов»), продумать поведение при недоступности системы контроля доступа.

**Что это значит.** «Tool sprawl» — когда в системе 5 разных способов вызвать инструменты: MCP для одних, custom REST для других, gRPC для третьих. Каждая вариативность — это сложность эксплуатации и мониторинга. CNCF призывает: выберите один-два протокола и держитесь их.

Поведение при недоступности Access Control (системы контроля доступа — например, OPA, Keycloak): что должен делать агент, если не может проверить права? Отказать? Работать с кэшированными правами? Какие сервисы остаются доступны? Это нужно продумать заранее, не в момент сбоя.

**Как сегодня.** MCP стал де-факто стандартом для доступа к инструментам (подробнее в 3.6). Для access control — [OPA](https://www.openpolicyagent.org/) (Open Policy Agent, CNCF Graduate) с Rego-политиками, или [Kyverno](https://kyverno.io/) для K8s-native политики.

### 3.3 Agent connectivity to AI Models — связь агента с моделями

CNCF рекомендует: продумать, какие процессы могут продолжаться при недоступности модели, а какие — нет. Рассмотреть Kubernetes custom watcher/controller для мониторинга критических ресурсов. Использовать gateway/proxy с observability для обнаружения сбоев.

**Что это значит.** Агент зависит от LLM-модели (GPT-4, Claude, self-hosted Llama). Если модель недоступна — что делать? В multi-agent системе с разными моделями: какие агенты могут работать без модели X, какие — нет?

Например: supervisor-агент использует GPT-4 для reasoning, coder-агент использует локальный CodeLlama для генерации кода. Если GPT-4 недоступен — supervisor не может делегировать, весь pipeline встаёт. Если CodeLlama недоступен — coder не работает, но researcher может продолжать.

**Kubernetes custom watcher/controller** — кастомный контроллер, который следит за состоянием model provider'а (health endpoint, метрики) и при сбое переключает на fallback-модель или приостанавливает работу.

**Как сегодня.** Gateway API Inference Extensions (раздел 2.4) решает часть этой проблемы — InferencePool с несколькими endpoint'ами. Для cross-provider fallback (OpenAI → Anthropic) — кастомная логика в агенте или AI Gateway ([LiteLLM](https://www.litellm.ai/), [Gloo AI Gateway](https://www.solo.io/products/gloo-ai-gateway)).

### 3.4 Agents to other Agents — связь между агентами

CNCF рекомендует: протокол A2A от Google для peer-to-peer взаимодействия агентов, даже в гетерогенных экосистемах.

**Что это значит.** Peer-to-peer (P2P — равноправная сеть, где каждый узел может быть и клиентом, и сервером) взаимодействие агентов. В отличие от MCP (где агент-клиент вызывает инструмент-сервер), A2A предполагает, что оба участника — агенты, со своим reasoning, своим state, своими инструментами. Они могут быть построены на разных фреймворках (LangChain, AutoGen, Eino), на разных языках, в разных кластерах.

**Как сегодня.** [A2A](https://github.com/a2aproject/A2A) — open-source протокол под Linux Foundation, разработан Google. Версия v1.0.1 (май 2026), 24.4k звёзд на GitHub. Ключевые концепции:

- **Agent Card** — JSON-документ с описанием возможностей агента (навыки, поддерживаемые модальности, endpoint). Похож на service discovery в микросервисах.
- **Транспорт**: JSON-RPC 2.0 поверх HTTP(S) — текстовый протокол удалённого вызова процедур.
- **Модальности**: синхронный request/response, streaming через SSE (Server-Sent Events), асинхронные push-уведомления.
- **Принцип opacity** (непрозрачность): агенты взаимодействуют без раскрытия внутреннего state, памяти, логики, инструментов. Это и для безопасности, и для защиты IP.

SDK для Python, Go, JavaScript, Java, .NET, Rust. DeepLearning.AI выпустил [бесплатный курс](https://goo.gle/dlai-a2a) по A2A в партнёрстве с Google Cloud и IBM Research.

### 3.5 Filtering and schema validation — валидация схем

CNCF рекомендует: определять схемы через JSON Schema, Protobuf или OpenAPI для валидации payloads при tool calls и вызовах внешних сервисов. Это повышает предсказуемость и избегает cascading failures (каскадных сбоев — когда отказ одного компонента вызывает отказы других по цепочке).

**Что это значит.** Агент вызывает инструмент `send_email(to, subject, body)`. Без schema validation агент может передать `to: null` или `body: 123` (число вместо строки). Инструмент упадёт, агент получит непонятную ошибку, попробует retry — и так до исчерпания budget.

Со schema validation инструмент заранее определяет: `to` — строка, email-формат; `subject` — строка, max 200 символов; `body` — строка. Если аргументы не соответствуют — ошибка на этапе валидации, понятная агенту, без вызова инструмента.

**Три технологии схем:**

| Технология | Формат | Где используется | Сильная сторона |
|-----------|--------|------------------|-----------------|
| **JSON Schema** | JSON-документ | REST API, MCP | Читаемость, повсеместность |
| **Protobuf** | Бинарный + .proto | gRPC, внутренние сервисы | Производительность, типизация |
| **OpenAPI** | YAML/JSON | REST API спецификация | Богатая экосистема инструментов |

**Как сегодня.** MCP поддерживает schema validation нативно — каждый tool определяет `inputSchema` как JSON Schema. Для gRPC — protoc генерирует Go-код с типами. Для REST — OpenAPI-спецификация + code generation.

Пример валидации tool call на Go:

```go
type ToolValidator struct {
    schemas map[string]*jsonschema.Schema
}

func (v *ToolValidator) ValidateCall(toolName string, args json.RawMessage) error {
    schema, ok := v.schemas[toolName]
    if !ok {
        return fmt.Errorf("неизвестный инструмент: %s", toolName)
    }
    if err := schema.Validate(args); err != nil {
        return fmt.Errorf("валидация схемы для %s: %w", toolName, err)
    }
    return nil
}
```

### 3.6 Protocols today — протоколы сегодня

CNCF перечисляет четыре протокола: MCP, A2A, AP2, и упоминает ACP. Разберём каждый подробно.

#### MCP (Model Context Protocol)

[MCP](https://modelcontextprotocol.io/) — разработан Anthropic, передан в Agentic AI Foundation (AAIF — фонд поддержки agentic AI проектов под эгидой Linux Foundation). «Де-факто стандарт» для доступа агентов к инструментам и ресурсам, по оценке [Yang et al. (2025)](https://arxiv.org/abs/2504.16736).

**Архитектура**: client-server. MCP Server предоставляет доступ к инструментам (tools), ресурсам (resources) и промптам (prompts). MCP Client (обычно внутри агента) вызывает их.

{{< mermaid >}}
flowchart LR
    AGENT["Agent<br/>(MCP Client)"] -->|"JSON-RPC 2.0<br/>over HTTPS"| SERVER["MCP Server<br/>(tool provider)"]
    SERVER --> TOOL1["Tool: github_search"]
    SERVER --> TOOL2["Tool: db_query"]
    SERVER --> RES["Resource: docs"]

    style AGENT fill:#2e7d32,color:#fff
    style SERVER fill:#1565c0,color:#fff
{{< /mermaid >}}

**Транспорт**: JSON-RPC 2.0 (протокол удалённого вызова процедур в формате JSON) поверх HTTPS + streamable HTTP. Версия спецификации: [2025-06-18](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization).

**Authorization**: OAuth 2.1 (стандарт авторизации, где клиент получает access token от authorization server и предъявляет его resource server). MCP Server выступает как OAuth resource server, MCP Client — как OAuth client. PKCE обязателен (Proof Key for Code Exchange — защита от перехвата authorization code). Audience binding через RFC 8707 (Resource Indicators — привязка токена к конкретному ресурсу, чтобы токен для одного MCP-сервера нельзя было использовать с другим). Dynamic Client Registration через RFC 7591.

**Ключевая особенность**: star architecture — агент в центре, MCP-серверы по периферии. [Yang et al.](https://arxiv.org/abs/2504.16736) отмечают: «весь обмен данными должен проходить через центрального агента, что потенциально создаёт узкое место производительности» (bottleneck). Для большинства use cases это не проблема (bottleneck — LLM inference, не IPC), но для high-throughput pipeline'ов — учитывать.

#### A2A (Agent2Agent)

[Разобран в разделе 3.4.](#34-agents-to-other-agents--связь-между-агентами) Peer-to-peer взаимодействие агентов, JSON-RPC 2.0 поверх HTTPS, Agent Card для discovery. Linux Foundation, v1.0.1.

#### AP2 (Agent Payments Protocol)

[AP2](https://github.com/google-agentic-commerce/AP2) — протокол платежей, совершаемых AI-агентами. Разработан Google. Версия v0.2.0 (апрель 2026), 3.1k звёзд на GitHub.

**Что это значит.** Агент покупает товар, заказывает услугу, оплачивает подписку — autonomously, без человека в цикле. AP2 определяет, как это делать безопасно: криптографически подписанные сертификаты, real-time и delegated operation models.

**Ключевые технологии:**
- **x402 extensions** — расширения протокола x402 для decentralized environments (децентрализованные среды — без центрального посредника)
- **Verifiable Credentials** (VC — проверяемые удостоверения, W3C-стандарт: цифровые удостоверения, криптографически подписанные, которые можно проверить без обращения к эмитенту)
- **Decentralized Identifiers** (DID — децентрализованные идентификаторы, W3C-стандарт: идентификаторы, не привязанные к центральному реестру, как `did:web:example.com` или `did:ethr:0x...`)

**Зрелость**: ранняя. v0.2 — значит API может меняться. Niche use case — агенты-покупатели. Но направление перспективное: по мере роста agent-автономности платежи станут необходимостью.

#### ACP (Agent Communication Protocol)

[ACP](https://www.ibm.com/think/topics/agent-communication-protocol) — разработан IBM Research и BeeAI (комьюнити под Linux Foundation). REST-based, HTTP-native протокол для связи между агентами.

**Что это значит.** В отличие от A2A (JSON-RPC 2.0), ACP использует привычный REST — HTTP-запросы к endpoint'ам агентов. Поддерживает синхронные и асинхронные паттерны (async-first, sync supported). Discovery через capability descriptions, объявляемые при сборке (offline discovery — метаданные встраиваются в пакет агента, discovery работает даже когда агент не запущен, что удобно для scale-to-zero окружений). Не требует SDK — можно вызывать через cURL или Postman.

**Важное обновление**: ACP официально слит с A2A под Linux Foundation. Команда ACP сворачивает активную разработку и передаёт технологии и экспертизу в A2A. Текущие пользователи ACP должны мигрировать на A2A. Таким образом, ACP и A2A конвергируют — A2A как umbrella-протокол, ACP как один из подходов внутри.

**Зрелость**: разработка сворачивается, миграция на A2A. BeeAI (i-am-bee на GitHub) остаётся активным как reference implementation, framework-agnostic SDK для Python и TypeScript.

### 3.7 Authentication and Authorization protocols — протоколы идентичности

CNCF перечисляет три: SPIFFE/SPIRE, Agntcy, и упоминает OAuth2/OIDC. Разберём подробно.

#### SPIFFE / SPIRE

[SPIFFE](https://spiffe.io/) (Secure Production Identity Framework For Everyone — «безопасный фреймворк идентичности для всех») и [SPIRE](https://spiffe.io/) (SPIFFE Runtime Environment — среда выполнения SPIFFE) — CNCF Graduate projects с 2022 года. Production-ready, используются в Amazon, Google, Netflix, Uber, Bloomberg, Twilio.

**Что это значит.** В zero-trust архитектуре каждый сервис нуждается в криптографической идентичности — не в статичных API-ключах, а в динамически выдаваемых, короткоживущих удостоверениях. SPIFFE определяет формат такой идентичности, SPIRE — реализацию.

**SVID** (SPIFFE Verifiable Identity Document — проверяемый документ идентичности SPIFFE) — криптографическое удостоверение workload. Два формата:

- **JWT-SVID** — JWT-токен (JSON Web Token) с полем `sub` (subject) содержащим SPIFFE ID. Подходит для crossing trust domains, проверки через публичный ключ.
- **X.509-SVID** — сертификат X.509 с SPIFFE ID в URI SAN (Subject Alternative Name). Подходит для mTLS (mutual TLS — двустороннее TLS), где оба участника аутентифицируют друг друга.

**SPIFFE ID** — URI-формат:

```
spiffe://{trust-domain}/{workload-identifier}
```

Например:

```
spiffe://prod.example.com/agent/payments/supervisor
```

Trust domain — домен доверия (кластер, организация), внутри которого SVID'ы валидны.

{{< mermaid >}}
sequenceDiagram
    participant P as Agent Pod
    participant SR as SPIRE Agent
    participant SS as SPIRE Server
    participant T as Tool Server

    Note over P: Запуск пода
    P->>SR: Workload API<br/>(attest via pod UID, namespace, SA)
    SR->>SS: Аттестация workload
    SS-->>SR: Подтверждение + trust bundle
    SR-->>P: JWT-SVID<br/>(TTL=1ч, audience=mcp-tools)

    Note over P: Первый вызов инструмента
    P->>T: Запрос + Bearer SVID
    T->>T: Проверка SVID<br/>(подпись, audience, TTL)
    T-->>P: Ответ

    Note over P: TTL истёк
    P->>SR: Новый SVID<br/>(авторотация)
    SR-->>P: Свежий JWT-SVID
{{< /mermaid >}}

**Federation** — доверие между trust domains. SPIRE Server одного домена публикует свой trust bundle (набор публичных ключей), SPIRE Server другого домена его скачивает. Так агенты из разных кластеров могут проверять SVID'ы друг друга.

#### Agntcy identity framework

[Agntcy](https://docs.agntcy.org/) — проект, донированный в [Linux Foundation в июле 2025](https://www.linuxfoundation.org/press/linux-foundation-welcomes-the-agntcy-project-to-standardize-open-multi-agent-system-infrastructure-and-break-down-ai-agent-silos). Сам CNCF-документ признаёт: «still in relatively nascent stages» (в относительно ранних стадиях).

**Что это значит.** BYOI (Bring Your Own Identity — «приноси свою идентичность») — подход, где разные identity providers могут использоваться в одной системе. Не привязка к одному решению (как SPIFFE), а каркас, в который можно подключить разные провайдеры.

**Уникальная особенность**: поддержка Web3 DID-стандарта (W3C Decentralized Identifiers — децентрализованные идентификаторы, как в блокчейн-экосистеме). Это позволяет агентам иметь идентичность, не привязанную к центральному реестру — полезно для cross-организационного взаимодействия.

**Agent Directory Service** (ADS) — реестр агентов с метаданными: возможности, состояние, endpoint'ы. Использует [OCI registry infrastructure](https://arxiv.org/html/2509.18787v1) (Open Container Initiative — та же инфраструктура, что для Docker-образов) и криптографическую подпись.

**Зрелость**: nascent. Production-деплой преждевременен, но за проектом стоит Linux Foundation, направление перспективное. Complementary к SPIFFE, не замена — SPIFFE для workload identity внутри кластера, Agntcy для cross-org agent discovery.

#### OWASP Agent Name Service (ANS)

[ANS v1.0](https://genai.owasp.org/resource/agent-name-service-ans-for-secure-al-agent-discovery-v1-0/) — разработан под эгидой OWASP (Open Web Application Security Project — открытый проект безопасности веб-приложений) GenAI Security Project, Agentic Security Initiative.

**Что это значит.** DNS-подобный сервис для обнаружения агентов — но с криптографической проверкой идентичности. Когда агент ищет другого агента по имени, ANS гарантирует, что найденный агент — тот, за кого себя выдаёт.

**Технологии:**
- **PKI** (Public Key Infrastructure — инфраструктура открытых ключей) — система сертификатов для верификации
- **Structured JSON schemas** — структурированные схемы для описания агентов
- **Zero-Knowledge Proofs** (ZKP — доказательства с нулевым разглашением: можно доказать, что утверждение истинно, не раскрывая никаких дополнительных данных) — для проверки идентичности и возможностей агента без раскрытия лишнего

**Защита от атак:** agent impersonation (подмена агента — когда злоумышленник выдаёт свой агент за легитимный) и registry poisoning (отравление реестра — когда злоумышленник модифицирует записи в реестре, подменяя endpoint'ы).

**Зрелость:** research stage. [IETF draft](https://www.ietf.org/archive/id/draft-narajala-ans-00.html), [IEEE paper](https://ieeexplore.ieee.org/document/11395757). Protocol-agnostic — поддерживает A2A, MCP, ACP. Перспективно для agent discovery, но в production рано.

### 3.8 Message and communication design — проектирование обмена сообщениями

CNCF рекомендует: Kafka/Flink для асинхронной event-driven коммуникации, gRPC для streaming, REST для простого request/response. Обращать внимание на влияние streaming data на token limits.

**Что значит каждый вариант:**

**Kafka** — распределённая event streaming платформа. CNCF упоминает для асинхронной коммуникации (long-running tasks — долгие задачи) и event-driven архитектуры (телеметрия, логи решений, координационные сигналы). Гарантии доставки: at-least-once (хотя бы один раз — сообщение может доставиться дважды, но не потеряется). [Flink](https://flink.apache.org/) — stream processing, обработка данных в потоке, на лету.

**gRPC** — бинарный RPC-протокол на основе HTTP/2 и Protobuf. Эффективнее REST (бинарный, не текстовый), поддерживает bidirectional streaming. CNCF обращает внимание: при streaming data нужно оценить ELT vs ETL подход (Extract-Load-Transform vs Extract-Transform-Load — порядок загрузки и трансформации данных). И влияние объёма streaming data на token limits агента — если агент получает данные через gRPC-stream, они попадают в context window, и при большом объёме токены заканчиваются.

**REST** — простой, interoperable, request/response. Минусы: большие payloads влияют на latency, менее производителен чем gRPC, нет native streaming (нужно для long-lived actions — долгоживущих действий, когда агент ждёт ответа минутами).

**Как сегодня.** Kafka — [Strimzi](https://strimzi.io/) (CNCF, Kafka на K8s), [Confluent](https://www.confluent.io/). gRPC — нативно в Go. Для multi-agent event-driven — Kafka как event bus, агенты как consumers/producers.

### 3.9 Discovery / agent registries — обнаружение сервисов

CNCF рекомендует: DNS-based discovery, service meshes, purpose-built agent registries, static registration для air gap сценариев.

**Что это значит.** Discovery — как агент находит другие агенты и инструменты. Четыре подхода:

**DNS-based** — Kubernetes-native DNS (CoreDNS) или service mesh registry. Агент обращается к `supervisor-agent.agents.svc.cluster.local`, DNS возвращает IP. Просто, но только сетевой адрес — без метаданных.

**Service meshes** (Istio, Linkerd) — динамический реестр запущенных сервисов с метаданными и endpoint'ами. Плюс mTLS, traffic management, observability из коробки.

**Purpose-built agent registries** — специализированные реестры агентов и инструментов, хранящие не только endpoint, но и метаданные: возможности, здоровье, статус. Позволяют агентам выбирать оптимальный ресурс во время выполнения.

**Static registration / multicast** — для air gap (изолированные среды без доступа к центральным реестрам). Ручная конфигурация или локальное обнаружение.

**Как сегодня.** DNS + service mesh — для K8s-internal discovery. Agntcy Directory Service (раздел 3.7) — emerging purpose-built registry. OWASP ANS — research stage. Для production сейчас: DNS + service mesh, кастомный реестр при необходимости.

**Что почитать.** [OWASP Secure Agent Registry](https://genai.owasp.org/secure-agent-registry/), [Agntcy Directory](https://agntcy.github.io/dir/latest/), [Agent Skills](https://agentskills.io/home).

## 4. Observability — наблюдаемость

CNCF расширяет классический MELT-стек на agent-специфичные метрики: токены, TTFT, стоимость, точность. И добавляет trajectory correlation через OpenTelemetry baggage.

### 4.1 Metrics — метрики

CNCF рекомендует метрики: токены для inference (input/output/reasoning), TTFT/TPOT/ITL, взаимодействия с внешними tool/LLM, длительность выполнения, стоимость, точность (precision), rate limits hits, use-case-специфичные метрики (галлюцинации, success rate).

**Что значит каждая метрика:**

| Метрика | Расшифровка | Зачем |
|---------|-------------|-------|
| **Input tokens** | Токены на входе модели | Стоимость запроса |
| **Output tokens** | Токены на выходе | Стоимость ответа |
| **Reasoning tokens** | Токены reasoning (o1-style) | Стоимость размышления модели |
| **TTFT** | Time To First Token — время до первого токена | Latency для пользователя |
| **TPOT** | Time Per Output Token — время на выходной токен | Throughput генерации |
| **ITL** | Inter-Token Latency — задержка между токенами | Smoothness streaming |
| **Cost** | Стоимость inference | FinOps, budget enforcement |
| **Precision** | Точность ответов | Качество модели |
| **Rate limit hits** | Попадания в лимиты API | Адаптация архитектуры |

**Token accounting** — основа cost governance (управление затратами). Каждый вызов LLM тарифицируется по токенам: input ~$0.01-0.06/1K, output ~$0.03-0.12/1K (зависит от модели). Один ReAct-цикл — 5-20K токенов, multi-agent debate (Part 4) — 50-200K токенов. Без per-trajectory token accounting невозможно понять, сколько стоит один запрос пользователя.

**Как сегодня.** OpenTelemetry с [gen-ai semantic conventions](https://github.com/open-telemetry/semantic-conventions-genai) (experimental, dedicated repo) определяет атрибуты для gen-ai spans: `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.system`, `gen_ai.request.model`. Для rate limits и cost — кастомные метрики поверх.

Пример инструментирования на Go:

```go
func (a *Agent) callLLM(ctx context.Context, messages []Message) (Response, error) {
    ctx, span := otel.Tracer("agent").Start(ctx, "llm.call",
        trace.WithAttributes(
            attribute.String("gen_ai.system", "openai"),
            attribute.String("gen_ai.request.model", a.model),
            attribute.String("gen_ai.session.id", a.sessionID),
        ),
    )
    defer span.End()

    resp, err := a.client.Complete(ctx, messages)
    if err != nil {
        span.RecordError(err)
        return resp, err
    }

    span.SetAttributes(
        attribute.Int("gen_ai.usage.input_tokens", resp.Usage.InputTokens),
        attribute.Int("gen_ai.usage.output_tokens", resp.Usage.OutputTokens),
        attribute.Float64("gen_ai.cost.usd", resp.Cost),
    )
    return resp, nil
}
```

### 4.2 Traces and spans — трейсы и спаны

CNCF рекомендует: custom instrumentation через OpenTelemetry, auto instrumentation в K8s, hooks для per-process executions (SPANs), baggage для propagation идентификаторов (user ID, session ID, context activity, container-ID).

**Что это значит.** Трейс — дерево вызовов, показывающее полный путь запроса через систему. Span — одна операция в дереве. Для агентов один пользовательский запрос порождает дерево: request → supervisor → researcher → web_search → LLM call → coder → code_gen → LLM call → reviewer → LLM call → response. Без трейсов это чёрный ящик.

{{< mermaid >}}
flowchart TB
    REQ["Пользовательский запрос"] --> SP1["Span: supervisor.run"]
    SP1 --> SP2["Span: researcher.run"]
    SP2 --> SP3["Span: web_search.execute"]
    SP2 --> SP4["Span: llm.call<br/>tokens: 1523 in / 847 out"]
    SP1 --> SP5["Span: coder.run"]
    SP5 --> SP6["Span: code_gen.execute"]
    SP5 --> SP7["Span: llm.call<br/>tokens: 2100 in / 3400 out"]
    SP1 --> SP8["Span: reviewer.run"]
    SP8 --> SP9["Span: llm.call<br/>tokens: 3200 in / 1200 out"]
    SP1 --> SP10["Span: response.generate"]

    style SP4 fill:#1565c0,color:#fff
    style SP7 fill:#1565c0,color:#fff
    style SP9 fill:#1565c0,color:#fff
{{< /mermaid >}}

**Baggage** — механизм OpenTelemetry для propagation метаданных через границы сервисов. Когда supervisor-агент вызывает researcher-агента, baggage содержит: `trajectory.id` (идентификатор траектории — всей последовательности шагов), `session.id` (идентификатор сессии пользователя), `user.id`. Это позволяет в Jaeger отфильтровать все спаны одной траектории, даже если они прошли через 5 разных подов.

**Как сегодня.** [OpenTelemetry GenAI semantic conventions](https://github.com/open-telemetry/semantic-conventions-genai) — dedicated repo, 83 star, без официальных релизов (experimental). Определяет span attributes для gen-ai: `gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.*`, `gen_ai.tool.*`. [OpenTelemetry LLM observability blog](https://opentelemetry.io/blog/2024/llm-observability/) — практическое руководство. [OTEL baggage](https://opentelemetry.io/docs/concepts/signals/baggage/) — спецификация.

### 4.3 Logs — логи

CNCF рекомендует: логи в «естественном языке» (natural language — для будущей обработки AI-системами), общесистемное время, структурированный формат данных, canonical logging.

**Что это значит.** «Естественный язык» — необычная рекомендация. CNCF предполагает, что логи будут читаться не только людьми, но и AI-моделями (для автоматического дебага). Формулировка: «authoring of viable and usable logs which are authored with a natural language in mind to allow for future re-usability by co-op AI model based systems for debugging».

**Canonical logging** — единый формат логов во всей системе: одинаковые поля, одинаковые значения, одинаковые единицы измерения. Позволяет строить log-based metrics (метрики на основе логов), упрощает post-mortem анализ.

**Общесистемное время** — NTP-синхронизация всех узлов. Без этого логи с разных подов нельзя выстроить в хронологию.

**Как сегодня.** Structured logging (JSON) — стандарт. В Go — `slog` (structured logging, стандартная библиотека с Go 1.21). Для K8s — Loki (CNCF, лог-агрегатор, интеграция с Prometheus и Grafana).

### 4.4 Continuous monitoring and adaptive control — мониторинг и адаптация

CNCF рекомендует: OTEL с semantic conventions для непрерывного мониторинга agent trajectories, периодическая переоценка для обнаружения drift (дрейфа — постепенного изменения поведения), feedback loops с reinforcement learning для self-correction.

**Что это значит.** Drift — когда поведение агента постепенно меняется со временем. Причины: обновление модели (GPT-4 → GPT-4o), изменение данных в RAG (Retrieval-Augmented Generation — генерация с дополнением извлечёнными данными), накопление episodic memory. Без мониторинга drift незаметен — агент «чуть хуже» работает каждую неделю, пока не станет критичным.

**Feedback loops** — агент получает обратную связь (от пользователя, от Agent-as-a-Judge) и адаптирует своё поведение. Reinforcement learning (обучение с подкреплением) — технически это fine-tuning модели на основе reward signal. В production для агентов это сложно (нужен training pipeline), но концептуально CNCF указывает правильное направление.

**Как сегодня.** OTEL pipeline → Prometheus → Alerting на основе SLO. Для drift detection — сравнение метрик (success rate, token usage, latency) по временным окнам: эта неделя vs прошлая. Для feedback loops — storing user feedback (thumbs up/down) в Postgres, periodic analysis.

## 5. Governance — управление

CNCF определяет governance как обязательный foundational слой для agentic-систем. Это не «добавим потом», а «проектируем с начала» — из-за regulatory adherence (соответствие регуляторным требованиям, как EU AI Act).

### 5.1 Agentic governance foundations — основы

CNCF утверждает: governance — mandatory foundational layer, нужен динамический и гибкий подход (в отличие от статичного software governance), regulatory adherence нужно проектировать с начала.

**Что это значит.** Традиционный software governance — статичные политики: code review, CI/CD checks, access control. Для агентов этого недостаточно: агенты проявляют emergent behavior (эмерджентное поведение — поведение, не предусмотренное явно, но возникающее из взаимодействия компонентов). Нужно continuously мониторить и адаптировать политики.

**Регуляторный контекст**: EU AI Act (закон ЕС об ИИ), [EU AI Data Act](https://digital-strategy.ec.europa.eu/en/policies/data-act) — требуют explainability (объяснимости: способность объяснить, почему принято конкретное решение) и auditability (аудируемости: возможность проверить историю решений). Проектировать с начала — значит: с первого дня хранить audit trail (историю решений), логировать reasoning (ход рассуждений), поддерживать data lineage (происхождение данных — какие данные использовались для решения).

**Что почитать.** [Reuel et al. (2024) «Open Problems in Technical AI Governance»](https://arxiv.org/abs/2407.14981) — TMLR 2025, 35 авторов включая Bengio, Hooker, Solaiman. Определяет Technical AI Governance через три функции: (a) выявление областей, требующих вмешательства, (b) оценка эффективности действий, (c) разработка механизмов compliance. CNCF ссылается на эту работу как академическое обоснование governance-секции.

### 5.2 Evaluation approach — подход к оценке

CNCF рекомендует: учитывать специфику use case (конкретного сценария применения), чёткие критерии успеха, полную стоимость использования (вычислительная, финансовая, экологическая, человеческая, на хранение данных), надёжность и устойчивость, безопасность и соответствие ценностям, качество взаимодействия, единые и унифицированные протоколы оценки.

**Что значит каждая категория оценки:**

| Категория | Что измеряет | Пример метрики |
|-----------|-------------|----------------|
| **Success rate** | Доля успешных выполнений | 87% задач решено корректно |
| **Computational cost** | Ресурсы | CPU-часы, GPU-часы, MB-RAM |
| **Financial cost** | Деньги | $0.50 за ReAct-цикл, $15 за multi-agent debate |
| **Environmental cost** | Углерод | CO2-след, carbon credits |
| **Human cost** | Человеко-часы | Время oversight, setup, калибровки |
| **Reliability** | Стабильность | Success rate в edge cases, при adversarial атаках |
| **Safety** | Безопасность | Отсутствие harmful outputs, alignment с человеческими ценностями |
| **Interaction quality** | UX | Естественность, связность, user-centeredness |

**Standard and uniform evaluation protocols** — CNCF подчёркивает: нужны чёткие правила проведения тестов, scoring, конфигурации окружения, чтобы результаты были сравнимы и воспроизводимы. Без этого «87% success rate» из одного теста несравнимо с «87%» из другого.

### 5.3 Synthetic data for testing — синтетические данные

CNCF рекомендует: diverse, policy-driven синтетические датасеты и fault scenarios, тесты выровнены на real-world scenarios, HITL (Human-in-the-loop — человек в цикле), генерация с entropy (энтропией — случайностью).

**Что это значит.** Синтетические данные — сгенерированные, не реальные. Нужны для тестирования агента: вместо реальных пользовательских запросов (которые могут содержать PII — Personal Identifiable Information, персональные данные) — синтетические, которые покрывают те же сценарии, но без рисков.

**Entropy** — случайность в синтетических данных. Если все тестовые запросы похожи друг на друга — агент может «выучить» их (overfit), и тесты перестанут ловить реальные баги. Entropy обеспечивает разнообразие.

**HITL** — человек участвует в тестовом цикле: калибрует тест-кейсы, оценивает edge cases, помечает некорректные результаты. Полностью автоматическая оценка недостаточна — особенно для subjective категорий (interaction quality, safety).

### 5.4 Granular and trajectory-based assessment — детальная оценка

CNCF рекомендует два подхода: **stepwise evaluation** (постепенная оценка — детальный анализ каждого шага агента), и **trajectory-based assessment** (оценка по траектории — анализ всей последовательности шагов относительно ожидаемого оптимального пути).

**Что это значит.** Stepwise — агент сделал шаг 1 (вызвал tool X), шаг 2 (получил observation Y), шаг 3 (сделал reasoning Z). Каждый шаг оценивается: правильно ли выбран tool? Правильно ли интерпретирован результат? Правильно ли reasoning? Это помогает найти root cause (корневую причину) ошибок.

Trajectory-based — сравнение реальной траектории с оптимальной. Агент должен был: query DB → filter results → generate answer. А сделал: web_search → web_search → query DB → filter → generate → filter again. Траектория длиннее оптимальной — агент «блуждал». Это может быть допустимо (агент исследовал), или признаком проблемы (агент не понимает задачу).

**Как сегодня.** τ-bench ([Yao et al., 2024](https://arxiv.org/abs/2406.12045)) — benchmark для tool-agent-user interaction, эмулирует динамические диалоги с domain-specific APIs. Метрика **pass^k** — reliability over multiple trials: прогоняем K раз, success = все K passed. gpt-4o показывает <50% success rate, pass^8 <25% в retail domain — то есть тот же агент даёт разные результаты на одних и тех же данных. Это важный finding: один прогон недостаточен для оценки.

[SWE-bench](https://arxiv.org/abs/2310.06770) ([Jimenez et al., 2024](https://arxiv.org/abs/2310.06770)) — 2,294 real-world SE problems из GitHub issues, 12 Python репозиториев. На момент публикации Claude 2 решал лишь 1.96%. Сейчас state-of-the-art значительно выше, но benchmark страдает от contamination (загрязнения — когда модель видела тестовые данные при обучении). SWE-bench Verified — revisited version с исправленными annotation errors.

[AgentBench](https://arxiv.org/abs/2308.03688) ([Liu et al., 2023](https://arxiv.org/abs/2308.03688)) — 8 environments, оценивает reasoning + decision-making. Significant disparity: top commercial LLMs сильны, open-source ≤70B — большой gap.

### 5.5 Data privacy and minimization — приватность данных

CNCF рекомендует: data minimization (минимизация данных — собирать только необходимое), transparent data governance policies, layered security.

**Что это значит.** Data minimization — принцип «не храни то, что тебе не нужно». Агент получает доступ к данным для конкретной задачи — после выполнения задачи, данные не сохраняются в памяти агента. Это и security (меньше данных = меньше риск утечки), и compliance (GDPR, CCPA требуют minimization).

**Layered security** — несколько слоёв защиты: NetworkPolicy (сетевая изоляция), RBAC (ролевой доступ), encryption at rest (шифрование на диске), encryption in transit (шифрование при передаче), audit logging (журналирование доступа).

### 5.6 Explainability and auditability — объяснимость и аудируемость

CNCF рекомендует: Model Openness Framework (MOF) для прозрачной документации, model cards и data cards, криптографическая подпись артефактов через Sigstore, automated auditing через Agent-as-a-Judge.

#### Model Openness Framework (MOF)

[MOF](https://arxiv.org/abs/2403.13784) ([White et al., 2024](https://arxiv.org/abs/2403.13784)) — трёхуровневая классификация моделей по полноте и открытости. Разработан Linux Foundation, University of Oxford, Columbia University, IBM.

**Три класса (по возрастанию полноты):**

| Класс | Название | Что включено |
|-------|----------|--------------|
| **Class III** | Open Model | Архитектура, финальные параметры, технический отчёт, результаты оценки, model card, data card |
| **Class II** | Open Tooling | Всё из Class III + код обучения/валидации/тестирования, код инференса, код оценки, данные оценки, вспомогательные библиотеки |
| **Class I** | Open Science | Всё из Class II + исследовательская работа, наборы данных, код препроцессинга, промежуточные контрольные точки, метаданные |

**Model card** — документация возможностей и ограничений модели, характера обучающих данных. **Data card** — документация наборов данных. Оба — обязательные компоненты Class III (минимальный уровень).

17 компонентов суммарно, плюс конфигурационный файл `mof.json`. Self-reporting (самоотчётность) — нет внешнего аудита, продюсеры моделей сами заполняют. [Model Openness Tool (MOT)](https://github.com/lfai/quality-assurance) — веб-форма для оценки.

**Важный нюанс**: MOF [явно исключает model provenance](https://arxiv.org/abs/2403.13784) (происхождение модели — криптографическую верификацию целостности артефактов) из scope (раздел 9.2). CNCF **композитит** MOF (документация) с Sigstore (криптографическая подпись) — это правильная композиция, которой нет ни в одной из работ по отдельности.

**Статистика для контекста**: 64.67% моделей и 72.13% датасетов на Hugging Face Hub не имеют лицензии. OSS (open-source software) используется в 96% кодовых баз и составляет до 90% программных стеков — MOF хочет того же для AI.

#### Agent-as-a-Judge

CNCF рекомендует: «explore and implement automated evaluation approaches using Agent-as-a-Judge».

[Agent-as-a-Judge](https://arxiv.org/abs/2410.10934) ([Zhuge et al., 2024](https://arxiv.org/abs/2410.10934), Meta/FAIR) — расширение LLM-as-a-Judge на agentic системы. В отличие от LLM-as-a-Judge (которая оценивает только финальный результат), Agent-as-a-Judge оценивает **intermediate feedback на весь task-solving process** — использует planning, tool-augmented verification, оценивает каждый шаг.

**LLM-as-a-Judge** ([Zheng et al., 2023](https://arxiv.org/abs/2306.05685), NeurIPS 2023) — канонический источник. GPT-4 как judge достигает >80% agreement с human preferences — на уровне inter-human agreement. Но имеет известные biases:

1. **Position bias** — judge предпочитает первый/последний вариант. Mitigation: swapping positions, averaging.
2. **Verbosity bias** — judge предпочитает более длинные ответы. Mitigation: [Length-Controlled AlpacaEval](https://arxiv.org/abs/2404.04475) (Dubois et al., 2024) — GLM-regression, «what would preference be if outputs had same length». Spearman correlation с Chatbot Arena поднимается с 0.94 → 0.98.
3. **Self-enhancement bias** — judge предпочитает ответы, похожие на свои собственные. Mitigation: cross-model judging.

Agent-as-a-Judge «dramatically outperforms LLM-as-a-Judge» и as reliable as human baseline — по Zhuge et al. Но [Ming et al. (2025)](https://arxiv.org/abs/2506.03332) показывают: judge может hallucinate, exhibit bias, act adversarially. Even strongest agents **switch correct answers after a single round of misleading feedback** — deceptive judge (обманчивый судья). Taxonomy по intent (constructive→malicious) × knowledge (parametric→RAG).

**Практический вывод**: Agent-as-a-Judge — один из сигналов, не единственный. Комбинируется с deterministic rules (schema validation, policy checks) и periodic human review. Использовать с осторожностью, особенно для high-stakes решений.

### 5.7 Integrated lifecycle governance — интегрированное управление жизненным циклом

CNCF рекомендует: управление (governance) — не разовая акция, а непрерывный процесс на протяжении всего жизненного цикла эксплуатации LLM (LLMOps). Техническая реализация, политические рамки и постоянный надзор должны работать в симбиозе. На уровне агента — повторные попытки (retry) с ограничением числа, предохранители (circuit breakers — автоматическое отключение при сбоях) и постепенная деградация (graceful degradation — переключение на упрощённый режим при отказе основного).

**Что это значит.** Жизненный цикл эксплуатации LLM (LLMOps — практики эксплуатации LLM-систем, аналог DevOps для LLM): подготовка данных → обучение модели → оценка → упаковка → развёртывание → мониторинг → обратная связь → переобучение. Управление (governance) должно присутствовать на каждом этапе, не только в production.

**Повторные попытки и предохранители на уровне агента** — в отличие от обычных микросервисов (где повтор обычно безопасен), повтор для агентов может быть дорогим (каждый повтор — вызов LLM = токены = деньги). Предохранитель (circuit breaker) — если N последовательных вызовов упали, перестать пытаться и вернуть запасной вариант (fallback). Постепенная деградация (graceful degradation) — если основной агент недоступен, переключиться на упрощённый.

## 6. Security — безопасность

CNCF определяет три цели: authentication (аутентификация — проверка, кто ты), authorization (авторизация — проверка, что тебе можно), trust (доверие). И три области: agent identity, tenancy, data access.

### 6.1 Agent identity — идентичность агента

CNCF рекомендует: назначать каждому агенту уникальную идентичность нагрузки (workload identity), короткоживущие автоматически обновляемые учётные данные, аудит и логирование использования идентичности, проверку идентичности перед каждым действием, границы принуждения (service meshes — сервисные сетки, NetworkPolicy — сетевые политики, API gateways — API-шлюзы), безопасное именование (OWASP ANS).

**Когда user identity alone, когда agent identity?**

CNCF даёт чёткое различение:

{{< mermaid >}}
flowchart TB
    START["Какая идентичность нужна?"] --> Q1{"Агент живёт только<br/>пока user logged in?"}
    Q1 -->|"Да"| USER["User identity alone<br/>Агент наследует<br/>права пользователя"]
    Q1 -->|"Нет"| Q2{"Агент делает<br/>autonomous decisions?"}
    Q2 -->|"Да"| AGENT["Agent identity required<br/>Собственная идентичность<br/>+ own permissions"]
    Q2 -->|"Нет"| Q3{"Действия за пределами<br/>прав пользователя?"}
    Q3 -->|"Да"| AGENT
    Q3 -->|"Нет"| USER

    style USER fill:#2e7d32,color:#fff
    style AGENT fill:#c62828,color:#fff
{{< /mermaid >}}

**User identity alone**: short-lived, user-initiated tasks. Агент живёт пока пользователь залогинен. Пример: «объясни этот код» — агент использует права пользователя, никаких autonomous actions. Когда пользователь logout — агент прекращает работу.

**Требуется идентичность агента**: автономные решения (запуск рабочих процессов, вызовы API, размещение заказов), сохраняется после завершения сессии пользователя, межотдельческий доступ (доступ к данным других отделов). Пример: «разверни этот сервис в staging» — агент должен иметь права на развёртывание, не наследуемые от пользователя (который может не иметь прав на deploy).

**Практики agent identity:**

- **Unique workload identity** — каждый agent instance получает уникальный SVID (SPIFFE, раздел 3.7). Не shared service accounts с broad permissions. Scoped, ephemeral, least privileged.
- **Short-lived credentials** — OIDC tokens с TTL, SPIFFE SVID certificates, ephemeral API credentials. Время жизни привязано к lifetime агента: агент умер — креды истекли.
- **Audit and log** — какой агент какую идентичность использовал, когда, для какой цели. Append-only logs (логи только для добавления — нельзя изменить или удалить запись) для non-repudiation (неотрицаемости — невозможности отрицать действие).
- **Verify before each action** — re-authenticate и re-authorize mid-session для чувствительных действий. Не «проверил при старте и забыл».
- **Enforcement boundaries** — service meshes (mTLS, identity-aware routing), NetworkPolicy (агенты общаются только с авторизованными tools/services), API gateways. Layered defense против lateral movement (перемещения злоумышленника по сети).
- **Secure naming** — [OWASP ANS](https://genai.owasp.org/resource/agent-name-service-ans-for-secure-al-agent-discovery-v1-0/) (раздел 3.7) для криптографически верифицируемого discovery и naming.

**MCP Authorization.** MCP [specification 2025-06-18](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization) определяет OAuth 2.1 flow: MCP server как resource server, MCP client как OAuth client. PKCE обязателен. Audience binding через RFC 8707. Token — Bearer в каждом request. Это для user-scoped tool access — в дополнение к SVID (который для service-to-service).

### 6.2 Agent tenancy — тенантство

CNCF рекомендует: JIT access provisioning, PoLP, ABAC/PBAC, strict workload partitioning (namespace, container, network, hardware isolation).

**Что значит.**

**JIT** (Just-in-Time access — доступ «точно в срок») — права выдаются на время конкретной задачи, после выполнения — отзываются. Агент не имеет постоянных прав, только ephemeral (короткоживущие).

**PoLP** (Principle of Least Privilege — принцип минимальных привилегий) — агент получает минимальные необходимые права. CNCF особенно подчёркивает: «just in case» permissions («на всякий случай») опасны для агентов, потому что агенты **сконструированы исследовать варианты** (designed to explore options). Если у агента есть лишнее разрешение «на всякий случай» — он может его использовать.

**ABAC** (Attribute-Based Access Control — доступ на основе атрибутов) — права определяются атрибутами: кто (agent identity), что (resource type), когда (time of day), откуда (network location), зачем (task context). Более гибкий чем RBAC (Role-Based — по ролям).

**PBAC** (Policy-Based Access Control — доступ на основе политик) — права определяются политиками, описанными в декларативном виде. OPA/Rego — пример.

**Isolation patterns:**
- **Namespace separation** — разные агенты в разных K8s namespaces
- **Container isolation** — разные контейнеры, желательно с sandboxing (gVisor, Kata Containers)
- **Network segmentation** — NetworkPolicy, ограничивающая трафик
- **Hardware partitioning** — особенно GPU: MIG (Multi-Instance GPU, раздел 2.4) для изоляции

**Service mesh capabilities** — mTLS (двустороннее TLS), identity-aware routing (маршрутизация с учётом идентичности), authorization policies (политики авторизации). Istio/Linkerd предоставляют это из коробки.

### 6.3 Agent data access — доступ к данным

CNCF рекомендует: ограниченный доступ к хранилищам данных, защиту от prompt injection, ограничение вызова инструментов, нулевое доверие (zero-trust) для внутренних API, изолированные среды выполнения (TEE — доверительные среды выполнения, secure enclaves — защищённые анклавы, GPU confidential computing — конфиденциальные вычисления на GPU), защиту среды выполнения агента (утечка системного промпта, доступ к исходному коду).

#### Защита от prompt injection

**Что значит.** Prompt injection (внедрение промпта — атака, при которой злоумышленник внедряет вредоносную инструкцию в данные, которые LLM обрабатывает) — экзистенциальная угроза для агентных систем. Агент, который может вызывать инструменты и принимать автономные решения — главная мишень.

[Greshke et al. (2023)](https://arxiv.org/abs/2302.12173) — семинальная работа: косвенное внедрение промпта (indirect prompt injection), демонстрация атак на Bing Chat/GPT-4. Обработка полученных данных равносильна «произвольному выполнению кода» — то есть обработка результатов (например, web_search) эквивалентна выполнению произвольного кода.

CNCF рекомендует в общем виде: «строгая валидация и очистка входных данных, контекстно-зависимые шаблоны промптов и защитные ограждения (guardrails), мониторинг и обнаружение аномалий». Но конкретные техники эшелонированной защиты (defence-in-depth — несколько слоёв, чтобы пробить один было недостаточно):

{{< mermaid >}}
flowchart LR
    INPUT["Tool Output<br/>(untrusted)"] --> L1["Layer 1: Input Screening<br/>rule-based patterns<br/>+ anomaly classifier"]
    L1 --> L2["Layer 2: Instruction Hierarchy<br/>system above user above tool output"]
    L2 --> L3["Layer 3: Output Sanitization<br/>validate LLM output<br/>before tool execution"]
    L3 --> L4["Layer 4: Tool Sandboxing<br/>pre-approved tools only<br/>+ audit log"]
    L4 --> SAFE["Safe Execution"]

    style L1 fill:#c62828,color:#fff
    style L2 fill:#e65100,color:#fff
    style L3 fill:#f57f17,color:#000
    style L4 fill:#2e7d32,color:#fff
{{< /mermaid >}}

**Layer 1: Input Screening.** [Saleem et al. (2026)](https://arxiv.org/abs/2606.19660) — layered defense framework: rule-based patterns (фильтрация известных атак) + fine-tuned anomaly classifier (ML-модель для обнаружения аномалий). Attack Success Rate (ASR — доля успешных атак) снижен с 71.4% → 11.3%, median latency overhead 61.2 ms.

**Layer 2: Instruction Hierarchy.** [Wallace et al. (OpenAI, 2024)](https://arxiv.org/abs/2404.13208) — «The Instruction Hierarchy»: system > user > tool output. LLM обучается selectively ignore lower-privileged instructions. «Drastically increases robustness» даже для unseen attack types, с минимальной деградацией capabilities. Это каноническая техника: модель знает, что instructions в tool output — lowest priority, и может их игнорировать.

**Layer 3: Output Sanitization.** OWASP [LLM Top 10](https://genai.owasp.org/llm-top-10/): LLM02 — Insecure Output Handling. Валидация LLM output перед передачей в tools. Schema validation (раздел 3.5) — если output не соответствует ожидаемой схеме, reject.

**Layer 4: Tool Sandboxing.** Pre-approved tools only (allowlist — белый список, не blacklist — чёрный). Audit log всех tool executions. Restricted permissions: каждый tool имеет минимальные права.

[ISE](https://arxiv.org/abs/2410.09102) (Instructional Segment Embedding, ICLR 2025) — +18.68% robust accuracy на Instruction Hierarchy benchmark. [RETA](https://arxiv.org/abs/2606.15441) — per-attack ASR <10%, average ASR 2.92%.

#### Tool hijacking prevention

**Что значит.** Tool hijacking (перехват инструмента — злоумышленник получает контроль над tool'ом агента, заставляя его возвращать вредоносные данные). CNCF рекомендует: strict permission boundaries на tool invocation (только authorized tools), только pre-approved tool interfaces, audit и log всех tool execution requests.

В практике это значит: агент имеет allowlist инструментов (не может вызвать любой tool, только одобренные). Каждый tool call логируется: кто вызвал, когда, с какими аргументами, какой результат. Если tool начинает возвращать аномальные данные (например, web_search вдруг возвращает injection-payloads) — alert.

#### Zero-trust для internal APIs

CNCF рекомендует: limit API surface area, segregate APIs по agent roles/tasks, network segmentation, firewall rules, continuously monitor API usage, mTLS для всех inter-service и agent-tool communications.

**Что значит.** Internal APIs (внутренние API — сервисы внутри кластера, не exposed наружу) часто считаются «безопасными по умолчанию». В zero-trust — нет. Каждый API аутентифицирует и авторизует каждый запрос, даже от «своих» сервисов. mTLS (mutual TLS — двустороннее TLS, где обе стороны проверяют сертификаты друг друга) — обязательный.

#### Isolated runtime environments — TEEs

CNCF рекомендует: deploy agents в изолированных runtime-окружениях, hardware-based isolation: TEEs, secure enclaves, GPU-based confidential computing. Минимизировать system prompt leakage, restrict access к source code.

**Что значит.** TEE (Trusted Execution Environment — доверительная среда выполнения) — аппаратно-изолированная область процессора, где код выполняется с гарантиями конфиденциальности и целостности. Даже администратор хоста не может прочитать данные внутри TEE. Secure enclave — разновидность TEE (Intel SGX, AMD SEV-SNP, ARM CCA).

**GPU-based confidential computing** — NVIDIA H100/Blackwell поддерживают CC mode: GPU execution, memory, register states изолированы. Модели, training data, inference prompts защищены.

**Когда оправданы.** [PipeLLM (Tan et al., ASPLOS 2025)](https://arxiv.org/abs/2411.03357) измеряет: naive H100 CC = **52.8% throughput drop** (30B model), **88.2%** (66B). С PipeLLM optimization — <19.6%. Multi-GPU training с TEE — **8-41× runtime overhead** ([Lee et al., 2025](https://arxiv.org/abs/2501.11771)).

Оправданы для: multi-tenant inference с sensitive prompts (Apple Private Cloud Compute — canonical example), model IP protection, regulated industries (healthcare/finance). Не оправданы для: internal tooling, non-sensitive workloads, когда cost > compliance benefit.

**Side-channel caveat**: [OTRO](https://arxiv.org/abs/2606.17358) демонстрирует end-to-end recovery of user prompts из tokenizer access patterns на Intel TDX. TEE ≠ full protection. Defence-in-depth, не silver bullet.

**Confidential Containers** — [CNCF Sandbox project](https://confidentialcontainers.org/), pod-level CC для K8s. Стандартизирует CC на уровне пода, vendor-neutral.

**Что почитать.** [AMD EPYC Confidential Computing](https://www.amd.com/en/products/processors/server/epyc/confidential-computing.html), [NVIDIA GPU Operator Confidential Containers](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-confidential-containers.html).

#### Protect agent execution environment

CNCF рекомендует: minimize system prompt leakage (не экспонировать system prompts через APIs, logs, client-side code), restrict access к source code и runtime binaries, redact sensitive flows.

**Что значит.** System prompt (системный промпт — инструкция, задающая поведение агента) — часто содержит proprietary logic, business rules, security constraints. Если злоумышленник получит system prompt — он понимает, как агент принимает решения, и может искать уязвимости.

Практики:
- Не логировать system prompts целиком (redact, логировать только hash)
- Не ship readable scripts (использовать compiled artifacts, signed containers, encrypted packages)
- Context-scoped prompts (разные prompts для разных контекстов, не один monolithic)
- Prompt moderation/transformation слой до/после LLM (дополнительная фильтрация)

## 7. Заключение

CNCF «Cloud Native Agentic Standards» — карта территории agentic-систем в Kubernetes. Документ охватывает пять областей: от контейнерных основ до TEEs, от MCP до OWASP ANS. Написан плотно — каждый пункт одна-две фразы, за которыми стоит много контекста.

В этой статье я разобрал каждый пункт подробно. Что CNCF рекомендует, что это значит на практике, как реализовано сегодня, куда читать глубже. Главные выводы:

1. **Документ — хороший фундамент.** Zero-trust как базис, MELT для наблюдаемости, SPIFFE для идентичности, MCP для инструментов — правильные технологические выборы. Production-ready сегодня.

2. **Протоколы — в активной разработке.** MCP — де-факто стандарт (передан в AAIF). A2A — v1.0.1 под Linux Foundation, перспективен для cross-framework коммуникации. AP2 — ранняя стадия (v0.2), niche use case. ACP — конвергирует с A2A. Выбирайте MCP для tool access, A2A для agent-to-agent, остальное — watch.

3. **Идентичность — три варианта с разной зрелостью.** SPIFFE/SPIRE — CNCF Graduate, готово для production (Amazon, Netflix, Uber). Agntcy — ранняя стадия (nascent), Linux Foundation, BYOI («принеси свою идентичность») с Web3 DID (децентрализованные идентификаторы). OWASP ANS — стадия исследования, ZKP (доказательства с нулевым разглашением). Для production сегодня: SPIFFE. Agntcy и ANS — следить.

4. **Управление (governance) — обязательно с начала.** EU AI Act требует объяснимости (explainability) и аудируемости (auditability). Agent-as-a-Judge (агент-судья) — перспективен, но имеет предвзятости (position bias — позиционная, verbosity bias — к многословию, self-enhancement bias — к собственным ответам) и риск обманчивого судьи (deceptive judge). Комбинировать с детерминированными правилами и человеческой проверкой. MOF (Model Openness Framework) + Sigstore — правильная композиция для документации модели и проверки целостности.

5. **Безопасность — эшелонированная защита (defence-in-depth).** Prompt injection (внедрение промпта) — экзистенциальная угроза. Четыре слоя: фильтрация входных данных (input screening), иерархия инструкций (instruction hierarchy), очистка выходных данных (output sanitization), песочница для инструментов (tool sandboxing). TEE (доверительные среды выполнения) — для нагрузок, обусловленных compliance (соответствием требованиям), но накладные расходы 52-88% (без оптимизации) или <19.6% (с оптимизацией). Не серебряная пуля — побочные каналы утечки (side-channels) существуют.

Серия «AI Agent Design Patterns» прошла путь от ReAct (Part 1) до cloud-native standards (Part 7). Parts 1-6 — паттерны внутри агента. Part 7 — паттерны эксплуатации агентов в Kubernetes. Следующий уровень абстракции — но принцип тот же: deep dive, honest limitations, code examples.

**Что почитать дополнительно:**

- [CNCF «Cloud Native Agentic Standards»](https://www.cncf.io/blog/2026/03/23/cloud-native-agentic-standards/) — первоисточник
- [Yang et al. (2025) «A Survey of AI Agent Protocols»](https://arxiv.org/abs/2504.16736) — таксономия 16 протоколов
- [White et al. (2024) «Model Openness Framework»](https://arxiv.org/abs/2403.13784) — 3 класса, 17 компонентов
- [Reuel et al. (2024) «Open Problems in Technical AI Governance»](https://arxiv.org/abs/2407.14981) — TMLR 2025, 35 авторов
- [Zheng et al. (2023) «Judging LLM-as-a-Judge»](https://arxiv.org/abs/2306.05685) — biases, MT-bench, Chatbot Arena
- [Zhuge et al. (2024) «Agent-as-a-Judge»](https://arxiv.org/abs/2410.10934) — extension на agentic systems
- [Wallace et al. (2024) «Instruction Hierarchy»](https://arxiv.org/abs/2404.13208) — OpenAI, system > user > tool output
- [Greshke et al. (2023) «Indirect Prompt Injection»](https://arxiv.org/abs/2302.12173) — семинальная работа по prompt injection
- [Saleem et al. (2026) «Layered Defense Framework»](https://arxiv.org/abs/2606.19660) — ASR 71.4% → 11.3%
- [Khan (2026) «63 LLM-Agent Budget-Overrun Incidents»](https://arxiv.org/abs/2606.04056) — каталог production инцидентов
- [Jimenez et al. (2024) «SWE-bench»](https://arxiv.org/abs/2310.06770) — 2,294 SE problems
- [Yao et al. (2024) «τ-bench»](https://arxiv.org/abs/2406.12045) — pass^k metric
- [Liu et al. (2023) «AgentBench»](https://arxiv.org/abs/2308.03688) — 8 environments
- [Tan et al. (ASPLOS 2025) «PipeLLM»](https://arxiv.org/abs/2411.03357) — TEE overhead numbers
