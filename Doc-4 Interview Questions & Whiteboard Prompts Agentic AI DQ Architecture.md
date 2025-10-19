# **Doc-4 Addendum: Likely Interview Questions & Whiteboard Prompts Based on Agentic AI DQ Architecture**

---

## **1. Problem Framing**

### **Q1. What is the key limitation of the current Ab Initio DQ gating process that justifies an agentic AI replacement?**

**Answer:**
The legacy Ab Initio–based DQ process operates as a **static gate**: SQL/Python scripts run *pre-transformation* and halt ingestion on failure.
This model is:

* **Siloed** (no cross-dataset intelligence),
* **Rigid** (rule change → redeploy job),
* **Non-learning** (same issues re-occur; no memory of past fixes),
* **Opaque** (hard to trace lineage or justify rule failure).

An **agentic AI layer** fixes these by adding:

* Autonomous detection with *reasoning*,
* Contextual rule discovery (via RAG + embeddings),
* Adaptive remediation and feedback learning,
* Governance alignment via Dataplex + Vertex AI metadata.

**In short:**
We move from *“rules that stop the pipeline”* → *“agents that diagnose, explain, and heal with traceable intelligence.”*

---

### **Q2. How does Agentic AI differ from a conventional rule-based data quality engine?**

| Aspect                | Conventional DQ Engine      | Agentic AI DQ System                    |
| --------------------- | --------------------------- | --------------------------------------- |
| Rule management       | Hard-coded SQL/Python rules | Dynamic, LLM-derived hybrid rules       |
| Adaptability          | Manual                      | Self-learning via feedback loops        |
| Context understanding | Schema-based only           | Semantic (embeddings, lineage)          |
| Remediation           | Manual scripts              | AI-assisted or auto-executed actions    |
| Governance            | Human gate                  | Policy-aware agents (Governance Agent)  |
| Intelligence source   | Predefined logic            | Vertex AI Gemini reasoning + RAG memory |

---

### **Q3. What is the one-line business value proposition of this transformation?**

**Answer:**
A **self-learning, policy-aware DQ system** that reduces manual intervention by 60-70%, increases rule reuse and explainability, and enables *predictive* DQ scoring across the data estate — turning DQ from a gatekeeper into an **autonomous reliability layer**.

---

## **2. Architecture Rationale**

### **Q4. Why Google Cloud (GCP) and why specifically ADK + Vertex AI + AgentSpace?**

**Answer:**

* **GCP ADK (Agent Development Kit):** simplifies multi-agent orchestration, shared memory, and event-driven workflows.
* **AgentSpace:** provides visual governance and human-in-the-loop feedback portals.
* **Vertex AI (Gemini):** adds reasoning and RAG capabilities with enterprise-grade safety.
* **Dataplex + Data Catalog:** unify metadata, rules, and lineage into one semantic layer.
* **BigQuery:** the canonical data + long-term memory store for incidents and rules.

Together, they create an **MCP-compatible, agent-centric mesh** where each agent has autonomy, context, and persistent state — something traditional workflow managers cannot achieve.

---

### **Q5. How do Dataplex and Data Catalog help with context and governance?**

**Answer:**

* **Dataplex** provides *authoritative metadata, data classification, and policy enforcement* for each dataset.
  The Detector Agent queries Dataplex APIs to fetch schema, PII tags, and DQ thresholds before running checks.
* **Data Catalog** serves as a **semantic anchor**: all incidents, rules, and feedback are tied to catalog entries via lineage.
* Together, they enable **traceability (“why did this rule fire?”)** and **policy inheritance** across domains.

---

### **Q6. Why do we use both Matching Engine and BigQuery Embeddings for RAG?**

**Answer:**

* **Matching Engine:** for production—handles millions of high-dimensional vectors with fast nearest-neighbor search.
* **BigQuery embeddings:** used during experimentation and quick prototyping; cheaper and integrated with SQL.
* This dual setup supports **progressive maturity:** teams start with BigQuery RAG, then move to Vertex Matching Engine as LTM scales.

---

## **3. Agent Design**

### **Q7. How does the Orchestrator Agent differ from a conventional pipeline scheduler?**

**Answer:**
A scheduler merely triggers static DAGs.
The **Orchestrator Agent**:

* Parses event context (intent extraction via Gemini),
* Determines which agents to activate (Detector, Remediator, etc.),
* Maintains **state in Firestore**,
* Applies governance policies dynamically, and
* Records a full event lineage in BigQuery.

Thus, it acts as a **cognitive router**, not a cron job.

---

### **Q8. What is the role of the Detector vs. Reasoner vs. Remediator agents?**

| Agent                    | Function                                | Tools                      | Key Output          |
| ------------------------ | --------------------------------------- | -------------------------- | ------------------- |
| **Detector**             | Runs hybrid DQ checks (rules + AI)      | SQL, Dataplex, Vertex AI   | dq_incident         |
| **Reasoner (Explainer)** | Root cause + natural-language rationale | Vertex AI (Gemini)         | explanation JSON    |
| **Remediator**           | Generates SQL/Dataflow fixes            | Vertex AI + dry-run engine | fix + rollback plan |

Together, these form the **Detect → Explain → Heal** lifecycle.

---

### **Q9. How does the system prevent unsafe automatic remediation?**

**Answer:**
Safety is layered:

1. **Remediator Agent policy:** all changes go through *dry-run + diff + rollback*.
2. **Governance Agent:** checks PII & critical dataset rules.
3. **Steward Approval Workflow:** AgentSpace human-in-the-loop.
4. **Audit & DLP logs:** recorded in BigQuery for traceability.

No write occurs without passing these controls.

---

### **Q10. How do agents share knowledge and learn over time?**

**Answer:**

* Each approved remediation → **embedding** stored in Vertex Matching Engine.
* Feedback Agent captures steward corrections → updates prompt registry.
* Recurrent LLM fine-tuning or prompt chaining evolves rule precision.

This forms a **closed learning loop**, turning human feedback into model memory.

---

## **4. Data & State Management**

### **Q11. What are the major state stores and their responsibilities?**

| Store                      | Responsibility                                  |
| -------------------------- | ----------------------------------------------- |
| **Firestore**              | Incident state machine (open → review → closed) |
| **BigQuery**               | Long-term metrics, lineage, and audit           |
| **Vertex Matching Engine** | Embeddings (contextual memory)                  |
| **Dataplex / Catalog**     | Governance + metadata policy                    |
| **Cloud Storage**          | Raw evidence, DQ reports                        |

---

### **Q12. How is lineage maintained between datasets, rules, and incidents?**

**Answer:**
Each **dq_incident** is tagged with:

* `dataset_id`, `rule_id`, `incident_id`, and `parent_job_id`.
  The Reasoner Agent enriches this with upstream/downstream impact metadata fetched from **Dataplex lineage APIs**.

This enables **causal analytics**: “Which upstream feed caused repeated missing values in the billing pipeline?”

---

## **5. RAG, Prompting, and Explainability**

### **Q13. What exactly goes into the RAG store?**

**Answer:**

* Historical incidents and their resolutions,
* Business rule documentation,
* Steward feedback,
* Schema + data dictionary context,
* Prior explanations.

The **Reasoner** pulls nearest examples from Matching Engine and fuses them into the Vertex AI prompt for contextual understanding.

---

### **Q14. How is prompt engineering governed and versioned?**

**Answer:**

* Prompts are treated as **configuration artifacts**, not code.
* Stored in a **Prompt Registry** in Cloud Storage with semantic versioning.
* Feedback Agent updates metadata (e.g., prompt success rate, steward satisfaction).
* Integrated with CI/CD (Cloud Build → Artifact Registry → rollout).

---

### **Q15. How do you ensure explainability for AI-generated remediations?**

**Answer:**

* Every fix proposal includes:

  * A **diff summary** (rows affected, columns updated),
  * **Confidence score** from the model,
  * **References** to source incidents and documents.
* Vertex AI Gemini explanation APIs can output **reasoning traces** for each decision.
* Steward UI (AgentSpace) visualizes this before approval.

---

## **6. Safety, Governance & Compliance**

### **Q16. How do you handle sensitive data (PII, financial info)?**

**Answer:**

* **Governance Agent** enforces classification rules.
* Cloud DLP API redacts sensitive fields before sending data to LLM.
* All vertex calls use **VPC-SC** (Service Perimeter) for data exfiltration prevention.
* Logging is structured; no raw data leaves secure projects.

---

### **Q17. What’s your audit strategy?**

**Answer:**
Every event (detect, explain, remediate, approve) emits an **audit entry** to BigQuery + Cloud Logging.
A **temporal graph** (Future-state) links agents, incidents, and datasets → supports forensics and trust reporting.

---

## **7. CI/CD, MLOps & Observability**

### **Q18. How are agents deployed and versioned?**

**Answer:**

* Source: Git + Cloud Build.
* Build: ADK agent container → Artifact Registry.
* Deploy: Cloud Run / Vertex AI Endpoints.
* Config: Firestore + Prompt Registry.
* Canary Rollouts: New agents run in “shadow mode” first, validated via policy.

---

### **Q19. How do you measure the effectiveness of the agentic layer?**

**Answer:**
Key KPIs:

1. **MTTD (Mean Time to Detect)** ↓
2. **MTTR (Mean Time to Resolve)** ↓
3. **Steward acceptance rate** ↑
4. **DQ incident recurrence rate** ↓
5. **Cost per incident (compute + manual hours)** ↓
6. **LLM call success / failure rate**

---

### **Q20. What observability stack is used?**

**Answer:**

* Cloud Monitoring + Logging for infra metrics,
* Vertex AI model telemetry for usage & drift,
* Custom BigQuery dashboards for rule acceptance and DQ trends,
* Optional integration with Looker Studio or Grafana for exec-level insights.

---

## **8. Future Evolution**

### **Q21. What’s next after this TO-BE state?**

**Answer:**
**Future-state (12–24 months):**

* **Self-healing pipelines** (low-risk remediations auto-executed),
* **Predictive DQ scoring** via historical embeddings,
* **Cross-domain semantic rule sharing** (transfer learning),
* **Temporal graph lineage** → causal anomaly detection,
* **CrewAI portability:** same agent mesh deployable to AWS or Azure via MCP bridges.

---

### **Q22. How could this generalize to other Agentic AI reference patterns?**

**Answer:**
This DQ pattern exemplifies a **reference template** for any **Agentic AI control layer**:

1. **Perception (Detector)**
2. **Reasoning (Explainer)**
3. **Action (Remediator)**
4. **Feedback (Learning)**
5. **Governance (Safety net)**

Swap the domain:

* DQ → IT Ops, FinCrime, Supply Chain, Claims Adjudication —
  and the same **agent loop + memory + policy control** architecture applies.

---

## **9. Whiteboard Prompts & Expected Talking Flow**

**Prompt 1:** *“Whiteboard the full data flow — ingest → detect → explain → remediate → learn.”*
➡ Emphasize Pub/Sub → Orchestrator → Detector → Reasoner → Remediator → Steward → Feedback → LTM.

**Prompt 2:** *“Draw where RAG fits and how it enhances reasoning.”*
➡ Place RAG store (Matching Engine) beside Vertex AI, feeding context for reasoning agents.

**Prompt 3:** *“Show how you’d ensure governance in auto-remediation.”*
➡ Chain Governance Agent → DLP → Steward approval → Audit → Firestore state close.

**Prompt 4:** *“How would you containerize and roll out agents?”*
➡ Cloud Build → Artifact Registry → Cloud Run → version tags → Firestore config control.

**Prompt 5:** *“If a steward rejects a remediation, how is that used later?”*
➡ Feedback Agent writes to Prompt Registry → retrains / tunes Gemini prompts → new generation becomes more accurate.

---

✅ **Summary:**
This Doc-4 Addendum arms you for any **technical deep-dive, solution architect, or platform AI interview** — covering *why*, *how*, and *what-if* layers around your **Agentic AI Data Quality system on GCP**.
It also generalizes naturally into an **enterprise reference blueprint** for future Agentic AI transformations.

---

