# **Doc-2: Deep Dive – Agentic AI Implementation for GenAI-Driven Data Quality on GCP**

---

## **1. Problem Context & Why Agentic AI**

Traditional data quality (DQ) systems — rule-based, reactive, and mostly static — are effective for **known and deterministic errors** (duplicates, nulls, invalid formats).
However, they fail in **contextual and semantic dimensions**:

* **Rule bias**: Hardcoded rules ignore business context.
* **Dynamic schema drift**: New data sources and evolving business logic require rule redesign.
* **Blind spots**: Semantic mismatches (e.g., “policy start date before approval date”) often escape detection.
* **No prescriptive intelligence**: Even when issues are detected, systems don’t suggest fixes.

Our **GenAI-infused DQ program** addresses these by introducing **agentic intelligence** — a network of autonomous yet collaborative AI agents that *observe, reason, and act* using GCP’s AI ecosystem.

---

## **2. Solution Overview**

The implemented solution uses a **multi-agent architecture** built on:

* **GCP Agentic Dev Kit (ADK)** and **AgentSpace** for agent orchestration.
* **Vertex AI / Gemini Pro** for reasoning, prompt-based inference, and business rule discovery.
* **Dataplex + Data Catalog + BigQuery + Dataflow** for metadata, profiling, and quality pipelines.
* **Pub/Sub + Cloud Functions** for asynchronous triggers and event pipelines.
* **Cloud Run + Firestore (State Store)** for persistent memory and interaction logs.
* **Prompt Registry (Cloud Storage + Metadata store)** for managing reusable prompts.
* **Governance Layer (DLP + Audit Logs)** for security and observability.

---

## **3. The Agentic Framework**

### **3.1. Agent Roles**

| Agent                           | Description                                                              | Core GCP Tools/Functions Used                                                            |
| ------------------------------- | ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| **Orchestrator Agent**          | Coordinates other agents, interprets user goals, manages task delegation | GCP ADK Orchestrator Class, Cloud Run, Vertex AI (Gemini Pro), Firestore                 |
| **DQ Detector Agent**           | Detects anomalies using hybrid rule + LLM reasoning                      | BigQuery Data Profiling APIs, Vertex AI Text Embedding API, Gemini Model Calls, Dataflow |
| **DQ Remediator Agent**         | Suggests or applies fixes via MCP tools and APIs                         | Cloud Functions, BigQuery DML via Python Client, Pub/Sub for alerts                      |
| **Governance Agent (optional)** | Ensures all actions comply with policy and DLP rules                     | Cloud DLP API, IAM Policy Binding, Cloud Audit Logs                                      |
| **Feedback Agent**              | Captures user feedback, updates rules and fine-tunes prompts             | Firestore, Prompt Registry, Pub/Sub                                                      |

---

## **4. How Agentic Flow Works**

### **Step 1 – User Intent Capture**

A Data Steward interacts via **AgentSpace UI**, specifying:

> “Run data quality scan on policy transactions for last quarter and identify anomalies beyond existing rules.”

This input is received by the **Orchestrator Agent**, which parses the intent via Gemini-powered semantic understanding.

### **Step 2 – Task Delegation**

* The Orchestrator determines sub-tasks:

  * Retrieve metadata from **Dataplex**
  * Fetch existing rule catalog
  * Trigger **DQ Detector Agent**
* It uses **A2A communication (Agent-to-Agent)** through ADK APIs to initiate the Detector.

### **Step 3 – Detection & Contextual Reasoning**

* **DQ Detector Agent** executes hybrid analysis:

  * **Statistical profiling** using BigQuery SQL.
  * **LLM contextual inference** using Gemini:

    ```python
    prompt = f"""
    You are a data quality analyst. Analyze the below schema and sample data,
    identify potential quality issues beyond explicit rules. Suggest semantic and business-context anomalies.
    Schema: {schema}
    Sample Data: {sample_records}
    """
    response = gemini_model.generate_content(prompt)
    ```
  * The model infers **contextual errors**, e.g., “Claim closure date before accident date”.

* Detector stores anomalies in **Firestore (State Store)** with metadata and confidence scores.

### **Step 4 – Prescriptive Remediation**

* **DQ Remediator Agent** is triggered automatically via **Pub/Sub**.
* It uses **MCP Tools** (GCP Cloud Functions) to execute:

  * Automated corrections for nulls, invalid formats.
  * Context-based recommendations for semantic issues (logged as suggestions).
* Example Function:

  ```python
  def correct_null_policy_id(event, context):
      bq_client.query("""
      UPDATE `policy_dataset.transactions`
      SET policy_id = 'UNKNOWN'
      WHERE policy_id IS NULL
      """)
  ```

### **Step 5 – Feedback Loop & Memory Update**

* **Feedback Agent** logs human-in-loop feedback (“Accept / Reject fix”) into Firestore.
* **Prompt Registry** stores:

  * Proven effective prompts.
  * Performance metrics (accuracy, F1 improvement).
* Over time, the system builds **contextual intelligence**, learning which DQ suggestions align best with business logic.

---

## **5. Key Design Components**

### **5.1. State Management**

* **Firestore** holds:

  * Agent interaction states.
  * Issue ID, dataset, anomaly summary, confidence score.
  * Timestamps and responsible agent.
* This persistent state enables **multi-turn collaboration** among agents even across sessions.

### **5.2. Memory Architecture**

* Each agent maintains:

  * **Short-term memory** → context for current task.
  * **Long-term memory** → historical anomalies, accepted rule patterns.
* Memory is retrieved by Orchestrator using:

  ```python
  context = memory_retriever.get_recent_context(dataset_id)
  ```

### **5.3. Prompt Registry**

* Implemented via **Cloud Storage bucket** + **Metadata table in BigQuery**.
* Each entry includes:

  | Field            | Description                 |
  | ---------------- | --------------------------- |
  | prompt_id        | Unique ID                   |
  | task_type        | e.g., "detect", "remediate" |
  | model            | Gemini-1.5-Pro              |
  | tokens           | Avg tokens used             |
  | accuracy_score   | Historical performance      |
  | feedback_summary | User feedback logs          |

---

## **6. Sample Prompts and Responses**

### **Prompt (Detection Agent)**

> “Review the following customer claims dataset. Identify inconsistencies between policy issue date and claim date. Flag contextual anomalies with reasoning.”

**Response (Gemini):**

> * 18% of records show claim date earlier than policy issue date.
> * 6% contain missing approval status though payment completed.
> * Possible semantic mismatch: “claim_type = hospitalization” without admission date.

### **Prompt (Remediation Agent)**

> “Suggest business-context-appropriate fix strategies for missing or inconsistent claim status fields.”

**Response:**

> * For `NULL` claim_status with valid payment_date, infer status as “Settled”.
> * For mismatched claim dates, flag to business steward for manual review.

---

## **7. Observations from POC → Production**

| Aspect                | POC Learning                                                       | Production Outcome                             |
| --------------------- | ------------------------------------------------------------------ | ---------------------------------------------- |
| **LLM context**       | Initially too generic; refined using domain-specific prompt tuning | Improved semantic detection accuracy by ~35%   |
| **A2A orchestration** | Occasional state loss                                              | Fixed using Firestore persistence              |
| **Latency**           | LLM calls added 8–10s overhead                                     | Optimized using caching & selective invocation |
| **User acceptance**   | 60% of auto-fixes accepted                                         | Increased to 82% post feedback-loop tuning     |

---

## **8. Governance & Observability**

* **DLP API** masks sensitive columns before prompt submission.
* **Cloud Audit Logs** track every agent interaction.
* **Looker Studio Dashboard** summarizes:

  * DQ anomalies by dataset.
  * Auto-fix success rates.
  * Agent performance metrics.

---

## **9. Benefits Realized**

| Benefit                   | Description                                      |
| ------------------------- | ------------------------------------------------ |
| Contextual rule discovery | LLM discovered 12 new hidden data relationships. |
| Continuous learning       | Feedback loop improved rule base dynamically.    |
| Reduced manual workload   | 60% automation in DQ rule creation.              |
| Predictive DQ insights    | Early drift detection for schema evolution.      |

---

## **10. Future Enhancements**

* **Agent fine-tuning** via Vertex AI Reinforcement Learning with Human Feedback (RLHF).
* **Integration with Data Governance Fabric** for cross-domain policy enforcement.
* **Conversational UI** for natural language data quality ops.
* **Predictive anomaly detection** via Vertex AI Tabular models integrated into Detector Agent.

---

## **11. Next Documents in the Series**

| Doc #     | Title                                             | Focus                                                       |
| --------- | ------------------------------------------------- | ----------------------------------------------------------- |
| **Doc-3** | *Prompt Engineering & Rule Inference in GenAI DQ* | Fine-tuning, prompt templates, Gemini prompt best practices |
| **Doc-4** | *MCP Tools and API Integration Guide*             | How MCP, Cloud Functions, and APIs enable agent actions     |
| **Doc-5** | *Governance, DLP, and Feedback Loops*             | Compliance, observability, and feedback learning            |
| **Doc-6** | *Scaling and Productionization Blueprint*         | Deployment pipelines, CI/CD, monitoring, cost optimization  |

---

## **12. Summary**

This GenAI-Agentic DQ system on GCP demonstrates how LLM-driven reasoning can **transcend traditional rule-based DQ**, unlocking **contextual, predictive, and prescriptive insights**.
By using **multi-agent collaboration**, **LLM prompt learning**, and **stateful orchestration**, the solution operationalizes AI-assisted data governance — paving the way for self-evolving data ecosystems.


# 📄 **Addendum to Doc-2: Interview Preparation Guide — Prompt Engineering & Contextual Rule Inference**

---

## **1. Interviewer Intent**

This section prepares you to answer questions around:

* How you integrated LLMs in real-world DQ pipelines
* Your prompt engineering design choices
* Trade-offs between accuracy, cost, explainability, and autonomy
* Real examples of what prompts did, what results you got, and how you iterated

---

## **2. Core Concept Questions**

### **Q1: What differentiates Prompt Engineering for DQ from general NLP prompting?**

**Answer:**

> In DQ, prompts are not about text summarization or Q&A — they are **data-contextual**.
> Each prompt must convey schema, data sample, domain ontology, and expected logic.
> Instead of “What is the sentiment?”, we ask “Given schema X and sample Y, infer if record violates domain consistency rules.”
> Hence, we design *structured prompts* with placeholders for schema + metadata + business glossary.

**Follow-up:**
*How do you ensure stability across dataset variations?*

> By using **prompt templates** registered in our Prompt Registry with version control and by abstracting schema tokens dynamically.

---

### **Q2: How do you ensure consistency and avoid hallucination in GenAI-based DQ checks?**

**Answer:**

> We anchor the model using contextual delimiters and format-bound output.
> Each prompt enforces **“Return JSON with confidence and rationale”**, so free-flow hallucinations are minimized.
> We also add a **post-validation SQL layer** — an LLM-inferred rule is only accepted if validated syntactically and semantically in BigQuery.

---

### **Q3: How did you evaluate which LLM works best?**

**Answer:**

> We ran A/B experiments using **Gemini 1.5-Pro**, **Gemini Flash**, and **PaLM2** across multiple data domains.
> We compared inference accuracy, token cost, and response latency.
> Gemini-Pro achieved best contextual precision (~87%) and lowest false positives when fine-tuned with rule-based priors.

**Follow-up:**
*What metrics did you use for comparison?*

> Precision, recall, steward validation score, token efficiency (tokens/output).
> All logged in Vertex AI Experiments for traceability.

---

### **Q4: What are some example prompts your team used in production?**

**Answer:**

> Example for semantic validation in claims data:

```
You are a data quality expert specializing in insurance claims.
Schema: [claim_id, claim_date, approval_date, payout_amount, policy_status]
Task: Identify contextual inconsistencies.
Output: JSON with rule_inferred, rationale, confidence.
```

**Model Response:**

```json
{
 "rule_inferred": "approval_date >= claim_date",
 "rationale": "Approval cannot precede submission",
 "confidence": 0.94
}
```

**Follow-up:**
*How do you store and reuse such prompt-response pairs?*

> In **Prompt Registry** on Cloud Storage — used as few-shot exemplars for future tasks.

---

### **Q5: How do you maintain interpretability of AI-generated rules?**

**Answer:**

> Every LLM-inferred rule is linked to a **rationale and lineage trace ID**, stored in Dataplex metadata.
> Stewards can inspect the logic, feedback into a “rule acceptance queue,” and the feedback updates the prompt weight in registry.
> This ensures transparency, accountability, and continuous improvement.

---

### **Q6: How is prompt optimization automated?**

**Answer:**

> Through an **Agent Feedback Loop**:
>
> * Detector Agent → executes inference
> * Feedback Agent → evaluates steward response
> * Orchestrator Agent → adjusts prompts (adds examples, reduces token complexity)
>   We used Gemini API’s “context window monitoring” and Vertex metadata for comparing token-efficiency across prompt versions.

---

## **3. Practical Scenario Questions**

### **Q7: Suppose your GenAI model infers a wrong DQ rule — what do you do?**

**Answer:**

> The workflow triggers a remediation path:
>
> * Steward flags “invalid rule” via AgentSpace UI
> * The rule’s feedback entry is logged
> * The Feedback Agent retrains the prompt context by excluding misleading examples
> * Validation threshold for similar prompts is temporarily raised (confidence ≥ 0.95)

---

### **Q8: How does contextual prompting enable cross-domain intelligence?**

**Answer:**

> The same base model can infer rules in multiple domains — billing, CRM, telemetry — because prompts supply contextual metadata (like “domain = network_telemetry”).
> This allows **transfer learning across datasets** without retraining the LLM — only prompt tuning.

---

### **Q9: How did you manage multi-turn conversation prompts in the DQ workflow?**

**Answer:**

> Memory is persisted in **Firestore**.
> When a steward asks a follow-up question (“Show me all anomalies for policy X”), the system recalls previous rule context and dataset lineage.
> This enables **context-continuity** across multi-turn sessions.

---

### **Q10: What was the most challenging prompt design problem you faced?**

**Answer:**

> Balancing specificity and generality — too narrow prompts miss anomalies, too broad cause hallucination.
> The fix was to dynamically generate prompts with metadata tokens (“@schema_field_name”) injected contextually from Dataplex metadata.

---

## **4. Cross-Functional Discussion Triggers**

Interviewers often pivot to see **architecture-wide understanding**. Be ready for these:

| Question                                                                      | Hint for Framing                                                                           |
| ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **How do your prompts interact with other agents (DQ Detector, Remediator)?** | Through Orchestrator Agent; prompt definitions stored centrally, reused by Detector Agent. |
| **What’s your fallback when LLM is unavailable or times out?**                | Default to rule-based SQL checks; cached prompts used for redundancy.                      |
| **How do you control cost of token usage?**                                   | Prompt truncation, batching, and caching of identical inputs.                              |
| **What were your scaling challenges?**                                        | Async invocation via Cloud Functions and Pub/Sub batching solved LLM rate-limit issues.    |
| **What type of governance was required?**                                     | Prompt versioning, steward approvals for production rules, audit logs for compliance.      |

---

## **5. Advanced Deep-Dive Interview Questions**

These are high-bar questions aimed at **senior or principal architect-level interviews**:

### **Q11: How do you measure the “business interpretability” of AI-generated rules?**

**Answer:**

> We log every LLM suggestion along with its textual rationale and steward verdict (accepted/rejected).
> Using these logs, we compute an **Interpretability Index (acceptance ratio × explanation clarity)**.
> It quantifies how usable GenAI outputs are in real workflows.

---

### **Q12: How did you ensure non-deterministic LLM responses don’t impact data reliability?**

**Answer:**

> Introduced **determinism layer**:
>
> * Fixed temperature (0.2–0.3)
> * Added deterministic regex validation
> * Stored canonical response templates per prompt version
>   Thus, we ensured consistent outputs for production-critical DQ rules.

---

### **Q13: Explain how you versioned prompts in CI/CD pipelines.**

**Answer:**

> Prompt versions were maintained in a Git-like registry; any change to a prompt or output schema required an approval PR by data governance.
> Cloud Build deployed new prompt versions to Cloud Storage via tagged releases (`DQ_DETECT_Vx.y`).

---

### **Q14: What insights did GenAI uncover that rule-based systems missed?**

**Answer:**

> * *Temporal violations* (“Payment posted before activation date”)
> * *Cross-system mismatches* (CRM region vs. Billing region conflict)
> * *Hidden dependency drift* (new network product not mapped to existing code taxonomy)
>   These context-driven issues were invisible to hardcoded rules.

---

## **6. Quick Memory Hooks for Interview**

| Topic                  | 5-Second Summary                                          |
| ---------------------- | --------------------------------------------------------- |
| **Prompt Registry**    | Cloud-stored versioned library of structured prompts      |
| **Output Enforcement** | JSON schema-bound responses for reliability               |
| **Feedback Loop**      | Steward → Agent → Prompt refinement cycle                 |
| **Contextual Drift**   | Handled via schema embeddings + temporal prompts          |
| **LLM Cost Control**   | Token budget enforcement + caching + async scaling        |
| **Governance**         | IAM, audit logs, approval workflow for production prompts |

---

## **7. Key Interview Sound-Bites**

* “We didn’t just engineer prompts — we **architected contextual understanding**.”
* “Every LLM call in our system produces an auditable, explainable, and measurable output.”
* “We used **LLMs not as black boxes**, but as semantic reasoning engines with embedded observability.”
* “Our prompt registry acts like a **dynamic rules engine**, continuously learning from data steward feedback.”

