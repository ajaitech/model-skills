---
name: aws-multitenant-saas
description: Use when designing, implementing, or auditing AWS multi-tenant SaaS isolation, tenant context, pooled or siloed architecture, Cognito, API Gateway, Lambda, RDS, S3, KMS, IAM ABAC, quotas, metering, or cross-tenant leak prevention.
---

# AWS multi-tenant SaaS isolation

**REQUIRED LIVE REFERENCE:** Use `live-official-docs` for current AWS service, IAM, SDK, and architectural guidance before implementation.

The core loop: **tenant identity is established once at the edge (from the verified
JWT), flows through every layer as context, and every data store filters by it.**
A single missing filter is a breach. Never trust a tenant id from the request body.

## Isolation models (pick per resource, document the choice)
| Model | Isolation | Cost/ops | Use when |
|---|---|---|---|
| **Pooled** (shared infra, tenant column/filter) | logical | low | many small tenants |
| **Silo** (separate stack/DB/account per tenant) | physical | high | regulated / large tenants |
| **Bridge** (pooled compute, siloed data) | mixed | medium | common middle ground |

## Tenant context propagation
1. **Cognito** carries the tenant: `custom:tenant_id` claim (set at sign-up / via pre-token-gen Lambda).
2. **API Gateway** JWT authorizer validates the token; the claim reaches the integration.
3. **Lambda** reads the claim from `event.requestContext.authorizer.jwt.claims['custom:tenant_id']`
   — and uses ONLY that, never a body field.

## Per-store enforcement (the patterns that matter)
```text
DynamoDB   → partition key prefixed with tenantId (PK = "TENANT#acme#…"); or
             leading-key IAM condition: dynamodb:LeadingKeys = ${aws:PrincipalTag/tenant}
RDS/Postgres → Row-Level Security: ALTER TABLE t ENABLE ROW LEVEL SECURITY;
             CREATE POLICY p ON t USING (tenant_id = current_setting('app.tenant'));
             set per request: SET app.tenant = '<verified tenant>';  (read-only role)
S3         → key prefix per tenant (tenants/acme/…); IAM condition on
             s3:prefix = ${aws:PrincipalTag/tenant}; SSE-KMS per-tenant CMK
OpenSearch → filtered alias / document-level security by tenant field
Kinesis/SQS→ tenant in the message; partition key = tenantId for ordering
```

## ABAC with STS session tags (the scalable IAM pattern)
Instead of a role per tenant, assume ONE role with a tenant **session tag**:
```bash
aws sts assume-role --role-arn <role> --role-session-name s --tags Key=tenant,Value=acme
```
Then resource policies/IAM use `${aws:PrincipalTag/tenant}` in conditions so the same
role can only touch the caller's tenant data. Verify with `aws iam simulate-principal-policy`.

## Cross-cutting
- **Encryption:** per-tenant KMS CMK (silo) or shared CMK + tenant in encryption context (pooled); enforce TLS.
- **Quotas / noisy-neighbor:** per-tenant usage plans (API Gateway), token-bucket metering in your app, per-tenant concurrency.
- **Audit:** CloudTrail + structured logs that stamp the verified tenantId on every request; per-tenant log streams.
- **Onboarding/offboarding:** automate tenant provisioning (IaC per silo / control-plane record per pool) and **data erasure** on offboard (GDPR).

## Leak-test before "done"
1. Tenant A token → request B's id/prefix/partition → returns nothing.
2. Body tenant spoofed → server uses the **token** tenant.
3. `simulate-principal-policy` proves the role reaches only its tenant's ARNs/keys.
4. RDS: connect with the app role and `SELECT` cross-tenant rows → RLS blocks it.
5. Unauthenticated request → 401, never reaches data.
