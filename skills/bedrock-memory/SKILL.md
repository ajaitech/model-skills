---
name: bedrock-memory
description: Memory and conversation-state management for Amazon Bedrock agents — short-term session continuity (sessionId), long-term cross-session memory (SESSION_SUMMARY, memoryId), the persistent Sessions/Invocations API (create-session, invocation steps), session vs prompt attributes, idle TTL, retention, and context-window control. Use whenever a task involves remembering across turns or sessions, agent memory, conversation history, "memoryId", session state, or how to persist/retrieve/clear what an agent knows about a user.
---

# Bedrock agent memory & sessions

Three distinct layers — pick the right one:

| Layer | Scope | Mechanism |
|---|---|---|
| **Short-term** | within one conversation | reuse the same `sessionId` on each `invoke_agent` |
| **Long-term** | across sessions for a user | agent **memory** (`SESSION_SUMMARY`) keyed by `memoryId` |
| **Persistent store** | full conversation records you manage | **Sessions/Invocations API** |

## 1. Short-term: keep `sessionId` stable
Same `sessionId` → the agent keeps conversational context for that session
(bounded by `idleSessionTTLInSeconds`, set at create-agent). New topic → new
`sessionId`. `sessionId` min length is 2.

## 2. Long-term agent memory (cross-session summaries)
Enable on the agent, then pass a `memoryId` (e.g. the user/tenant id) per invoke:
```bash
aws bedrock-agent update-agent --agent-id <ID> --agent-name <n> --foundation-model <profile> \
  --agent-resource-role-arn <role> --instruction "..." \
  --memory-configuration '{"enabledMemoryTypes":["SESSION_SUMMARY"],"storageDays":30}'
aws bedrock-agent prepare-agent --agent-id <ID>
```
```python
# boto3 invoke with a stable memoryId → agent recalls prior sessions for that user
client.invoke_agent(agentId="<ID>", agentAliasId="<A>", sessionId="sess-7",
                    memoryId="user-acme-123", inputText="continue where we left off")
```
Read / clear it (these ARE real CLI commands):
```bash
aws bedrock-agent-runtime get-agent-memory   --agent-id <ID> --agent-alias-id <A> \
  --memory-type SESSION_SUMMARY --memory-id user-acme-123
aws bedrock-agent-runtime delete-agent-memory --agent-id <ID> --agent-alias-id <A> --memory-id user-acme-123
```

## 3. Persistent Sessions / Invocations API (you own the records)
For audit, replay, hand-off, or building your own memory on top:
```bash
S=$(aws bedrock-agent-runtime create-session --query sessionId --output text)
I=$(aws bedrock-agent-runtime create-invocation --session-identifier $S --query invocationId --output text)
aws bedrock-agent-runtime put-invocation-step --session-identifier $S --invocation-identifier $I \
  --invocation-step-time "$(date -u +%FT%TZ)" --payload '{"contentBlocks":[{"text":"user asked X; agent did Y"}]}'
aws bedrock-agent-runtime list-sessions
aws bedrock-agent-runtime list-invocation-steps --session-identifier $S --invocation-identifier $I
aws bedrock-agent-runtime end-session    --session-identifier $S
aws bedrock-agent-runtime delete-session --session-identifier $S   # GDPR / right-to-erasure
```

## 4. Per-turn state (not memory, but related)
`sessionState.sessionAttributes` (persist for the session, visible to action-group
Lambdas) and `promptSessionAttributes` (injected into the prompt this turn). Use for
tenantId, locale, user role — NOT for long-term facts.

## 5. Context-window control
- The model summarizes long sessions automatically with `SESSION_SUMMARY`; for very
  long chats, start a fresh `sessionId` and seed it from the summary/KB rather than
  replaying the whole transcript.
- For "remembered facts" that must be retrievable and grounded, store them in a
  **knowledge base** and `retrieve` them — that's durable, queryable, and citeable
  (see bedrock-multitenant-security for tenant-scoped retrieval).

## Multi-tenant memory (critical)
`memoryId` and the Sessions store MUST be namespaced by verified tenant
(e.g. `memoryId = "<tenantId>:<userId>"`), and `delete-agent-memory` /
`delete-session` must be reachable per tenant for erasure. Never let one tenant's
`memoryId` be guessable/usable by another. See bedrock-multitenant-security.
