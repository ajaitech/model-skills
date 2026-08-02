# AiVibeClaw

## Goal

AiVibeClaw is AiFactory's platform for appointing/operating AI agents as full "employees" — each gets
its own identity, phone number, email, AI model subscriptions ("brains"), a physical device, scoped
business-system access, and a governed communication policy, per `AIVIBECLAW-PRODUCT-REQUIREMENTS.md`.
It ships white-labeled as **AiVibeClaw** under `aivibeclaw.com`, telemetry-free, single Super-Admin
owner (Aravind) in v1. `/Users/aj/Dev-Apps/aivibeclaw` is a **satellite-tooling monorepo**: the Control
Center app, one example agent config, and ~12 independent CLI/bot/service repos supporting the platform
(Google Workspace, WhatsApp, GitHub maintenance bots, a skills registry, screen automation, chat, docs).
The core agent-runtime/gateway (an "OpenClaw" fork, npm package `aivibeclaw`, per `tech-plan.md`) and
the main `aivibeclaw/aivibeclaw` product repo are **not present as source in this checkout** — no
`aivibeclaw-code/`/`openclaw/` directory exists; the gateway installs via `npm install -g aivibeclaw@latest`.

## Products

| Product | What it does | Language / runtime | Path | Maturity |
|---|---|---|---|---|
| AIVIBECLAW-CONTROL-CENTER | Portal (PRD §15): hire/configure/govern agents, 85-method RPC API, PIN login, SQLite. ~360K lines. | Node≥26/TS7 + Lit 3.3.3 + Python≥3.14 FastAPI + Android | `development/` | Working; no git/CI; thin tests |
| agents (ajs-beste) | One markdown spec, example agent "AJ's Beste" (owner's Chief of Staff). Out-of-scope per PRD §22. | Markdown only | `agents/ajs-beste/` | Config doc, no code |
| clawhub | "ClawHub" registry: publish/search/install text skills + code/bundle plugins; vector search. | TS: React 19.2.7 + TanStack Start + Convex 1.41.0 | `clawhub/` | Fork (MIT, P.Steinberger); CI off; 336 tests |
| clawsweeper | GitHub issue/PR triage/repair bot for `aivibeclaw/aivibeclaw`, `clawhub`, self; via Codex CLI. | Node≥24, TS (tsgo); CF Worker dashboard | `clawsweeper/` | Fork; CI off; ~1,204 tests |
| clickclack | Self-hostable team chat (threads, DMs, magic-link+GitHub OAuth). | Go 1.26 (chi/SQLite/pgx) + Svelte 5; 1 binary | `clickclack/` | Fork; CI off; 30 tests, 85% gate |
| clownfish | Narrower sibling to clawsweeper — reviews pre-grouped clusters on one repo only. | Node≥24, zero npm deps | `clownfish/` | Fork of "projectclownfish"; CI off; tests broken |
| CodexBar | macOS 14+ menu-bar app: AI coding-provider usage/limits across 53 providers. | Swift 6.2 | `CodexBar/` | Fork; v0.37.3; CI off |
| claude-code-mcp | MCP server (stdio), one tool `claude_code`, shells to local `claude` CLI. | Node ESM, TS 5.8.3, MCP SDK ^1.11.2 | `claude-code-mcp/` | Fork; v1.10.12; CI off |
| gogcli | Go CLI (`gog`) for Google Workspace + Zoom; also MCP server. | Go 1.26.2, kong, mcp-go 0.55.0 | `gogcli/` | Fork; v0.31.1-dev; CI off |
| mcporter | TS CLI/runtime: discover/call/record/replay/codegen MCP servers; can bridge as aggregate server. | Node≥24 (Bun=secondary), MCP SDK ^1.29.0 | `mcporter/` | Fork ("Sweetistics"); v0.12.2; CI off |
| peekaboo | macOS screen-capture + GUI-automation CLI/app/MCP server; MCP client for Chrome DevTools MCP. | Swift 6.2, macOS 15+ | `peekaboo/` | v3.5.3; 5 sibling Swift pkgs empty |
| wacli | Scriptable WhatsApp CLI (QR pairing, SQLite+FTS5) on `whatsmeow`. | Go 1.25.0, cobra | `wacli/` | Fork; v0.11.1; CI off |
| deploy | Ops scripts: GPU-box bootstrap (gateway via npm, Postgres+pgvector, vision-id) + live face-ID service. | Bash + Python FastAPI | `deploy/` | Working; no tests found |
| docs | Static docs-site generator/deployer for `docs.aivibeclaw.com`. | Node (markdown-it/mermaid/pagefind) + CF Worker | `docs/` | Own MIT license; CI off |

## Core requirements

- Each agent: own identity, phone, email, device, isolated environment, up to 4 "brains" via
  **subscription login** (not API) sharing **one memory** (§2, §6).
- Access granted in levels (identity/skills, then Level-2 business-system access), logged, revocable (§7).
- **Mode 1** (agent as itself) vs **Mode 2** (on owner's behalf); nothing ever sent from the owner's own
  accounts without explicit per-item approval — hard rule (§8, FR-CM2-4).
- Voice/telephony real-time, any-language, inbound + outbound (§10).
- Device = body (camera/mic/speaker + lip-synced avatar); personal-phone vs office-device (3D depth
  camera) profiles (§11).
- Go-live evaluation blocks on policy conflicts (§13); 24/7 self-recovery; self-improvement bounded by
  granted access (§14).
- Full white-label + telemetry-free; contacts `aravind@ / developer@ / support@aivibe.in` (§3).

## Tech stack

| Layer | Technology | Version | Source of truth |
|---|---|---|---|
| Agent gateway (planned base) | OpenClaw fork → npm pkg `aivibeclaw` | `@latest`, unpinned | `tech-plan.md`, `deploy-gpu-server.sh:83` |
| Control Center BE/FE/aux | Node/TS + Lit/Vite + Python FastAPI | Node≥26, TS 7.0.2, lit 3.3.3, Python≥3.14, fastapi 0.139.2 | `AIVIBECLAW-CONTROL-CENTER/development/package.json` |
| Skill registry | TanStack Start + Convex | react 19.2.7, convex 1.41.0 | `clawhub/package.json` |
| Chat app | Go + Svelte | go 1.26, svelte 5.56.3 | `clickclack/go.mod` |
| Workspace/WhatsApp CLIs | Go | gogcli 1.26.2, wacli 1.25.0 | `gogcli/go.mod` |
| macOS automation | Swift | tools-version 6.2 | `peekaboo/Package.swift` |
| MCP SDKs | TS/Go/Swift official SDKs | ^1.11.2–1.29.0(TS), 0.55.0(Go), 0.12.x(Swift) | resp. manifests |
| Vision/face-ID | FastAPI + insightface (SCRFD/ArcFace) | fastapi≥0.111, insightface≥0.7.3 | `deploy/vision-id/requirements.txt` |
| Vector DB | PostgreSQL 17 + pgvector | `pgvector/pgvector:pg17` | `deploy/db-pgvector.sh` |
| Docs site | Cloudflare Worker + R2 | compat date 2026-05-06 | `docs/wrangler.toml` |

## Architecture

Each product subdirectory is its **own independent git repo** (`github.com/aivibe-org/<name>`) with a
single commit — "AiVibeClaw: white-label fork, rebranded and sanitized" — nearly every product is a
rebranded fork of a pre-existing OSS project (mostly by Peter Steinberger; one from "Sweetistics"; one
from "projectclownfish"). `/Users/aj/Dev-Apps/aivibeclaw` itself is **not** a git repo, nor are
`AIVIBECLAW-CONTROL-CENTER/`, `agents/`, `deploy/`. Products are **independent, coupled only by
convention** (env-var prefixes, `aivibeclaw.com` subdomains) — no root workspace ties them together.
Deployment targets: a GPU box (systemd, Docker Postgres, gateway via npm) for the gateway + vision-ID;
Cloudflare Workers/Containers for clawsweeper's dashboard, clickclack, docs; Homebrew/npm/AUR for CLIs.

## Naming conventions

- Domains: `<product>.aivibeclaw.com`, e.g. `"docs.aivibeclaw.com"`, `"clawsweeper.aivibeclaw.com"`
  (clickclack keeps its own `"app.clickclack.chat"`).
- Env-var prefixes per product: `AIVIBECLAW_*`, `CLAWSWEEPER_*`, `CLOWNFISH_*`, `CLAWHUB_*`/`CLAWDHUB_*`
  (legacy alias), `GOG_*`, `WACLI_*`, `PEEKABOO_*`, `MCPORTER_*`, `CODEXBAR_*`, `CLICKCLACK_*`, `VISION_*`.
- Identical fork commit across every forked repo: `"AiVibeClaw: white-label fork, rebranded and sanitized"`.
- Fixed product/system contacts: `"aravind@aivibe.in"`, `"developer@aivibe.in"`, `"support@aivibe.in"` (FR-WL-5).

## Data types & models

| Entity | Fields (name : type) | Store | Defined in |
|---|---|---|---|
| AgentDraft | agentId, employeeNo, name, roleId, tasks, selfBrainId, sleepWindow | SQLite | `src/components/contracts/core.ts` |
| skills / packages | slug, displayName, ownerUserId, latestVersionId, tags (~60 tables incl. stars, auditLogs) | Convex | `clawhub/convex/schema.ts:757,1416` |
| messages / workspaces / channels / users | id: ULID text, name, slug, body, created_at, deleted_at, role | SQLite/Postgres | `clickclack/apps/api/.../migrations/*.sql` |
| codex-result / job | status, repo, cluster_id, actions[], merge_preflight[], allowed_actions[] | JSON / Markdown+YAML | `clownfish/schemas/*.schema.json` |

## API surface

| Operation | Product | Method/Path/Tool | Auth | Defined in |
|---|---|---|---|---|
| RPC gateway (85 methods: `agent.hire`, `chat.message.send`) | Control Center | RPC/HTTP | PIN + session cookie | `rpc-descriptors/controlCenterRpc.ts` |
| Registry API (56 routes) | clawhub | REST `/api/v1/{search,skills,packages,...}` | Convex Auth (GitHub OAuth) | `packages/schema/src/routes.ts` |
| Chat REST/WS (55+ routes) | clickclack | REST `/api/*` + WS `/api/realtime/ws` | session cookie + CSRF header | `apps/api/internal/httpapi/server.go` |
| Dashboard status/triage | clawsweeper | REST `/api/{health,status,triage}` + webhook | GitHub App / bearer | `dashboard/worker.ts` |
| Tool `claude_code` | claude-code-mcp | MCP/stdio | local `claude` CLI login | `src/server.ts` |
| 25 tools (ImageTool, ClickTool,...) | peekaboo | MCP/stdio | none (local) | `MCP/Server/MCPToolCatalog.swift` |
| Allowlisted tools wrapping `gog` subcommands | gogcli | MCP/stdio (`gog mcp`) | none (local) | `internal/cmd/mcp.go` |
| enroll/identify/events | deploy/vision-id | REST `/enroll` `/identify` + WS `/events` | none (127.0.0.1 only) | `app.py` |

## CORS & headers

No CORS configuration found anywhere (clickclack uses same-origin SPA + custom `X-ClickClack-CSRF`
header instead) — **none configured — GAP** across the monorepo.

## Security boundary

- Auth varies per product: Control Center = 4-digit operator PIN + HttpOnly session cookie; clawhub =
  GitHub OAuth via Convex Auth; clickclack = magic-link + optional GitHub OAuth; wacli/gogcli/peekaboo
  rely on the provider's own login (WhatsApp QR, Google OAuth, local `claude` CLI session) — no CLI/MCP
  tool holds its own credential store.
- Secret sources (names only): `AIVIBECLAW_PRODUCT_POLICY_TOKEN`, `AIVIBECLAW_OPERATOR_PIN`,
  `CLAWSWEEPER_APP_PRIVATE_KEY`, `CLAWSWEEPER_WEBHOOK_SECRET`, `CLOWNFISH_GH_TOKEN`, `GITHUB_TOKEN`,
  `OPENAI_API_KEY`, `CLICKCLACK_GITHUB_CLIENT_SECRET`, `CLICKCLACK_R2_SECRET_ACCESS_KEY`,
  `GOG_ACCESS_TOKEN`, `ANTHROPIC_API_KEY`.
- Public vs private: clawhub, clickclack, docs, clawsweeper's dashboard are internet-facing
  (`*.aivibeclaw.com`); vision-id binds `127.0.0.1` only; Control Center is a private local-auth portal.

## Known gaps & risks

- **No version control at the top level** — `/Users/aj/Dev-Apps/aivibeclaw` is not a git repo; product
  subdirs are, each a single squashed commit. `AIVIBECLAW-CONTROL-CENTER/`, `agents/`, `deploy/` have no
  git history at all.
- **The gateway/agent-runtime and main product repo are absent from source** — no
  `aivibeclaw-code/`/`openclaw/` anywhere; installed via `npm install -g aivibeclaw@latest`; clawsweeper/
  clownfish job records reference PR activity on a separate `aivibeclaw/aivibeclaw` repo not present here.
- **CI is universally disabled** — every `.github/workflows/*.yml` in every forked product is renamed
  `*.yml.disabled`.
- **peekaboo cannot build standalone** — 5 sibling Swift packages (AXorcist, Tachikoma, Commander,
  TauTUI, Swiftdansi) are empty, no `.gitmodules` recorded.
- **clownfish has reproducible test failures** from the rebrand (expects org `aivibeclaw/clownfish`,
  finds `aivibe-org/clownfish`); **no CORS policy configured anywhere**.
- **Thin test coverage on the largest component** — Control Center ~360K lines, ~6 test files.
- Licensing: most products retain upstream MIT copyright (Peter Steinberger), not AiVibeClaw-original.
