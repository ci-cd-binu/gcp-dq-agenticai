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

---

