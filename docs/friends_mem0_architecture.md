# 🧠 Mem0 Friends Demo — Architecture & Component Diagram

This document presents the system architecture, component layout, data flow, and read/write path lifecycles strictly for the **Mem0 Friends Demo Series (Phases 1–5)**.

---

## 1. High-Level System Architecture Diagram

```mermaid
graph TB
    subgraph ClientLayer ["Client & Application Layer (Friends Demo Notebooks)"]
        NB1["Phase 1: Core Features<br/>(mem0_friends_phase1_features.ipynb)"]
        NB2["Phase 2: RAG & Scoping<br/>(mem0_friends_phase2_generation_conflict_scoping.ipynb)"]
        NB3["Phase 3: Read Path & Scores<br/>(mem0_friends_phase3_readpath.ipynb)"]
        NB4["Phase 4: Temporal Decay<br/>(mem0_friends_phase4_decay.ipynb)"]
        NB5["Phase 5: Auto-Operations<br/>(mem0_friends_phase5_add_update_delete_noop.ipynb)"]
        AWM["answer_with_memory()<br/>RAG Generation Pipeline"]
    end

    subgraph LLMConfigLayer ["Lab Configuration & LLM Adapter"]
        CFG["lab_llm_config.py"]
        ENV[".env Configuration<br/>(OPENAI_API_KEY, HF_TOKEN, MEM0_API_KEY)"]
        ADK["Google ADK LiteLlm Integration"]
    end

    subgraph MemoryPipelines ["Mem0 Pipeline Engine (Write & Read Paths)"]
        WRITE_PATH["Write Path (add)<br/>• LLM Fact Summarizer<br/>• SentenceTransformers Embedding<br/>• Auto-Deduplication Engine"]
        READ_PATH["Read Path (search / get_all)<br/>• Cosine Vector Similarity<br/>• BM25 Keyword Scoring<br/>• Score Threshold & Top-K Cutoff<br/>• Temporal Decay Reranker (0.3x-1.5x)"]
        AUTO_OPS["Auto-Decision Engine<br/>• ADD | UPDATE | DELETE | NOOP"]
    end

    subgraph StorageModes ["Storage Execution Modes"]
        subgraph ModeA ["Mode A: Local Open-Source (Memory.from_config)"]
            OSS_MEM["mem0ai OSS Core"]
            CHROMA[("Local ChromaDB<br/>./chroma_db")]
            LAB_VLLM["Lab vLLM Endpoint<br/>http://10.0.10.51:8000/v1<br/>(openai/gpt-oss-20b)"]
        end

        subgraph ModeB ["Mode B: Cloud Hosted (MemoryClient)"]
            M0_CLIENT["MemoryClient (REST SDK)"]
            M0_CLOUD["Mem0 Cloud Platform API<br/>https://api.mem0.ai/v3/"]
            M0_ASYNC["Async Cloud Queue<br/>(PENDING -> COMPLETED)"]
        end
    end

    NB1 & NB2 & NB3 & NB4 & NB5 -->|"Cloud API Key"| M0_CLIENT
    AWM -->|"1. Search"| M0_CLIENT
    AWM -->|"2. LLM Completion"| CFG

    CFG -->|"load_lab_env()"| ENV
    CFG -->|"complete()"| LAB_VLLM
    CFG -->|"LiteLlm"| ADK

    M0_CLIENT --> WRITE_PATH & READ_PATH
    WRITE_PATH --> AUTO_OPS --> M0_CLOUD
    READ_PATH --> M0_CLOUD
    M0_CLOUD --> M0_ASYNC

    OSS_MEM --> CHROMA
    OSS_MEM -->|"Extract Facts"| LAB_VLLM
```

---

## 2. End-to-End RAG Generation & Memory Flow (`answer_with_memory`)

```mermaid
sequenceDiagram
    autonumber
    actor User as User / Notebook Cell
    participant App as answer_with_memory()
    participant Mem0 as Mem0 Client / Store
    participant Config as lab_llm_config
    participant vLLM as Lab LLM Router

    User->>App: Call answer_with_memory(question, user_id="maya")
    App->>Mem0: client.search(query=question, filters={"user_id": "maya"})
    Mem0->>Mem0: Compute Cosine + BM25 Composite Score & Apply Threshold/Top-K
    Mem0-->>App: Return ranked facts [{"score": 0.291, "memory": "Maya likes hiking..."}]

    App->>App: Format System Prompt + Insert Memory Bullets

    App->>Config: complete(full_prompt)
    Config->>vLLM: POST /v1/chat/completions (model: openai/gpt-oss-20b)
    vLLM-->>Config: Response content ("Maya is an active outdoors person...")
    Config-->>App: Return answer string

    App->>Mem0: client.add([{"role":"user",...},{"role":"assistant",...}], user_id="maya")
    Mem0-->>App: Return Async Event Status {"event_id": "...", "status": "PENDING"}

    App-->>User: Return (answer, used_memories)
```

---

## 3. Multi-Entity Scope Architecture & Auto-Decision Router

```mermaid
graph TD
    subgraph InputStatements ["Incoming Statements"]
        ST1["'I started pottery on Tuesdays'"]
        ST2["'Aviation safety rules must be verified'"]
        ST3["'Current session troubleshooting state'"]
    end

    subgraph ScopeRouting ["Entity Scope Routing"]
        U_SCOPE["user_id Scope<br/>(Personal facts & user preferences)"]
        A_SCOPE["agent_id Scope<br/>(System instructions & agent rules)"]
        R_SCOPE["run_id Scope<br/>(Transient session / turn context)"]
    end

    subgraph DecisionEngine ["Mem0 Contradiction & Action Decision Router"]
        OP_ADD["ADD<br/>(New unique fact)"]
        OP_UPD["UPDATE<br/>(Reconcile contradiction)"]
        OP_DEL["DELETE<br/>(Remove invalidated fact)"]
        OP_NOOP["NOOP<br/>(Duplicate fact ignored)"]
    end

    ST1 --> U_SCOPE
    ST2 --> A_SCOPE
    ST3 --> R_SCOPE

    U_SCOPE & A_SCOPE & R_SCOPE --> DecisionEngine
    DecisionEngine --> OP_ADD & OP_UPD & OP_DEL & OP_NOOP
```

---

## 4. Key Component Matrix

| Component | Path / Notebook | Responsibilities | Key API Syntax |
| :--- | :--- | :--- | :--- |
| **Lab LLM Config** | [`lab_llm_config.py`](file:///Users/sangeetha/SV-Summer2026/agents/lab/mem0_lite/lab_llm_config.py) | Environment loading, endpoint config, LLM completion, telemetry suppression. | `load_lab_env()`, `complete(prompt)` |
| **Phase 1 Demo** | `mem0_friends_phase1_features.ipynb` | Full CRUD lifecycle, user isolation, metadata filtering. | `client.add()`, `get_all()`, `update()`, `delete()` |
| **Phase 2 Demo** | `mem0_friends_phase2_generation_conflict_scoping.ipynb` | RAG answer generation & multi-entity scoping (`user_id`, `agent_id`, `run_id`). | `answer_with_memory()`, `filters={"user_id": ...}` |
| **Phase 3 Demo** | `mem0_friends_phase3_readpath.ipynb` | Read path inspection, composite scoring ($0.05$ to $0.35$), score thresholding & Top-K limits. | `client.search(query, filters=..., top_k=...)` |
| **Phase 4 Demo** | `mem0_friends_phase4_decay.ipynb` | Time-weighted recency & access frequency decay (`0.3x` to `1.5x` score multipliers). | `client.project.update(decay=True)` |
| **Phase 5 Demo** | `mem0_friends_phase5_add_update_delete_noop.ipynb` | Automatic contradiction resolution & action classification (`ADD`, `UPDATE`, `DELETE`, `NOOP`). | `client.add(message, user_id=...)` |
