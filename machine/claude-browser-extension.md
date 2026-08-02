# Claude in Chrome — Browser Extension

Scope: operating the `claude-in-chrome` MCP tools safely and effectively for browser automation — clicking, forms, screenshots, console/network reading, multi-tab flows.

## Load order — call tab context FIRST
Before any interaction, load the core tool set in ONE batched `ToolSearch` call, never one tool at a time (each separate call wastes a round trip):
`select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__tabs_create_mcp`
Add task-specific tools to that same call when obviously needed: `read_console_messages`/`read_network_requests` (debugging), `form_input` (forms), `gif_creator` (multi-step recordings), `javascript_tool` (page scripting). Only issue a second `ToolSearch` if a later, unanticipated need arises.
Always call `tabs_context_mcp` first to establish which tab/session is being driven — never assume a tab ID.

## Hard rule — never trigger JS dialogs
NEVER cause a page to fire `alert()`, `confirm()`, or `prompt()`. A native JS dialog blocks the entire extension — it cannot be dismissed programmatically — and stalls the whole session. Before clicking a button with an unknown handler on an unfamiliar page, read page state/console first, or inspect for dialog-triggering code via `javascript_tool` rather than clicking blind.

## Console reading — filter, don't dump
`read_console_messages` on a busy page returns high noise (framework warnings, analytics, third-party scripts). Apply a regex filter scoped to what's being debugged — the app's own log prefix, an error code, a specific file path — rather than reading the raw firehose. Cuts context usage and surfaces the actual signal.

## Multi-step flows — GIF capture
For flows spanning multiple screens or steps (onboarding, checkout, multi-page forms), use `gif_creator` to produce one shareable artifact of the whole flow, rather than a series of disconnected screenshots — the clearer deliverable when reporting "here's what happened."

## Site-permission model
The extension requires **site-level permission**, granted by the user in the extension's own settings, before it can act on a given origin. A tool call against an unpermitted site fails at the permission layer, not from a code bug — if actions silently fail on a new domain, the fix is the user granting that site permission in the extension UI, not retrying the call differently.

## Tab ID discipline
Never reuse a tab ID captured in an earlier session or turn. Tab IDs are only valid for the browser session that issued them via `tabs_context_mcp`/`tabs_create_mcp` — always re-resolve current tabs at the start of a new task rather than trusting a cached ID from prior context.

## Practical sequencing
1. `tabs_context_mcp` — confirm current tab(s).
2. `navigate`, or reuse an existing tab.
3. `read_page`/`find` to locate targets before `computer` (click/type) — avoids blind-clicking into a dialog-triggering element.
4. For forms, prefer `form_input` over simulated keystrokes where the tool supports the field type.
5. Screenshot or GIF at completion as the delivered evidence, not just a text claim of success.
