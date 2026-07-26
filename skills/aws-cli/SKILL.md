---
name: aws-cli
description: Use when discovering, executing, debugging, or verifying AWS CLI operations for any AWS account, service, resource, deployment, IAM, logs, networking, storage, database, serverless, container, or Bedrock task.
---

# AWS CLI — direct execution

**REQUIRED LIVE REFERENCE:** Use `live-official-docs` for current AWS documentation, then verify syntax with the installed CLI's exact `aws <service> <command> help`.

You execute real AWS operations through the `aws` CLI. The CLI is the source of
truth — **never assume a resource, permission, route, or value; read it first.**
For Bedrock specifically, defer to the `aws-bedrock` skill.

## Execution discipline (non-negotiable)
1. **Region + identity first:** `aws sts get-caller-identity` and pass `--region`
   on every command (resources are regional).
2. **Read before write:** before any `create/update/put/delete`, run the matching
   `get-/describe-/list-` to see current state. After a mutation, **re-read and
   quote the real output** — never report success without it.
3. **Tight output:** `--query '<JMESPath>'` + `--output table|text`; set
   `export AWS_PAGER=""` so commands don't block on a pager.
4. **Destructive ops** (`delete-*`, `remove-permission`, IAM detach, `put-bucket-policy`,
   `s3 rm --recursive`): show the exact command and the blast radius, confirm intent, then run.
5. **Idempotency / dry-run:** use `--dry-run` where supported (EC2), `--cli-input-json`
   for repeatable calls, and `change-set` flows for CloudFormation.

## Service quick-map (the commands you reach for most)

```bash
export AWS_PAGER=""
# Identity / account
aws sts get-caller-identity ; aws sts assume-role --role-arn <arn> --role-session-name s1

# Lambda — code + config + permissions
aws lambda get-function --function-name <fn>
aws lambda update-function-code --function-name <fn> --zip-file fileb://fn.zip
aws lambda update-function-configuration --function-name <fn> --environment 'Variables={K=V}' --timeout 60 --memory-size 512
aws lambda add-permission --function-name <fn> --statement-id <id> --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com --source-arn <arn>   # (or bedrock.amazonaws.com for agents)
aws lambda invoke --function-name <fn> --payload '{"k":"v"}' --cli-binary-format raw-in-base64-out out.json

# API Gateway HTTP API (v2) — routes / integrations / authorizers / CORS
aws apigatewayv2 get-apis ; aws apigatewayv2 get-routes --api-id <id> ; aws apigatewayv2 get-integrations --api-id <id>
aws apigatewayv2 get-authorizers --api-id <id>
aws apigatewayv2 update-api --api-id <id> --cors-configuration AllowOrigins=...,AllowMethods=GET,POST,OPTIONS,AllowHeaders=...,AllowCredentials=true
aws apigatewayv2 create-route --api-id <id> --route-key 'OPTIONS /chat' --target integrations/<intId>   # public preflight
aws apigatewayv2 update-route --api-id <id> --route-id <rid> --authorization-type JWT --authorizer-id <aid>   # POST only
# REST API (v1): aws apigateway get-rest-apis / get-resources / put-method / put-integration / create-deployment

# IAM — roles / policies (the usual fix for AccessDenied)
aws iam get-role --role-name <r> ; aws iam list-attached-role-policies --role-name <r>
aws iam put-role-policy --role-name <r> --policy-name inline --policy-document file://policy.json
aws iam update-assume-role-policy --role-name <r> --policy-document file://trust.json
aws iam simulate-principal-policy --policy-source-arn <roleArn> --action-names bedrock:InvokeModel --resource-arns '*'

# Cognito (user pools) — used for the gateway JWT auth
aws cognito-idp describe-user-pool --user-pool-id <pool>
aws cognito-idp list-user-pool-clients --user-pool-id <pool>
aws cognito-idp admin-get-user --user-pool-id <pool> --username <u>

# Secrets Manager (NEVER print full secret values into logs/transcripts)
aws secretsmanager list-secrets --query 'SecretList[].Name'
aws secretsmanager get-secret-value --secret-id <id> --query 'SecretString' --output text   # handle, don't echo

# CloudWatch Logs — the first stop when a Lambda 500s
aws logs tail /aws/lambda/<fn> --since 15m --follow
aws logs filter-log-events --log-group-name /aws/lambda/<fn> --filter-pattern '"ERROR"'

# S3 / CloudFront / Route53 / SES / STS / DynamoDB / SQS / EventBridge — discover with:
aws <service> help | col -b | sed -n '/AVAILABLE COMMANDS/,$p' | grep -E '^ +[a-z]'   # list real subcommands
```

## Discovering the exact command surface (don't guess)
Any service: `aws <service> help` (commands), `aws <service> <cmd> help` (args).
List subcommands programmatically:
```bash
AWS_PAGER="" aws <service> help | col -b | awk '/^AVAILABLE COMMANDS/{f=1;next}/^[A-Z]/{f=0}f&&/^ +o /{print $2}'
```
**Streaming/event-stream APIs (e.g. `bedrock-agent-runtime invoke-agent`, `converse-stream`)
are NOT in the AWS CLI — use a boto3 one-liner for those** (see aws-bedrock skill).

## Output → fix (common)
- `AccessDenied` → `aws iam simulate-principal-policy` to find the missing action; add it with `put-role-policy`.
- `ResourceNotFoundException` → wrong region/id; re-`list-`.
- `ThrottlingException` → backoff; check Service Quotas.
- CLI v2 blob/body errors → add `--cli-binary-format raw-in-base64-out`.
