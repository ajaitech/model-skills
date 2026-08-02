# AWS Platform

## Applies when
- `cdk.json` exists, or `infrastructure/`/`cdk/` holds CDK stacks.
- `template.yaml` (SAM) or `serverless.yml` present.
- Deps on `aws-cdk-lib`, `boto3`, `aws-sdk`/`@aws-sdk/*`.
- Terraform files with an `aws` provider block.

## Authoritative sources
| Need | URL |
|---|---|
| IAM | https://docs.aws.amazon.com/iam/ |
| Bedrock | https://docs.aws.amazon.com/bedrock/ |
| AppSync | https://docs.aws.amazon.com/appsync/ |
| API Gateway | https://docs.aws.amazon.com/apigateway/ |
| DynamoDB | https://docs.aws.amazon.com/amazondynamodb/ |
| Cognito | https://docs.aws.amazon.com/cognito/ |
| CDK v2 guide | https://docs.aws.amazon.com/cdk/v2/guide/home.html |
| CLI reference | https://docs.aws.amazon.com/cli/latest/ |
| SDK reference | https://docs.aws.amazon.com/sdkref/latest/guide/overview.html |
| Well-Architected | https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html |
| aws-cdk (GitHub) | https://github.com/aws/aws-cdk |

## Non-obvious rules
- **Explicit Deny always wins**, regardless of source (SCP, boundary, identity/resource policy) or order — one stray Deny anywhere blocks the call.
- **Resource policies matter by type.** Cross-account needs identity AND resource policy Allow. Same-account S3 needs only identity policy — but KMS key policies apply even same-account; a key policy missing the caller blocks decryption regardless of IAM.
- **Bedrock model access is opt-in per region per model**, separate from IAM. A 403 with full `bedrock:*` is almost always missing model access, not a policy bug.
- **Bedrock on-demand has per-model TPM/RPM quotas** that throttle silently — production needs Provisioned Throughput, not just retries.
- **AppSync resolvers run sandboxed**, no arbitrary outbound calls. `$context.identity` shape differs completely per auth mode — code defensively per mode.
- **REST, HTTP, and WebSocket APIs are different products.** REST has validators/usage plans; HTTP API is cheaper, fewer features; WebSocket needs `$connect`/`$disconnect`/`$default` routes plus a connections table since Lambda holds no socket state between invocations.
- **WebSocket replies need the Management API** (`@connections` + stored connectionId), not the client SDK. Stale IDs throw `GoneException` — catch and prune.
- **DynamoDB reads default to eventually consistent**; strong reads cost 2x RCU and are unavailable on GSIs.
- **Hot partitions throttle regardless of table throughput.** Low-cardinality keys (single tenant, monotonic timestamp) concentrate load — fix with a sharded key suffix.
- **Cognito `custom:` attributes must be declared mutable at pool creation.**
- **ID token proves identity; access token proves API scope** — never authorize off the ID token. Pre-token-generation Lambda injects claims like `tenant_id`.
- **Cost Explorer lags ~24h; Budgets is near-real-time** — same-day guardrails need Budgets + SNS. Bedrock volume and Lambda concurrency are top silent-cost vectors.

## Production checklist
- [ ] IAM least-privilege; no `Resource: "*"` on write actions in prod
- [ ] Bedrock model access enabled per target region before deploy
- [ ] API Gateway throttling + usage plans set; WAF on public APIs
- [ ] WebSocket connections table has TTL and GoneException pruning
- [ ] DynamoDB access patterns documented before table design; PITR on
- [ ] Cognito MFA set; `tenant_id` injected via pre-token-generation trigger
- [ ] Budgets + SNS alert wired before Bedrock/Lambda production traffic
- [ ] `cdk diff` reviewed against live stack before every deploy

## Never
- Never grant broad `bedrock:*` — scope to `InvokeModel*` plus specific model ARNs.
- Never authorize off the ID token payload alone — verify signature, use access-token scopes.
- Never design a DynamoDB table before the access-pattern list is final.
- Never leave a WebSocket API without pruning dead connectionIds.
- Never deploy to a shared/prod account without reviewing `cdk diff` first.
