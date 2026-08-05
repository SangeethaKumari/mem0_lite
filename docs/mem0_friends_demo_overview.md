# 🌟 Mem0 Friends Demo — Complete Executive Overview

> **Presentation & Showcase Guide**  
> *A 5-Phase Walkthrough of Long-Term AI Memory for AI Agents*

---

## 🎯 Purpose & Story Overview

Standard LLMs are **stateless** — every new conversation resets context.  
**Mem0** provides a persistent, dynamic memory layer that allows AI applications to:
1. **Extract Facts Automatically**: Convert raw conversations into structured user facts.
2. **Search by Meaning**: Perform semantic retrieval across user histories.
3. **Scope & Isolate Data**: Keep individual user, agent, and session memories strictly segregated.
4. **Resolve Conflicts**: Reconcile changing user preferences over time.

### 👥 The Demo Characters & User Isolation
Throughout the demo, three friends illustrate strict user memory isolation:
- 🏔️ **Maya**: Hikes in the Rockies, takes pottery on Tuesdays, later switches to painting.
- 🍳 **Jordan**: Cooks Italian dinners, plays jazz piano to unwind.
- 🥜 **Sam**: Plays video games, has a **peanut allergy** (safety-critical constraint).

---

## 🧠 Memory Scopes vs. Memory Types (`memory_type`)

It is crucial to distinguish between **who a memory belongs to** (Entity Scope) and **what kind of memory it is** (Memory Type):

```mermaid
graph TD
    subgraph Dimension1 ["1. Entity Scope (WHO / WHERE)"]
        U_ID["user_id Scope<br/>Personal facts & preferences"]
        A_ID["agent_id Scope<br/>Agent persona & behavioral guidelines"]
        R_ID["run_id Scope<br/>Transient session context"]
    end

    subgraph Dimension2 ["2. Memory Type (WHAT KIND OF MEMORY)"]
        SEM["Semantic Memory (DEFAULT)<br/>Extracted facts & knowledge<br/>e.g. 'Maya likes hiking'"]
        EPI["Episodic Memory (DEFAULT)<br/>Events & conversation logs<br/>e.g. 'User hiked on Aug 1-2'"]
        PRO["Procedural Memory (OPT-IN)<br/>Agent behavior & workflow rules<br/>Requires agent_id<br/>e.g. 'Always double check safety specs'"]
    end
```

### Key Differences & Rules:
- **Semantic & Episodic Memories (Default)**: Created automatically by `client.add()`. You don't have to request them — Mem0's extraction engine generates both during normal conversation processing.
- **Procedural Memory (Opt-in)**: Created explicitly by passing `memory_type="procedural_memory"`.
  - **Requirement**: **Requires `agent_id`**. Procedural memory defines *"how the assistant should behave"*, which is agent-scoped rather than person-scoped.

---

## 🏗️ High-Level Architecture Diagram

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

## 🗺️ The 5-Phase Demo Roadmap

```mermaid
flowchart LR
    P1["Phase 1<br/><b>Core Features</b><br/>Store, Search, Scope,<br/>Update, Delete, Metadata"] --> P2["Phase 2<br/><b>RAG & Scoping</b><br/>answer_with_memory,<br/>user/agent/run layers"]
    P2 --> P3["Phase 3<br/><b>Readpath & Scores</b><br/>Relevance scores,<br/>Thresholding & Top-K"]
    P3 --> P4["Phase 4<br/><b>Temporal Decay</b><br/>Time-weighted recency,<br/>Memory staleness"]
    P4 --> P5["Phase 5<br/><b>Auto-Operations</b><br/>ADD, UPDATE, DELETE,<br/>NOOP Contradictions"]
```

---

## 🚀 5-Phase Walkthrough Details

### 1️⃣ Phase 1 — Core Features & Lifecycle ([mem0_friends_phase1_features.ipynb](file:///Users/sangeetha/SV-Summer2026/agents/lab/mem0_lite/docs/notebooks/mem0_friends_phase1_features.ipynb))
- **Automatic Fact Extraction**: Pass raw chat sentences $\rightarrow$ Mem0 extracts clean facts automatically.
- **Strict User Isolation**: `user_id` ensures Maya's, Jordan's, and Sam's memories never cross paths.
- **Explicit CRUD**: `update()` and `delete()` memories in-place using `memory_id`.
- **Metadata Tagging**: Filter memories by custom key-value pairs (e.g. `topic: "cooking"`).

### 2️⃣ Phase 2 — RAG Generation & Multi-Entity Scoping ([mem0_friends_phase2_generation_conflict_scoping.ipynb](file:///Users/sangeetha/SV-Summer2026/agents/lab/mem0_lite/docs/notebooks/mem0_friends_phase2_generation_conflict_scoping.ipynb))
- **`answer_with_memory()`**: Combine vector retrieval with LLM completion to answer questions naturally.
- **Multi-Entity Layers**: Segregate preferences into `user_id` (personal facts), `agent_id` (system compliance/safety rules), and `run_id` (transient session history).

### 3️⃣ Phase 3 — The Read Path & Score Tuning ([mem0_friends_phase3_readpath.ipynb](file:///Users/sangeetha/SV-Summer2026/agents/lab/mem0_lite/docs/notebooks/mem0_friends_phase3_readpath.ipynb))
- **Inspect Scores**: View composite similarity scores ($0.05$ to $0.35$ scale).
- **Thresholding & Top-K**: Use a score cutoff (e.g. `0.25`) + `top_k` limit to keep prompt context clean.

### 4️⃣ Phase 4 — Temporal Decay & Recency ([mem0_friends_phase4_decay.ipynb](file:///Users/sangeetha/SV-Summer2026/agents/lab/mem0_lite/docs/notebooks/mem0_friends_phase4_decay.ipynb))
- **Time-Weighted Recency**: Enable `decay=True`. Frequently accessed/recent memories get up to a `1.5x` score boost, while idle memories dampen toward `0.3x`.

### 5️⃣ Phase 5 — Automatic Operations ([mem0_friends_phase5_add_update_delete_noop.ipynb](file:///Users/sangeetha/SV-Summer2026/agents/lab/mem0_lite/docs/notebooks/mem0_friends_phase5_add_update_delete_noop.ipynb))
- **Auto-Reconciliation**: Watch Mem0 classify incoming statements into `ADD`, `UPDATE`, `DELETE`, or `NOOP` when facts change or repeat.

---

## 💡 Quick API Cheat-Sheet

```python
# 1. Store Semantic/Episodic Memory (Default - user_id is top-level)
client.add("I'm learning pottery on Tuesdays.", user_id="maya")

# 2. Store Procedural Memory (Opt-in - requires agent_id)
client.add(
    "Always double-check safety specifications.",
    agent_id="maintenance_agent",
    memory_type="procedural_memory",
)

# 3. Get All Memories (filters dict)
client.get_all(filters={"user_id": "maya"})

# 4. Search Memories (filters dict)
client.search("What are her hobbies?", filters={"user_id": "maya"})

# 5. RAG Generation (Retrieve -> Inject -> Complete)
memories = client.search(question, filters={"user_id": user_id})
answer = complete(f"Context:\n{memories}\n\nQuestion: {question}")
```
