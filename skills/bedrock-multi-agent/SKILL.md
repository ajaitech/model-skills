---
name: bedrock-multi-agent
description: Configure Amazon Bedrock multi-agent collaboration — a supervisor agent that orchestrates specialist collaborator agents (routing, delegation, conversation-history relay). Use when building or debugging supervisor/collaborator topologies, choosing SUPERVISOR vs SUPERVISOR_ROUTER, wiring associate-agent-collaborator, passing tenant/session state across agents, or composing several specialist agents (e.g. ArjunA → aicippy-io, or invoices/ledger/inventory agents) into one orchestrated system.
---

# Bedrock multi-agent collaboration

One **supervisor** agent receives the user turn and delegates to **collaborator**
agents (each a normal, prepared, aliased agent specialized to a domain). Verified
against the CLI — the knobs are real.

## Two supervisor modes (`--agent-collaboration`)
- **`SUPERVISOR`** — the supervisor reasons, may call MULTIPLE collaborators per
  turn, synthesizes a combined answer. Use for complex tasks needing several
  specialists (the powerful, slightly slower mode).
- **`SUPERVISOR_ROUTER`** — the supervisor just ROUTES the turn to the single best
  collaborator (intent classification). Use for "pick the right specialist" with
  lower latency/cost. Falls back to full supervision when intent is ambiguous.
- `DISABLED` (default) = a normal single agent.

## Wire it (order matters)
```bash
# 1. Each collaborator is a normal agent: create-agent → action groups/KB → prepare-agent → create-agent-alias
# 2. Turn the orchestrator into a supervisor (on DRAFT):
aws bedrock-agent update-agent --agent-id <SUP_ID> --agent-name orchestrator \
  --foundation-model us.anthropic.claude-3-7-sonnet-20250219-v1:0 \
  --agent-resource-role-arn <role> --instruction "Route/decompose to specialists; never answer ERP facts yourself." \
  --agent-collaboration SUPERVISOR

# 3. Attach each collaborator (by its ALIAS arn — collaborators must be prepared+aliased)
aws bedrock-agent associate-agent-collaborator --agent-id <SUP_ID> --agent-version DRAFT \
  --agent-descriptor '{"aliasArn":"arn:aws:bedrock:<r>:<acct>:agent-alias/<COLLAB_ID>/<ALIAS>"}' \
  --collaborator-name invoices \
  --collaboration-instruction "Use for invoice/AR questions. Always tenant-scope your queries." \
  --relay-conversation-history TO_COLLABORATOR     # or DISABLED — controls if the collaborator sees prior turns

# 4. Repeat for each specialist, then:
aws bedrock-agent prepare-agent --agent-id <SUP_ID>
aws bedrock-agent create-agent-alias --agent-id <SUP_ID> --agent-alias-name prod
# Manage: list-agent-collaborators / get-agent-collaborator / update-agent-collaborator / disassociate-agent-collaborator
```

## Invoke (the user only calls the supervisor)
`invoke_agent` is SDK-only (event stream) — boto3:
```python
r = client.invoke_agent(agentId="<SUP_ID>", agentAliasId="<ALIAS>", sessionId="s1",
        inputText="overdue invoices over 5 lakh and current cash position",
        enableTrace=True,
        sessionState={"sessionAttributes":{"tenantId":"acme"}})
# The trace shows the routing/delegation: which collaborator(s) fired and their sub-traces.
```

## Design rules
- **Collaboration-instruction is the routing signal** — write each as a crisp
  "use this collaborator WHEN …". Vague instructions → bad routing.
- **Specialists stay specialized** — one domain + its own action groups/KB each;
  don't give a collaborator the whole world.
- **Tenant/session state flows down**: `sessionAttributes` (e.g. tenantId) propagate
  to collaborators and their action-group Lambdas — enforce isolation at every hop
  (see bedrock-multitenant-security).
- **`relay-conversation-history`**: `TO_COLLABORATOR` when context matters; `DISABLED`
  to keep specialists stateless/cheaper.
- **Depth**: collaborators can themselves be supervisors (hierarchies) — keep it
  shallow (1–2 levels) for latency and debuggability.
- **IAM**: the supervisor role needs `bedrock:InvokeAgent`/`GetAgentAlias` on each
  collaborator alias; each collaborator role needs its own model/KB/Lambda perms.

## Debug
- Supervisor ignores a collaborator → it wasn't `prepare`d/aliased, the
  collaboration-instruction is too vague, or the supervisor wasn't re-`prepare`d
  after associating. Read the `--enable-trace` orchestration trace to see routing.
