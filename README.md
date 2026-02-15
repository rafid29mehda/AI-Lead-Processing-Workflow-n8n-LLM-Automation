# AI Lead Processing Workflow — n8n + LLM Automation

## Project Overview

This project is an end-to-end AI-driven lead processing system built entirely within **n8n** using native workflow nodes. A lead arrives through a webhook (via an HTML form or API call), gets classified by a locally-hosted LLM, has key fields extracted, gets stored in a PostgreSQL database, and receives an AI-generated email response — all automatically, with zero manual intervention.

**Tech Stack:** n8n (self-hosted via Docker) · Ollama + Llama 3.2 (local LLM) · Neon PostgreSQL · Gmail OAuth2 · HTML form (web interface)

**Architecture — 17 nodes:**

```
HTML Form / API
      ↓ POST
Webhook → Prepare Lead Data → Classify Intent (LLM) → Switch (Route by Intent)
                                   ↕                    ├── Sales → LLM Response ──────┐
                            Ollama (temp 0.1)            ├── Support → LLM Response ────┤
                            Output Parser                ├── Spam → Static Reply ───────┤
                                                         └── Fallback → Static Reply ───┘
                                                                                        ↓
                                                      Assemble Record → PostgreSQL → IF (Has Email?)
                                                                                     ├── Yes → Gmail → Respond to Webhook
                                                                                     └── No ────────→ Respond to Webhook
```

---

## Prompt Strategy

The workflow uses a **two-stage prompting architecture** with deliberate temperature separation and proper System/User message roles.

**Stage 1 — Classification (temperature: 0.1).** The classification LLM Chain splits its instructions across n8n's System Message and User Message fields. The System Message defines the LLM's role as a "strict lead classification system" and provides explicit keyword-to-category mapping: buying/pricing/demo → Sales, issues/bugs/help → Support, irrelevant/promotional → Spam. The User Message carries only the dynamic lead data (message text, sender name, email). This separation matters because LLMs treat System Messages as higher-priority directives, making classification rules harder to override by ambiguous user input. A Structured Output Parser sub-node enforces a strict JSON schema with four fields (intent, name, company, requirement), ensuring the model cannot produce free-form text.

**Stage 2 — Response Generation (temperature: 0.7, branched).** Rather than using a single prompt with conditional instructions like "if Sales, write X; if Support, write Y," the Switch node routes each intent to a dedicated LLM Chain with its own System/User message pair. The Sales chain's System Message defines a "friendly sales assistant" role with guardrails: acknowledge interest, promise 24-hour follow-up, and explicitly prohibit pricing commitments. The Support chain takes an empathetic tone and injects an auto-generated ticket number (TK-XXXXXX). Spam leads skip the LLM entirely — a static Set node returns a generic acknowledgment, saving compute and eliminating hallucination risk completely for that branch.

**Why branching over conditional prompting:** Each LLM call has exactly one job, which reduces the probability of the model ignoring instructions. Prompts stay short and focused. Each branch can be independently tuned without affecting the others.

---

## Hallucination Reduction

Multiple layers work together to constrain LLM output. The **Structured Output Parser** enforces a JSON schema on classification, rejecting any response that doesn't conform to the expected four fields. **Temperature 0.1** for classification produces near-deterministic results — the same input consistently yields the same intent. The prompt includes **explicit keyword-to-category mapping** so the model follows defined rules rather than reasoning from scratch. **Intent-specific branching** means each response prompt has a single objective with no conditional logic to misinterpret. **Hard length constraints** ("3-4 sentences maximum") prevent the model from rambling and inventing facts. **Explicit prohibitions** ("Do NOT make specific promises about pricing") block the most common hallucination pattern in sales contexts: over-commitment. The **Fallback branch** on the Switch node catches any unrecognized intent with a safe static response, ensuring edge cases never produce nonsensical output.

---

## Error Handling

The workflow implements error handling at three levels. At the **node level**, all three LLM Chain nodes and the Postgres node have Retry on Fail enabled (3 attempts with 5-second intervals for LLM calls, 2 attempts with 3-second intervals for database operations). This handles Ollama cold-start delays when the model is first loading into memory and transient network issues with the cloud database. At the **data validation level**, the Structured Output Parser rejects malformed LLM output before it reaches the Switch node, the IF node prevents Gmail errors by checking whether a sender email exists before attempting to send, and the Switch node's Fallback output catches any intent that doesn't match the three expected categories. At the **workflow level**, a dedicated Error Handler workflow (connected via n8n's Error Workflow setting) captures any unhandled exception with the Error Trigger node and logs the timestamp, failing node name, error message, and execution ID. All errors are also visible in n8n's built-in Execution History for debugging.

---

## Submission Files

| File | Description |
|------|-------------|
| `AI_Lead_Processing_Workflow.json` | Main workflow |
| `error-handler-workflow.json` | Error handler workflow |
| `lead-form.html` | HTML form for submitting leads via browser |
| `README.md` | This document |
| `Demo Video.mp4` | Live Workflow Execution |

