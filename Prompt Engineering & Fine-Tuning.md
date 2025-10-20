# **Doc-3: Prompt Engineering & Fine-Tuning – Practical Guide for Agentic AI in Data Quality**

---

## **1. Context & Objective**

In our GenAI-based **Agentic Data Quality (DQ)** solution, prompt engineering determines how effectively each agent — **Detector**, **Remediator**, **Feedback**, and **Governance** — reasons about, remediates, and validates data anomalies.

This document explains:

* How prompts are structured across the multi-agent workflow
* Patterns used for **Data Quality reasoning, rule synthesis, and SQL generation**
* How to **fine-tune and evaluate** model behavior
* What **prompt design principles** help in interviews and real-world deployments

---

## **2. Prompt Engineering in the Agentic DQ Architecture**

Each agent in the **Multi-Agent Control Plane (MCP)** uses a distinct prompt layer. Prompts are **modular, contextual, and role-driven** — built dynamically from Dataplex metadata, historical DQ results, and user feedback.

| Agent                | Purpose                                          | Input Context                              | Output                                |
| -------------------- | ------------------------------------------------ | ------------------------------------------ | ------------------------------------- |
| **Detector Agent**   | Identify anomalies in source or staging datasets | Schema + sample data + prior DQ thresholds | DQ Incident Summary JSON              |
| **Remediator Agent** | Suggest SQL / ETL corrections                    | Incident summary + transformation logic    | SQL patch + explanation               |
| **Feedback Agent**   | Validate or refine remediation suggestions       | Proposed patch + steward comments          | Approval decision + reason            |
| **Governance Agent** | Ensure compliance with data policy               | Data tags + DLP metadata                   | Risk classification + mitigation plan |

---

## **3. Prompt Patterns by Agent Role**

### **3.1 Detector Agent Prompt Pattern**

**Goal:** Detect anomalies beyond static thresholds using semantic context.

**Prompt Template Example:**

```text
You are a Data Quality Detection Agent for {domain}. 
You are given:
1. Dataset schema: {schema}
2. Sample data: {sample_rows}
3. Historical thresholds: {dq_rules}
4. Recent anomalies: {recent_incidents}

Analyze and summarize:
- Which columns deviate significantly?
- What kind of anomaly is likely (missing, range, type, semantic)?
- Confidence score (0–1)
- Suggested rule improvement.

Output JSON with fields:
[{"column": "...", "issue": "...", "confidence": ..., "suggested_rule": "..."}]
```

**Prompt Pattern Used:**
✅ *Role-based context*
✅ *Structured JSON output constraint*
✅ *Dynamic data injection (schema, samples)*

---

### **3.2 Remediator Agent Prompt Pattern**

**Goal:** Propose automated remediation logic and explain rationale.

**Prompt Template Example:**

```text
You are a Data Remediation Assistant.
Given:
1. DQ Incident: {incident_json}
2. Existing transformation logic: {etl_code}
3. Data catalog context: {metadata_summary}

Propose:
1. SQL or ETL fix for the identified issue.
2. Step-by-step rationale.
3. Risk or data loss likelihood (Low/Medium/High).

Output JSON:
{
 "fix_sql": "...",
 "explanation": "...",
 "risk": "..."
}
```

**Prompt Pattern Used:**
✅ *Tool Augmentation Pattern* (Agent generates SQL → validated by tool before execution)
✅ *Chain-of-thought summarization* (explanation required)
✅ *Guardrails for risk scoring*

---

### **3.3 Feedback Agent Prompt Pattern**

**Goal:** Convert steward validation into model learning signals.

**Prompt Template Example:**

```text
You are a Data Quality Feedback Agent.
You will be shown:
1. Original issue: {dq_issue}
2. Proposed fix: {remediation_sql}
3. Steward feedback: {comments}

Based on this, summarize:
- Was the proposed remediation accepted?
- Why or why not?
- What should future remediation prompts include or avoid?

Output JSON:
{
 "accepted": true/false,
 "reason": "...",
 "learning_summary": "..."
}
```

**Prompt Pattern Used:**
✅ *HITL (Human-in-the-Loop) feedback assimilation*
✅ *Conversational summarization for few-shot updates*

---

### **3.4 Governance Agent Prompt Pattern**

**Goal:** Enforce compliance and classify data sensitivity.

**Prompt Template Example:**

```text
You are a Governance Agent monitoring DQ workflows.
Given:
1. Dataset metadata: {dataplex_tags}
2. DQ issue details: {dq_incident}
3. Remediation plan: {remediation_summary}

Determine:
- Does this dataset contain sensitive data (PII, financial, etc.)?
- Is the remediation compliant with policy?
- Should approval be manual or automated?

Output JSON:
{
 "contains_sensitive_data": true/false,
 "compliance_score": 0-1,
 "approval_type": "manual" or "auto"
}
```

**Prompt Pattern Used:**
✅ *Policy reasoning pattern*
✅ *Embedded metadata grounding via Dataplex tags*

---

## **4. System Prompt Strategies**

| Prompt Layer            | Purpose                          | Notes                                                     |
| ----------------------- | -------------------------------- | --------------------------------------------------------- |
| **System Prompt**       | Defines immutable role and tone  | “You are a Data Quality Detector for Financial Datasets.” |
| **Instruction Prompt**  | Specific query for the task      | e.g., “Identify likely anomalies using schema + sample.”  |
| **Context Prompt**      | Inject metadata & recent results | Pulled from Firestore + Vertex Embeddings                 |
| **Output Guard Prompt** | Enforce structured format        | e.g., “Return JSON with schema {…} only.”                 |

> 🔹 *Interview Tip:* Always mention that you separate “contextual grounding” and “output formatting” — this shows architectural discipline in LLM usage.

---

## **5. Fine-Tuning Strategies**

Fine-tuning is used when prompts alone cannot consistently enforce logic or style.
For DQ Agentic AI, fine-tuning targets **domain consistency and remediation quality**.

| Model                    | Fine-tuned For            | Training Data              | Outcome                                  |
| ------------------------ | ------------------------- | -------------------------- | ---------------------------------------- |
| `text-bison` (Vertex AI) | DQ Anomaly classification | 10k labeled DQ issues      | Better precision in “semantic anomalies” |
| `code-bison`             | SQL remediation           | ETL patches + explanations | Higher syntactic correctness             |
| `gemini-pro`             | Policy reasoning          | Compliance dataset         | Consistent compliance scoring            |

**Fine-Tuning Dataset Composition:**

* Input: prompt + schema + data sample
* Output: validated JSON response
* Split: 80/10/10 (train/val/test)
* Evaluation: BLEU (for text), execution accuracy (for SQL)

---

## **6. Lessons Learned**

| Challenge                             | Lesson                                                                |
| ------------------------------------- | --------------------------------------------------------------------- |
| Prompt drift due to schema complexity | Use *template modularization* — keep role + context separate          |
| SQL hallucination                     | Use *tool feedback loop* — SQL generated → validated via ADK Tool API |
| Over-explaining in JSON               | Explicitly define max response length                                 |
| Forgetting prior incidents            | Add *vector-based RAG grounding* via Vertex Matching Engine           |
| High latency on long prompts          | Use *compressed schema summaries* and sample only top anomalies       |

---

## **7. Prompt Evaluation Metrics**

| Metric                     | Description                        | How Calculated                  |
| -------------------------- | ---------------------------------- | ------------------------------- |
| **Semantic Accuracy**      | Does output logically match issue? | Manual scoring or evaluator LLM |
| **Compliance Consistency** | Is policy scoring repeatable?      | Evaluate across 50 test cases   |
| **Format Adherence**       | JSON validity rate                 | Parsing success / total runs    |
| **SQL Executability**      | % of generated SQL that compiles   | Cloud BQ dry-run validation     |
| **Feedback Incorporation** | Steward suggestions retained?      | Compare pre/post fine-tuning    |

---

## **8. Prompt Engineering Tools (on GCP)**

| Tool                           | Purpose                                     |
| ------------------------------ | ------------------------------------------- |
| **Vertex AI Prompt Design UI** | Iterative prompt testing                    |
| **ADK PromptRegistry**         | Version control for prompt templates        |
| **Vertex Evaluation SDK**      | Evaluate LLM responses vs reference outputs |
| **Dataplex Metadata APIs**     | Dynamic grounding context                   |
| **BigQuery Data Sampler**      | Generate schema-context dynamically         |

---

## **9. Example – Detector to Remediator Conversation**

**Flow:**

1. **Detector Agent Prompt →** identifies “missing product_price for region = EU”.
2. **Remediator Prompt →** proposes:

   ```sql
   UPDATE sales_data
   SET product_price = regional_avg_price
   WHERE product_price IS NULL AND region = 'EU';
   ```
3. **Feedback Agent Prompt →** marks it approved with reason: “Valid based on last 3 months.”
4. **Governance Agent Prompt →** ensures “No PII columns touched.”

Each agent reuses the prior agent’s output as context — forming a **prompt chain with contextual carryover**.

---

## **10. “Interview Talk Track” Summary**

> “Our prompt design is modular and role-specific.
> Each agent has a clear system identity, grounded in Dataplex metadata and past DQ incidents.
> We used patterns like structured JSON output, tool-augmented reasoning, and feedback incorporation.
> Over time, feedback loops and fine-tuning converted human approval data into structured learning examples — making the system smarter, safer, and explainable.”

---

## **11. Common Interview Questions (with Suggested Answers)**

| Question                                                                      | Suggested Answer                                                                                                                 |
| ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **What’s the biggest difference between prompt engineering and fine-tuning?** | Prompting changes *input context*; fine-tuning changes *model weights*. Use prompts for control and fine-tuning for consistency. |
| **How do you handle hallucination in SQL generation?**                        | Use structured output prompts + post-generation validation tool. If SQL fails dry-run, rerun with revised context.               |
| **What’s few-shot prompting, and how do you apply it here?**                  | Include 2–3 known incident-resolution examples to guide the model — especially useful for new data domains.                      |
| **How do you version and test prompts?**                                      | Every prompt has a version tag in ADK PromptRegistry; tests run via Vertex Evaluation SDK comparing JSON output consistency.     |
| **What’s your grounding strategy?**                                           | Dataplex metadata + previous incident embeddings (Vertex Matching Engine) form RAG context.                                      |

---

## **12. Diagram: Prompt Flow in Multi-Agent DQ System**

*(Visualize when whiteboarding)*

```
 ┌──────────────────────────────┐
 │ Dataplex Metadata & Schema   │
 └─────────────┬────────────────┘
               │
        [Context Injection]
               │
 ┌─────────────▼────────────────────┐
 │  Detector Agent (Prompt 1)       │
 │  → Identify anomaly               │
 └─────────────┬────────────────────┘
               │ JSON Issue
               ▼
 ┌─────────────▼────────────────────┐
 │  Remediator Agent (Prompt 2)     │
 │  → Suggest fix                   │
 └─────────────┬────────────────────┘
               │
               ▼
 ┌─────────────▼────────────────────┐
 │  Feedback Agent (Prompt 3)       │
 │  → Validate & Learn              │
 └─────────────┬────────────────────┘
               │
               ▼
 ┌─────────────▼────────────────────┐
 │ Governance Agent (Prompt 4)      │
 │ → Policy compliance              │
 └──────────────────────────────────┘
```

# **Doc-3 Addendum: Likely Interview Questions & Whiteboard Prompts on Prompt Engineering & Fine-Tuning**

---

## **1. Section A — Prompt Engineering Fundamentals**

### **Q1. What is Prompt Engineering, and why is it critical in an Agentic AI DQ system?**

**Answer:**
Prompt Engineering is the practice of designing structured, context-aware instructions that guide a large language model (LLM) toward reliable, predictable, and auditable outputs.
In the Agentic DQ system:

* Each agent (Detector, Remediator, Governance, Feedback) acts as a specialized role with its own **prompt contract**.
* Prompts inject **schema, metadata, and historical DQ context** to anchor the model’s reasoning.
* Structured JSON outputs ensure **machine-readable** interoperability between agents.
  Without disciplined prompt design, responses would drift, become verbose, or violate schema boundaries.

> **Whiteboard Cue:** Draw the “Prompt Layer Stack” (System → Instruction → Context → Output Guard) and show how Dataplex metadata feeds into the Context layer.

---

### **Q2. Explain the difference between system prompts, instruction prompts, and contextual prompts.**

| Layer                   | Description                                       | Example                                                        |
| ----------------------- | ------------------------------------------------- | -------------------------------------------------------------- |
| **System Prompt**       | Defines the agent’s role and tone.                | “You are a Data Quality Detector for financial data.”          |
| **Instruction Prompt**  | Specific task description.                        | “Identify columns with semantic inconsistencies.”              |
| **Context Prompt**      | Injects dynamic runtime info (schema, anomalies). | “Schema: customer_id INT, purchase_dt DATE…”                   |
| **Output Guard Prompt** | Defines format, structure, and boundaries.        | “Respond only in JSON with fields: column, issue, confidence.” |

> **Tip:** Emphasize how modular layering makes it easier to A/B test prompt variants without retraining models.

---

### **Q3. How do you handle prompt drift when schemas or data types change frequently?**

**Answer:**

1. Maintain **parameterized templates** — e.g., `{schema}`, `{sample_rows}`, `{dq_rules}`.
2. Dynamically build context via Dataplex API and Firestore before prompt dispatch.
3. Implement **prompt evaluation checks** (token length, missing placeholders).
4. Version prompts in **ADK PromptRegistry** and use rollback if performance degrades.

> **Whiteboard Cue:** Show schema change detection → trigger prompt rebuild workflow via Cloud Function.

---

## **2. Section B — Designing Robust Prompts**

### **Q4. What patterns are used in the Detector and Remediator agents’ prompts?**

**Answer:**

* **Detector Pattern:** Hybrid reasoning (SQL + contextual LLM) — uses *structured observation* template.
* **Remediator Pattern:** *Tool-augmented reasoning* — model outputs SQL → validated via dry-run (MCP Tool).
* **Feedback Pattern:** *Reflection prompting* — converts steward comments into structured learning events.
* **Governance Pattern:** *Policy reasoning* — ensures rule compliance via Dataplex metadata grounding.

| Pattern                    | Example                                                                |
| -------------------------- | ---------------------------------------------------------------------- |
| **Chain-of-Thought (CoT)** | “Explain your reasoning before producing JSON output.”                 |
| **RAG Grounding**          | “Use Dataplex schema & historical DQ incidents as reference.”          |
| **JSON Guardrail**         | “Output must follow format: [{column, issue, confidence, suggestion}]” |
| **Multi-turn Correction**  | “If SQL validation fails, re-evaluate with error context.”             |

---

### **Q5. How do you test and evaluate prompt quality?**

**Answer:**
Use both **quantitative** and **qualitative** metrics:

| Metric                          | Purpose                                       | Tool                      |
| ------------------------------- | --------------------------------------------- | ------------------------- |
| **JSON Validity Rate**          | Ensures structured format compliance.         | Python JSON parser tests  |
| **Semantic Accuracy**           | Measures match with human-labeled truth.      | Vertex Eval SDK           |
| **SQL Executability**           | % of generated queries that run successfully. | BigQuery dry-run API      |
| **Consistency Across Versions** | Detects drift.                                | PromptRegistry diff tests |

> **Whiteboard Cue:** Draw evaluation pipeline — Prompt Variant A/B → Validation Tests → Score Logging → Registry Update.

---

### **Q6. Give an example of a poorly designed prompt and its fix.**

| Poor Prompt                       | Issue                                  | Fix                                              |
| --------------------------------- | -------------------------------------- | ------------------------------------------------ |
| “Check for errors in dataset.”    | Too vague; no role or format guidance. | Add role, schema context, and output constraint. |
| “Find anomalies using AI.”        | No grounding; leads to hallucination.  | Inject Dataplex schema + rule context.           |
| “Suggest fixes for missing data.” | Lacks validation guardrail.            | Add “generate SQL + dry-run diff required.”      |

---

## **3. Section C — Fine-Tuning Strategy**

### **Q7. When do you decide to fine-tune instead of improving prompts?**

**Answer:**
Fine-tuning is needed when:

* Responses must be **consistent across many schemas/domains**.
* Prompt tokens become too long or repetitive.
* You need **behavioral specialization** (e.g., risk classification).

> **Rule of Thumb:**
> Use **prompt tuning** for *control* and **fine-tuning** for *stability*.

---

### **Q8. Describe the fine-tuning workflow on Vertex AI for this project.**

**Answer:**

1. Collect validated prompt-response pairs (approved steward cases).
2. Store them in **BigQuery LTM dataset**.
3. Export to **Cloud Storage → Vertex Fine-Tuning job**.
4. Evaluate new checkpoint with Vertex Evaluation SDK.
5. Register fine-tuned model version in **ADK Model Registry**.
6. Update inference endpoint in the Orchestrator Agent.

| Step      | Tool       | Output           |
| --------- | ---------- | ---------------- |
| Data Prep | BigQuery   | Training JSONL   |
| Training  | Vertex AI  | Fine-tuned model |
| Eval      | Vertex SDK | BLEU + accuracy  |
| Deploy    | Cloud Run  | API endpoint     |

---

### **Q9. What are the trade-offs of fine-tuning versus using retrieval-augmented prompts (RAG)?**

| Aspect           | Fine-Tuning                            | RAG                                 |
| ---------------- | -------------------------------------- | ----------------------------------- |
| **Cost**         | High upfront                           | Lower (no retraining)               |
| **Adaptability** | Slower to adapt                        | Instantly updated with embeddings   |
| **Consistency**  | Very stable                            | Depends on embedding relevance      |
| **Best For**     | Policy classification, rule suggestion | Schema reasoning, anomaly detection |

> **Whiteboard Cue:** Draw “Prompt vs Fine-Tuning Boundary” — what’s solved by context vs what’s baked into weights.

---

## **4. Section D — Integration & Memory**

### **Q10. How do prompts leverage memory or embeddings in this system?**

**Answer:**
Each agent uses a **Retrieval-Augmented Generation (RAG)** mechanism:

1. Query **Vertex Matching Engine** for embeddings related to the dataset or prior incident.
2. Inject retrieved snippets into the context prompt.
3. Model uses the combined context + instruction for reasoning.
4. Output stored in Firestore and appended to LTM.

> **Whiteboard Cue:**
> “Incident → Embedding → RAG Fetch → Prompt Assembly → Response → LTM Update” loop.

---

### **Q11. How do you ensure prompt safety and governance compliance?**

**Answer:**

* Enforce **PII redaction** via Cloud DLP API before prompt dispatch.
* Maintain **prompt version lineage** in Firestore (who changed what, when).
* Use **approval thresholds** in Governance Agent: high-risk prompts require dual sign-off.
* Evaluate **LLM toxicity and leakage** risk via Vertex Safety Filters.

---

## **5. Section E — Applied Whiteboard Prompts**

### **Prompt 1 — Detector Reasoning**

> “Show how you would design a prompt for detecting null-rate anomalies in customer transactions using Dataplex metadata.”

**Expected Whiteboard Answer:**

1. System Role → “You are a Data Quality Detector.”
2. Context → Inject schema & prior thresholds.
3. Task → “Identify columns violating null-rate > threshold.”
4. Format → JSON output.
5. Grounding → Link Dataplex tags (customer_id, region).
6. Example Output → show structured JSON with column, issue, confidence.

---

### **Prompt 2 — Remediation Flow**

> “Design a prompt for an agent that fixes duplicate records automatically but validates against risk policy.”

**Expected Steps:**

1. Ingest dq_incident JSON.
2. Propose SQL remediation + rollback script.
3. Assign risk score = HIGH if primary key involved.
4. Require manual approval path if `risk >= 0.6`.
5. Output structured JSON: `{sql_patch, rollback, risk}`.

---

### **Prompt 3 — Feedback Loop**

> “How do you incorporate human feedback into the learning loop?”

**Answer:**

* Feedback Agent captures steward decisions as structured JSON.
* Map `accepted/rejected + reason` into **PromptRegistry** metadata.
* When multiple rejections occur for a prompt version → auto-flag for refinement.
* Store context in LTM for fine-tuning candidates.

---

## **6. Section F — Evaluation & Governance Questions**

### **Q12. How do you track and audit prompt performance across agents?**

**Answer:**

* Every prompt call logs:

  * Version ID
  * Input context hash
  * Token usage
  * Output quality metrics
* Firestore stores per-agent telemetry.
* Dataplex metadata updates capture lineage of rule → incident → remediation → approval.

---

### **Q13. How do you ensure consistent JSON structure across multiple agents?**

**Answer:**

* Use **Prompt Templates with explicit JSON schema** validation before submission.
* Each agent has a **JSON schema validator** class (ADK built-in).
* Failed validations trigger fallback → safe “explanation-only” mode.

---

### **Q14. What’s your approach to prompt lifecycle management?**

| Stage       | Activity                    | Tool            |
| ----------- | --------------------------- | --------------- |
| **Design**  | Draft template              | PromptRegistry  |
| **Test**    | Evaluate against gold set   | Vertex Eval SDK |
| **Deploy**  | Attach version tag          | Firestore       |
| **Monitor** | Track drift / errors        | Cloud Logging   |
| **Retire**  | Archive deprecated versions | GCS             |

---

## **7. Section G — “Explain Like You’re Whiteboarding” Flow**

> “Imagine you’re whiteboarding this for an interviewer — walk through step by step.”

1. **Draw the flow:** Dataplex metadata → RAG store → Detector prompt → dq_incident JSON → Remediator prompt → fix → Feedback prompt → Governance validation.
2. **Explain roles:** “Each box represents an agent with its own system prompt, tuned for a narrow reasoning scope.”
3. **Show feedback cycle:** “All accepted remediations feed back into fine-tuning corpus.”
4. **Summarize:** “This converts a static DQ gate into a self-learning agentic ecosystem with explainable prompts, safe execution, and governance alignment.”

---

## **8. Section H — Interview Wrap-Up Summary**

> “In short — prompt engineering here isn’t just about crafting clever instructions.
> It’s a **design discipline** involving context injection, policy compliance, structured output, and measurable improvement.
> Fine-tuning comes later — once we’ve built enough validated prompt-response data to justify stable, domain-specific behavior.”

---


