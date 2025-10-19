# Doc-3 — **Agent Lifecycle, State & Memory Management**

**Enterprise GenAI Data Quality (DQ) on GCP**
*(GCP ADK-centric with CrewAI addendum — interview-ready, implementation-grade)*

---

## Contents (quick jump)

1. Executive summary
2. AS-IS → TO-BE → Future-state evolution table
3. High-level architecture (narrative + component map)
4. Agents: taxonomy, responsibilities, inputs/outputs, tools
5. Agent lifecycle & orchestration patterns (event-driven, goal-driven)
6. State & memory design — STM vs LTM, storage, schemas, lifecycles
7. RAG and embedding patterns for DQ context
8. Prompt engineering patterns, templates, examples (production-grade)
9. Tool integrations (MCP tools, Cloud Functions, BigQuery, Dataplex) — concrete examples & code snippets
10. Safety, governance, DLP, auditability, approvals, policy engine
11. Observability, metrics, monitoring, SLOs, cost controls
12. CI/CD, IaC, testing, canary rollout, prompt/versioning lifecycle
13. Common failure modes, mitigations, operational runbook
14. Interview Addendum — focused Q&A for senior GenAI architects
15. CrewAI Addendum — mapping, differences, when to use it
16. Appendices: sample JSONs, SQL, field schemas, prompt registry entry

---

## 1. Executive summary

This document gives an implementation-grade blueprint for replacing a legacy, static DQ gate (pre-transformation Ab Initio BDM checks) with an **agentic AI layer** built on Google Cloud. The system uses **GCP ADK + AgentSpace** for agents, **Vertex AI (Gemini)** for reasoning, **Dataplex / Data Catalog** for authoritative metadata + lineage, **BigQuery** as canonical store, and **Cloud Functions / Dataflow** as actionable MCP tool mechanisms. It covers agent lifecycle, state & memory management, RAG and embeddings, prompt engineering, integration patterns, safety/governance, monitoring, CI/CD practices, and the CrewAI portability addendum.

---

## 2. AS-IS → TO-BE → Future-state evolution (enterprise view)

| Aspect              | AS-IS (Today)                                                             | TO-BE (This Project)                                                                                                  | Future-state (12–24mo)                                                                              |
| ------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| DQ checks           | Ab Initio SQL/python scripts run pre-transform; fail stops medallion load | Agentic DQ layer: Detector Agent runs hybrid rule+LLM checks; Remediator suggests or executes fixes subject to policy | Self-healing pipelines: agents auto-resolve low-risk issues; predictive DQ scores prevent ingestion |
| Governance          | Manual steward review; siloed rules                                       | Dataplex + AgentSpace + approval workflows; rules cataloged & versioned                                               | Auto-promotion of vetted AI-discovered rules to Dataplex policy library                             |
| Context / semantics | Not captured; ad-hoc docs                                                 | RAG store: embeddings (past incidents, docs) + Data Catalog context                                                   | Cross-domain transfer learning; semantic rule reuse across datasets                                 |
| State               | Logs + dashboards                                                         | Firestore (incident state), BigQuery (LTM, audit), Vertex Matching Engine (vectors)                                   | Temporal graph store for lineage + causal analytics                                                 |
| Remediation         | Manual scripts                                                            | MCP Tools via Cloud Functions; staged dry-run + rollback support                                                      | Full automated remediation with roll-forward + roll-back SLOs                                       |
| Monitoring          | Rule counts                                                               | LLM usage metrics, steward acceptance, incident MTTD/MTTR                                                             | Predictive SLO violation alerts and cost-aware scaling                                              |

---

## 3. High-level architecture (narrative + component map)

**Flow summary** (ingest → detect → reason → suggest/act → feedback):

1. **Ingest & Preprocessing**: Data enters GCS / Pub/Sub → Dataflow → BigQuery staging (Ab Initio BDM or existing ETL may still run).
2. **Baseline DQ**: Dataplex scheduled profiling + existing SQL checks produce DQ baseline metrics.
3. **Event Trigger**: If thresholds or scheduled windows trigger, event to `dq-ingest` Pub/Sub topic → Cloud Run / API Gateway → Orchestrator Agent (ADK).
4. **Detection**: Orchestrator calls Detector Agent (A2A). Detector runs deterministic checks (SQL) + LLM reasoning via Vertex AI (with RAG context from Matching Engine or BigQuery embeddings). Detector creates a `dq_incident` document in Firestore + entry in `dq_incidents` BigQuery table.
5. **Explanation**: Explainer/Reasoner Agent reads incident + RAG context, returns human-friendly explanation (JSON with root_causes, confidence).
6. **Remediation Proposal**: Remediator Agent generates remediation SQL or Dataflow template, runs **dry-run** in staging (BigQuery `dry_run`), stores diff, and posts into AgentSpace UI.
7. **Governance/Approval**: Steward reviews in AgentSpace. Approve → Orchestrator triggers MCP Tools (Cloud Functions or Dataflow) to execute remediation; log everything. Reject/Modify → Feedback Agent updates memory and prompt registry.
8. **Learn & Persist**: Approved remediation and steward feedback are turned into embeddings (LTM) and prompt tuning updates. Audit logs written to BigQuery.

**Key GCP pieces**: BigQuery, Dataplex, Data Catalog, Dataflow, Pub/Sub, Cloud Run, Cloud Functions, Firestore (state), Vertex Matching Engine (prod embeddings) / BigQuery embeddings (POC), Vertex AI (Gemini), AgentSpace & ADK (agents), Cloud Monitoring / Logging, Secret Manager, Cloud DLP.

---

## 4. Agents — taxonomy, responsibilities, API contracts

### 4.1 Orchestrator Agent

* **Purpose**: Entry point for DQ runs, routing, priority decisions, policy enforcement.
* **Inputs**: Ingestion event (dataset, job_id, timestamp) or steward request.
* **Outputs**: Dispatch tasks (Detector), aggregate results, drive approval flows.
* **Tools**: ADK orchestrator runtime, Firestore (state write), Pub/Sub, Vertex AI for intent parsing.
* **Contract** (example):

  ```json
  {
    "task_id":"orch-20251018-001",
    "dataset":"billing.prod_charges",
    "trigger":"ingest_event",
    "priority":"high",
    "requested_checks":["null_rate","duplicates","temporal"]
  }
  ```

### 4.2 Detector Agent (DQ Detector)

* **Purpose**: Execute checks (SQL templates + Dataplex profiles) and produce structured incidents.
* **Actions**:

  * Run deterministic SQL checks (partition-aware).
  * Run Dataplex profiling if configured.
  * Call Vertex AI for contextual detection (send canonicalized sample + schema).
* **Outputs**: `dq_incident` JSON stored in Firestore + BigQuery.
* **Sample SQL check**: null rate per partition.
* **Safety**: Read-only for data; writes to incident stores only.

### 4.3 Explainer / Reasoner Agent

* **Purpose**: Convert raw detection into high-value explanations & root-cause hypotheses.
* **Inputs**: `dq_incident` id, RAG context (catalog text, previous incidents, business rules).
* **Outputs**: Explanation JSON as:

  ```json
  {
    "incident_id":"",
    "summary":"",
    "root_causes":[{"cause":"","confidence":0.7}],
    "investigation_steps":[...],
    "references":[...]
  }
  ```

### 4.4 Rule-Generator Agent

* **Purpose**: Suggest formalizable rules (for Dataplex / BigQuery) given recurring incidents.
* **Work**: Batch job that analyzes repeated patterns in `dq_incidents` LTM, proposes SQL checks with estimated FP/FN.

### 4.5 Remediator Agent

* **Purpose**: Propose remediation (SQL/Dataflow/connector calls), run dry-run, provide rollback script.
* **Safety**: Must run dry-run, produce diff and rollback before any production write.
* **Execution options**: (a) generate artifact for steward approval, (b) for low-risk tasks auto-execute with policy.

### 4.6 Feedback Agent (HITL learning)

* **Purpose**: Capture steward decisions, map to LTM and Prompt Registry updates, supervise fine-tuning triggers.

### 4.7 Governance Agent

* **Purpose**: Enforce policy (PII, no auto-write for billing without 2 approvals), call Cloud DLP for redaction, manage audit logs.

---

## 5. Agent lifecycle & orchestration patterns

### 5.1 Two orchestration paradigms

* **Event-driven (common for DQ)**: Ingest event → Orchestrator → Detector → Explainer → Remediator → Steward. Good for real-time or delta-based checks.
* **Goal-driven (useful for periodic audits)**: Steward issues goal (“audit all billing tables this month”), Orchestrator decomposes into sub-tasks, parallel detectors run, Rule-Generator runs, then a synthesis agent compiles results.

### 5.2 A2A (Agent-to-Agent) interactions

* **Mechanism**: ADK messaging or Pub/Sub topics (e.g., `agent.tasks`, `agent.incidents`, `agent.approvals`).
* **Pattern**:

  1. Orchestrator publishes task to `agent.tasks`.
  2. Detector subscribes; runs checks; publishes `agent.incidents`.
  3. Explainer consumes `agent.incidents` and writes `agent.incident_explanations`.
  4. Remediator listens for explanations with `actionable=true`.
  5. Steward UI listens to `agent.incident_explanations` and posts approvals to `agent.approvals`.
  6. Governance agent listens and enforces policies.

**Idempotency**: Every task includes an `idempotency_key` to avoid double execution.

### 5.3 Multi-turn & stateful flows

* **Use case**: Steward asks follow-up question about same incident. Orchestrator retrieves STM (session context) from Firestore and re-hydrates prompt with previous messages and top-k RAG hits.

---

## 6. State & Memory design — STM vs LTM

### 6.1 Definitions

* **STM (Short-term memory)**: session-level, conversation history, active incident context. TTL: 1–7 days (configurable). Stored in **Firestore** (fast reads/writes).
* **LTM (Long-term memory)**: historical incidents, accepted rules, remediation artifacts, embedding vectors. Stored in **BigQuery** (structured) + **Vertex Matching Engine** (vectors) or BigQuery embeddings table.

### 6.2 Why split?

* STM reduces token cost and provides immediate conversational continuity.
* LTM supports retrieval for RAG (learned patterns over time), offline analytics, and rule generation.

### 6.3 Data models — suggested schemas

**`dq_incidents` (BigQuery)**:

```sql
CREATE TABLE dq.dq_incidents (
  incident_id STRING,
  dataset STRING,
  table STRING,
  column STRING,
  finding STRING,
  severity STRING,
  confidence FLOAT64,
  sample_rows ARRAY<STRUCT<row_json STRING>>,
  lineage ARRAY<STRUCT<process STRING, upstream STRING>>,
  created_at TIMESTAMP
)
```

**`agent_memory` (Firestore document)**:

```json
{
  "session_id": "sess-123",
  "incident_stack": ["dq-2025-10-18-1"],
  "last_prompt": "...",
  "last_response_summary":"...",
  "created_at":"2025-10-18T..."
}
```

**Embeddings table (BigQuery)**

```sql
CREATE TABLE dq.embeddings (
  embedding_id STRING,
  vector ARRAY<FLOAT64>,
  source_type STRING, -- 'incident','doc','rule'
  source_id STRING,
  created_at TIMESTAMP
)
```

### 6.4 Embedding lifecycle & refresh policy

* **On event**: Detector creates embedding for incident summary; store in Matching Engine.
* **Refresh**: Monthly for stable domains; weekly for high-change domains (billing).
* **Eviction / compaction**: Remove embeddings for deprecated datasets after retention period (e.g., 12 months) or compress older embeddings.

### 6.5 Retrieval strategy

1. Metadata filter (dataset, domain) — reduce search space.
2. Vector similarity (top-k).
3. Short summarization of retrieved items (concatenate up to ~400 tokens) — this canonicalized summary becomes the RAG context passed to Gemini.

---

## 7. RAG & embedding patterns for DQ context

### 7.1 What to index

* Past incidents (summary + sample rows)
* Dataplex profiling outputs (table summaries)
* Data Catalog descriptions (business glossary)
* Business rules documents (SharePoint / Confluence extracts)
* Approved remediation SQL templates & past remediation diffs

### 7.2 Canonicalization

* Convert retrieved raw items into a normalized snippet:

  ```
  [SOURCE:dataplex_profile] row_count=1.2m null_pct_plan_code=0.18 last_profile_run=2025-10-15
  [SOURCE:incident] id=dq-2025-10-12-23 summary="plan_code null spike 17%"
  ```
* This reduces LLM hallucination risk and token usage.

### 7.3 Using Vertex Matching Engine vs BigQuery embeddings

* **Prod: Vertex Matching Engine** — low-latency, scalable nearest neighbor.
* **POC: BigQuery Embeddings Table** — simpler, lower ops overhead; use approximate matching via cosine using SQL UDFs.

---

## 8. Prompt engineering — patterns, templates, examples

### 8.1 Principles for DQ prompts

* **Role clarity**: Begin with explicit role and output schema.
* **Contextualization**: Always include metadata + sample rows + top-k RAG snippet.
* **Structured output**: Enforce JSON with schema and confidence.
* **Calibration**: Ask model for confidence and reasons.
* **Conservatism**: If below threshold, ask for investigation steps rather than remediation code.

### 8.2 Explainer prompt (production template)

```
SYSTEM: You are a Data Quality Analyst for Telecom Billing systems.
INPUT:
- Schema: {schema}
- Latest Dataplex profile summary: {profile_summary}
- Sample rows (up to 20): {sample_rows}
- Past similar incidents (canonicalized): {rag_snippets}

TASK:
1) Provide a short summary (1 sentence)
2) List up to 3 probable root causes with confidence scores (0-1)
3) Suggest 3 investigation steps (SQL snippets or logs to check)
4) If confident (>0.8), propose a remediation approach (not direct production SQL) and provide a safe dry-run SQL template.

OUTPUT: JSON only with fields: summary, root_causes[], investigation_steps[], remediation_proposal{}, confidence
```

### 8.3 Remediator prompt (production template)

```
SYSTEM: You are a Data Remediation Assistant.
INPUT: As above + policy constraints (policy: no direct update to production for billing without 2 approvals).
TASK:
- Provide dry_run_sql (COUNT/SELECT preview)
- Provide remediation_sql (CREATE OR REPLACE staging table or MERGE INTO staging)
- Provide rollback_sql
- Estimate affected_rows (SQL snippet to count)
OUTPUT: JSON {dry_run_sql, remediation_sql, rollback_sql, estimated_affected_rows, confidence}
```

### 8.4 Example model output (explainer)

```json
{
  "summary": "plan_code null spike to 18% in partition ds=2025-10-17",
  "root_causes":[
    {"cause":"Connector schema change in CRM (plan_id field moved)", "confidence":0.72},
    {"cause":"Upstream join failure (master_plans missing entries)", "confidence":0.45}
  ],
  "investigation_steps":[
    {"step":"Run join between billing_staging and master_plans to identify missing keys", "sql":"SELECT COUNT(1) ..."},
    {"step":"Check Dataflow job logs ingest_billing_2025-10-17 for errors"}
  ],
  "remediation_proposal":{"approach":"left-join map plan_code from master_plans in staging; fallback 'UNKNOWN'","confidence":0.65}
}
```

---

## 9. Tool integrations — code snippets & patterns

### 9.1 BigQuery checks (Detector Agent)

Run partitioned null rate check:

```python
from google.cloud import bigquery
bq = bigquery.Client()
sql = f"""
SELECT partition_id,
  COUNTIF(plan_code IS NULL)/COUNT(1) as null_rate
FROM `{project}.{dataset}.__TABLES__`
WHERE _PARTITIONTIME = TIMESTAMP('{date}')
GROUP BY partition_id
"""
job = bq.query(sql)
results = job.result()
```

### 9.2 Vertex AI LLM call (Gemini)

```python
from google.cloud import aiplatform
client = aiplatform.gapic.PredictionServiceClient()
endpoint = client.endpoint_path(project, location, endpoint_id)
response = client.predict(endpoint=endpoint, instances=[{"content": prompt}])
llm_text = response.predictions[0]["content"]
```

### 9.3 Creating embedding & storing in Matching Engine (pseudo)

* Generate embedding via Vertex Embeddings API, then index into Matching Engine.

### 9.4 Firestore incident write (Orchestrator)

```python
from google.cloud import firestore
db = firestore.Client()
doc_ref = db.collection('dq_incidents').document(incident_id)
doc_ref.set({
  "incident_id": incident_id,
  "dataset": dataset,
  "table": table,
  "finding": finding,
  "status": "NEW",
  "created_at": firestore.SERVER_TIMESTAMP
})
```

### 9.5 Cloud Function to execute remediation (MCP)

Example wrapper that validates policy and runs BigQuery job:

```python
def execute_remediation(request):
    body = request.get_json()
    # validate policy
    if not is_allowed(body): return {"status":"forbidden"}, 403
    dry_run = run_bigquery(body['dry_run_sql'])
    if dry_run['rows']>0:
        job = run_bigquery(body['remediation_sql'])
        log_job(job)
        return {"status":"executed","job_id":job.job_id}
    return {"status":"no-op"}
```

---

## 10. Safety, governance & DLP

### 10.1 PII handling

* Use **Cloud DLP** to scan payloads before sending to LLM. Replace/transform sensitive tokens when necessary.
* Only pass minimal sample rows (token limited) and redacted values when possible.

### 10.2 Policy engine & approval gating

* Policy table in BigQuery indicates allowed auto-remediation thresholds.
* Example policy:

  ```json
  {"domain":"billing","auto_write":false,"approval_required":2,"confidence_threshold":0.95}
  ```

### 10.3 Audit trail

* For each LLM call store: prompt_id, canonical_context_id (embedding snapshot), model_id, tokens_used, steward_id (if any), result hash, timestamp. Store in BigQuery `dq_audit_llm_calls`.

---

## 11. Observability, metrics & SLOs

### 11.1 Core KPIs

* **Detection precision/recall** (sample-based labeling).
* **LLM usage**: calls/day, avg tokens/call, cost/day.
* **Steward acceptance rate**: % remediations accepted.
* **MTTD / MTTR**: mean time to detect and remediate.
* **Incident backlog size**.
* **False Positive rate**.

### 11.2 Dashboards & Alerts

* Cloud Monitoring dashboards for service latency, error rates.
* BigQuery-driven dashboards (Looker Studio) for DQ KPIs.
* Alerts for: sudden LLM cost spike, incident backlog > threshold, critical incident with severity=high not acknowledged in X minutes.

### 11.3 SLO examples

* Orchestrator latency < 5s for route decision (95th percentile).
* Detector end-to-end check complete within 2 min for partitioned checks.
* LLM calls per incident ≤ 3 (budget enforcement).

---

## 12. CI/CD, IaC, testing & deployment

### 12.1 IaC

* **Terraform** modules for: BigQuery datasets, Dataplex zones, Vertex AI endpoints, Matching Engine, Pub/Sub topics, Cloud Run services, Firestore indexes, IAM roles.

### 12.2 Code & Prompt Repo

* Monorepo with: agents (Python), tools, infra (Terraform), prompts (YAML), tests.
* **Prompt Registry**: Each prompt stored as a text file with metadata; PR-based review for prompt changes.

### 12.3 Testing

* Unit tests for SQL checks (sample datasets & fixtures).
* Integration tests simulating ingestion events and asserting incident creation.
* LLM response regression tests (golden outputs for key prompts).
* Dry-run pipeline tests for remediation.

### 12.4 Deploy flow

* Dev → staging → canary → prod for all services.
* Prompt changes: test in staging with shadow LLM calls; steward review; then promote.

---

## 13. Failure modes, mitigations, operational runbook

### 13.1 Common failure modes

* **LLM hallucination** → validate via SQL cross-check and require steward approval.
* **Embedding drift** → monitor retrieval hit-rate & reindex embeddings.
* **Network outages / Vertex AI rate limits** → fallback to deterministic checks; enqueue tasks.
* **State store inconsistency** → Firestore transaction retries & dead-letter topics for corrupted events.
* **Cost overrun** → alert and throttle LLM calls; daily budget caps.

### 13.2 Operational runbook (short)

* **Critical DQ incident**: Pager, Orchestrator escalates to on-call steward, mark incident `SEV=1`.
* **LLM degraded**: Switch to deterministic check path; notify ML ops.
* **Remediation failure**: Auto-rollback using stored rollback SQL, mark stewardship ticket.

---

## 14. Interview Addendum — Senior GenAI Architect Q&A (focused)

Below are high-value Qs you will likely face and succinct ways to answer (with depth cues).

### Q: Why ADK + AgentSpace vs building custom orchestration?

**Answer**: ADK provides enterprise-grade primitives (tool APIs, agent lifecycle, security integration) and lowers time-to-value. Using ADK ensures consistent A2A patterns and better integration with Vertex AI. Custom orchestration adds maintenance and security surface area.

**If pressed**: Explain agent orchestration alternatives (CrewAI, LangChain orchestration) and trade-offs (portability vs native integration).

### Q: How do you manage LLM hallucination risk for remediation SQL?

**Answer**: Multi-layer validation: (1) require JSON schema outputs, (2) run static SQL linter + explain plan, (3) run dry-run in staging with row-level diff, (4) require approval if beyond policy thresholds.

### Q: How is state consistency ensured across distributed agents?

**Answer**: Use Firestore with transactions for critical state updates plus Pub/Sub for eventual-consistency flows. Implement idempotency keys and sequence numbers; leverage Cloud Tasks for ordered retries.

### Q: How do you measure ROI of GenAI DQ?

**Answer**: Map to business KPIs: reduced billing disputes, improved model stability (AUC), analyst time saved, decreased incident backlog. Quantify remediation acceptance and error reduction.

### Q: What is your strategy for LLM prompt drift?

**Answer**: Prompt Registry with versioning, prompt A/B experiments, stewardship feedback loop, and scheduled prompt reviews. Maintain golden examples and regression tests.

### Q: How to scale Vertex Matching Engine cost-effectively?

**Answer**: Tiered strategy: hot indices in Matching Engine for most used datasets; cold storage in BigQuery embeddings for historical retrieval. Prune vectors and limit top-k.

(See Appendix for many more sample Qs with concise but defensible answers.)

---

## 15. CrewAI Addendum — mapping, differences, when to use it

### 15.1 Why include CrewAI

* Useful when: you require vendor-agnostic multi-agent choreography, richer agent graphs, or need a cross-cloud orchestration plane.
* CrewAI provides higher-level multi-agent planning features and built-in orchestration control.

### 15.2 Conceptual mapping (ADK ↔ CrewAI)

| Capability                 | GCP ADK/AgentSpace                            | CrewAI Equivalent                            |
| -------------------------- | --------------------------------------------- | -------------------------------------------- |
| Agent runtime              | ADK (Cloud Run / Cloud Functions)             | CrewAI runtime (agents as pods/services)     |
| Tool API integration       | ADK tool wrappers (BigQuery, Dataplex)        | CrewAI connectors (HTTP, SDK)                |
| A2A orchestration          | Pub/Sub + ADK mesh                            | CrewAI orchestrator (native)                 |
| Prompt & memory management | Prompt Registry + Firestore + Matching Engine | CrewAI memory store / vector DB + connectors |
| Governance                 | IAM, Dataplex policy                          | External policy engine integration           |

### 15.3 Example: porting Detector Agent to CrewAI

* Detector logic stays same: SQL checks + LLM inference.
* Replace ADK A2A calls with CrewAI tasks; use CrewAI’s built-in retry/backpressure controls.
* Use same Vertex AI endpoints; CrewAI calls Vertex via HTTP or SDK.

### 15.4 When to prefer CrewAI

* Multi-cloud deployments (one orchestrator across AWS/GCP).
* Complex inter-agent planning and scheduling needs that exceed ADK orchestration features.

### 15.5 Interview-ready talking points

* “We targeted ADK for native integration and minimal ops. If the customer asked for multi-cloud portability or advanced agent graphs we prototyped a CrewAI variant and documented equivalence with a migration playbook.”

---

## 16. Appendices (practical artifacts)

### Appendix A — Sample `dq_incident` JSON (full)

```json
{
  "incident_id":"dq-20251018-0001",
  "dataset":"billing",
  "table":"prod_charges",
  "column":"plan_code",
  "finding":"null_rate_spike",
  "value":0.18,
  "baseline":0.005,
  "severity":"high",
  "confidence":0.68,
  "sample_rows":[{"row_id":12345,"plan_id":9999,"plan_code":null}],
  "lineage":[{"process":"dataflow-ingest-billing","upstream":"crm_raw"}],
  "created_at":"2025-10-18T07:30:00Z",
  "state":"NEW",
  "orchestrator_task":"orch-20251018-001"
}
```

### Appendix B — Prompt Registry entry (YAML)

```yaml
prompt_id: DQ_EXPLAIN_V1
model: gemini-1.5-pro
task: explain_incident
avg_tokens: 825
examples:
  - inputs: {schema: "...", sample_rows: "..."}
    output: {summary: "...", root_causes:[...]}
owner: "dq-team"
last_updated: "2025-10-10"
```

### Appendix C — Example dry-run SQL snippet

```sql
SELECT COUNT(1) affected
FROM `project.billing.prod_charges_staging` pc
LEFT JOIN `project.master.master_plans` m
  ON pc.plan_id = m.plan_id
WHERE pc.plan_code IS NULL
```

### Appendix D — Minimal Terraform snippet (idea)

```hcl
resource "google_pubsub_topic" "dq_tasks" {
  name = "dq-tasks"
}
resource "google_bigquery_dataset" "dq" {
  dataset_id = "dq"
  location   = "US"
}
```

---

## Closing notes

This Doc-3 provides a complete, implementable, interview-grade explanation of how to build, operate, and defend an **agentic GenAI Data Quality platform** on GCP. It is intentionally prescriptive — with schemas, code snippets, prompts, and governance patterns — so you can speak confidently to both the **strategic architecture** and the **tactical runbook**.


## **1. End-to-End Architecture Recap**

The implemented GenAI DQ platform runs on **GCP**, combining:

* **Vertex AI / Gemini Pro** → reasoning and natural-language inference
* **Agentic AI framework (GCP ADK)** → orchestrated multi-agent coordination
* **Dataplex, BigQuery, Data Catalog, Pub/Sub, Cloud Functions** → metadata, profiling, and remediation pipelines
* **Cloud Run, Firestore, Cloud Storage** → persistence, APIs, prompt registry
* **AgentSpace UI** → unified human-AI interaction portal

### Architectural Principles:

1. **Autonomy + Alignment**: Each agent works independently but aligns with business objectives via Orchestrator’s governance policies.
2. **Event-Driven Modularity**: Pub/Sub decouples agents.
3. **Statefulness**: Firestore maintains conversation & remediation memory.
4. **Traceability**: Every agent action logged for audit & explainability.
5. **LLM-assisted reasoning**: Gemini infers hidden semantic and contextual rules.

---

## **2. Interview-Level View: How LLM Augments DQ**

Traditional DQ → deterministic, rule-driven.
GenAI DQ → inferential, semantic, predictive.

| Dimension             | Traditional              | GenAI Approach                                          |
| --------------------- | ------------------------ | ------------------------------------------------------- |
| **Rule Definition**   | Manually written, static | LLM discovers contextual and temporal rules dynamically |
| **Rule Application**  | Deterministic match      | Probabilistic inference (confidence scoring)            |
| **Issue Remediation** | Manual or script-based   | Autonomous or recommendation-based (via MCP Tools)      |
| **Learning**          | None                     | Continuous — feedback & memory updates                  |

### Example of Business-Context Inference

Prompt:

> “Given policy claim data, infer potential rule violations involving temporal or conditional dependencies.”

Gemini Response:

> “If claim_approval_date < claim_submission_date → likely workflow anomaly. Suggest rule: approval_date ≥ submission_date.”

---

## **3. Prompt Engineering & Rule Inference Design**

### **3.1. Prompt Taxonomy**

| Prompt Type             | Purpose            | Example                                                                 |
| ----------------------- | ------------------ | ----------------------------------------------------------------------- |
| **Descriptive Prompt**  | Data understanding | “Summarize schema and identify high-risk attributes for inconsistency.” |
| **Diagnostic Prompt**   | DQ detection       | “Find anomalies where claim status and payment date conflict.”          |
| **Prescriptive Prompt** | Remediation        | “Suggest remediation steps to fix invalid policy references.”           |
| **Learning Prompt**     | Meta-reflection    | “From past 10 remediation logs, identify repeating anomaly patterns.”   |

---

### **3.2. Prompt Template Design Pattern**

All prompts follow a **structured template** to maintain consistency and reduce token drift:

```text
[Role Definition]
You are a {domain} data quality expert specializing in {context} datasets.

[Context Input]
Dataset Schema: {schema}
Sample Data: {sample_data}

[Task Definition]
Identify issues related to {specific_aspect}, reasoning about contextual rules.

[Output Format]
Return structured JSON with fields: rule_inferred, confidence, rationale.
```

---

### **3.3. Prompt Evaluation Metrics**

| Metric                      | Description                              | Example Target |
| --------------------------- | ---------------------------------------- | -------------- |
| **Precision**               | % of true anomalies correctly identified | >85%           |
| **Recall**                  | % of total anomalies caught              | >80%           |
| **Business Relevance**      | Qualitative score from steward feedback  | >4/5           |
| **Prompt Token Efficiency** | Tokens / Useful Output Ratio             | <1.5           |

Prompts are versioned in **Prompt Registry (Cloud Storage)** with metadata:

```json
{
  "prompt_id": "DQ_DETECT_V3",
  "model": "gemini-1.5-pro",
  "task": "contextual-dq-detection",
  "avg_tokens": 950,
  "accuracy": 0.87,
  "last_modified": "2025-10-10"
}
```

---

## **4. LLM and Model Lifecycle**

### **4.1. Model Selection Rationale**

| Model                        | Purpose                                      | Notes                                                   |
| ---------------------------- | -------------------------------------------- | ------------------------------------------------------- |
| **Gemini 1.5 Pro**           | Reasoning + Contextual inference             | Core of rule inference and text-to-schema understanding |
| **Vertex AI Embeddings API** | Similarity search for semantic rule matching | Used by Feedback Agent                                  |
| **Vertex AI Tabular Model**  | Predictive DQ risk scoring                   | Future integration for proactive alerts                 |

---

### **4.2. Model Invocation Pattern**

* **Synchronous** for small data & reasoning prompts
* **Batch async via Cloud Functions** for large dataset scans

```python
from google.cloud import aiplatform
model = aiplatform.gapic.PredictionServiceClient()
prompt = "Identify semantic inconsistencies in claim data..."
response = model.predict(endpoint=vertex_endpoint, instances=[{"prompt": prompt}])
```

---

### **4.3. Caching & Cost Optimization**

* Responses cached in Firestore for identical prompts.
* Vertex AI budget guardrails limit token usage.
* Agents evaluate “LLM necessity” before invocation (lightweight heuristics first).

---

## **5. Agent Lifecycle & State/Mem Management**

### **Agent Lifecycle**

| Phase             | Function                              | Example                              |
| ----------------- | ------------------------------------- | ------------------------------------ |
| **Perception**    | Agent observes data/metadata via APIs | Detector Agent reads Dataplex schema |
| **Reasoning**     | Agent invokes LLM or internal logic   | Gemini infers hidden rule            |
| **Action**        | Agent triggers remediation via MCP    | Cloud Function executes fix          |
| **Reflection**    | Agent evaluates performance           | Feedback loop updates registry       |
| **Memory Update** | Persist key learnings                 | Firestore entry updated              |

---

### **Memory Management Example**

```python
memory_entry = {
  "agent_id": "DQ_DETECT_01",
  "dataset": "claims_2024",
  "rule_inferred": "approval_date >= submission_date",
  "confidence": 0.92,
  "feedback": "Accepted",
  "timestamp": "2025-10-18"
}
firestore.collection("agent_memory").add(memory_entry)
```

Long-term memory stores **rule evolution** across datasets for **transfer learning** within domain.

---

## **6. Integration Patterns (Interview-Critical)**

### **6.1. Data Ingestion → Profiling → DQ Pipeline**

* Ingestion via **Pub/Sub → Dataflow**
* Profiling via **Dataplex + BigQuery**
* DQ Check triggered as **event → Cloud Function → Orchestrator Agent**

### **6.2. A2A Communication**

Agents interact via GCP ADK’s **Agent Mesh** interface:

```python
orchestrator.send_task("dq_detector", payload)
```

### **6.3. LLM-in-Loop**

LLM used for:

* Schema inference
* Rule discovery
* Root cause reasoning
* Summarized stewardship reports

---

## **7. Error Handling & Observability**

* **Cloud Logging + Pub/Sub Dead Letter Queues** for failed events
* **Trace IDs** attached to each anomaly detection for lineage
* **Looker Studio Dashboard** shows per-agent latency, fix success, LLM utilization

---

## **8. Security, Privacy, and DLP**

* Sensitive fields masked before LLM prompt using **Cloud DLP**:

```python
deidentify_config = {"info_type_transformations": {"transformations": [{"primitive_transformation": {"replace_with_info_type_config": {}}}]}}
```

* Role-based access (IAM) restricts agent operations.
* **Audit logs** map every auto-fix or rule generation event to steward approval.

---

## **9. Example Interview-Style Q&A**

| Question                                                                                 | Expected Depth of Answer                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Q:** How does your GenAI DQ solution differ from rule-based DQ tools like Informatica? | “We introduce contextual, LLM-assisted reasoning to discover hidden data semantics, with agents orchestrated through GCP ADK. The LLM learns evolving rules instead of relying solely on hardcoded ones.” |
| **Q:** How do you ensure cost control when using Vertex AI LLMs?                         | “Through caching, selective invocation heuristics, and prompt token budgets enforced by orchestration policies.”                                                                                          |
| **Q:** How do agents maintain context between sessions?                                  | “State persisted in Firestore with dataset-level memory; Orchestrator retrieves it before assigning new tasks.”                                                                                           |
| **Q:** What happens when LLM gives inaccurate suggestions?                               | “Feedback Agent captures human correction; this retrains prompt weights in registry, improving precision over time.”                                                                                      |
| **Q:** How do you ensure security and compliance?                                        | “Cloud DLP for masking, IAM for role segregation, and audit logs for traceability — embedded within the agent workflow.”                                                                                  |

---

## **10. Lessons Learned (Architectural Insights)**

* **Prompt drift** can degrade inference → mitigated via template standardization.
* **LLM hallucination** managed via **validation filters** using deterministic SQL cross-checks.
* **Latency** improved using **parallelized LLM calls** via Cloud Functions.
* **Cross-agent synchronization** requires careful state serialization (Firestore + Pub/Sub).
* **Explainability** is vital — hence structured JSON responses over free text.

---

## **11. Future Enhancements**

* RLHF fine-tuning of Gemini prompts using steward feedback.
* Integration with **LangChain or CrewAI** for complex reasoning chains.
* Multi-modal expansion (schema + data + lineage graphs).
* Predictive DQ scoring via time-series learning.

---

## **12. Key Takeaways (Interview Sound Bites)**

* “We built a **self-evolving, LLM-guided DQ engine** on GCP.”
* “Agents reason, act, and learn — replacing static rule catalogs.”
* “Context persistence via Firestore enables **multi-turn AI governance**.”
* “LLMs now serve as **semantic auditors** for enterprise data.”
* “Feedback loops close the gap between detection and continuous improvement.”

---
Perfect — this is an excellent move.

Let’s now generate **Doc-4: Agentic AI Reference Pattern — Live Whiteboard Talk Script**, i.e. the narrative you’d deliver if a panel of senior GenAI architects asked you to *“walk us through your Agentic AI architecture for enterprise data quality modernization on GCP.”*

---

# **Doc-4: Agentic AI Reference Pattern – Live Whiteboard Talk Script**

---

### 🎤 **Opening Context – Setting the Stage**

> “Let me start by framing the business problem before I draw the architecture.”

Most large enterprises today — especially in regulated domains like **insurance, banking, and automotive** — run massive **data pipelines** built on **Ab Initio**, **Informatica**, or **GCP Dataflow**, orchestrated through **BDM (ETL)** or **Airflow**.
These pipelines include static **data quality (DQ)** rules — null checks, referential integrity, duplicates, threshold validation, schema drift, etc.

However, the problem is —

1. The rules are **hand-crafted** and **static**.
2. They miss **contextual errors** — e.g., data that looks correct syntactically but is **semantically wrong** in business context.
3. The DQ teams spend months adding new rules for every product or region.
4. And, the **DQ decisioning logic** is not intelligent enough to “learn” new anomalies.

That’s where **Agentic AI** comes in — where **autonomous GenAI agents** augment or replace these brittle scripts.
They don’t just *validate data* — they *reason* about it, *learn* from lineage and metadata, and *recommend actions*.

---

### 🧩 **Whiteboard Level 1 — Layers of the Solution**

*(As you draw, start from bottom up — data to intelligence to orchestration)*

#### **Layer 1 – Data Sources & Landing**

* Enterprise data arrives from **operational systems**, **APIs**, **streaming feeds**.
* Landed into **GCP Cloud Storage / BigQuery** under **Dataplex Zones (Raw, Curated, Governed)**.
* **Metadata** (schema, lineage, quality metrics) captured in **Dataplex Catalog** and **Data Governance APIs**.

> “So this is our base layer — the raw substrate that our agents will later analyze.”

---

#### **Layer 2 – Data Transformation & Existing DQ Flow (AS-IS)**

* Legacy **Ab Initio ETL** performs cleansing and transformations.
* Pre-check scripts run *DQ Rules* before BDM jobs execute.
* If thresholds fail → **the downstream medallion stage doesn’t load**.

> “This gives us deterministic rule-based quality, but not adaptive intelligence.”

---

#### **Layer 3 – Agentic AI Infusion (TO-BE)**

Here’s where **Google Agent Development Kit (ADK)** comes in.

We create a **multi-agent system**, deployed in **Vertex AI Agent Runtime**, connected via **Dataplex** and **BigQuery**.

##### Agents Overview:

1. **Data Profiler Agent** –
   Uses **BigQuery Data Profiling API** + **custom Python transformers** to generate statistical summaries.
   Calls **LLM (Tuned Gemini 1.5 Pro)** via ADK Tools to interpret outliers, drift, and hidden correlations.

2. **Rule Inference Agent** –
   Reads historical DQ logs, metadata, and schema changes to *infer new business rules*.
   For example: “If claim amount > premium for 3 months in a row → anomaly.”
   This is learned via **prompt-chaining** and **RAG** over a **Data Quality Knowledge Base** stored in **Vertex Vector Search**.

3. **DQ Orchestration Agent** –
   Integrates with **Dataform / Airflow** or **Ab Initio API**.
   Dynamically decides which transformation job to block or proceed with — using both rule-based thresholds and LLM-derived insights.
   Has **stateful memory** using **Firestore / Vertex Memory Store**, to persist prior decisions.

4. **Governance Agent** –
   Writes back results and explanations into **Dataplex DQ Metrics Dashboard** and **Looker Studio**.
   Also suggests which rules to retire or optimize.

---

#### **Layer 4 – Context & Intelligence**

* Agents share a **Message Passing Channel (MCP)** via **Pub/Sub**.
  Example: Profiler → Rule Inference Agent sends `profile_summary`, receives `inferred_rules`.
* A **central Orchestrator** (either ADK Coordinator Agent or CrewAI Supervisor) ensures loop completion and feedback.
* **Memory Store** ensures *state persistence* across daily runs.
* **Fine-tuning and prompt templates** stored in **Vertex Model Registry**.

---

#### **Layer 5 – Interfaces & Observability**

* **Looker Studio** dashboard for DQ KPIs, LLM confidence, and anomaly distribution.
* **Vertex Model Monitoring** tracks agent drift and response latency.
* **Ops Hooks** send alerts to **Slack / PagerDuty** when DQ violations cross critical levels.

---

### 🧠 **Whiteboard Level 2 — Agent Interactions (Walkthrough Scenario)**

> “Let’s say a new batch of policy data lands in the Curated Zone…”

1. **Profiler Agent** samples data → computes statistical profile.
2. Sends metadata to **Rule Inference Agent** → which detects an unseen pattern.
3. Rule Inference Agent queries **RAG index** built from historical logs → finds similar anomalies from 2023.
4. It proposes a **contextual rule**: *‘Policy with claim ratio > 95% in 3 months = potential fraud’*.
5. **DQ Orchestrator Agent** injects this into **Dataform validation stage**.
6. If breach detected → it blocks ETL execution and sends LLM-generated reasoning to Governance Agent.
7. Governance Agent logs DQ Metrics and creates a human-readable explanation.

> “So, instead of static thresholds, the system now learns new context-driven rules and explains its reasoning — this is the leap that Agentic AI brings.”

---

### ⚙️ **Implementation Snapshot**

| Function         | GCP Component                             | Notes                          |
| ---------------- | ----------------------------------------- | ------------------------------ |
| Agentic Runtime  | Vertex AI Agent Runtime / ADK             | Deploy, manage, monitor agents |
| Memory           | Firestore / Vertex Memory Store           | Persistent state between runs  |
| Communication    | Pub/Sub (MCP Channel)                     | Inter-agent messaging          |
| Storage          | Dataplex + BigQuery                       | Data Zones + Catalog           |
| Model Invocation | Vertex AI Model Endpoint (Gemini 1.5 Pro) | LLM reasoning                  |
| Observability    | Cloud Logging + Looker Studio             | Metrics and alerts             |

---

### 🚧 **Challenges Encountered**

* **Cost and Latency** — early inference was slow; we batched profiling jobs.
* **Rule Explosion** — too many inferred rules; we used priority scoring (support × confidence).
* **Explainability** — added structured output templates to make LLM responses audit-friendly.
* **Integration with Ab Initio** — had to use REST API wrappers since direct connector unavailable.

---

### 🌱 **Outcomes & Learnings**

* Reduced manual rule-writing by 60%.
* Discovered 18 contextual DQ issues missed by static rules.
* Improved SLA compliance (ETL failures ↓ 30%).
* Created reusable **GenAI Agent Framework** template for other data domains.

---

### 🧩 **General Reference Pattern Takeaway**

This pattern isn’t only for Data Quality — it generalizes to any enterprise domain where:

* Agents interpret structured + unstructured metadata,
* Communicate via a shared context layer,
* Learn from historical knowledge bases,
* Execute adaptive workflows with human-in-loop feedback.

We can extend the same design for:

* **Data Lineage Explanation Agents**
* **Schema Evolution Advisors**
* **Master Data Deduplication Agents**

---

### 🎯 **Closing Script**

> “So, to summarize:
> We evolved from a static, rule-based DQ framework to a contextual, self-learning, explainable Agentic AI system.
> It fuses GCP native services (Dataplex, BigQuery, Vertex AI) with the Agent Development Kit, orchestrated via MCP channels, and optionally CrewAI when richer multi-agent coordination is required.
> The architecture is not just about better data quality; it’s about intelligent data governance that learns and evolves.”


