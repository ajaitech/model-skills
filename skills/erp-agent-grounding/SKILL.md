---
name: erp-agent-grounding
description: Ground Bedrock agents against ERP systems with high confidence and zero hallucinated business data — multi-tenant, multi-skill orchestration that connects an agent to ERP records (invoices, ledgers, inventory, orders, customers) via structured queries, action-group API calls, and citeable retrieval. Use when wiring an agent to an ERP/accounting/business system, building NL→SQL over an ERP database, enforcing "answer only from the source / refuse if not grounded", composing multiple agents/skills, or guaranteeing tenant-isolated, auditable, grounded answers for ERP workflows.
---

# Grounding Bedrock agents for ERP connections

ERP answers are facts (₹ amounts, balances, stock, statuses) — **a wrong number is
worse than no answer.** This skill makes an agent answer ERP questions ONLY from
the real system, per tenant, with citations, or refuse. Pairs with:
`aws-bedrock` (commands), `bedrock-multitenant-security` (isolation),
`bedrock-memory` (state), `aws-cli` (deploy).

## The grounding contract (bake into the agent instruction)
> "Answer ERP questions ONLY from tool results or retrieved sources. For any
> number/status, call the structured-query or API action group — never compute or
> recall it. If the data isn't returned by a tool, say you don't have it and how to
> get it. Always cite the source (table/record id / document)."
This + the tactics below is what gets you to "don't make up ERP data."

## Two grounding channels — use the right one

1. **Structured / transactional facts → SQL action group or structured KB (NL→SQL).**
   ERP truth lives in a database. For exact figures, query it, don't retrieve prose.
   - **Structured KB**: create a KB over the ERP DB and use `generate-query`
     (TEXT_TO_SQL) so "overdue invoices above ₹5 lakh" → a real SQL query →
     real rows. (`aws bedrock-agent-runtime generate-query …`, see aws-bedrock.)
   - **Or an action-group Lambda** that runs parameterized, read-only, tenant-scoped
     SQL against the ERP DB and returns rows. Define it with a typed `function-schema`
     (e.g. `get_overdue_invoices(min_amount, tenant)`).

2. **Policy / document knowledge → vector KB with citations.**
   For "what's our refund policy / GST rule", retrieve from an S3-backed vector KB
   and use `retrieve-and-generate`, which returns `citations` → render them so the
   user sees the source. No citation ⇒ treat as ungrounded ⇒ refuse.

## Connect the agent to the ERP (action groups)
```bash
aws bedrock-agent create-agent-action-group --agent-id <ID> --agent-version DRAFT \
  --action-group-name erp-query \
  --action-group-executor '{"lambda":"arn:aws:lambda:<r>:<acct>:function:erp-bridge"}' \
  --function-schema '{"functions":[
    {"name":"get_overdue_invoices","description":"Overdue invoices over a threshold for the tenant",
     "parameters":{"min_amount":{"type":"number","required":true}}},
    {"name":"get_account_balance","description":"Ledger balance for an account",
     "parameters":{"account_id":{"type":"string","required":true}}}]}'
aws lambda add-permission --function-name erp-bridge --statement-id bedrock --action lambda:InvokeFunction \
  --principal bedrock.amazonaws.com --source-arn arn:aws:bedrock:<r>:<acct>:agent/<ID>
aws bedrock-agent prepare-agent --agent-id <ID>
```
The `erp-bridge` Lambda: reads `event.sessionAttributes.tenantId` (from the verified
token), runs **read-only, parameterized** queries scoped to that tenant, returns
structured rows. It NEVER trusts a tenant id from the model.

## Multi-tenant (every ERP customer isolated)
- One agent, tenant in `sessionState.sessionAttributes.tenantId` (set server-side).
- The ERP Lambda and every SQL/KB filter scope to that tenant (row-level / schema /
  KB metadata filter). See bedrock-multitenant-security § isolation + leak-test.
- Memory/sessions namespaced by `<tenantId>:<userId>` (bedrock-memory).

## Multi-skill / multi-agent composition
- **Supervisor + collaborators**: an orchestrator agent delegates to specialist
  agents (e.g. an "invoices" agent, a "ledger" agent) via
  `associate-agent-collaborator` (aws-bedrock § multi-agent). Each collaborator is
  grounded to its own ERP domain + KB.
- Or one agent with multiple action groups (query, post, report) + a KB.

## Confidence guardrails (push toward 100% correctness)
- **Guardrail** on every call: deny ungrounded financial advice, anonymize PII,
  block prompt injection (`apply-guardrail` / `--guardrail-config`).
- **Automated Reasoning policy** (aws-bedrock § advanced): encode ERP business rules
  (e.g. "an invoice is overdue iff due_date < today AND status != paid") so Bedrock
  formally checks the agent's claims against the rules — turns "probably right" into
  "verified against policy".
- **Cite or refuse**: surface `citations` / queried row ids; if a turn produced no
  tool/retrieval evidence, the agent must say so rather than guess.

## Ship + verify checklist
1. Deploy `erp-bridge` Lambda (read-only DB role, tenant-scoped) — `aws-cli` skill.
2. Create agent + action group(s) + KB; `prepare-agent`; alias.
3. boto3 `invoke_agent` with `enableTrace=True`; read the trace to confirm the tool
   actually fired and returned real rows (not a hallucinated answer).
4. Tenant-leak test (bedrock-multitenant-security): tenant A can never see B's figures.
5. Ask a fact NOT in the ERP → agent must refuse, not invent.
