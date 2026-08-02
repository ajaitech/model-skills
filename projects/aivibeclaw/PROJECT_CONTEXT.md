# AiVibeClaw

## Goal

Appoint AI agents as full "employees" — each with its own identity, phone, email, model subscriptions
("brains"), physical device, scoped business-system access and a governed communication policy, per
`AIVIBECLAW-PRODUCT-REQUIREMENTS.md`. White-labeled under `aivibeclaw.com`, telemetry-free, one
Super-Admin owner in v1.

`/Users/aj/Dev-Apps/aivibeclaw` is a **satellite-tooling monorepo**: Control Center, one agent config,
12 independent CLI/bot/service repos. The core agent-runtime/gateway (an "OpenClaw" fork, npm package
`aivibeclaw`, per `tech-plan.md`) and the main `aivibeclaw/aivibeclaw` repo are **not source here** —
the gateway installs via `npm install -g aivibeclaw@latest` (`deploy-gpu-server.sh:83`).

**Read first: nearly every product is a single-commit white-label fork of a pre-existing OSS project.**
Each product subdir is its own repo (`github.com/aivibe-org/<name>`) whose entire history is one commit:
`"AiVibeClaw: white-label fork, rebranded and sanitized"` — upstream mostly Peter Steinberger, mcporter
from "Sweetistics", clownfish from "projectclownfish". Trust upstream docs; local history is gone.

## Products

| Product (path) | What it does | Runtime / key versions | State |
|---|---|---|---|
| AIVIBECLAW-CONTROL-CENTER/development | Portal (PRD §15): hire/govern agents, 80-method RPC, PIN login, SQLite, ~350K TS lines | node>=26, typescript 7.0.2, pkg v2026.7.23-r10; UI lit 3.3.3 (`ui/`); python pkg `aivaruna` v2026.7.23.post9, requires-python>=3.14, fastapi==0.139.2 (`aivaruna/`); @aws-sdk/client-lambda+sesv2 3.1093.0; `android/` | Works; no git, no CI; 6 test files |
| agents/ajs-beste | Markdown spec for example agent "AJ's Beste"; out-of-scope per PRD §22 | Markdown | Doc only |
| clawhub | Registry: publish/search/install skills + code/bundle plugins; vector search | react 19.2.7, @tanstack/react-start 1.168.26, convex 1.41.0 | Fork; CI off; 312 test files |
| clawsweeper | GitHub issue/PR triage+repair bot for `aivibeclaw/aivibeclaw`, `clawhub`, self; drives Codex CLI | Node>=24 TS; CF Worker dashboard | Fork; v0.3.1; CI off; 100 test files |
| clickclack | Self-hostable team chat (threads, DMs), single binary | go 1.26, chi/SQLite/pgx-v5 5.9.2, svelte ^5.56.3 | Fork; CI off; 25 Go test files |
| clownfish | Narrower clawsweeper sibling — reviews pre-grouped clusters on one repo | Node>=24, zero npm deps | Fork; v0.1.0; CI off; tests broken |
| CodexBar | macOS menu-bar app: AI coding-provider usage/limits | swift-tools 6.2 | Fork; v0.37.3; CI off |
| claude-code-mcp | MCP stdio server, one tool `claude_code`, shells to local `claude` CLI | Node ESM, MCP SDK ^1.11.2 | Fork; v1.10.12; CI off |
| gogcli | Go CLI `gog` for Google Workspace + Zoom; also an MCP server | go 1.26.2, mcp-go 0.55.0 | Fork; v0.31.1-dev; CI off |
| mcporter | TS CLI/runtime: discover/call/record/replay/codegen MCP servers; aggregate bridge | Node>=24, MCP SDK ^1.29.0 | Fork; v0.12.2; CI off |
| peekaboo | macOS screen-capture + GUI automation CLI/app/MCP server; also a Chrome DevTools MCP client | swift-tools 6.2 | v3.5.3; 5 sibling pkgs EMPTY |
| wacli | Scriptable WhatsApp CLI (QR pairing, SQLite+FTS5) on `whatsmeow` | go 1.25.0, cobra | Fork; v0.11.1; CI off |
| deploy | GPU-box bootstrap + live face-ID service | Bash; FastAPI>=0.111 + insightface>=0.7.3 (SCRFD/ArcFace); `pgvector/pgvector:pg17` | Works; no git; no tests |
| docs | Static generator/deployer for `docs.aivibeclaw.com` | Node (markdown-it/mermaid/pagefind) + CF Worker, compatibility_date 2026-05-06 | Own MIT license; CI off |

## Core requirements (PRD)

- Agent gets own identity, phone, email, device, isolated env, up to 4 "brains" via **subscription
  login** (not API keys) sharing **one memory** (§2, §6). Access is level-gated, logged, revocable (§7).
- **Mode 1** (agent as itself) vs **Mode 2** (on owner's behalf); nothing leaves the owner's own
  accounts without explicit per-item approval — hard rule (§8, FR-CM2-4).
- Real-time voice/telephony, any language, in + out (§10). Device = body (camera/mic/speaker +
  lip-synced avatar); personal-phone vs office-device (3D depth camera) profiles (§11).
- Go-live blocks on policy conflicts (§13); 24/7 self-recovery; self-improvement bounded by granted
  access (§14).

## Build / run / deploy

| Target | Command | Prerequisite |
|---|---|---|
| Control Center | in `.../development/`: `npm run build` (tsc); `npm run dev:gateway` (tsx) | Node >= 26 |
| clawhub | `bun run build` (llms:generate → vite build) | **Bun** — scripts call `bun`, not node |
| clickclack | `docker build` (`Dockerfile`: node:25-alpine web stage → golang:1.26-alpine api stage) | pnpm 11.0.7, Go 1.26 |
| Go CLIs | `go build ./cmd/...` in `gogcli/`, `wacli/` | Go >= 1.26 / 1.25 |
| Swift | `swift build` in `CodexBar/`; peekaboo does NOT build (see gaps) | Swift 6.2 toolchain |
| GPU box | `./deploy-gpu-server.sh`, `deploy/db-pgvector.sh` | Docker, systemd host |

**`./install-deps.sh` is inoperable here** — it targets `aivibeclaw-code/` and `zoozoo-android/`,
neither of which exists in this checkout. Do not run it; install per product.

## Architecture & naming

- Products are **independent, coupled only by convention** (env prefixes, `aivibeclaw.com` subdomains);
  no root workspace, no shared lockfile.
- Deploy: GPU box (systemd, Docker Postgres, npm gateway) for gateway + vision-ID; CF Workers for
  clawsweeper dashboard, clickclack, docs; Homebrew/npm for CLIs.
- `clickclack/go.mod` sits at the **clickclack root** — one Go module covering `apps/api`, not under it.
- Domains `<product>.aivibeclaw.com`; clickclack keeps `app.clickclack.chat`.
- Env prefixes: `AIVIBECLAW_*`, `CLAWSWEEPER_*`, `CLOWNFISH_*`, `CLAWHUB_*`/`CLAWDHUB_*` (legacy alias),
  `GOG_*`, `WACLI_*`, `PEEKABOO_*`, `MCPORTER_*`, `CODEXBAR_*`, `CLICKCLACK_*`, `VISION_*`.
- Go module paths and repo owner are `github.com/aivibe-org/<name>`, NOT `aivibeclaw/`.
- Fixed contacts: `aravind@aivibe.in`, `developer@aivibe.in`, `support@aivibe.in` (FR-WL-5).

## Data types & models

| Entity | Fields | Store | Defined in |
|---|---|---|---|
| AgentDraft | agentId, employeeNo, name, roleId, tasks, selfBrainId, sleepWindow | SQLite | `src/components/contracts/core.ts` |
| skills / packages | slug, displayName, ownerUserId, latestVersionId, tags (~60 tables) | Convex | `clawhub/convex/schema.ts` |
| messages / workspaces / channels / users | id: ULID text, name, slug, body, created_at, deleted_at, role | SQLite/PG | `clickclack/apps/api/.../migrations/*.sql` |
| codex-result / job | status, repo, cluster_id, actions[], merge_preflight[], allowed_actions[] | JSON / MD+YAML | `clownfish/schemas/*.schema.json` |

## API surface

| Operation | Product | Transport | Auth | Defined in |
|---|---|---|---|---|
| RPC gateway, 80 dotted methods (`agent.activate`, `agent.calendar.event.save`) | Control Center | RPC/HTTP | PIN + session cookie | `src/gateway/rpc-descriptors/controlCenterRpc.ts` |
| Registry API: 30 `/api/v1/*` paths + 10 legacy `/api/*` | clawhub | REST | Convex Auth (GitHub OAuth) | `packages/schema/src/routes.ts` |
| Chat, 68 route registrations | clickclack | REST `/api/*` + WS `/api/realtime/ws` | session cookie + CSRF header | `apps/api/internal/httpapi/server.go` |
| Dashboard status/triage | clawsweeper | REST `/api/{health,status,triage}` + webhook | GitHub App/bearer | `dashboard/worker.ts` |
| Tool `claude_code` | claude-code-mcp | MCP/stdio | local `claude` CLI login | `src/server.ts` |
| 27 tools (ImageTool, CaptureTool, AnalyzeTool, BrowserTool…) | peekaboo | MCP/stdio | none (local) | `…/PeekabooAgentRuntime/MCP/Server/MCPToolCatalog.swift` |
| Tools wrapping `gog` subcommands | gogcli | MCP/stdio (`gog mcp`) | none (local) | `internal/cmd/mcp.go` |
| enroll / identify / events | deploy/vision-id | REST + WS `/events` | none (127.0.0.1 only) | `app.py` |

## CORS, headers, security boundary

- clawhub `corsHeaders(origin = "*")` — **wildcard default** (`clawhub/convex/lib/httpHeaders.ts:12-13`).
- clawsweeper dashboard sets `access-control-allow-origin: *`, methods `GET,POST,OPTIONS`, headers
  `authorization,content-type` (`dashboard/worker.ts:4602-4604`) — wildcard beside bearer auth.
- clickclack: no CORS; same-origin SPA + custom `X-ClickClack-CSRF` header. Control Center, docs,
  vision-id: none configured.
- Auth: Control Center = operator PIN + HttpOnly session cookie; clawhub = GitHub OAuth via Convex
  Auth; clickclack = magic-link + optional GitHub OAuth; wacli/gogcli/peekaboo/claude-code-mcp ride the
  provider's own login (WhatsApp QR, Google OAuth, local `claude` session) — no CLI/MCP tool has its
  own credential store.
- Secret **names** only: `AIVIBECLAW_PRODUCT_POLICY_TOKEN`, `AIVIBECLAW_OPERATOR_PIN`,
  `AIVIBECLAW_RAZORPAY_KEY_ID`, `CLAWSWEEPER_APP_PRIVATE_KEY`, `CLAWSWEEPER_WEBHOOK_SECRET`,
  `CLOWNFISH_GH_TOKEN`, `GITHUB_TOKEN`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
  `CLICKCLACK_GITHUB_CLIENT_SECRET`, `CLICKCLACK_R2_SECRET_ACCESS_KEY`, `GOG_ACCESS_TOKEN`.
- Internet-facing: clawhub, clickclack, docs, clawsweeper dashboard. vision-id binds `127.0.0.1`;
  Control Center is a private local-auth portal.

## Known gaps & risks

- **No VCS at top level** — repo root is not a git repo; `AIVIBECLAW-CONTROL-CENTER/`, `agents/`,
  `deploy/` have none either. The 11 product subdirs hold one commit each: nothing to bisect, no
  upstream remote.
- **Gateway/main repo absent** (see Goal); clawsweeper and clownfish job records reference PRs on
  `aivibeclaw/aivibeclaw`, which is not in this checkout.
- **CI universally disabled** — every `.github/workflows/*.yml` renamed `*.yml.disabled` in every fork.
  Nothing is verified on push; run checks locally.
- **peekaboo cannot build standalone** — sibling Swift packages AXorcist, Tachikoma, Commander, TauTUI,
  Swiftdansi are empty dirs, no `.gitmodules`.
- **clownfish tests fail reproducibly** post-rebrand (expects org `aivibeclaw/clownfish`, finds
  `aivibe-org/clownfish`).
- **Wildcard CORS** on clawhub's default and clawsweeper's bearer-authenticated dashboard.
- **Thin coverage on the largest component** — Control Center ~350K TS lines, 6 test files.
- Licensing: most products keep upstream MIT copyright (Peter Steinberger), not AiVibeClaw-original.
