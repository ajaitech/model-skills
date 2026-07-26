# Bedrock Agent reference — `aws bedrock-agent` + `aws bedrock-agent-runtime`

Build plane = `bedrock-agent` (create/configure). Data plane = `bedrock-agent-runtime`
(invoke/retrieve). All commands take `--region`. Agent edits target version `DRAFT`;
publish by `prepare-agent` then pointing an alias.

## 1. Agents (lifecycle)

```bash
# Service role first (trust bedrock.amazonaws.com). Then:
aws bedrock-agent create-agent --agent-name odin \
  --foundation-model us.anthropic.claude-3-7-sonnet-20250219-v1:0 \
  --instruction "You are OdiN. Emit json:local-action / json:ui-payload per contract." \
  --agent-resource-role-arn arn:aws:iam::<acct>:role/AmazonBedrockExecutionRoleForAgents_odin \
  --idle-session-ttl-in-seconds 600 \
  --memory-configuration '{"enabledMemoryTypes":["SESSION_SUMMARY"],"storageDays":30}'

aws bedrock-agent get-agent   --agent-id <ID>
aws bedrock-agent update-agent --agent-id <ID> --agent-name odin --foundation-model <profile> --instruction "..." --agent-resource-role-arn <role>
aws bedrock-agent list-agents --query 'agentSummaries[].{id:agentId,name:agentName,status:agentStatus}' --output table
aws bedrock-agent prepare-agent --agent-id <ID>          # REQUIRED after every change
aws bedrock-agent delete-agent  --agent-id <ID> --skip-resource-in-use-check
```

## 2. Aliases (what runtime calls)

```bash
aws bedrock-agent create-agent-alias --agent-id <ID> --agent-alias-name prod
aws bedrock-agent list-agent-aliases  --agent-id <ID>
aws bedrock-agent update-agent-alias  --agent-id <ID> --agent-alias-id <ALIAS> --agent-alias-name prod \
  --routing-configuration '[{"agentVersion":"3"}]'
```
`TSTALIASID` always points at the latest `DRAFT` for testing.

## 3. Action groups (tools the agent can call)

Executor is either a Lambda or `customControl: RETURN_CONTROL` (you handle the call).
Schema is either a typed `function-schema` or an OpenAPI `api-schema`.

```bash
# Function-schema style
aws bedrock-agent create-agent-action-group --agent-id <ID> --agent-version DRAFT \
  --action-group-name run-shell \
  --action-group-executor '{"lambda":"arn:aws:lambda:<r>:<acct>:function:agent-tools"}' \
  --function-schema '{"functions":[{"name":"run_command","description":"Run a shell command","parameters":{"command":{"type":"string","required":true}}}]}'

# OpenAPI style (api-schema from S3 or inline)
aws bedrock-agent create-agent-action-group --agent-id <ID> --agent-version DRAFT \
  --action-group-name api --action-group-executor '{"lambda":"<arn>"}' \
  --api-schema '{"s3":{"s3BucketName":"bkt","s3ObjectKey":"openapi.json"}}'

aws bedrock-agent list-agent-action-groups --agent-id <ID> --agent-version DRAFT
# Remember: lambda must allow bedrock to invoke it:
aws lambda add-permission --function-name agent-tools --statement-id bedrock --action lambda:InvokeFunction \
  --principal bedrock.amazonaws.com --source-arn arn:aws:bedrock:<r>:<acct>:agent/<ID>
```

## 4. Knowledge base (RAG)

```bash
# KB needs: a role, an embeddings model, and a vector store (e.g. OpenSearch Serverless)
aws bedrock-agent create-knowledge-base --name docs-kb --role-arn <kb-role> \
  --knowledge-base-configuration '{"type":"VECTOR","vectorKnowledgeBaseConfiguration":{"embeddingModelArn":"arn:aws:bedrock:<r>::foundation-model/amazon.titan-embed-text-v2:0"}}' \
  --storage-configuration '{"type":"OPENSEARCH_SERVERLESS","opensearchServerlessConfiguration":{"collectionArn":"<arn>","vectorIndexName":"idx","fieldMapping":{"vectorField":"vec","textField":"text","metadataField":"meta"}}}'

aws bedrock-agent create-data-source --knowledge-base-id <KB> --name s3-src \
  --data-source-configuration '{"type":"S3","s3Configuration":{"bucketArn":"arn:aws:s3:::docs-bkt"}}'
aws bedrock-agent start-ingestion-job --knowledge-base-id <KB> --data-source-id <DS>
aws bedrock-agent list-knowledge-bases ; aws bedrock-agent list-data-sources --knowledge-base-id <KB>

# Attach KB to an agent, then re-prepare
aws bedrock-agent associate-agent-knowledge-base --agent-id <ID> --agent-version DRAFT \
  --knowledge-base-id <KB> --description "product docs" --knowledge-base-state ENABLED

# Direct document ingestion (no S3 sync) + KB/data-source lifecycle
aws bedrock-agent ingest-knowledge-base-documents --knowledge-base-id <KB> --data-source-id <DS> \
  --documents '[{"content":{"dataSourceType":"CUSTOM","custom":{"customDocumentIdentifier":{"id":"doc1"},"sourceType":"IN_LINE","inlineContent":{"type":"TEXT","textContent":{"data":"..."}}}}}]'
aws bedrock-agent list-knowledge-base-documents --knowledge-base-id <KB> --data-source-id <DS>
aws bedrock-agent list-ingestion-jobs --knowledge-base-id <KB> --data-source-id <DS>   # watch IN_PROGRESS→COMPLETE
aws bedrock-agent update-knowledge-base ... ; aws bedrock-agent delete-knowledge-base --knowledge-base-id <KB>
# Agent/version/KB also have get-*, update-*, delete-*, list-agent-versions, disassociate-* siblings.
```

## 5. Multi-agent collaboration (supervisor → collaborators)

```bash
# Make an agent a supervisor, then attach collaborators (e.g. ArjunA → aicippy-io)
aws bedrock-agent update-agent --agent-id <SUP> --agent-collaboration SUPERVISOR ... 
aws bedrock-agent associate-agent-collaborator --agent-id <SUP> --agent-version DRAFT \
  --agent-descriptor '{"aliasArn":"arn:aws:bedrock:<r>:<acct>:agent-alias/<COLLAB_ID>/<ALIAS>"}' \
  --collaborator-name aicippy-io --collaboration-instruction "Delegate CLI/shell command generation."
aws bedrock-agent prepare-agent --agent-id <SUP>
```

## 6. Flows & prompt management

```bash
aws bedrock-agent create-prompt --name greet --variants '[{"name":"v1","templateType":"TEXT","templateConfiguration":{"text":{"text":"Hi {{name}}"}}}]'
aws bedrock-agent create-prompt-version --prompt-identifier <id>
aws bedrock-agent create-flow --name pipe --execution-role-arn <role> --definition '{"nodes":[...],"connections":[...]}'
aws bedrock-agent prepare-flow --flow-identifier <id>
aws bedrock-agent create-flow-alias --flow-identifier <id> --name prod --routing-configuration '[{"flowVersion":"1"}]'
```

## 7. Runtime (`aws bedrock-agent-runtime`)

**`invoke-agent`, `invoke-inline-agent`, `invoke-flow` are SDK-only (event
streams) — NOT CLI commands.** Use boto3 from the shell:
```bash
python3 - <<'PY'
import boto3
r = boto3.client("bedrock-agent-runtime", region_name="ap-south-1").invoke_agent(
    agentId="<ID>", agentAliasId="<ALIAS>", sessionId="sess-001", inputText="Reply OK",
    enableTrace=True,
    sessionState={"sessionAttributes":{"tenantId":"acme"},"promptSessionAttributes":{"locale":"en"}})
for e in r["completion"]:
    if "chunk" in e: print(e["chunk"]["bytes"].decode(), end="")
    if "trace" in e: pass  # inspect e["trace"] for tool/KB/reasoning steps
PY
```

These ARE real CLI commands:
```bash
# Non-streaming RAG
aws bedrock-agent-runtime retrieve --knowledge-base-id <KB> \
  --retrieval-query '{"text":"refund policy"}' \
  --retrieval-configuration '{"vectorSearchConfiguration":{"numberOfResults":5}}'
aws bedrock-agent-runtime retrieve-and-generate \
  --input '{"text":"What is the refund policy?"}' \
  --retrieve-and-generate-configuration '{"type":"KNOWLEDGE_BASE","knowledgeBaseConfiguration":{"knowledgeBaseId":"<KB>","modelArn":"arn:aws:bedrock:<r>::foundation-model/us.anthropic.claude-3-7-sonnet-20250219-v1:0"}}'

# Rerank retrieved passages by relevance
aws bedrock-agent-runtime rerank --queries '[{"type":"TEXT","textQuery":{"text":"refunds"}}]' \
  --sources '[{"type":"INLINE","inlineDocumentSource":{"type":"TEXT","textDocument":{"text":"..."}}}]' \
  --reranking-configuration '{"type":"BEDROCK_RERANKING_MODEL","bedrockRerankingConfiguration":{"modelConfiguration":{"modelArn":"arn:aws:bedrock:<r>::foundation-model/amazon.rerank-v1:0"}}}'

# NL → query (structured / SQL knowledge bases)
aws bedrock-agent-runtime generate-query --query-generation-input '{"type":"TEXT","text":"top 10 customers by revenue"}' \
  --transformation-configuration '{"mode":"TEXT_TO_SQL","textToSqlConfiguration":{...}}'
```

### Persistent sessions & agent memory (the new memory API)
```bash
aws bedrock-agent-runtime create-session                         # → sessionId/sessionArn
aws bedrock-agent-runtime create-invocation --session-identifier <s>
aws bedrock-agent-runtime put-invocation-step --session-identifier <s> --invocation-identifier <i> \
  --invocation-step-time <ts> --payload '{"contentBlocks":[{"text":"..."}]}'
aws bedrock-agent-runtime list-sessions ; aws bedrock-agent-runtime list-invocation-steps --session-identifier <s> --invocation-identifier <i>
aws bedrock-agent-runtime end-session --session-identifier <s> ; aws bedrock-agent-runtime delete-session --session-identifier <s>
# Long-term agent memory (SESSION_SUMMARY) per memoryId:
aws bedrock-agent-runtime get-agent-memory --agent-id <ID> --agent-alias-id <ALIAS> --memory-type SESSION_SUMMARY --memory-id <m>
aws bedrock-agent-runtime delete-agent-memory --agent-id <ID> --agent-alias-id <ALIAS> --memory-id <m>
```

### Async flow execution
```bash
aws bedrock-agent-runtime start-flow-execution --flow-identifier <id> --flow-alias-identifier <alias> --inputs '[...]'
aws bedrock-agent-runtime list-flow-executions --flow-identifier <id> --flow-alias-identifier <alias>
aws bedrock-agent-runtime get-flow-execution --flow-identifier <id> --flow-alias-identifier <alias> --execution-identifier <e>
aws bedrock-agent-runtime stop-flow-execution --flow-identifier <id> --flow-alias-identifier <alias> --execution-identifier <e>
```

## End-to-end agent (copy-paste order)
1. Create the **service role** (trust `bedrock.amazonaws.com`; perms `bedrock:InvokeModel*` on the model/inference-profile, `bedrock:Retrieve` on KBs, `lambda:InvokeFunction`).
2. `create-agent` (with the inference-profile id, not the bare model).
3. `create-agent-action-group` (+ `lambda add-permission` so Bedrock can call the Lambda).
4. (optional) `create-knowledge-base` → `create-data-source` → `start-ingestion-job` → `associate-agent-knowledge-base`.
5. `prepare-agent`.
6. `create-agent-alias`.
7. Smoke-test via the boto3 `invoke_agent` snippet (§7) with `enableTrace=True`; read the trace to confirm tool/KB calls fire. (No `aws` CLI invoke-agent exists.)

## Debugging
| Symptom | Fix |
|---|---|
| `accessDeniedException` on `invoke-agent` | caller lacks `bedrock:InvokeAgent` on the alias ARN; OR agent role lacks `bedrock:InvokeModel*` on the model/profile (see core.md) |
| agent answers but never calls the tool | action group not on the served version → `prepare-agent` + check alias `routing-configuration`; verify the Lambda `add-permission` |
| `…on-demand throughput isn't supported` | `--foundation-model` must be the inference-profile id, not the bare model |
| KB returns nothing | ingestion job still `IN_PROGRESS` (`list-ingestion-jobs`) or empty index; re-run `start-ingestion-job` |
| `ValidationException` sessionId | min length 2 |
| supervisor ignores collaborator | `agent-collaboration SUPERVISOR` not set, or collaborator not `prepare`d/aliased |
