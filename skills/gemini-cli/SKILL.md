---
name: gemini-cli
description: Complete reference for the Gemini CLI itself — its commands and syntax (mcp, extensions, skills, hooks, gemma), flags (--model, --yolo, --approval-mode, --extensions, --include-directories, -p/-i), settings.json schema, model/thinking config, custom slash-commands (TOML), GEMINI.md context, MCP servers, and skill authoring. Use whenever the task is about configuring, extending, scripting, or operating the `gemini` CLI on this machine — adding an MCP server, writing a custom command or skill, tuning the model/approval mode, or explaining how a Gemini CLI feature works.
---

# Gemini CLI — commands, syntax & config

This machine: `gemini` 0.44.x, config in `~/.gemini/`. The CLI is an interactive
agent (`gemini`) or headless (`gemini -p "…"`).

## Top-level commands & key flags
```bash
gemini                         # interactive
gemini -p "prompt"             # headless / non-interactive (pipe stdin too)
gemini -i "prompt"             # run a prompt then stay interactive
gemini -m gemini-3.1-pro-preview   # model override (also GEMINI_MODEL env, settings model.name)
gemini --approval-mode default|auto_edit|yolo|plan   # tool approval (yolo = auto-approve all; plan = read-only)
gemini -y                      # = --approval-mode yolo
gemini --include-directories ../shared,../api   # add dirs to the workspace
gemini --debug                 # debug console (F12)

gemini mcp add|remove|list|enable <…>      # MCP servers
gemini extensions <list|install|…>         # extensions  (-e to scope; -l to list)
gemini skills <list|enable|disable|install|link|uninstall>   # agent skills (SKILL.md)
gemini hooks <…>               # lifecycle hooks
gemini gemma                   # local Gemma routing
```

## Config precedence (model)
`-m flag` > `GEMINI_MODEL` env > `~/.gemini/settings.json → model.name` > built-in default.
Confirm with `gemini --debug -p "ok" | grep Routing` (shows the resolved model).

## settings.json (`~/.gemini/settings.json`)
```jsonc
{
  "security": { "auth": { "selectedType": "oauth-personal" } },  // Google account (Ultra) vs api key
  "model":    { "name": "gemini-3.1-pro-preview", "maxSessionTurns": -1 },  // -1 = unlimited
  "ui":       { "theme": "Default" },
  "mcpServers": { "<name>": { "command": "…", "args": ["…"], "env": {…} } }  // or via `gemini mcp add`
}
```
Scopes: user (`~/.gemini/settings.json`) and project (`<repo>/.gemini/settings.json`, overrides user).

## Custom slash-commands (TOML)
Put `*.toml` in `~/.gemini/commands/` (user) or `<repo>/.gemini/commands/` (project).
File `deploy.toml` → `/deploy`:
```toml
description = "Deploy the lambda"
prompt = "Run: bash deploy/deploy_lambda.sh {{args}} and report the result."
```
`{{args}}` injects whatever follows the command. Nest dirs for namespacing
(`commands/aws/sync.toml` → `/aws:sync`).

## GEMINI.md (project/user context, like CLAUDE.md)
`~/.gemini/GEMINI.md` (global) or `<repo>/GEMINI.md` (project) is auto-loaded as
standing context. Use it for repo conventions, stack, do/don't — concise.

## MCP servers (extend tools)
```bash
gemini mcp add aws-docs uvx awslabs.aws-documentation-mcp-server   # stdio server
gemini mcp add myhttp https://host/mcp                            # http/sse server
gemini mcp list
```
Scope tools per run with `--allowed-mcp-server-names a,b`.

## Skills (SKILL.md) — what you author for domain expertise
`~/.gemini/skills/<name>/SKILL.md` with frontmatter:
```markdown
---
name: my-skill
description: <when to use — this is the trigger the model reads>
---
# concise instructions; reference references/*.md for depth; scripts/ for code
```
A strong `description` (clear "use when …") is what makes the skill auto-activate.
Keep SKILL.md short; push detail into `references/`. Manage with `gemini skills`.

## Hooks
`gemini hooks` configures lifecycle hooks (pre/post tool, session events) — use to
enforce policy, inject context, or run checks automatically around tool calls.

## Operating tips
- Headless automation: `echo "data" | gemini -p "summarize"` (stdin + prompt combine).
- `--approval-mode plan` for safe read-only analysis; `yolo` only in trusted automation.
- Skills + GEMINI.md + MCP compose: skills = procedural knowledge, GEMINI.md =
  standing context, MCP = live tools.
