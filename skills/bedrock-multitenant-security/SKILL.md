---
name: bedrock-multitenant-security
description: Multi-tenant isolation and security for Amazon Bedrock agents, knowledge bases, and model access. Use when designing or debugging tenant separation (one tenant must never see another's data, sessions, KB content, or quota), passing/enforcing tenantId, scoping IAM least-privilege, filtering KB retrieval by tenant, applying guardrails/PII controls, encrypting with KMS, and auditing. Trigger on anything involving tenantId, "multi-tenant", "tenant isolation", "cross-tenant", session attributes, per-tenant KB/guardrail, IAM scoping, or securing a Bedrock backend.
---

# Bedrock multi-tenant isolation & security

Goal: a single shared agent/KB serving many tenants where **no tenant can ever
read another's data, session, memory, or KB content, or spend another's quota.**
The two cardinal rules:

1. **Tenant identity comes from the verified token, NEVER from the request body.**
   The edge (Cognito JWT authorizer / your Lambda) validates the JWT and extracts
   the tenant claim (e.g. `custom:tenant_id`). Pass THAT downstream. A body
   `tenantId` is a hint at best and a spoof vector at worst.
2. **Every data path is filtered by that tenant id** — session, memory, KB
   retrieval, S3 prefix, and IAM. One missing filter = a cross-tenant leak.

## Carry tenant through the agent (session attributes)
`invoke_agent` (boto3 — it's SDK-only) takes `sessionState`:
```python
sessionState={
  "sessionAttributes":       {"tenantId": "acme"},     # available to action-group Lambdas
  "promptSessionAttributes": {"tenantId": "acme"}       # injected into the model prompt
}
```
Action-group Lambdas read `event["sessionAttributes"]["tenantId"]` and MUST scope
every downstream query (DB, API, S3) to it. Never let the model choose the tenant.

## Knowledge-base tenant isolation (pick one, enforce always)
- **Metadata filtering (shared KB, preferred for many small tenants):** tag every
  ingested doc with `{"tenantId":"acme"}` (a `<file>.metadata.json` sidecar in S3),
  then filter on retrieve:
  ```bash
  aws bedrock-agent-runtime retrieve --knowledge-base-id <KB> \
    --retrieval-query '{"text":"..."}' \
    --retrieval-configuration '{"vectorSearchConfiguration":{"filter":{"equals":{"key":"tenantId","value":"acme"}}}}'
  ```
  The filter MUST be set server-side from the verified tenant — never client-supplied.
- **Separate KB or data-source per tenant:** strongest isolation, higher cost/ops.
- For `retrieve-and-generate`, put the same `filter` in `retrievalConfiguration`.

## IAM least-privilege & per-tenant scoping
- **No long-lived keys.** Lambda/agent uses its execution **role**; cross-account
  uses `sts:AssumeRole`. (See aws-cli skill.)
- **Session tags for attribute-based access control (ABAC):** assume a role with
  `--tags Key=tenant,Value=acme` and gate resources with a condition:
  `"Condition":{"StringEquals":{"aws:PrincipalTag/tenant":"${s3:ExistingObjectTag/tenant}"}}`.
- Scope agent/KB perms tightly: `bedrock:InvokeModel*` only on the needed
  model/inference-profile ARN; `bedrock:Retrieve` only on the tenant's KB ARN;
  `bedrock:InvokeAgent` only on the specific agent-alias ARN.
- Verify a policy actually allows/denies: `aws iam simulate-principal-policy`.

## Quota / cost isolation
- Per-tenant **application inference profiles** (`aws bedrock create-inference-profile`
  with `--tags tenant=acme`) → cost allocation + dedicated routing per tenant.
- Track + cap usage per tenant in your own metering (the gateway), not just AWS quotas.

## Guardrails & data protection
- Apply a **guardrail** on every invocation (`--guardrail-config` / `apply-guardrail`)
  for PII anonymization, denied topics, prompt-injection mitigation — per-tenant
  guardrail ids where policies differ.
- **Encryption:** KMS CMK on the agent (`--customer-encryption-key-arn`), the KB
  vector store, and S3 data sources. Enforce TLS-only via bucket/endpoint policy.
- **Audit:** enable Bedrock model-invocation logging + CloudTrail; log the verified
  tenantId on every turn for traceability.

## Leak-test checklist (run before "done")
1. Tenant A's token → request data tagged tenant B → must return **nothing**.
2. Body `tenantId` spoofed to B while token says A → server uses **A** (ignores body).
3. KB retrieve without a filter → must be **impossible** in code (filter injected server-side).
4. IAM: `simulate-principal-policy` shows the role can reach ONLY its tenant's resources.
5. Unauthenticated request → **401**, never reaches Bedrock.
