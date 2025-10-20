# **Doc-5: MCP & A2A (Agent-to-Agent) Design Patterns**

---

## 🧩 **1. Objective and Scope**

### **Purpose**

To define the **Multi-Agent Control Plane (MCP)** architecture and **Agent-to-Agent (A2A)** coordination patterns that allow autonomous agents to:

* Collaborate on DQ lifecycle tasks (detect → explain → remediate → learn).
* Operate within governance constraints.
* Maintain observability, rollback, and fail-safe execution.

### **Why It Matters**

While Docs 1-4 described agent roles, prompts, and governance integration, this document explains **how agents communicate, synchronize, and escalate decisions**, ensuring the platform behaves as a **self-governing AI system** rather than disconnected components.

---

## ⚙️ **2. Multi-Agent Control Plane (MCP) Overview**

### **2.1 Definition**

MCP is the **coordination and execution backbone** that manages inter-agent messaging, state propagation, and execution sequencing.

### **2.2 Core Principles**

| Principle          | Description                                                                                      |
| ------------------ | ------------------------------------------------------------------------------------------------ |
| **Event-Driven**   | Agents act on Pub/Sub events instead of synchronous calls.                                       |
| **Contract-Based** | All agent interactions use structured JSON contracts with schema validation.                     |
| **Composable**     | New agents can register dynamically via MCP registry without code redeploy.                      |
| **Safe Execution** | Policies and dry-run checks embedded in the Governance Agent intercept every mutating operation. |
| **Traceable**      | Each A2A call generates immutable audit entries in Firestore + BigQuery.                         |

---

### **2.3 MCP Core Components**

| Component                 | Description                                            | GCP Services                           |
| ------------------------- | ------------------------------------------------------ | -------------------------------------- |
| **Event Bus**             | Routes triggers (e.g., new dq_incident, rule proposal) | Pub/Sub                                |
| **State Store**           | Holds task, agent, and workflow states                 | Firestore                              |
| **Agent Registry**        | Stores agent metadata (type, version, topic)           | Firestore / Cloud Run config           |
| **Workflow Orchestrator** | Sequencing logic, retries, SLA tracking                | Cloud Composer / Workflows             |
| **Policy Enforcer**       | Validates agent actions before commit                  | Governance Agent + Dataplex            |
| **Telemetry Pipeline**    | Captures metrics, tokens, logs                         | Cloud Logging + BigQuery               |
| **Feedback Loop**         | Closes incident lifecycle with learning                | Feedback Agent + Vertex AI fine-tuning |

---

## 🧠 **3. Agent-to-Agent (A2A) Design Patterns**

Each pattern is a repeatable micro-interaction schema.
Agents communicate **through events, not direct calls**, following “*speak via contracts, not code*”.

---

### **3.1 Pattern #1 — Orchestrator → Detector (Task Dispatch)**

**Trigger:** Dataset ingestion or steward request.
**Flow:**

1. Orchestrator publishes a `dq_task.create` event.
2. Detector subscribes and performs data quality checks.
3. Detector emits structured `dq_incident` to Firestore and Pub/Sub.

**Contract Example:**

```json
{
  "task_id": "orch-20251018-001",
  "dataset": "billing.prod_charges",
  "priority": "high",
  "requested_checks": ["null_rate", "duplicates"],
  "callback_topic": "dq_incident.results"
}
```

**Design Notes:**

* Idempotent execution (same dataset event cannot run twice).
* Task priority drives BigQuery slot allocation.

---

### **3.2 Pattern #2 — Detector → Explainer (Incident Interpretation)**

**Purpose:** Convert numeric anomalies into natural-language explanations.
**Flow:**

1. Detector publishes incident event with schema summary.
2. Explainer subscribes and retrieves incident metadata + lineage context.
3. Vertex AI (Gemini Pro) reasons over the data to infer likely root causes.

**Contract Example:**

```json
{
  "incident_id": "inc-20251018-14",
  "dataset": "billing.prod_charges",
  "symptom": "null_rate > 0.25 in partition 2025Q1",
  "context": ["schema drift", "new ingestion source"]
}
```

**Outputs:** `dq_explanation` JSON with confidence scores.

---

### **3.3 Pattern #3 — Explainer → Rule Generator**

**Purpose:** Translate recurring root causes into formalized DQ rules.

**Flow:**

* Explainer emits `pattern_detected` events when a rule candidate repeats.
* Rule Generator Agent transforms this into parameterized SQL + policy metadata.
* Writes proposal to Dataplex Tag Template (in “draft” state).

**Safety:** Governance Agent auto-blocks active deployment until steward approval.

---

### **3.4 Pattern #4 — Rule Generator → Governance Agent**

**Purpose:** Validate proposed rule compliance and enforce two-person approval.

**Steps:**

1. Rule proposal triggers governance validation.
2. DLP scan confirms no policy violation.
3. Approval event sent back with status = approved / rejected.

**Contract Example:**

```json
{
  "rule_id": "rule-dup-detect-002",
  "dataset": "billing.prod_charges",
  "approval_status": "pending",
  "requires_dlp_scan": true
}
```

---

### **3.5 Pattern #5 — Governance Agent → Remediator**

**Purpose:** Governed hand-off for corrective actions.

**Flow:**

1. Governance Agent approves remediation plan.
2. Publishes `remediation.approved` event.
3. Remediator runs dry-run, produces diff report.
4. On success, executes and logs rollback script.

**Rollback JSON Example:**

```json
{
  "job_id": "rem-20251018-07",
  "dry_run": true,
  "affected_rows": 2541,
  "rollback_script": "DELETE FROM staging_table_backup WHERE batch_id=20251018"
}
```

---

### **3.6 Pattern #6 — Feedback Agent Loop (HITL Learning)**

**Purpose:** Close the learning loop between human stewards and LLM reasoning.

**Flow:**

1. Steward reviews incident + remediation outcome in UI.
2. Feedback captured as structured `feedback_event`.
3. Vertex AI fine-tuning job retrains reasoning prompts.

**Sample Feedback Event:**

```json
{
  "incident_id": "inc-20251018-14",
  "decision": "approved",
  "comments": "root cause confirmed, rule generalized correctly",
  "agent_ids": ["explainer", "rule_generator"]
}
```

---

## 🧩 **4. Advanced A2A Design Patterns**

| Pattern                          | Description                                                             | Implementation                               |
| -------------------------------- | ----------------------------------------------------------------------- | -------------------------------------------- |
| **Supervisor Pattern**           | Orchestrator supervises a group of agents via policy checkpoints.       | Cloud Workflows + ADK Supervisor API         |
| **Chain of Thought Validation**  | Explainer → Governance double-check using Vertex AI reasoning trace.    | Vertex AI + Firestore trace store            |
| **Voting Pattern**               | Multiple agents evaluate same issue; Orchestrator aggregates consensus. | Pub/Sub topic fan-out + Firestore tally      |
| **Sub-Agent Delegation**         | Detector delegates specialized profiling sub-agents.                    | Cloud Run micro-containers registered in MCP |
| **Human-in-the-Loop Escalation** | Governance defers to steward for complex decisions.                     | Cloud Tasks + Vertex AI Prompt Registry      |

---

## 🧠 **5. MCP Execution Lifecycle**

**Lifecycle Steps:**

1. **Registration** – Each agent registers metadata (capabilities, topics).
2. **Discovery** – MCP registry publishes service manifest.
3. **Execution** – Orchestrator triggers DAG via Pub/Sub topics.
4. **Observation** – Logging + metrics collected for each agent invocation.
5. **Adaptation** – Feedback Agent updates prompts or rules.
6. **Audit** – Governance Agent validates trace completeness.

---

### **State Machine Diagram (Descriptive)**

```
┌──────────────┐
│   Trigger    │
└──────┬───────┘
       ↓
┌──────────────┐
│ Orchestrator │
└──────┬───────┘
       ↓
┌──────────────┐
│   Detector   │
└──────┬───────┘
       ↓
┌──────────────┐
│   Explainer  │
└──────┬───────┘
       ↓
┌──────────────┐
│ RuleGenerator│
└──────┬───────┘
       ↓
┌──────────────┐
│  Governance  │
└──────┬───────┘
       ↓
┌──────────────┐
│  Remediator  │
└──────┬───────┘
       ↓
┌──────────────┐
│  Feedback    │
└──────────────┘
```

---

## 🧩 **6. MCP Reliability & Error-Handling Patterns**

| Category                 | Mechanism                        | Description                                    |
| ------------------------ | -------------------------------- | ---------------------------------------------- |
| **Retry Policy**         | Cloud Pub/Sub dead-letter topics | Automatic re-execution after transient failure |
| **State Replay**         | Firestore state versioning       | Recover incomplete runs                        |
| **Compensation**         | Rollback scripts                 | Reverse unsafe writes                          |
| **Rate Limiting**        | Token buckets in Orchestrator    | Prevent LLM overuse                            |
| **Circuit Breaker**      | Cloud Run health probes          | Suspend agent groups on anomalies              |
| **Graceful Degradation** | Fallback to rule-based checks    | Maintain continuity under LLM outage           |

---

## 📡 **7. Security and Policy Alignment**

* IAM service accounts per agent.
* All A2A contracts signed with **HMAC tokens** (MCP internal trust).
* DLP enforcement before any dataset mutation.
* All events timestamped and logged with correlation ID.

---

## 📘 **8. MCP Observability**

| Metric                       | Source                        | Dashboard               |
| ---------------------------- | ----------------------------- | ----------------------- |
| Agent Latency                | Cloud Logging                 | Looker Studio           |
| Message Throughput           | Pub/Sub metrics               | GCP Monitoring          |
| LLM Token Cost               | Vertex AI usage logs          | Cost Dashboard          |
| Incident Resolution SLA      | Firestore incident collection | Data Steward Console    |
| Governance Policy Violations | Audit logs                    | Security Command Center |

---

## 🧠 **9. Interview Questions & Whiteboard Prompts**

| **Topic**                   | **Question**                                                         | **Detailed Answer Summary**                                                                 |
| --------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **A2A Flow**                | How does Orchestrator know which agents to trigger?                  | Uses MCP registry metadata; filters by event type and dataset tag.                          |
| **Resilience**              | What if Detector fails mid-pipeline?                                 | Firestore marks state as `error`; Orchestrator replays message after cooldown.              |
| **Governance Integration**  | How is Governance Agent injected in A2A flow?                        | Every mutating topic passes through `policy.enforcement` Pub/Sub interceptor.               |
| **MCP vs. Workflow Engine** | Why not use only Composer?                                           | MCP adds AI-specific intelligence—dynamic reasoning, adaptive routing—beyond static DAGs.   |
| **Observability**           | How can you trace one dq_incident across agents?                     | Use correlation ID across all logs; stored in BigQuery audit table.                         |
| **Prompt Governance**       | How is prompt versioning linked to MCP events?                       | Feedback Agent updates Prompt Registry keyed by incident type and agent role.               |
| **Whiteboard Prompt**       | Draw event-driven A2A flow and highlight where LLM reasoning occurs. | Show Detector → Explainer → Governance; LLM used for interpretation and policy enforcement. |

---

## 🧭 **10. Summary**

The **MCP** transforms the DQ agent ecosystem from point-to-point API calls into a **self-orchestrating, policy-aware, and explainable mesh**.
It ensures:

* **Autonomy with accountability**
* **Adaptivity through learning**
* **Compliance via governance hooks**

This design forms the **foundation of next-generation agentic architectures** — scalable, observable, and governed.

# **Doc-5 Addendum: Likely Interview Questions & Whiteboard Prompts — MCP & A2A Design Patterns**

---

### **1. Conceptual Grounding**

**Q1. What does MCP mean in the context of your GenAI Data Quality solution?**
**A:** MCP stands for **Multi-Agent Control Plane** — it’s the layer that coordinates interactions among agents (Detector, Remediator, Feedback, Governance). It standardizes communication, manages context propagation, handles safety/policy enforcement, and tracks execution state. Essentially, it’s what turns a collection of independent agents into a **coherent, governed system** rather than ad-hoc LLM calls.

---

**Q2. Why is an MCP required instead of letting agents directly call each other (A2A)?**
**A:** Direct A2A works for small PoCs but doesn’t scale. Without MCP, you lose **traceability, throttling, role isolation, and failure recovery**. MCP adds:

* A **shared state store** (Firestore + Pub/Sub topics)
* **Uniform schema contracts** between agents
* **Queue-based orchestration** for retries and concurrency
* Centralized **audit & DLP policies**

In short: A2A = local autonomy; MCP = enterprise governance.

---

**Q3. What are the core components of your MCP implementation?**
**A:**

1. **Orchestrator Layer** → built using GCP ADK Orchestrator + Cloud Run
2. **Task Bus** → Pub/Sub with schema-validated payloads
3. **State Store** → Firestore (short-term), BigQuery (long-term audit)
4. **Policy Filter** → Cloud Functions intercept layer invoking DLP / IAM check
5. **Metrics Hooks** → Cloud Monitoring + custom LLM token usage logs
6. **Agent Registry** → Firestore collection + Dataplex metadata for versioning

---

### **2. A2A Patterns**

**Q4. How do agents communicate with each other — synchronous or asynchronous?**
**A:** Mostly **asynchronous** via Pub/Sub topics. Each agent has a subscription to the MCP’s task bus with a **topic prefix** like `dq.detector.task` or `dq.remediator.task`.
For **real-time feedback loops** (e.g., rule suggestion → approval → apply), a **temporary direct A2A gRPC channel** is created using ADK’s session context.

---

**Q5. How is state and memory passed between agents?**
**A:**

* **Short-term state (STM):** Firestore documents — e.g., incident_id, dataset_id, last action.
* **Long-term memory (LTM):** BigQuery tables + Vertex Matching Engine embeddings (for RAG context).
* **RAG context propagation:** Orchestrator injects prior incident embeddings + relevant catalog text into the next agent’s prompt envelope.

---

**Q6. What GCP ADK classes or components were used to build these A2A links?**
**A:**

* `AgentOrchestrator` (manages context, policies)
* `TaskHandler` (per agent class)
* `A2AConnector` (for controlled direct handoffs)
* `MemoryStore` (abstraction over Firestore + Vertex Matching Engine)
* `PromptRegistry` (versioned templates and HITL prompts)

---

**Q7. How do you handle agent retries, idempotency, and error isolation?**
**A:**

* Each agent publishes structured `status` events (`pending`, `completed`, `error`) to Pub/Sub.
* The MCP assigns a **correlation ID** per workflow run.
* Failures are retried with exponential backoff (Cloud Tasks).
* Agents are **idempotent** — no double writes; state transitions recorded in Firestore.

---

### **3. Whiteboard-Style “Explain Live” Questions**

**Q8. Walk me through a complete A2A interaction — say Detector → Remediator → Feedback.**
**A:**

1. **Detector Agent** identifies a DQ issue → writes `dq_incident` to Firestore + publishes event to `dq.incident.detected`.
2. **MCP Policy Filter** checks dataset sensitivity (via Dataplex tags).
3. **Remediator Agent** picks event → drafts SQL remediation (dry-run) → posts diff to Firestore.
4. **Feedback Agent** listens to `dq.remediation.proposed`, captures steward approval via AgentSpace UI.
5. On approval → **Orchestrator** invokes Cloud Function (MCP Tool) to apply fix.
6. Results logged → embeddings updated in LTM.

You can draw it as a **Pub/Sub event chain** with Firestore as sidecar state.

---

**Q9. How do you maintain context between agents (memory consistency)?**
**A:**

* Each agent references the **same `task_id`** in Firestore.
* The Orchestrator merges agent responses into a composite object.
* Prompt context = last N incidents + steward notes from Feedback Agent (RAG context).
* Vertex Matching Engine indexes embeddings; retrieval ensures semantic continuity across A2A flows.

---

**Q10. What’s your approach for performance and concurrency control in multi-agent runs?**
**A:**

* Limit 1:1 agent concurrency via Pub/Sub push subscription quotas.
* Use **Cloud Run revisions** for horizontal scaling.
* Deploy async jobs with **Cloud Tasks** to throttle throughput.
* Token + cost metrics sent to Cloud Monitoring dashboards.

---

### **4. Safety, Governance, and CI/CD**

**Q11. How do you ensure one rogue agent doesn’t perform unauthorized actions?**
**A:**

* Every agent has a **service account with scoped IAM roles** (principle of least privilege).
* Governance Agent inspects action manifests before execution.
* Any `write` operation triggers a DLP scan via Cloud DLP API.
* Manual approval required for high-risk datasets (billing, PII).

---

**Q12. How do you test or simulate MCP interactions?**
**A:**

* Use **ADK’s Agent Emulator** for dry runs.
* Pub/Sub replay mode for event-driven tests.
* Mocked Vertex AI responses using sample embeddings.
* CI/CD pipeline in Cloud Build triggers automated regression scenarios.

---

**Q13. What are the main challenges you faced in building this MCP?**
**A:**

* Getting **prompt consistency** across asynchronous calls.
* Managing **latency** in Vertex AI reasoning when chained.
* Ensuring **memory coherency** (Firestore vs embeddings).
* Designing **fine-grained rollback policies** for partial remediations.

Each was mitigated by adding caching, async buffering, and rollback diffing.

---

### **5. “Tough-Round” Deep Dives**

**Q14. How would this MCP design change if using CrewAI instead of GCP ADK?**
**A:** CrewAI provides a similar concept with **CrewManager + Task** primitives. We’d replace ADK’s Orchestrator with **CrewManager**, Firestore with **CrewMemory**, and Pub/Sub with a queue (e.g., Redis).
But the **semantic pattern remains identical** — Orchestrator → Detector → Remediator → Feedback → Governance, all mediated via event bus and memory store.

---

**Q15. Can you evolve this pattern toward an “Autonomous DataOps” future state?**
**A:** Yes — next step is **predictive remediation**:

* Agents generate DQ risk scores before jobs run.
* LLM predicts likely anomalies using embeddings from prior incidents.
* Low-risk issues auto-healed; high-risk flagged for approval.
  This moves from **reactive data quality** → **proactive data reliability management**.

---

✅ **Quick Interview Summary (How to Conclude)**

> “Our MCP pattern turns rule-based DQ gates into an intelligent, self-learning control plane.
> It brings together ADK’s agent runtime, Pub/Sub orchestration, Vertex AI reasoning, and Dataplex governance.
> The design is modular, observable, and policy-aware — built for scale and explainability.”

---

Would you like me to now generate a **diagram** (MCP + A2A topology, showing event flow and memory) for this Doc-5 to accompany your whiteboard walkthrough?


