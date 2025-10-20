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


