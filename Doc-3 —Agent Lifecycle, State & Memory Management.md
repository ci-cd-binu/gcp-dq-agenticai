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

---

