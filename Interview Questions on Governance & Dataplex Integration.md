# **Addendum: Interview Questions on Governance & Dataplex Integration**

---

## 🧩 **1. Problem Framing and Context**

### **1.1 What governance gaps exist in legacy Ab Initio–based DQ setups?**

**Answer:**
Legacy DQ systems (e.g., Ab Initio) often have **hard-coded business rules** and limited **metadata context awareness**.
Key gaps include:

* Rules stored in scripts, not version-controlled metadata.
* Manual lineage tracking—no automated impact analysis.
* Lack of PII awareness or DLP integration.
* No federated governance—each data domain manages independently.
* High operational overhead for policy compliance audits.

**Agentic AI Resolution:**

* DQ governance policies shift from “scripts” to **knowledge graphs** and **metadata-driven enforcement**.
* Dataplex catalogs + Vertex AI + DLP = autonomous policy application.
* Agents self-validate governance alignment and trigger alerts for violations.

---

### **1.2 How does Dataplex strengthen governance in Agentic AI DQ systems?**

**Answer:**
Dataplex acts as a **metadata control fabric**:

* Provides **unified metadata registry** for tables, rules, lineage, and PII flags.
* Enables **policy-based orchestration** across zones (raw, curated, governed).
* Integrates with **DLP API** for automated scanning and tagging of sensitive columns.
* Supplies **lineage APIs** that feed into Orchestrator/Feedback Agents for impact traceability.
* Ensures DQ rules run **within governed data zones** only.

**Whiteboard Tip:**
Draw the three Dataplex zones (raw → curated → governed) and show Governance Agent validating access tokens and PII policies before Remediator acts.

---

### **1.3 How do agents map to governance functions in Dataplex?**

| **Agent**            | **Governance Function**                       | **Dataplex or GCP Integration**     |
| -------------------- | --------------------------------------------- | ----------------------------------- |
| Orchestrator Agent   | Enforces dataset-level policy routing         | Dataplex Metadata API, IAM          |
| Detector Agent       | Reads metadata to determine profiling level   | Dataplex Data Profiles, Catalog API |
| Explainer Agent      | Queries lineage for contextual interpretation | Dataplex Lineage API                |
| Rule Generator Agent | Writes back rule metadata                     | Dataplex Tag Templates              |
| Remediator Agent     | Requires governance approval for execution    | Cloud DLP + Policy Tags             |
| Feedback Agent       | Captures audit trail & steward feedback       | Firestore + Dataplex Metadata Store |
| Governance Agent     | Oversees DLP compliance & audit               | Cloud DLP, Cloud Audit Logs         |

---

## 🧭 **2. Architecture Rationale & Design**

### **2.1 Why centralize metadata in Dataplex vs. custom metadata DB?**

**Answer:**
Dataplex ensures:

* **Native lineage + DLP integration** (no custom ETL).
* **Policy tagging** at column and dataset granularity.
* **Scalable zone abstraction**—raw, curated, governed—mirrors AI agent states.
* **Integration hooks for Vertex AI and BigQuery**—agents can query policies in real time.

A custom DB would duplicate capabilities without leveraging GCP’s managed lineage or IAM controls.

---

### **2.2 How does the Governance Agent interact with Dataplex APIs?**

**Answer:**
It acts as a **policy enforcement broker**:

1. Subscribes to Dataplex change notifications (Pub/Sub).
2. Queries dataset tags for compliance attributes (e.g., `pii_flag=true`).
3. Intercepts any Remediator or Rule-Generator request.
4. Checks DLP + IAM policy before execution.
5. Logs all actions in Cloud Audit Logs and Firestore.

**Example Policy Check JSON:**

```json
{
  "dataset": "billing.prod_charges",
  "operation": "update_column",
  "columns": ["customer_name", "payment_token"],
  "policy_check": {
    "pii": true,
    "approval_required": true,
    "status": "pending"
  }
}
```

---

### **2.3 How is auditability maintained across agent actions?**

**Answer:**

* All agent invocations emit **structured events** (Pub/Sub).
* Governance Agent subscribes and writes **immutable audit logs**.
* Each record includes:

  * agent_id
  * action
  * dataset
  * timestamp
  * result (approved/denied)
  * lineage link (Dataplex object)
* Stored in **BigQuery Audit Dataset** → visualized via **Looker Studio** dashboards.

---

## 🧠 **3. Data Governance Scenarios**

### **3.1 Scenario: PII Column Discovered in a Non-Governed Zone**

**Question:**
What happens if Detector Agent finds customer_name in the raw zone?

**Answer:**

1. **Detector Agent** raises a dq_incident tagged `pii_violation`.
2. **Governance Agent** queries Dataplex tag template for compliance level.
3. **Remediator Agent** is paused (no write ops allowed).
4. **DLP API** triggers column redaction or masking.
5. Incident logged + steward notified via **Feedback Agent**.

**Outcome:**
Violation mitigated, incident traceable, no human-in-loop delay for basic redaction.

---

### **3.2 Scenario: New Rule Proposal Requires Steward Approval**

**Question:**
How do we ensure new data quality rules align with governance standards?

**Answer:**

1. **Rule Generator Agent** proposes a rule and writes draft metadata to Dataplex tag.
2. **Governance Agent** checks:

   * Dataset zone (`curated` or `governed`)
   * Steward group approval
   * DLP clearance
3. Once approved, rule is versioned and moved to **active_ruleset** in Firestore.
4. Version number references both **Dataplex Tag ID** and **BigQuery View**.

---

### **3.3 Scenario: Lineage-based Root Cause Analysis**

**Question:**
How can the Explainer Agent use Dataplex lineage to justify a DQ issue?

**Answer:**

1. Explainer queries **Dataplex Lineage API** using `incident.dataset_id`.
2. Builds a trace graph (source → transform → output).
3. Uses Vertex AI reasoning to infer probable cause (e.g., upstream schema drift).
4. Renders cause map in structured JSON with confidence scores.

---

## ⚙️ **4. Policy Implementation Patterns**

### **4.1 Dataplex Tag Template Example**

| Field           | Type    | Description                    |
| --------------- | ------- | ------------------------------ |
| pii_flag        | Boolean | True if contains personal data |
| data_owner      | String  | Responsible steward            |
| retention_days  | Integer | Data lifecycle duration        |
| dq_rule_version | String  | Active DQ rule metadata        |
| zone            | Enum    | raw / curated / governed       |

Agents read/write these via **Dataplex Metadata API**.

---

### **4.2 Governance Flow**

**Whiteboard Flow:**

```
[Ingest Trigger]
   ↓
[Detector Agent → dq_incident]
   ↓
[Governance Agent: Validate DLP + Policy Tag]
   ↓
[Remediator Agent: Execute if compliant]
   ↓
[Feedback Agent: Record outcome + steward comment]
   ↓
[Dataplex Metadata: Update lineage and tags]
```

---

## 🧩 **5. Whiteboard Prompts**

1. **Draw the Data Governance Flow in Agentic AI DQ**

   * Show agent-to-Dataplex interaction.
   * Include IAM and DLP checkpoints.

2. **Illustrate how Dataplex Zones map to AI Agent responsibilities**

   * Raw → Detector focus
   * Curated → Explainer/Rule Generator
   * Governed → Remediator/Feedback

3. **Sketch policy check lifecycle**

   * Ingestion event → Rule proposal → DLP enforcement → Audit.

4. **Show integration points**

   * Dataplex ↔ Firestore ↔ Pub/Sub ↔ Vertex AI.

---

## 💬 **6. Likely Interview Questions & Answers**

| **Question**                                                  | **Detailed Answer**                                                                                                                                                                                                    |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **How does the Governance Agent prevent policy drift?**       | By continuously syncing Dataplex tag templates and validating agent metadata actions through a policy registry. Drift is detected by comparing Firestore audit snapshots with Dataplex metadata timestamps.            |
| **How does Dataplex DLP tagging improve explainability?**     | It allows the Explainer Agent to reason over data classification metadata (e.g., “this column is PII, hence redacted values cause null spikes”) — enabling contextual explanations instead of isolated numeric alerts. |
| **What happens if Dataplex metadata becomes inconsistent?**   | Governance Agent runs a reconciliation job daily, cross-verifying Firestore and Dataplex entries. Discrepancies create “meta_incidents” for Data Steward review.                                                       |
| **Can Governance Agent trigger remediation autonomously?**    | Yes, for low-risk, policy-compliant datasets (non-PII). It runs dry-run validation scripts and only auto-executes on pre-approved templates.                                                                           |
| **How does IAM tie into this ecosystem?**                     | Each agent has its own service account. Dataplex enforces least-privilege IAM; Governance Agent uses impersonation to perform cross-zone validation securely.                                                          |
| **How does audit data feed MLOps retraining or fine-tuning?** | Feedback Agent stores governance decision patterns; these are later fed into prompt retraining (Vertex AI fine-tuning jobs) for adaptive compliance reasoning.                                                         |

---

## 📘 **7. Summary Takeaway**

Governance & Dataplex Integration converts a data quality pipeline into a **policy-aware intelligent control loop**.
Agents operate with embedded compliance, real-time DLP scanning, and lineage-backed explainability.
The result is **autonomous governance**—data quality improves while stewardship effort decreases.

---

