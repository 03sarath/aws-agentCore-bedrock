# Smart Meal Planner Agent — Architecture

> **Notebook:** `meal_planner_with_memory_and_streaming.ipynb`
> **Model:** Amazon Nova 2 Lite (`global.amazon.nova-2-lite-v1:0`)
> **Framework:** Strands Agents + Amazon Bedrock AgentCore Runtime

---

## What Changed from the Original

| Original (Weather Agent) | New (Meal Planner Agent) |
|:---|:---|
| `get_weather()` tool | `search_recipe()` — TheMealDB API |
| `calculator()` tool | `get_meals_by_category()` — TheMealDB API |
| — | `calculator()` — retained for recipe scaling |
| Claude Haiku 4.5 | **Amazon Nova 2 Lite** |
| No memory | **AgentCore Memory** (cross-session persistence) |
| Full response only | **Streaming** (SSE token-by-token) |
| Single architecture phase | **3 phases**: Local → Configure → Deploy |

---

## Diagram 1 — Complete End-to-End Architecture

```mermaid
flowchart TD
    USER(["👤 User / Application"])

    subgraph LOCAL["① Local Environment  —  Development & Testing"]
        direction TB
        AGENT_L["🧬 Strands Agent\nstrands_meal_planner.py"]
        TL1["search_recipe(dish)"]
        TL2["get_meals_by_category(category)"]
        TL3["calculator()"]
        AGENT_L <-->|tool calls| TL1
        AGENT_L <-->|tool calls| TL2
        AGENT_L <-->|tool calls| TL3
    end

    subgraph DEPLOY["② Deployment Pipeline"]
        direction LR
        DEV(["</> Developer"])
        DOCKER["🐳 Dockerfile\n+ .dockerignore\nARM64"]
        CODEBUILD["AWS CodeBuild\nCloud build\nno local Docker needed"]
        ECR["Amazon ECR\nARM64 Container\nImage"]
        DEV -->|"configure()"| DOCKER
        DOCKER -->|"launch()"| CODEBUILD
        CODEBUILD -->|push image| ECR
    end

    subgraph AWS["③ AWS Cloud — Runtime"]
        direction TB

        subgraph RUNTIME["AgentCore Runtime"]
            direction TB
            EP["Runtime Endpoint\n/invocations  /ping"]
            AGENT_RT["🧬 Strands Agent\n@app.entrypoint\nPORT 8080"]
            EP --> AGENT_RT
        end

        subgraph TOOLS["Agent Tools"]
            T1["search_recipe(dish)\nTheMealDB API"]
            T2["get_meals_by_category()\nTheMealDB API"]
            T3["calculator()\nstrands_tools"]
        end

        subgraph MEMORY["AgentCore Memory"]
            MEM_GET["retrieve_memories()\nSemantic search\nper user_id"]
            MEM_PUT["ingest_events()\nStore conversation\nper user_id"]
        end

        NOVA["Amazon Bedrock\nAmazon Nova 2 Lite\nglobal.amazon.nova-2-lite-v1:0\n1M context window"]
        MEALDB[("TheMealDB API\nfree · no key\nthemealdb.com")]
        CW["CloudWatch Logs\n+ X-Ray Tracing\nObservability"]
    end

    %% Deploy pipeline feeds runtime
    ECR -->|"create / update\nAgentCore Runtime"| RUNTIME

    %% User invokes
    USER -->|"invoke payload\n{prompt, user_id}"| EP
    USER <-.->|"SSE Streaming\ntokens arrive live"| EP

    %% Memory flow
    AGENT_RT -->|"1 · BEFORE responding\nretrieve prefs"| MEM_GET
    MEM_GET -->|"past preferences\ninjected into system prompt"| AGENT_RT
    AGENT_RT -->|"4 · AFTER responding\nsave conversation"| MEM_PUT

    %% Tool calls
    AGENT_RT <-->|"2 · tool calls"| T1
    AGENT_RT <-->|"2 · tool calls"| T2
    AGENT_RT <-->|"2 · tool calls"| T3
    T1 & T2 -->|"HTTP GET"| MEALDB

    %% LLM call
    AGENT_RT <-.->|"3 · LLM inference\nstreaming tokens"| NOVA

    %% Observability
    RUNTIME -.->|auto-instrumented| CW

    %% Local also hits Bedrock
    AGENT_L <-.->|"LLM inference"| NOVA

    style LOCAL fill:#f0f4ff,stroke:#6366f1,stroke-width:2px
    style DEPLOY fill:#fff7ed,stroke:#f59e0b,stroke-width:2px
    style AWS fill:#f0fdf4,stroke:#16a34a,stroke-width:2px
    style RUNTIME fill:#dcfce7,stroke:#15803d,stroke-width:2px
    style MEMORY fill:#fef9c3,stroke:#ca8a04,stroke-width:2px
    style TOOLS fill:#f0f9ff,stroke:#0284c7,stroke-width:2px
```

---

## Diagram 2 — Deployment Pipeline (Configure → Launch → Invoke)

Mirrors the original `configure.png`, `launch.png`, and `invoke.png` — updated for the Meal Planner.

```mermaid
flowchart LR
    subgraph CODE["Agent Code\nstrands_meal_planner.py"]
        direction TB
        M["Nova 2 Lite\nBedrockModel"]
        F["Strands Framework\n@tool decorators"]
        D["@app.entrypoint\nBedrockAgentCoreApp"]
        MEM_CODE["Memory helpers\nretrieve / save"]
    end

    DEV(["</> Application\nDeveloper"])
    DOCKER["🐳 Dockerfile\nARM64\n+ .dockerignore"]
    CB["AWS CodeBuild\nCloud build"]
    ECR["Amazon ECR\nContainer Registry"]

    subgraph RT["AgentCore Runtime"]
        direction TB
        RA["Runtime Agent\nPORT 8080"]
        RE["/invocations\n/ping"]
    end

    APP(["👤 User\nApplication"])

    DEV -->|"runtime.configure()\nauto_create_ecr=True\nauto_create_execution_role=True\nenv: MEMORY_ID"| DOCKER
    CODE --> DOCKER
    DOCKER -->|"runtime.launch()\nCodeBuild mode"| CB
    CB -->|"ARM64 image push"| ECR
    ECR -->|"create_agent_runtime()"| RT
    APP -->|"invoke_agent_runtime()\n{prompt, user_id}"| RE
    RE --> RA
    RA -.->|"SSE streaming response"| APP

    style CODE fill:#f5f3ff,stroke:#7c3aed,stroke-width:2px
    style RT fill:#dcfce7,stroke:#15803d,stroke-width:2px
```

---

## Diagram 3 — Request Lifecycle (Sequence)

Shows exactly what happens inside the Runtime on every single request — the memory + tool + LLM flow.

```mermaid
sequenceDiagram
    actor User
    participant EP  as AgentCore Endpoint<br/>/invocations
    participant MEM as AgentCore Memory
    participant AGT as Strands Agent
    participant LLM as Amazon Nova 2 Lite<br/>(Bedrock)
    participant DB  as TheMealDB API

    User->>EP: POST {prompt, user_id}

    Note over EP,AGT: @app.entrypoint called

    EP->>MEM: retrieve_memories(user_id, query)
    MEM-->>AGT: past preferences<br/>(dietary restrictions, allergies, past meals)

    Note over AGT: Inject memory into system_prompt

    AGT->>LLM: Chat with enriched context
    LLM-->>AGT: decide to call tool

    AGT->>DB: search_recipe(dish)
    DB-->>AGT: recipe JSON (name, ingredients, instructions)

    AGT->>LLM: tool result → generate response
    LLM-->>AGT: streaming tokens

    AGT->>MEM: ingest_events(user_id, conversation)
    Note over MEM: AgentCore extracts & stores facts<br/>for future sessions

    AGT-->>EP: full response text
    EP-->>User: SSE streaming response<br/>(tokens arrive live)
```

---

## Diagram 4 — AgentCore Memory Flow Detail

Shows the before/after of memory across two separate sessions — the core teaching point.

```mermaid
flowchart TD
    subgraph S1["SESSION 1  —  Monday"]
        U1(["👤 User"])
        Q1["'I am vegetarian and\nallergic to nuts.\nSuggest a pasta dish.'"]
        A1["Agent responds with\nvegetarian, nut-free pasta"]
        SAVE["ingest_events()\n→ AgentCore Memory stores:\n• user is vegetarian\n• user allergic to nuts"]
        U1 --> Q1 --> A1 --> SAVE
    end

    subgraph MEM_STORE["AgentCore Memory Store\n(persistent · scoped per user_id)"]
        F1["✔ vegetarian"]
        F2["✔ allergic to nuts"]
        F3["✔ likes pasta"]
    end

    subgraph S2["SESSION 2  —  Tuesday  (new conversation)"]
        U2(["👤 User"])
        Q2["'What should I cook\nfor dinner tonight?'"]
        RETRIEVE["retrieve_memories()\n→ fetches stored facts\n→ injected into system_prompt"]
        A2["Agent suggests vegetarian,\nnut-free dinner options\n✅ WITHOUT being told again"]
        U2 --> Q2 --> RETRIEVE --> A2
    end

    SAVE --> MEM_STORE
    MEM_STORE --> RETRIEVE

    style S1 fill:#f0f9ff,stroke:#0284c7,stroke-width:2px
    style S2 fill:#f0fdf4,stroke:#16a34a,stroke-width:2px
    style MEM_STORE fill:#fef9c3,stroke:#ca8a04,stroke-width:2px
```

---

## Diagram 5 — Streaming vs Non-Streaming

```mermaid
flowchart LR
    subgraph NS["❌ Without Streaming"]
        direction LR
        U_NS(["👤 User"]) -->|"sends prompt"| W["[........waiting 3-5s........]"]
        W -->|"entire response at once"| R_NS["Full response\nappears at once"]
    end

    subgraph WS["✅ With Streaming  (SSE)"]
        direction LR
        U_WS(["👤 User"]) -->|"sends prompt"| T1_S["'Here...'"]
        T1_S --> T2_S["'is a...'"]
        T2_S --> T3_S["'great...'"]
        T3_S --> T4_S["'recipe...'"]
        T4_S --> T5_S["tokens arrive\nimmediately"]
    end

    style NS fill:#fff1f2,stroke:#e11d48,stroke-width:2px
    style WS fill:#f0fdf4,stroke:#16a34a,stroke-width:2px
```

---

## Component Summary

| Component | Service / Tool | Role |
|:---|:---|:---|
| **Agent Framework** | Strands Agents | Orchestrates tools + LLM loop |
| **LLM** | Amazon Nova 2 Lite (Bedrock) | Reasoning, response generation |
| **Tool 1** | `search_recipe()` → TheMealDB | Fetch recipe by dish name |
| **Tool 2** | `get_meals_by_category()` → TheMealDB | Browse meals by category |
| **Tool 3** | `calculator()` → strands_tools | Scale recipe quantities |
| **Memory** | AgentCore Memory | Persist user prefs cross-session |
| **Streaming** | SSE via AgentCore Runtime | Token-by-token response |
| **Container Build** | AWS CodeBuild | ARM64 image build (no local Docker) |
| **Image Registry** | Amazon ECR | Store ARM64 container image |
| **Hosting** | AgentCore Runtime | Serverless agent hosting, `/invocations` + `/ping` |
| **Observability** | CloudWatch + X-Ray | Logs, traces, GenAI dashboard |

---

## Key Code Touchpoints

| What | File | Pattern |
|:---|:---|:---|
| Local agent entry | `meal_planner.py` | `def meal_planner(payload)` |
| AgentCore entry | `strands_meal_planner.py` | `@app.entrypoint` |
| Memory retrieve | `strands_meal_planner.py` | `retrieve_preferences(user_id, query)` |
| Memory save | `strands_meal_planner.py` | `save_interaction(user_id, msg, reply)` |
| Model selection | both files | `BedrockModel(model_id="global.amazon.nova-2-lite-v1:0")` |
| Deploy configure | notebook cell | `runtime.configure(entrypoint=..., environment_variables={"MEMORY_ID": ...})` |
| Deploy launch | notebook cell | `runtime.launch()` — CodeBuild mode |
| Invoke | notebook cell | `agentcore_runtime.invoke({prompt, user_id})` |
| Streaming invoke | notebook cell | `boto3` EventStream parsing |
