# E-commerce Return/Refund Triage Agent

An n8n workflow that reads an incoming return request, has an AI agent judge it against
a written return policy, and — this is the part most demos skip — **shows you exactly
what happens when the automation hits a wall.**

## What happens on run #2

Run #1 is always the happy path. Here's what actually happened on a real run against a
real (empty) Shopify dev store, with no cherry-picking:

1. A return request comes in via webhook: order `1002`, reason `defective`, description
   `"Screen is dead on arrival, clearly damaged in shipping."`
2. The AI agent (Groq / `openai/gpt-oss-120b`) reads the return policy in its system
   prompt and decides:
   ```json
   {
     "decision": "approve",
     "confidence": 95,
     "reasoning": "The request cites a 'defective' reason. According to the return
       policy, defective items are always eligible for return regardless of purchase
       date. No conflicting signals were found.",
     "concerns": []
   }
   ```
3. Because confidence is high, the workflow tries to act on it automatically — a real
   `POST` to the Shopify Admin API.
4. The order doesn't exist in this dev store, so Shopify correctly returns `400 Bad
   Request — Required parameter missing or invalid`.
5. Instead of failing silently, the workflow's error branch fires and posts this to
   Slack, **with the AI's full reasoning trail attached**, not just "something broke":

   > 🚨 **Automatic return processing FAILED — manual review needed**
   > **Order:** 1002
   > **Customer:** buyer@example.com
   > **AI Decision:** approve
   > **Confidence:** 95
   > **Error:** Bad request - please check your parameters
   > *Automated with this n8n workflow*

That Slack message is real — it was posted by this exact workflow, not staged.

## Why it's built this way

Most "AI agent" automation demos show only the case where everything works. In
practice, the thing that determines whether an automation is safe to hand real business
processes to is **what it does when it's wrong or when the outside system it depends on
fails** — not the happy path. This workflow is deliberately built so that:

- The AI never gets to act silently. Every decision comes with a `reasoning` string and
  a `confidence` score, not just a verdict.
- Low-confidence or ambiguous decisions (`confidence < 70` or `decision: "escalate"`)
  route to a human via Slack **before** anything external is touched.
- High-confidence decisions do try to act for real (via the Shopify Admin API) — but
  the call is wrapped so that any failure routes to a Slack alert carrying the AI's
  original reasoning, not a bare "500 error" with no context.

## Architecture

```
Webhook (return request)
   │
   ▼
AI Agent (Groq, structured output: decision / confidence / reasoning / concerns)
   │
   ▼
Acknowledge customer immediately (webhook response)
   │
   ▼
Needs human review?  (decision == "escalate"  OR  confidence < 70)
   │                                   │
  yes                                  no
   │                                   │
   ▼                                   ▼
Slack: human review           Shopify API call (real action)
(full reasoning attached)             │
                                  on failure
                                       │
                                       ▼
                          Slack: processing failed alert
                          (reasoning + actual error attached)
```

## Stack

- **n8n** — orchestration
- **Groq** (`openai/gpt-oss-120b`) — the triage decision, via a structured output
  parser so the agent can never skip the reasoning/confidence fields
- **Shopify Admin API** — the real external action the workflow attempts
- **Slack** — both review and failure-alert destinations

## Import this workflow

`workflow.json` is a standard n8n export. In n8n: **Workflows → Create → ⋯ → Import
from File**, then reconnect the three credentials (Groq, Slack, Shopify) to your own
accounts — they're intentionally left as placeholders in the file.

## What I'd build next

- A small persistence layer (e.g. Postgres or a Google Sheet) that logs every AI
  decision and the human's final call, so future triage runs can pull in the most
  similar past cases as reference — nudging judgment without silently retraining
  anything.
- Proper Shopify Admin API return/refund payloads (this version intentionally keeps a
  simplified body so the failure path above stays reproducible against an empty dev
  store).

---

Built by [Sengoku Automation](https://github.com/sengoku-samurai) — n8n + AI workflow
automation.
