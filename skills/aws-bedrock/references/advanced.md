# Bedrock advanced / latest features (`aws bedrock`)

Newer capabilities not in core.md/agent.md. All take `--region`.

## Automated Reasoning policies (formal verification / hallucination guard)
Encode domain rules as a policy; Bedrock mathematically checks model outputs
against them (use inside a guardrail). Full lifecycle is CLI-driven:
```bash
aws bedrock create-automated-reasoning-policy --name refunds-policy \
  --policy-definition '{...rules/types/variables...}'
aws bedrock start-automated-reasoning-policy-build-workflow --policy-arn <arn> \
  --build-workflow-type INGEST_CONTENT --source-content '{...}'
aws bedrock get-automated-reasoning-policy-build-workflow --policy-arn <arn> --build-workflow-id <id>
aws bedrock create-automated-reasoning-policy-test-case --policy-arn <arn> --guard-content "..." --query-content "..."
aws bedrock start-automated-reasoning-policy-test-workflow --policy-arn <arn>
aws bedrock get-automated-reasoning-policy-test-result --policy-arn <arn> --build-workflow-id <id> --test-case-id <id>
aws bedrock create-automated-reasoning-policy-version --policy-arn <arn>
aws bedrock list-automated-reasoning-policies
# Attach to a guardrail via --automated-reasoning-policy-config on create-guardrail.
```

## Intelligent prompt routing (route cheap vs strong models automatically)
```bash
aws bedrock create-prompt-router --prompt-router-name cost-saver \
  --models '[{"modelArn":"<weak>"},{"modelArn":"<strong>"}]' \
  --routing-criteria '{"responseQualityDifference":0.3}' \
  --fallback-model '{"modelArn":"<strong>"}'
aws bedrock list-prompt-routers ; aws bedrock get-prompt-router --prompt-router-arn <arn>
# Invoke by passing the prompt-router ARN as the model-id to converse/invoke-model.
```

## Custom-model deployment (serve a fine-tuned/imported model on-demand)
```bash
aws bedrock create-custom-model --model-name my-claude --model-source-config '{...}'   # register weights
aws bedrock create-custom-model-deployment --model-deployment-name md1 --model-arn <custom-model-arn>
aws bedrock list-custom-model-deployments ; aws bedrock get-custom-model-deployment --custom-model-deployment-identifier <id>
# Then invoke the deployment ARN like any model id.
```

## Bedrock Marketplace model endpoints (3rd-party models from Marketplace)
```bash
aws bedrock create-marketplace-model-endpoint --model-source-identifier <arn> --endpoint-config '{...}' --endpoint-name ep1
aws bedrock register-marketplace-model-endpoint --endpoint-arn <arn> --model-source-identifier <id>
aws bedrock list-marketplace-model-endpoints ; aws bedrock deregister-marketplace-model-endpoint --endpoint-arn <arn>
```

## Resource policies (cross-account sharing of agents/KBs/profiles)
```bash
aws bedrock put-resource-policy --resource-arn <arn> --policy '{"Version":"2012-10-17","Statement":[...]}'
aws bedrock get-resource-policy --resource-arn <arn>
aws bedrock delete-resource-policy --resource-arn <arn>
```

## Enforced guardrails (org/account-level mandatory guardrail)
```bash
aws bedrock put-enforced-guardrail-configuration --guardrail-identifier <id> --guardrail-version <v>
aws bedrock list-enforced-guardrails-configuration ; aws bedrock delete-enforced-guardrail-configuration
```

## Lifecycle siblings you'll need (create-/list- shown in core.md/agent.md)
Every resource also has `get-`, `update-`, `delete-`, `stop-`, and
`list-tags-for-resource` / `tag-resource` / `untag-resource`:
- Guardrails: `update-guardrail`, `delete-guardrail`, `get-guardrail`, `list-guardrails`
- Customization: `get-model-customization-job`, `stop-model-customization-job`, `delete-custom-model`
- Batch: `get-model-invocation-job`, `stop-model-invocation-job`
- Evaluation: `get-evaluation-job`, `list-evaluation-jobs`, `stop-evaluation-job`, `batch-delete-evaluation-job`
- Provisioned throughput: `update-provisioned-model-throughput`, `delete-provisioned-model-throughput`
- Inference profiles: `delete-inference-profile`
- Imported/copied: `get-imported-model`, `list-imported-models`, `delete-imported-model`, `get-model-copy-job`, `get-model-import-job`
