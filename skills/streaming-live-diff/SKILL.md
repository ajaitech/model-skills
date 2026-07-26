---
name: streaming-live-diff
description: Consume Amazon Bedrock streaming responses and render them live — token-by-token conversational streaming (ConverseStream, InvokeModelWithResponseStream), agent event streams with traces and tool-use (InvokeAgent), and rendering proposed file changes as live unified diffs (before/after). Use whenever a task needs real-time streamed output, parsing Bedrock event streams in boto3, handling tool/trace/citation events as they arrive, or showing the user live file-change diffs from an agent's edits.
---

# Streaming Bedrock + live diffs

**These are SDK-only — there is no `aws` CLI for streaming** (`converse-stream`,
`invoke-model-with-response-stream`, `invoke-agent`, `invoke-flow` all return event
streams). Stream from boto3; render each event as it arrives.

## 1. Conversational token streaming (ConverseStream)
```python
import boto3
br = boto3.client("bedrock-runtime", region_name="ap-south-1")
resp = br.converse_stream(
    modelId="us.anthropic.claude-3-7-sonnet-20250219-v1:0",
    messages=[{"role":"user","content":[{"text":"Stream a haiku."}]}])
for ev in resp["stream"]:
    if "contentBlockDelta" in ev:
        print(ev["contentBlockDelta"]["delta"].get("text",""), end="", flush=True)  # live tokens
    elif "messageStop" in ev:
        print()  # turn complete; ev has stopReason
    elif "metadata" in ev:
        usage = ev["metadata"]["usage"]   # tokens in/out
```
Raw bytes variant: `invoke_model_with_response_stream` → iterate
`resp["body"]` events, `json.loads(e["chunk"]["bytes"])`.

## 2. Agent event stream (InvokeAgent) — text + trace + tool calls live
```python
ar = boto3.client("bedrock-agent-runtime", region_name="ap-south-1")
r = ar.invoke_agent(agentId="<ID>", agentAliasId="<A>", sessionId="s1",
                    inputText="...", enableTrace=True)
for ev in r["completion"]:
    if "chunk" in ev:
        print(ev["chunk"]["bytes"].decode(), end="", flush=True)        # streamed answer
    elif "trace" in ev:
        t = ev["trace"]["trace"]                                        # reasoning/tool/KB steps
        # orchestrationTrace → invocationInput (tool call) / observation (tool result)
    elif "returnControl" in ev:
        pass  # RETURN_CONTROL: you execute the tool, then continue the session
    elif "files" in ev:
        pass  # generated files
```
Surface citations from `trace …knowledgeBaseLookupOutput.retrievedReferences` so
streamed claims stay grounded (see erp-agent-grounding).

## 3. Live file-change diffs (the action contract)
When the agent proposes an edit, have it emit the change as structured data
(e.g. a `json:local-action` block with `diff_path` / `diff_before` / `diff_after`,
the aicippy contract), then render a unified diff BEFORE applying — and gate on
confirmation:
```python
import difflib, sys
def show_diff(path, before, after):
    d = difflib.unified_diff(before.splitlines(True), after.splitlines(True),
                             fromfile=f"a/{path}", tofile=f"b/{path}")
    for ln in d:
        c = "32" if ln.startswith("+") and not ln.startswith("+++") else \
            "31" if ln.startswith("-") and not ln.startswith("---") else "90"
        sys.stdout.write(f"\033[{c}m{ln}\033[0m")
# stream prose live (§1/§2); when a change block closes, show_diff(...), ask y/n, then write the file.
```
Pattern: **stream the explanation token-by-token → on a completed change block,
pause the stream, render the green/red diff, confirm, apply, then continue.**

## 4. Live-render discipline
- Flush every chunk (`flush=True`); never buffer the whole response then dump it
  (that's fake streaming — show the agent's real cadence).
- Separate channels visually: streamed **answer** vs dim **trace/thinking** vs
  red/green **diff** vs the tool **result**.
- Truncation is a bug: render the FULL streamed text and the FULL diff — never cap
  at N words/lines silently.
- On stream error mid-turn, surface it (don't swallow it) and stop cleanly.
