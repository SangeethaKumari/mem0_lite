# Mem0 Architecture & System Component Diagram

This document presents the system architecture, component layout, and data flow across both **Local Open-Source (`Memory`)** and **Cloud Hosted (`MemoryClient`)** modes in the lab environment.

---

## 1. High-Level System Architecture Diagram

```mermaid
graph TB
    subgraph ClientLayer ["Client & Application Layer"]
        NB1["Phase 1 Notebook<br/>(blackbox_phase1_mem0.ipynb)"]
        NB2["Phase 2 Notebook<br/>(blackbox_phase2_mem0.ipynb)"]
        AWM["answer_with_memory()<br/>RAG Generation Pipeline"]
    end

    subgraph LLMConfigLayer ["Lab Configuration & LLM Adapter"]
        CFG["lab_llm_config.py"]
        ENV[".env Configuration<br/>(OPENAI_API_KEY, HF_TOKEN, MEM0_API_KEY)"]
        ADK["Google ADK LiteLlm Integration"]
    end

    subgraph LocalMem0Mode ["Mode A: Local Open-Source (Memory.from_config)"]
        OSS_MEM["mem0ai OSS Core"]
        HF_EMB["SentenceTransformers<br/>(all-MiniLM-L6-v2)"]
        CHROMA[("Local ChromaDB<br/>./chroma_db")]
        LAB_VLLM["Lab vLLM Endpoint<br/>http://10.0.10.51:8000/v1<br/>(openai/gpt-oss-20b)"]
    end

    subgraph CloudMem0Mode ["Mode B: Cloud Hosted (MemoryClient)"]
        M0_CLIENT["MemoryClient"]
        M0_CLOUD["Mem0 Cloud Platform<br/>https://api.mem0.ai/v3/"]
        M0_ASYNC["Async Task Queue<br/>(PENDING -> COMPLETED)"]
    end

    NB1 -->|"Local Config"| OSS_MEM
    NB2 -->|"Cloud API Key"| M0_CLIENT

    CFG -->|"load_lab_env()"| ENV
    CFG -->|"complete()"| LAB_VLLM
    CFG -->|"LiteLlm"| ADK

    OSS_MEM --> HF_EMB
    OSS_MEM --> CHROMA
    OSS_MEM -->|"Extract Facts"| LAB_VLLM

    M0_CLIENT -->|"REST API"| M0_CLOUD
    M0_CLOUD --> M0_ASYNC
    AWM -->|"1. Search"| M0_CLIENT
    AWM -->|"2. LLM Completion"| CFG
```

---

## 2. End-to-End Generation & Memory Flow (`answer_with_memory`)

```mermaid
sequenceDiagram
    autonumber
    actor User as User / Notebook Cell
    participant App as answer_with_memory()
    participant Mem0 as Mem0 Client / Store
    participant Config as lab_llm_config
    participant vLLM as Lab LLM Router

    User->>App: Call answer_with_memory(question, user_id="eng_01")
    App->>Mem0: client.search(query=question, filters={"user_id": "eng_01"})
    Mem0-->>App: Return relevant facts [{"memory": "User works on B737..."}]

    App->>App: Format System Prompt + Insert Retrieved Memory Bullets

    App->>Config: complete(full_prompt)
    Config->>vLLM: POST /v1/chat/completions (model: openai/gpt-oss-20b)
    vLLM-->>Config: Response content ("The hydraulic backup system is...")
    Config-->>App: Return answer string

    App->>Mem0: client.add([{"role":"user","content":...},{"role":"assistant","content":...}], user_id="eng_01")
    Mem0-->>App: Async Event Status {"event_id": "...", "status": "PENDING"}

    App-->>User: Return (answer, used_memories)
```

---

## 3. Scope & Filter Architecture (`user_id`, `agent_id`, `run_id`)

Mem0 segments memory into distinct layers using entity scopes:

```mermaid
graph TD
    subgraph MemoryStore ["Mem0 Unified Storage"]
        U["User Layer (user_id='eng_01')<br/>User profile, domain preferences, aircraft role"]
        A["Agent Layer (agent_id='maintenance_agent')<br/>Safety rules, compliance standards, agent instructions"]
        R["Run / Session Layer (run_id='session_104')<br/>Transient session facts, current workflow context"]
    end

    subgraph QueryFilters ["Query & Search Filters (v2.0+ Syntax)"]
        F1["filters={'user_id': 'eng_01'}"]
        F2["filters={'agent_id': 'maintenance_agent'}"]
        F3["filters={'user_id': 'eng_01', 'run_id': 'session_104'}"]
    end

    F1 -->|Retrieves| U
    F2 -->|Retrieves| A
    F3 -->|Retrieves Multi-Scope| U
    F3 -->|Retrieves Multi-Scope| R
```

---

## 4. Key Component Matrix

| Component | Path / Function | Responsibilities | Key API Syntax |
| :--- | :--- | :--- | :--- |
| **Lab LLM Config** | [`lab_llm_config.py`](file:///Users/sangeetha/SV-Summer2026/agents/lab/mem0_lite/lab_llm_config.py) | Environment loading, lab endpoint configuration, LLM completion, telemetry suppression. | `load_lab_env()`, `complete(prompt)` |
| **Local Mem0** | [`blackbox_phase1_mem0.ipynb`](file:///Users/sangeetha/SV-Summer2026/agents/lab/mem0_lite/docs/notebooks/blackbox_phase1_mem0.ipynb) | Local memory management via ChromaDB & HuggingFace embeddings. | `Memory.from_config(config)` |
| **Cloud Mem0** | [`blackbox_phase2_mem0.ipynb`](file:///Users/sangeetha/SV-Summer2026/agents/lab/mem0_lite/docs/notebooks/blackbox_phase2_mem0.ipynb) | Hosted Mem0 API client for cloud storage & async memory extraction. | `MemoryClient(api_key=...)` |
| **Memory Extraction (`add`)** | `client.add(...)` | Accepts raw dialogue, uses LLM to extract key facts, writes to vector store. | `client.add(messages, user_id="...")` *(Top-level entity param)* |
| **Memory Listing (`get_all`)** | `client.get_all(...)` | Retrieves all stored facts within a given entity scope. | `client.get_all(filters={"user_id": "..."})` *(Dictionary filter)* |
| **Memory Search (`search`)** | `client.search(...)` | Performs semantic search against stored memory vectors. | `client.search(query, filters={"user_id": "..."})` *(Dictionary filter)* |
