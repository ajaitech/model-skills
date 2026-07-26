# Bedrock Core reference — `aws bedrock` + `aws bedrock-runtime`

All commands take `--region <r>`. Use `--query` (JMESPath) to trim output.

## 1. Model & inference-profile discovery (`aws bedrock`)

```bash
# Foundation models (filter by provider / modality / inference type)
aws bedrock list-foundation-models \
  --by-provider anthropic \
  --by-output-modality TEXT \
  --by-inference-type ON_DEMAND \
  --query 'modelSummaries[].{id:modelId,name:modelName,streaming:responseStreamingSupported}' --output table
aws bedrock get-foundation-model --model-identifier anthropic.claude-3-7-sonnet-20250219-v1:0

# Cross-region inference profiles (REQUIRED for most current models)
aws bedrock list-inference-profiles --type-equals SYSTEM_DEFINED
aws bedrock get-inference-profile --inference-profile-identifier us.anthropic.claude-3-7-sonnet-20250219-v1:0
# Application inference profile (your own, for cost-tracking tags / dedicated routing)
aws bedrock create-inference-profile --inference-profile-name prod-claude \
  --model-source '{"copyFrom":"arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-7-sonnet-20250219-v1:0"}' \
  --tags '[{"key":"team","value":"platform"}]'
```
Profile id prefixes by geography: `us.` `eu.` `apac.` — pick the one matching your region group.

## 2. Invocation (`aws bedrock-runtime`)

**Converse (model-agnostic, preferred — same shape for all providers):**
```bash
aws bedrock-runtime converse \
  --model-id us.anthropic.claude-3-7-sonnet-20250219-v1:0 \
  --messages '[{"role":"user","content":[{"text":"Summarize Bedrock in one line."}]}]' \
  --system '[{"text":"You are concise."}]' \
  --inference-config '{"maxTokens":512,"temperature":0.2,"topP":0.9}' \
  --query 'output.message.content[0].text' --output text

# NOTE: converse-stream / invoke-model-with-response-stream are SDK-only
# (event streams) — NOT aws CLI commands. For streaming from a shell use boto3.

# Token counting (estimate before sending)
aws bedrock-runtime count-tokens --model-id us.anthropic.claude-3-7-sonnet-20250219-v1:0 \
  --input '{"converse":{"messages":[{"role":"user","content":[{"text":"hello"}]}]}}'

# Async / long-running invocation (e.g. video, large gen) → results to S3
aws bedrock-runtime start-async-invoke --model-id <model> \
  --model-input '{...}' --output-data-config '{"s3OutputDataConfig":{"s3Uri":"s3://bkt/out/"}}'
aws bedrock-runtime list-async-invokes ; aws bedrock-runtime get-async-invoke --invocation-arn <arn>

# Tool use (function calling) via converse
aws bedrock-runtime converse --model-id us.anthropic.claude-3-7-sonnet-20250219-v1:0 \
  --messages '[{"role":"user","content":[{"text":"weather in Mumbai?"}]}]' \
  --tool-config '{"tools":[{"toolSpec":{"name":"get_weather","description":"...","inputSchema":{"json":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}}}}]}'
```

**invoke-model (raw, provider-specific body):**
```bash
aws bedrock-runtime invoke-model \
  --model-id us.anthropic.claude-3-7-sonnet-20250219-v1:0 \
  --content-type application/json --accept application/json \
  --body '{"anthropic_version":"bedrock-2023-05-31","max_tokens":256,"messages":[{"role":"user","content":"Reply OK"}]}' \
  /dev/stdout
```
Note: `--body` must be JSON bytes; with AWS CLI v2 add `--cli-binary-format raw-in-base64-out` if it complains about the blob. The streaming variant `invoke-model-with-response-stream` is SDK-only (not in the CLI).

## Model access via CLI (you do NOT need the console)

```bash
aws bedrock get-foundation-model-availability --model-id anthropic.claude-3-7-sonnet-20250219-v1:0
aws bedrock list-foundation-model-agreement-offers --model-id anthropic.claude-3-7-sonnet-20250219-v1:0
aws bedrock create-foundation-model-agreement --model-id <id> --offer-token <token-from-offers>
aws bedrock put-use-case-for-model-access --form-data <base64-eula-form>   # some 3rd-party models
aws bedrock delete-foundation-model-agreement --model-id <id>              # revoke
```

## 3. Guardrails

```bash
aws bedrock create-guardrail --name pii-guard \
  --blocked-input-messaging "Blocked." --blocked-outputs-messaging "Blocked." \
  --content-policy-config '{"filtersConfig":[{"type":"HATE","inputStrength":"HIGH","outputStrength":"HIGH"}]}' \
  --sensitive-information-policy-config '{"piiEntitiesConfig":[{"type":"EMAIL","action":"ANONYMIZE"}]}'
aws bedrock create-guardrail-version --guardrail-identifier <id>
aws bedrock list-guardrails
# Apply at inference time (independent of the model call)
aws bedrock-runtime apply-guardrail --guardrail-identifier <id> --guardrail-version 1 \
  --source INPUT --content '[{"text":{"text":"my email is x@y.com"}}]'
```
You can also pass `--guardrail-config` to `converse`/`invoke-model`.

## 4. Customization, throughput, batch, evaluation

```bash
# Fine-tuning / continued pre-training
aws bedrock create-model-customization-job --job-name ft1 --custom-model-name my-claude \
  --role-arn <role> --base-model-identifier <fm> --customization-type FINE_TUNING \
  --training-data-config '{"s3Uri":"s3://bkt/train.jsonl"}' --output-data-config '{"s3Uri":"s3://bkt/out/"}' \
  --hyper-parameters '{"epochCount":"1"}'
aws bedrock list-model-customization-jobs ; aws bedrock list-custom-models

# Provisioned throughput (dedicated capacity for a model/custom model)
aws bedrock create-provisioned-model-throughput --model-units 1 \
  --provisioned-model-name pt1 --model-id <model-or-custom-arn>
aws bedrock list-provisioned-model-throughputs

# Batch / async inference over S3
aws bedrock create-model-invocation-job --job-name batch1 --role-arn <role> \
  --model-id us.anthropic.claude-3-7-sonnet-20250219-v1:0 \
  --input-data-config '{"s3InputDataConfig":{"s3Uri":"s3://bkt/in/"}}' \
  --output-data-config '{"s3OutputDataConfig":{"s3Uri":"s3://bkt/out/"}}'
aws bedrock list-model-invocation-jobs

# Evaluation
aws bedrock create-evaluation-job --job-name eval1 --role-arn <role> \
  --evaluation-config '{...}' --inference-config '{...}' --output-data-config '{"s3Uri":"s3://bkt/eval/"}'

# Imported / copied models
aws bedrock create-model-import-job ...   # bring your own weights
aws bedrock create-model-copy-job ...     # copy across regions/accounts
```

## 5. Observability

```bash
# Turn on invocation logging (CloudWatch + S3)
aws bedrock put-model-invocation-logging-configuration --logging-config \
  '{"cloudWatchConfig":{"logGroupName":"/bedrock/invocations","roleArn":"<role>"},"textDataDeliveryEnabled":true}'
aws bedrock get-model-invocation-logging-configuration
```

## 6. Debugging (map error → fix)

| Error | Cause | Fix |
|---|---|---|
| `…on-demand throughput isn't supported. Retry … with an inference profile` | bare model id for a profile-only model | use `us.anthropic.…` inference-profile id/ARN |
| `AccessDeniedException` on InvokeModel | (a) model access not enabled, or (b) IAM | (a) console → Model access; (b) add `bedrock:InvokeModel*` on model + profile ARN |
| `ValidationException: …model identifier is invalid` | wrong id / region | `list-foundation-models` / `list-inference-profiles` in that region |
| `ThrottlingException` | rate/quota | exponential backoff; or `create-provisioned-model-throughput` |
| `ServiceQuotaExceededException` | account limit | request a quota increase (Service Quotas: "Amazon Bedrock") |
| blob/body errors (CLI v2) | binary format | add `--cli-binary-format raw-in-base64-out` |

Quick IAM sanity: `aws sts get-caller-identity` then `aws bedrock-runtime converse --model-id <profile> --messages '[{"role":"user","content":[{"text":"ok"}]}]'` — a 200 proves access end-to-end.
