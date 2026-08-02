# Backend: Python + AWS CDK

## Applies when
`cdk.json` exists AND a `requirements.txt` or `pyproject.toml` exists.

## Authoritative sources
| Source | URL |
|---|---|
| AWS CDK docs | https://docs.aws.amazon.com/cdk |
| AWS CDK repo | https://github.com/aws/aws-cdk |
| Python docs | https://docs.python.org |
| PyPI | https://pypi.org |

## Non-obvious rules
- CDK v2 collapses every service construct into one `aws-cdk-lib` package plus `constructs` — the v1 pattern of per-service packages (`@aws-cdk/aws-s3`, etc.) is end-of-life. A `requirements.txt` pinning individual `aws-cdk.aws_*` packages signals a stale v1 project needing migration, not a style choice.
- Lambda layers built with compiled dependencies (psycopg2, numpy, pillow, etc.) must be built for the Lambda runtime's target architecture (x86_64 or arm64/Graviton). Running `pip install` on a Mac (especially Apple Silicon) and zipping the result produces binaries incompatible with the Lambda runtime — build inside Docker matching the target, or use CDK's bundling options with a Lambda-compatible image.
- `BundlingOptions` with a Docker image gives reproducible, architecture-correct builds; local (non-Docker) bundling silently depends on the host machine's Python version and OS matching the Lambda runtime closely enough to work — treat local bundling as dev-only, not for production deploys.
- `cdk.json` context values cache into `cdk.context.json` (AZ lookups, VPC lookups, SSM lookups). A stale cached value causes deploy-time drift from reality with no warning — decide deliberately whether to commit this file, and refresh it (`cdk context --clear`) after any account/region-dependent infra change.
- Convenience grant helpers (`bucket.grantReadWrite(fn)`, etc.) often grant a broader IAM action set than the name implies. Run `cdk synth` and read the generated policy JSON before assuming the grant is minimal — least privilege is not the default output, it's a reviewed one.
- Cross-account or cross-region stack references require explicit `env: { account, region }` on each stack — CDK does not infer cross-account references automatically, and a missing `env` fails at synth or deploy with an unrelated-looking error.
- Many L2 constructs default `RemovalPolicy` to `DESTROY` for convenience in early development — stateful resources (databases, buckets with real data) need this set explicitly to `RETAIN` before production deploy, or a stack deletion silently takes the data with it.

## Production checklist
- [ ] `cdk synth` produces no unresolved tokens or synth-time warnings
- [ ] `cdk diff` reviewed by a human before every production deploy — no blind `--require-approval never` in the prod pipeline
- [ ] Lambda layers rebuilt for the exact runtime architecture in use (verify, don't assume x86_64)
- [ ] IAM policies scoped to specific resource ARNs; wildcard `Resource: "*"` justified in a comment if present at all
- [ ] Secrets sourced from Secrets Manager or SSM Parameter Store — never plaintext in `environment:` blocks
- [ ] `RemovalPolicy` explicitly set (not left to L2 defaults) on every stateful resource
- [ ] CloudWatch log retention set explicitly (default is indefinite and cost-accumulating)

## Never
- Never commit `cdk.context.json` or `cdk.out` if either contains account-specific identifiers treated as sensitive.
- Never leave a wildcard `Resource: "*"` IAM policy in a production stack without explicit justification.
- Never ship a Lambda layer built on a different OS/architecture than the target runtime without verifying compatibility.
- Never leave CloudWatch log groups at default (unlimited) retention on high-volume functions.
