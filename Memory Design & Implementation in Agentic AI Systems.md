# 🧠 Doc-8: Memory Design & Implementation in Agentic AI Systems (GCP ADK vs AWS Agents)

---

## 1. Overview

In agentic AI architectures, **memory** determines how an agent maintains context over time — whether that’s across:

* a **single conversation** (short-term),
* a **session or workflow** (medium-term), or
* a **project or system-level knowledge base** (long-term).

Without structured memory, agents behave like *stateless bots*, unable to reason, learn, or evolve over iterative tasks.

Both **Google’s Agentic Development Kit (ADK)** and **AWS Agents for Bedrock** offer primitives to support memory, but they differ in abstraction, scope, and persistence strategies.

---

## 2. Types of Memory in Agentic Systems

| Memory Type     | Scope                      | Example Use                                  | Storage Mechanism                                               |
| --------------- | -------------------------- | -------------------------------------------- | --------------------------------------------------------------- |
| **Short-Term**  | Within one agent task      | Summarizing user prompt history              | In-memory vector store (e.g., FAISS, Redis)                     |
| **Medium-Term** | Across workflow or episode | Context of previous validation steps         | Firestore / DynamoDB with TTL                                   |
| **Long-Term**   | Cross-project, reusable    | Corporate policy knowledge, previous tickets | Dataplex (GCP), S3 + Kendra (AWS), Vector DBs (Pinecone/Chroma) |

---

## 3. Memory Design Pattern (Generic)

```
┌──────────────────────────┐
│      Memory Manager      │
├──────────────────────────┤
│ Retrieve context → short │
│ Summarize → medium       │
│ Persist knowledge → long │
│ Refresh embeddings       │
│ Expire stale data        │
└──────────────────────────┘
```

Each agent interacts with the **Memory Manager** via API wrappers:

* `retrieve_context()`
* `store_memory()`
* `summarize_and_persist()`

---

## 4. Google Cloud ADK Memory Implementation

### 🔹 4.1 Architecture

In GCP ADK (Agent Development Kit), memory is implemented through:

* **Context Manager API** (part of ADK)
* **Vertex AI Vector Search**
* **Datastore / Firestore / BigQuery**
* Optional: **Memorystore (Redis)** for temporal caching

### 🔹 4.2 Example Code Snippet

```python
from google.cloud import aiplatform
from google.cloud import firestore
from google.cloud.aiplatform.experimental.agent import Agent, MemoryStore

# Initialize memory components
db = firestore.Client()
vector_store = aiplatform.VectorSearchIndex("projects/xyz/locations/us-central1/indexes/my-memory")

class DQAgent(Agent):
    def __init__(self, name):
        super().__init__(name)
        self.memory = MemoryStore(vector_store=vector_store, doc_store=db)

    def retrieve_context(self, query):
        # semantic retrieval
        results = self.memory.vector_store.search(query)
        return [r['content'] for r in results]

    def store_memory(self, key, content, metadata={}):
        self.memory.doc_store.collection("short_term").document(key).set({
            "content": content,
            "metadata": metadata
        })
        self.memory.vector_store.upsert([{"id": key, "content": content}])

# Usage
agent = DQAgent("DataQualityEvaluator")
agent.store_memory("last_prompt", "Checked null ratio for customer_age column")
context = agent.retrieve_context("data quality")
```

### 🔹 4.3 How It Works

* **Short-term** memory → Firestore temporary document (with TTL).
* **Medium-term** memory → Vertex Vector Search Index.
* **Long-term** memory → Dataplex metadata catalog linked to versioned embeddings.

---

## 5. AWS Agent Code Memory Implementation

### 🔹 5.1 Architecture

AWS Agent SDK (Bedrock Agents or Agents for Amazon Bedrock) provides:

* **`memory_config`** for session context
* **Amazon Kendra** for semantic retrieval
* **Amazon DynamoDB** for structured session persistence
* **Amazon S3** for long-term archival

### 🔹 5.2 Example Code Snippet

```python
from boto3 import client
from bedrock_agent_sdk import BedrockAgent

agent = BedrockAgent(
    model_id="anthropic.claude-3-sonnet",
    memory_config={
        "short_term": {"type": "in_memory"},
        "medium_term": {"type": "dynamodb", "table": "AgentContext"},
        "long_term": {"type": "kendra", "index": "AgentKnowledgeBase"}
    }
)

def handle_query(user_input):
    context = agent.retrieve_memory(user_input)
    response = agent.invoke_with_memory(prompt=user_input, context=context)
    agent.store_memory(user_input, response)
    return response
```

### 🔹 5.3 How It Works

* **Short-term:** transient RAM or Redis.
* **Medium-term:** contextual history persisted in DynamoDB (with conversation_id).
* **Long-term:** semantically indexed corpus in Kendra or OpenSearch.

---

## 6. Unified Memory Abstraction Across Clouds

When you design **multi-cloud agent frameworks**, abstract memory behind a simple interface:

```python
class MemoryAdapter:
    def retrieve(self, query): pass
    def store(self, key, value): pass
    def summarize(self): pass
```

Then implement:

* `GCPMemoryAdapter`
* `AWSMemoryAdapter`
* `RedisCacheAdapter`

Each adapter can internally map to:

* **GCP:** Firestore + Vertex Vector Search
* **AWS:** DynamoDB + Kendra
* **Neutral:** Redis + ChromaDB (for local tests)

---

## 7. Key APIs and Libraries Used

| Cloud       | Library                                      | Purpose                         |
| ----------- | -------------------------------------------- | ------------------------------- |
| **GCP**     | `google-cloud-aiplatform.experimental.agent` | Agent + memory management       |
|             | `google-cloud-firestore`                     | Medium-term structured memory   |
|             | `google-cloud-dataplex`                      | Governance & long-term memory   |
|             | `google-cloud-redis`                         | Temporal cache                  |
| **AWS**     | `bedrock_agent_sdk`                          | Core agent orchestration        |
|             | `boto3.kendra`, `boto3.dynamodb`             | Memory & retrieval              |
|             | `boto3.s3`                                   | Archive storage                 |
| **Neutral** | `langchain.memory`, `chromadb`, `redis`      | Portable memory implementations |

---

## 8. Common Challenges & Solutions

| Challenge                                  | Root Cause                                         | Solution                                                                      |
| ------------------------------------------ | -------------------------------------------------- | ----------------------------------------------------------------------------- |
| **Context bloat**                          | Too many memory tokens inflating cost              | Summarize conversation after N turns using `text-bison` or `claude-summarize` |
| **Cold-start latency**                     | Fetching embeddings from Vector DB on each request | Pre-cache recent context in Redis                                             |
| **Knowledge drift**                        | Outdated facts in long-term store                  | Periodically retrain embeddings & refresh Kendra/Vector index                 |
| **Multi-agent conflicts**                  | Agents overwriting shared context                  | Introduce *AgentContextID* partition key (per agent or role)                  |
| **Memory explosion in Firestore/DynamoDB** | Lack of TTL                                        | Implement auto-expiry (Firestore TTL, DynamoDB TimeToLive attribute)          |

---

## 9. Example Hybrid Pattern (Your Project Reference)

In your **Agentic DQ Platform (GCP)**:

* **Short-term:** Vertex AI ADK agent memory
* **Medium-term:** Firestore (per workflow)
* **Long-term:** Dataplex + Chroma Vector store for historic rules, DQ issues, and resolutions
  Used in:
* *DQ Detector Agent* for “recent anomaly context”
* *Remediator Agent* for “historical pattern references”

---

## 10. Interview Readiness Notes

| Question                                               | Short Answer                                                                   |
| ------------------------------------------------------ | ------------------------------------------------------------------------------ |
| *How do you persist agent memory in GCP ADK?*          | Firestore for structured memory + Vertex AI Vector Search for semantic context |
| *How do you control token cost from memory injection?* | Summarization and context filtering (only top-K relevant chunks)               |
| *How would you replicate this pattern in AWS?*         | Replace Firestore with DynamoDB and Vertex Vector Search with Kendra           |
| *Why use a memory registry?*                           | To manage prompt-memory pairs and track historical agent reasoning traces      |
| *What’s the role of Redis in this architecture?*       | Acts as a fast-access layer for active workflows to reduce retrieval latency   |

---

## 11. Summary

| Concept               | GCP ADK                          | AWS Agent SDK                   |
| --------------------- | -------------------------------- | ------------------------------- |
| Memory Storage        | Firestore + Vertex Vector Search | DynamoDB + Kendra               |
| Cache                 | Memorystore (Redis)              | Elasticache (Redis)             |
| Long-Term Persistence | Dataplex + BigQuery              | S3 + OpenSearch                 |
| Management API        | `MemoryStore` (ADK)              | `memory_config` (Bedrock Agent) |
| Observability         | Cloud Logging + Pub/Sub metrics  | CloudWatch + X-Ray              |

---
Great call — digging deep into memory APIs for the Agent Development Kit (ADK) is exactly what a seasoned Gen-AI architect must be ready for. Below are **detailed interview-ready sections** on *ADK memory management*, plus **code/API snippets**, plus **whiteboard prompts** you can use to demonstrate mastery.

---

## 🔍 ADK Memory – Key APIs & Concepts

Here are the major ADK-memory related APIs, classes, services and how/when to use them:

| API / Class                                                           | Purpose                                                                                                            | Use-Case & When to Use                                                                                                        |
| --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| `SessionService` (e.g., `VertexAiSessionService`) ([Google Cloud][1]) | Manages sessions: a user/agent conversation thread. Stores events, state, history.                                 | Use when you want to persist context within a session (short-term memory) across turns.                                       |
| `Session.state` property ([Medium][2])                                | Holds temporary data for a specific session (scratchpad).                                                          | Use for multi-turn flows where the agent needs to keep tasks, partial answers, user preferences (medium-term within session). |
| `BaseMemoryService` interface ([Google GitHub][3])                    | Core interface for memory stores that support long-term knowledge recall: `add_session_to_memory`, `search_memory` | Use when you want cross-session recall (long-term memory) — e.g., refer to past incidents, rules, artifacts.                  |
| `InMemoryMemoryService` (ADK implementation) ([Google GitHub][3])     | A non-persistent, in-memory store for prototyping.                                                                 | Use in POC/Dev contexts where persistence is not required.                                                                    |
| `VertexAiMemoryBankService` (ADK implementation) ([Google Cloud][4])  | Persistent long-term memory store managed by Vertex AI — semantic retrieval, embedding recall.                     | Use in production when you need semantic search memory across sessions or agents.                                             |
| `Memory tool` (e.g., `PreloadMemoryTool`) ([Google Cloud][4])         | A tool you assign to an agent so that it automatically retrieves/injects memory into prompts.                      | Use when agent needs memory retrieval/injection without custom code — e.g., at each turn pull up prior context.               |

---

## 🧑‍💻 Code Snippets – How You’d Actually Implement This

Here are sample code snippets you can walk through in an interview:

### Example: Setting up Memory in ADK (Short-Term + Long-Term)

```python
from google import adk
from google.adk.sessions import VertexAiSessionService
from google.adk.memory import VertexAiMemoryBankService
from google.adk.tools.preload_memory_tool import PreloadMemoryTool

# Setup Session service
session_service = VertexAiSessionService(
    project="my-project",
    location="us-central1"
)

# Setup Memory service for long-term recall
memory_service = VertexAiMemoryBankService(
    project="my-project",
    location="us-central1",
    agent_engine_id="my-agentengine-id"
)

# Define agent with a memory tool
agent = adk.Agent(
    model="gemini-2.0-flash",
    name="dq_detector_agent",
    instruction="""You are a data quality detector agent...""",
    tools=[ PreloadMemoryTool(memory_service=memory_service) ]
)

# Wrap in ADK App
from vertexai.agent_engines import AdkApp
app = AdkApp(agent=agent, session_service=session_service)
```

### Example: Using Memory Service API

```python
# After a session or task completes, add to memory (long‐term)
memory_service.add_session_to_memory(
    user_id="user123",
    app_name="dq_app",
    session_id="session789"
)

# Later, search memory for context
results = memory_service.search_memory(
    user_id="user123",
    app_name="dq_app",
    query="billing invoice null spike",
    top_k=5
)
for r in results:
    print(r.content, r.metadata)
```

### Example: Using Session state (within session)

```python
# When processing user input in a turn
session = session_service.create_session(user_id="user123", app_name="dq_app")
# … agent runs …
# At some point store state:
session.state["last_detected_issue"] = {
    "table": "billing.prod_charges",
    "issue_type": "null_rate_spike",
    "confidence": 0.78
}
session_service.update_session(session)
```

---

## 🕰 Memory Usage Across Short/Medium/Long-Term

Here’s how to map memory types to your DQ project and which APIs apply:

* **Short-term**: Within a single orchestration run or user conversation.

  * Use `Session.state` (scratchpad).
  * Example: While steward asks follow-up questions, you remember earlier parts of conversation.
  * Tool: SessionService + state fields.

* **Medium-term**: Across a workflow run, repeated tasks, or across related sessions (but within same domain/timeframe).

  * Use Session events + optionally MemoryService but with recent TTL.
  * Example: Detector agent triggers a remediation, and then next turn the Remediator agent recalling last incident details.
  * Tool: MemoryService or dedicated medium-term store (e.g., Firestore).

* **Long-term**: Across many workflows, datasets, incidents, even across domains — a knowledge base.

  * Use `BaseMemoryService` / `VertexAiMemoryBankService`.
  * Example: Past DQ incidents, remediation templates, learned rules, embeddings of prior incidents for RAG.
  * Tool: MemoryService search and retrieval.

---

## 🚧 Challenges & Solutions

When I’ve built memory into enterprise agentic systems (and you referenced this in your DQ project), here are the typical issues and how to handle them:

| Challenge                      | Reason                                                                         | Solution                                                                                            |
| ------------------------------ | ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| Memory bloat / token explosion | Injecting too much memory into prompt causing latency/cost                     | Apply retrieval filtering (top-k), summarize older memory entries, limit memory length.             |
| Relevance drift                | Old memory becomes irrelevant or incorrect                                     | Implement TTL or periodic refresh, version memory entries, embed time metadata.                     |
| Consistency across agents      | Different agents retrieving inconsistent memory leading to conflicting outputs | Use shared memory service and schema contract, apply agent-specific retrieval scopes.               |
| Latency in retrieval           | Semantic search on large vector store slows down agent turnaround              | Use caching (Redis / memory store) for recent memory, pre-fetch relevant memory prior to agent run. |
| Security / privacy of memory   | Memory store accidentally contains PII or sensitive info                       | Apply redaction before memory ingestion (via DLP), use IAM controls on memory service.              |

---

## 🎯 Whiteboard Prompts (Interview Use)

Here are some whiteboard style questions you can ask or answer:

1. **“Draw the architecture of how you implemented memory in the DQ Agentic AI system on GCP: show short-term, medium-term, long-term layers, and how the Agent interfaces.”**

2. **“Walk through the API flow when a new DQ incident is detected, how memory is updated and future retrieval works.”**

3. **“How would you design aging or eviction policy for long-term memory in ADK so that old, irrelevant incidents don’t pollute the model?”**

4. **“What considerations would you make if you wanted to port this memory design to AWS agents? Map equivalent services.”**

---

## ✅ Summary for Interview

> “In ADK, memory is built into the agent stack via Sessions (for state) and MemoryService (for long-term recall). You typically use `Session.state` and `Session.events` for short-term and intra-session context, and `VertexAiMemoryBankService` (or a custom MemoryService) to store/retrieve long-term knowledge.
> In the DQ project we designed, the Detector, Explainer and Feedback agents all share a unified memory service: new incidents get added via `add_session_to_memory`; later agents retrieve related incidents via `search_memory` and feed them into prompts for richer context.
> The major decisions are around retrieval filtering (top-k), memory summarization, caching for performance, and governance for sensitive data.”

---
Excellent — let’s build a **“Memory Management in Agentic AI using Google ADK”** reference that’s practical for interviews and consistent with your GenAI DQ project.
Below is a **concise yet implementation-oriented summary** combining the key ideas from the official ADK `Session` and `MemoryService` APIs and their application in your **Agentic Data Quality (DQ)** program.

---

## 🧠 Memory Management in Agentic AI (Google ADK Reference + Project Mapping)

### 1. Core Concepts in Google ADK

| Concept                     | Scope                                           | Lifetime                | Purpose                                                                 | Example Usage in DQ Project                                        |
| --------------------------- | ----------------------------------------------- | ----------------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **Session**                 | Conversation or workflow                        | Until closed or expired | Represents a logical conversation or execution (a “run”)                | A single DQ run triggered by dataset ingest                        |
| **State (`session.state`)** | Within-session memory                           | Ephemeral               | Tracks short-term variables used by the agents during one session       | Tracks which partitions have been checked, which rules triggered   |
| **MemoryService**           | Cross-session, long-term                        | Persistent              | Knowledge store for historical context, embeddings, or learned policies | Stores past DQ incidents, approved remediations, steward feedback  |
| **MemoryDocument**          | Entry in long-term memory                       | Persistent              | Stores text, structured info, or embeddings                             | “Invoice date anomaly” incident record with root cause explanation |
| **EmbeddingStore**          | Vector-based retrieval                          | Persistent              | Enables RAG retrieval and contextual recall                             | Search similar anomalies before making a new decision              |
| **SessionService**          | API layer to manage sessions and memory linkage | Persistent              | Controls creation, resumption, archival, and memory attach/detach       | Maintains link between current DQ run and prior context            |

---

### 2. Key ADK APIs and When to Use Them

| API / Class                                           | Purpose                                                     | Typical Call Pattern                                           | DQ Example                                                              |
| ----------------------------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `SessionService.create_session()`                     | Start a logical workflow or user interaction                | When a new DQ run starts or user triggers analysis             | Orchestrator Agent creates a session for dataset “billing.prod_charges” |
| `session.state.set(key, value)`                       | Store transient data during workflow                        | When an agent must share quick context with others in same run | Detector Agent sets `state["null_rate_detected"]=True`                  |
| `session.state.get(key)`                              | Retrieve state                                              | Other agents query runtime variables                           | Explainer Agent reads `state["detected_rules"]` to generate summary     |
| `MemoryService.store(document)`                       | Persist knowledge (structured, unstructured, or embeddings) | After a run, store all incidents + outcomes                    | Feedback Agent saves approved remediations as MemoryDocuments           |
| `MemoryService.query(query_text, top_k)`              | Retrieve similar historical knowledge                       | Before generating new rule or reasoning                        | Detector Agent queries similar anomalies before deciding if new         |
| `MemoryService.delete(document_id)`                   | Cleanup memory entries                                      | During retention management                                    | Governance Agent purges aged or PII data per DLP rules                  |
| `EmbeddingService.embed(text)`                        | Convert context into embeddings                             | When enriching long-term memory for semantic search            | Convert steward notes → vector embedding for retrieval                  |
| `SessionService.attach_memory(session_id, memory_id)` | Link long-term context to a current run                     | To provide continuity                                          | Attach prior billing dataset context to today’s run                     |

---

### 3. How It Works in Your GenAI DQ Project

| Phase                   | Memory Type                    | Implementation                                   | Tools / APIs                                  |
| ----------------------- | ------------------------------ | ------------------------------------------------ | --------------------------------------------- |
| **During Detection**    | Short-term (Session State)     | Keeps current job_id, partition, triggered rules | `session.state`                               |
| **During Explanation**  | Medium-term (Session + Memory) | Loads prior incidents, similar data context      | `MemoryService.query()`                       |
| **During Remediation**  | Long-term (Persistent Memory)  | Adds validated remediations + steward decisions  | `MemoryService.store()`                       |
| **Learning & Feedback** | Long-term with Embeddings      | Stores embeddings of prompts and outcomes        | `EmbeddingService.embed()`, `Matching Engine` |
| **Governance**          | Policy-based Cleanup           | Enforce retention, privacy                       | `MemoryService.delete()` + DLP API            |

---

### 4. Typical Memory Design Pattern (Pseudocode)

```python
from google.adk import SessionService, MemoryService, EmbeddingService

# Start a session
session = SessionService.create_session(name="DQ_Run_Billing_2025_10_20")

# Short-term: store context
session.state.set("dataset", "billing.prod_charges")
session.state.set("trigger", "daily_ingest")

# Agent performs detection
incident_summary = detector_agent.run(session)

# Long-term: persist result as memory document
doc = {
    "title": "Billing Null Value Anomaly",
    "content": incident_summary,
    "tags": ["billing", "null_rate", "2025-10-20"]
}
MemoryService.store(doc)

# Enrich with embeddings for contextual search
embedding = EmbeddingService.embed(incident_summary)
MemoryService.store({"embedding": embedding, "metadata": doc})

# Later in next run
past_context = MemoryService.query("null value anomalies in billing dataset", top_k=3)
```

---

### 5. Challenges & Solutions Observed in Real Projects

| Challenge                                    | Cause                             | Solution                                                                                      |
| -------------------------------------------- | --------------------------------- | --------------------------------------------------------------------------------------------- |
| **State loss between orchestrator restarts** | Non-persistent session backing    | Use Firestore as session persistence backend instead of in-memory                             |
| **Latency in embedding retrieval**           | Vector DB lookup cost             | Cache hot embeddings in Redis or Matching Engine local memory                                 |
| **Unbounded memory growth**                  | No retention policy               | Scheduled clean-up via Governance Agent using `MemoryService.delete()`                        |
| **Prompt drift**                             | Prompts evolved without history   | Store prompts + responses as MemoryDocuments with timestamps                                  |
| **Semantic mismatch in RAG**                 | Embeddings trained inconsistently | Normalize text before embedding; use domain-specific embedding model (Gemini Text Embeddings) |

---

### 6. Cross-Cloud Comparison (ADK vs AWS)

| Function           | GCP ADK                                  | AWS Equivalent                      | Notes                                   |
| ------------------ | ---------------------------------------- | ----------------------------------- | --------------------------------------- |
| Session & State    | `SessionService`, `session.state`        | Bedrock Agent Runtime (Session API) | Similar concept, different naming       |
| Long-term Memory   | `MemoryService`                          | KnowledgeBase / DynamoDB            | GCP has built-in embeddings integration |
| Embeddings         | Vertex AI Embedding API                  | Bedrock Embeddings (Titan, Claude)  | Both support cosine similarity          |
| Vector Store       | Vertex Matching Engine / BigQuery Vector | OpenSearch / Pinecone               | GCP has tighter BigQuery integration    |
| Policy Enforcement | IAM + DLP APIs                           | IAM + Macie                         | DLP native in GCP                       |

---

### 7. Example Interview Pointers

* **When to use `session.state` vs `MemoryService`?**
  → Use `session.state` for transient, workflow-scoped context (per DQ run). Use `MemoryService` for durable, cross-run knowledge (historical incidents, feedback, embeddings).

* **How do you ensure prompt + memory synchronization?**
  → The Feedback Agent writes each successful prompt and result into the memory registry; embeddings allow future prompts to reuse context semantically.

* **What is your memory scaling pattern?**
  → Active memories (last 30 days) in Firestore; archived in BigQuery for analytics; vector embeddings stored in Matching Engine for retrieval.

---


