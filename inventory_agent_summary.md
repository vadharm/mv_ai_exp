Inventory AI Agent — Phases Summary
Extracted from inventory_agent_implementation_guide.md.

This document summarizes all phases and their rationale (Why statements) for the Agentic Text-to-SQL architecture built on Nike's ~28B-row, ~3 TB marketplace inventory dataset.

Architecture Diagram
The diagram below shows how every component from the phases below fits together at runtime. Solid arrows = request/data flow, dotted arrows = metadata / build-time / supporting relationships.

flowchart TB
    %% =========================================================
    %% CONSUMER LAYER  (Phase 7)
    %% =========================================================
    subgraph CONSUMERS["Consumer Layer — Phase 7"]
        direction LR
        REST["REST API Client<br/>(7.1)"]
        SLACK["Slack Bot<br/>(7.2)"]
        NB["Databricks Notebook<br/>(7.3 dev/test)"]
    end

    %% =========================================================
    %% GATEWAY + SERVING  (Phase 4)
    %% =========================================================
    subgraph DEPLOY["Deployment & Gateway — Phase 4"]
        direction TB
        GW["AI Gateway (4.3)<br/>Rate Limits • PII Redaction • Cost Tracking"]
        EP["Model Serving Endpoint (4.2)<br/>inventory-ai-agent (Serverless, scale-to-zero)"]
        MLF["MLflow + Unity Catalog Registry (4.1)<br/>Versioned Agent Artifact"]
    end

    %% =========================================================
    %% AGENT CORE  (Phase 2)
    %% =========================================================
    subgraph AGENT["Agent Core — Phase 2"]
        direction TB
        ORCH["Agent Orchestrator<br/>(2.6 Mosaic AI ChatAgent / 2.7 LangChain)"]
        SP["System Prompt (2.5)<br/>Partition Rules • Source Rules • Few-Shots"]
        LLM["Foundation Model<br/>Llama 3.1 70B / DBRX / GPT-4o"]
        CFG["Config Constants (2.1)"]
    end

    %% =========================================================
    %% AGENT TOOLS  (Phase 2 + 5)
    %% =========================================================
    subgraph TOOLS["Agent Tools (UC Functions — 2.4)"]
        direction TB
        SQLTOOL["SQL Executor (2.2)<br/>Validation • Guardrails • LIMIT 1000 • 90s timeout"]
        SCHEMA["Schema Inspector (2.3)<br/>DESCRIBE TABLE EXTENDED"]
        RAGTOOL["RAG Retrieval (5.2 / 5.3)<br/>glossary_lookup"]
    end

    %% =========================================================
    %% RAG GLOSSARY  (Phase 5)
    %% =========================================================
    subgraph RAGSTORE["RAG Glossary — Phase 5"]
        direction TB
        VS["Vector Search Index (5.1)<br/>inventory_glossary_index"]
        BGE["Embedding Model<br/>databricks-bge-large-en"]
        GUIDE[["Business Data Guide<br/>(markdown chunks)"]]
    end

    %% =========================================================
    %% DATA LAYER  (Phase 1)
    %% =========================================================
    subgraph DATA["Data Layer — Phase 1"]
        direction TB
        WH["SQL Warehouse (1.3)<br/>Serverless Pro • Photon • Large"]
        VIEW[("total_marketplace_inventory_base_v1<br/>UNION ALL view • ~28B rows • ~3 TB")]
        T1[("dc_digital_3pl_inventory_v1")]
        T2[("pond_inventory_v1")]
        T3[("store_inventory_v1")]
        UCM["Unity Catalog Metadata (1.1)<br/>View + Column Comments"]
        OPT["OPTIMIZE + ANALYZE (1.2)<br/>Bin-pack • Column Stats"]
    end

    %% =========================================================
    %% EVALUATION  (Phase 3)
    %% =========================================================
    subgraph EVAL["Evaluation Loop — Phase 3 (build-time / CI)"]
        direction TB
        GOLD["Golden Queries (3.1)"]
        AGEV["Mosaic AI Agent Eval (3.2)"]
        SCORE["Custom SQL Scorer (3.3)"]
    end

    %% =========================================================
    %% OBSERVABILITY  (Phase 6)
    %% =========================================================
    subgraph OBS["Observability — Phase 6"]
        direction TB
        INF["Inference Tables (6.1)<br/>Request/Response audit logs"]
        ALERT["Health Checks & Alerts (6.2)<br/>error rate • p95 latency"]
    end

    %% ---------- REQUEST FLOW (solid) ----------
    REST --> GW
    SLACK --> GW
    GW --> EP
    EP --> MLF
    MLF --> ORCH
    NB -. dev/test .-> ORCH

    %% ---------- AGENT REASONING ----------
    ORCH <--> LLM
    SP --> ORCH
    CFG -. config .-> ORCH
    CFG -. config .-> SQLTOOL

    %% ---------- TOOL CALLS ----------
    ORCH -- "tool: execute SQL" --> SQLTOOL
    ORCH -- "tool: inspect schema" --> SCHEMA
    ORCH -- "tool: lookup glossary" --> RAGTOOL

    %% ---------- TOOLS → DATA ----------
    SQLTOOL --> WH
    SCHEMA --> WH
    WH --> VIEW
    VIEW --> T1
    VIEW --> T2
    VIEW --> T3
    UCM -. enriches .-> VIEW
    UCM -. fuels prompt .-> SP
    OPT -. tunes .-> T1
    OPT -. tunes .-> T2
    OPT -. tunes .-> T3

    %% ---------- RAG ----------
    RAGTOOL --> VS
    VS -. embeds via .-> BGE
    GUIDE -. chunked into .-> VS

    %% ---------- EVALUATION (build-time) ----------
    GOLD --> AGEV
    GOLD --> SCORE
    AGEV -. validates .-> ORCH
    SCORE -. validates .-> ORCH

    %% ---------- OBSERVABILITY ----------
    EP --> INF
    INF --> ALERT

    %% ---------- STYLING ----------
    classDef phase1 fill:#e3f2fd,stroke:#1565c0,color:#0d47a1
    classDef phase2 fill:#fff3e0,stroke:#e65100,color:#bf360c
    classDef phase3 fill:#f3e5f5,stroke:#6a1b9a,color:#4a148c
    classDef phase4 fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef phase5 fill:#fce4ec,stroke:#ad1457,color:#880e4f
    classDef phase6 fill:#fffde7,stroke:#f9a825,color:#f57f17
    classDef phase7 fill:#eceff1,stroke:#455a64,color:#263238

    class WH,VIEW,T1,T2,T3,UCM,OPT phase1
    class ORCH,SP,LLM,CFG,SQLTOOL,SCHEMA phase2
    class GOLD,AGEV,SCORE phase3
    class GW,EP,MLF phase4
    class RAGTOOL,VS,BGE,GUIDE phase5
    class INF,ALERT phase6
    class REST,SLACK,NB phase7
Runtime Request Flow (end-to-end)
Consumer (REST / Slack / Notebook) sends a natural language question.
AI Gateway (4.3) applies rate limits, PII redaction, and cost tracking, then forwards to the Serving Endpoint (4.2).
The endpoint loads the versioned agent artifact from the MLflow + UC registry (4.1).
The Agent Orchestrator (2.6/2.7) primes the LLM with the System Prompt (2.5) — which itself was built from the Unity Catalog metadata (1.1).
The LLM decides which tool to call:
Schema Inspector (2.3) to verify column names,
RAG Retrieval (5.2) to fetch partition-specific context from the Vector Search index (5.1),
SQL Executor (2.2) to validate + run the generated SQL.
The SQL Executor routes the validated query to the Serverless SQL Warehouse (1.3), which scans the UNION ALL view (1.1) over the three partitioned, OPTIMIZE-/ANALYZE-tuned Delta tables (1.2).
Results return to the LLM, which produces a natural-language answer with source caveats.
Every request/response pair is logged to Inference Tables (6.1); alerts (6.2) fire on error-rate or latency regressions.
Evaluation (3.1–3.3) runs out-of-band (CI / pre-deploy) to gate new prompt or tool changes against the Golden Query set.
Phase 1: Data Layer Setup
1.1 Unity Catalog Metadata — View & Column Comments
Why: The Mosaic AI Agent Framework reads Unity Catalog metadata (table/column comments) to understand the schema. Without rich comments, the LLM will hallucinate column names or misunderstand the UNION ALL structure. This is the single most important step — the LLM generates SQL quality directly proportional to metadata quality.

1.2 Optimize Underlying Tables
Why: The tables are already partitioned by 5 columns (region_partition_cd, source_partition_desc, inventory_type_partition_cd, inventory_month_year_nbr, report_dt) and this partitioning is performing well for both reads and writes. Do NOT add Liquid Clustering or ZORDER — both are incompatible with Hive-style partitioning on the same Delta table. Instead, run OPTIMIZE for bin-packing (compacts small files) and ANALYZE TABLE to generate column-level statistics for data skipping within partitions.

1.3 SQL Warehouse Configuration
Why: A dedicated Serverless SQL Warehouse isolates agent query workloads from BI dashboards and ETL pipelines. The agent generates unpredictable queries — auto-scaling handles burst traffic while scaling to zero eliminates idle costs.

Phase 2: Agent Tools — SQL Executor & Schema Inspector
2.1 Configuration Constants
Why: Centralizes all configuration — catalog/schema names, warehouse ID, partition column names, and source system mappings. Every other component references these constants.

2.2 SQL Executor Tool
Why: This is the agent's primary tool — it takes a SQL query string, validates it against guardrails, executes it on the SQL Warehouse, and returns the result as a formatted string. Registered as a Unity Catalog function so it is governed, versioned, and auditable.

2.3 Schema Inspector Tool
Why: Before generating SQL, the agent should inspect the actual table schema from Unity Catalog. This prevents hallucinating column names that don't exist. The tool returns column names, types, and comments — the comments are the "data dictionary" the LLM uses to write correct SQL.

2.4 Register Tools as Unity Catalog Functions
Why: Registering the tools as UC Functions makes them governed (access control), versioned (schema evolution), and auditable (lineage tracking). The Agent Framework discovers UC Functions automatically when they are associated with the agent.

Phase 2: System Prompt Construction
2.5 System Prompt as a Python Constant
Why: The system prompt is the most critical artifact — it encodes the partition scheme, source-awareness rules, measure availability, and few-shot examples that drive SQL accuracy. Stored as a Python constant so it can be versioned, tested, and A/B tested via MLflow.

Phase 2: Agent Construction (Mosaic AI Agent Framework)
2.6 Build the Agent with Mosaic AI Agent Framework
Why: The Agent Framework ties together the LLM, the system prompt, and the tools (SQL Executor, Schema Inspector) into a single deployable unit. It manages multi-turn conversation state, tool invocation, and response formatting. Logged with MLflow for versioning, A/B testing, and rollback.

2.7 Alternative: LangChain-Based Agent
Why: If you prefer LangChain's tool-calling abstraction over the native ChatAgent interface, this implementation provides the same functionality using LangChain's agent executor with Databricks-hosted models.

Phase 3: Evaluation Framework (Golden Queries)
3.1 Golden Query Evaluation Set
Why: The evaluation set is the ground truth for measuring SQL accuracy. Each entry contains a natural language question, the expected SQL, and the expected answer. The agent is tested against these to measure accuracy before deployment and after every prompt change.

3.2 Run Evaluation with Mosaic AI Agent Evaluation
Why: Mosaic AI Agent Evaluation provides automated scoring of the agent's SQL accuracy, response quality, and safety. It generates a scorecard that tracks improvements over time and catches regressions when prompts or tools change.

3.3 Custom SQL Accuracy Scorer
Why: The built-in evaluator checks response quality, but we also need to verify that the generated SQL includes the correct partition filters, targets the right sources, and avoids known traps. This custom scorer checks structural properties of the generated SQL.

Phase 4: MLflow Logging & Model Serving Deployment
4.1 Log the Agent with MLflow
Why: MLflow packages the agent (code + system prompt + tool definitions + dependencies) into a single versioned artifact. This enables A/B testing different prompts, rolling back to previous versions, and tracking which version produced which responses.

4.2 Deploy to Model Serving Endpoint
Why: The Model Serving endpoint exposes the agent as a production REST API. Serverless compute scales to zero when idle (no cost) and auto-scales under load. External consumers (Slack bots, custom apps, dashboards) call this endpoint via HTTP POST.

4.3 Configure AI Gateway
Why: The AI Gateway adds rate limiting, PII redaction, and cost tracking on top of the Model Serving endpoint. It protects the 28B-row dataset from excessive queries and prevents sensitive data from leaking in responses.

Phase 5: Optional RAG Glossary (Vector Search)
5.1 Embed the Business Data Guide as a RAG Index
Why: The system prompt cannot encode all 40 partition-specific nuances (record counts, date ranges, which measures are populated where, deprecated sources, IOH-driven vs coverage-driven processing, etc.). A small RAG index over the business data guide lets the agent retrieve partition-specific context at query time, significantly improving SQL accuracy for complex questions.

5.2 RAG Retrieval Tool for the Agent
Why: This tool lets the agent retrieve relevant business context from the glossary before generating SQL. When a user asks about "3PL inventory" or "NSP sell-through", the agent first retrieves the partition description, then generates more accurate SQL.

5.3 Integrate RAG Tool into the Agent
Why: With the RAG tool added, the agent can first retrieve partition-specific context, then generate SQL with full awareness of which measures are available, which sources are deprecated, and what caveats apply.

Monitoring & Observability (Ongoing)
6.1 Query Inference Tables for Monitoring
Why: Inference tables automatically capture every request/response pair, including latency, token usage, and errors. Monitoring these tables lets you identify failing query patterns, expensive queries, and accuracy degradation over time.

6.2 Alerting on Degraded Performance
Why: Automated alerts catch issues before users report them — rising error rates, increased latency, or new query patterns that fail consistently. Implemented as a Databricks SQL Alert or a scheduled notebook.

Consumer Integration Examples
7.1 REST API Call (Python)
Why: This is how external applications (dashboards, Slack bots, custom portals) consume the agent. A simple HTTP POST with a question; the agent returns a natural language answer.

7.2 Slack Bot Integration (Webhook)
Why: Many inventory analysts use Slack daily. A Slack bot provides the most natural integration — users type a question in a channel, and the agent responds in-thread.

7.3 Quick Test in a Databricks Notebook
Why: Before deploying to production, test the agent interactively in a notebook. This loop lets you iterate on questions and inspect the generated SQL.
