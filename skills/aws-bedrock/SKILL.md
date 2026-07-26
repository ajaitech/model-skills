---
name: aws-bedrock
description: Use when listing, invoking, building, deploying, debugging, reviewing, or securing Amazon Bedrock models, runtime APIs, inference profiles, guardrails, agents, knowledge bases, flows, prompts, evaluations, or AWS CLI operations.
---

# AWS Bedrock via AWS CLI — Agent + Core

**REQUIRED LIVE REFERENCE:** Use `live-official-docs` for current AWS documentation and the installed SDK or CLI reference before choosing Bedrock syntax, model identifiers, regions, or capabilities.

You are an expert AWS Bedrock operator working through the `aws` CLI. This skill
gives you the command surface, the non-obvious operational rules, and the
debugging playbook. **Verify every claim with a real `aws` call — never assume a
resource or permission exists.**

## The four CLI namespaces (know which one you need)

| `aws <service>` | Plane | Use for |
|---|---|---|
| `bedrock` | Core control plane | foundation models, inference profiles, guardrails, customization, batch, provisioned throughput, logging, evaluation |
| `bedrock-runtime` | Core data plane | `invoke-model`, `converse`, `*-stream`, `apply-guardrail` |
| `bedrock-agent` | Agent build plane | agents, aliases, action groups, knowledge bases, data sources, ingestion, collaborators, flows, prompts |
| `bedrock-agent-runtime` | Agent data plane | `invoke-agent`, `retrieve`, `retrieve-and-generate`, `invoke-flow`, `invoke-inline-agent` |

Full command catalog with arguments: **`references/core.md`** (bedrock + bedrock-runtime)
and **`references/agent.md`** (bedrock-agent + bedrock-agent-runtime). Read the
relevant reference file before composing a multi-step deploy.

## Non-negotiable operational rules (these cause most failures)

1. **Newer models REQUIRE a cross-region inference profile, not the bare model id.**
   Invoking `anthropic.claude-3-7-sonnet-20250219-v1:0` directly fails with
   *"Invocation … with on-demand throughput isn't supported. Retry … with an
   inference profile."* Use the system-defined profile id/ARN instead:
   `us.anthropic.claude-3-7-sonnet-20250219-v1:0` (or `eu.` / `apac.` by geo).
   For agents, set `--foundation-model` to that profile id.
   - Discover: `aws bedrock list-inference-profiles --type-equals SYSTEM_DEFINED`
   - On-demand-capable models only: `aws bedrock list-foundation-models --by-inference-type ON_DEMAND`

2. **`AccessDeniedException` has two distinct causes — check both:**
   - **Model access not granted** in the account/region. This CAN be done via CLI:
     `aws bedrock list-foundation-model-agreement-offers --model-id <id>` →
     `aws bedrock create-foundation-model-agreement --model-id <id> --offer-token <t>`
     (some models also need `put-use-case-for-model-access`). Check status with
     `aws bedrock get-foundation-model-availability --model-id <id>`.
   - **IAM** → the caller/role lacks `bedrock:InvokeModel*`, `bedrock:InvokeAgent`, or `bedrock-agent-runtime:*`. See the IAM block below.

3. **Region matters and resources are region-scoped.** Agents, KBs, aliases live
   in one region. A cross-region inference profile lets the *model* span regions,
   but the agent itself is regional. Always pass `--region`.

4. **An agent must be `prepare`d after every change** before an alias serves it:
   `aws bedrock-agent prepare-agent --agent-id <id>` → then point/refresh the alias.

5. **`sessionId` min length is 2** for `invoke-agent` (a 1-char id throws
   ValidationException).

## IAM you almost always need

- **Caller/Lambda invoking models:** `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`, `bedrock:Converse`, `bedrock:ConverseStream` on `arn:aws:bedrock:*::foundation-model/*` and the inference-profile ARN (`arn:aws:bedrock:<region>:<acct>:inference-profile/us.anthropic…`).
- **Caller invoking an agent:** `bedrock:InvokeAgent` on `arn:aws:bedrock:<region>:<acct>:agent-alias/<agentId>/<aliasId>`; for KB calls `bedrock:Retrieve` / `bedrock:RetrieveAndGenerate`.
- **Agent service role** (trust `bedrock.amazonaws.com`, condition `aws:SourceAccount`): `bedrock:InvokeModel*` on the model/inference-profile; `bedrock:Retrieve` on associated KBs; `lambda:InvokeFunction` for action groups.
- **Knowledge-base role:** read the S3 data source, `bedrock:InvokeModel` on the embeddings model, and access to the vector store (e.g. `aoss:APIAccessAll` for OpenSearch Serverless).
- **Cross-account model access:** do NOT store IAM keys — have the caller role `sts:AssumeRole` a `bedrock-invoker` role in the model account.

## Most-used commands (memorize these; full set in references/)

```bash
# CORE — discover + invoke
aws bedrock list-foundation-models --by-provider anthropic --query 'modelSummaries[].modelId'
aws bedrock list-inference-profiles --type-equals SYSTEM_DEFINED --query 'inferenceProfileSummaries[].inferenceProfileId'
aws bedrock-runtime converse --model-id us.anthropic.claude-3-7-sonnet-20250219-v1:0 \
  --messages '[{"role":"user","content":[{"text":"Reply OK"}]}]' \
  --inference-config '{"maxTokens":256,"temperature":0.2}'

# AGENT — list + prepare
aws bedrock-agent list-agents --query 'agentSummaries[].{id:agentId,name:agentName,status:agentStatus}'
aws bedrock-agent prepare-agent --agent-id <ID>
```

## CRITICAL: streaming APIs are NOT in the AWS CLI (use boto3)

`invoke-agent`, `invoke-flow`, `invoke-inline-agent`, `converse-stream`,
`invoke-model-with-response-stream`, and `retrieve-and-generate-stream` return
event streams and are **not** exposed as `aws` CLI commands (you get
`Found invalid choice`). To invoke an agent from a shell, use a boto3 one-liner:

```bash
python3 - <<'PY'
import boto3
r = boto3.client("bedrock-agent-runtime", region_name="ap-south-1").invoke_agent(
    agentId="<ID>", agentAliasId="<ALIAS>", sessionId="smoke-01",
    inputText="Reply OK", enableTrace=True)
print("".join(e["chunk"]["bytes"].decode() for e in r["completion"] if "chunk" in e))
PY
```
Non-streaming RAG (`retrieve`, `retrieve-and-generate`) and `converse` / `invoke-model`
ARE real CLI commands. See `references/agent.md` for the boto3 patterns.

## Workflows
- **Build an agent end-to-end** (role → create-agent → action group → KB → prepare → alias → invoke): `references/agent.md` § "End-to-end agent".
- **Stand up a knowledge base** (role → KB → data source → ingestion → retrieve): `references/agent.md` § "Knowledge base".
- **Invoke a model, add a guardrail, batch-infer, fine-tune:** `references/core.md`.
- **Debug AccessDenied / inference-profile / throttling:** `references/core.md` § "Debugging" and rule 1–2 above.
- **Latest/advanced features** (Automated Reasoning policy checks, persistent Sessions/memory API, Rerank, generate-query/NL→SQL, async invoke, prompt routers, marketplace + custom-model deployment, resource policies, enforced guardrails, flow executions): `references/advanced.md`.

## Discipline
- Prefer `--query` (JMESPath) + `--output table/text` to keep output tight.
- For anything that mutates (create/update/delete/prepare), show the exact command, run it, then **verify** with the matching `get-`/`list-` call. Quote the real output — never report success without it.
- Destructive ops (`delete-agent`, `delete-knowledge-base`, IAM changes): confirm intent first.
