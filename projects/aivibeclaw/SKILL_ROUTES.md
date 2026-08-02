Project: AiVibeClaw (`aivibeclaw`) — multi-product monorepo; route by product subtree.

| Condition | Skill URL |
|---|---|
| under `clickclack/` (go.mod at the clickclack ROOT; API code in `apps/api`), `gogcli/`, or `wacli/` | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/go-backend-services.md |
| under `clawhub/src` or `clawhub/packages` — `react` 19.2.7 + `vite` 8.0.16 in `clawhub/package.json` | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/web-typescript-react-vite.md |
| under `AIVIBECLAW-CONTROL-CENTER/development` — `@aws-sdk/client-lambda` / `client-sesv2` 3.1093.0 (`package.json:48-49`) | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/aws-platform.md |
| under `AIVIBECLAW-CONTROL-CENTER/development` — `AIVIBECLAW_RAZORPAY_*` config or `ui/src/app/razorpay-checkout.ts` in play | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/payments-india-razorpay.md |
| under `deploy/` (PostgreSQL 17 + pgvector, `deploy/db-pgvector.sh`) or `clickclack/apps/api` (pgx/v5) | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/data-postgres-dynamo.md |
| ad-hoc machine or system task, not product code | https://raw.githubusercontent.com/ajaitech/model-skills/main/machine/MACHINE_INDEX.md |

No row exists for these — work from the product's own manifest and upstream docs:
- Cloudflare Workers (`clawsweeper/dashboard`, `clickclack/infra`, `docs/workers`) — the node-serverless
  file covers Serverless Framework/Lambda, and this repo has no `serverless.yml` and no `aws-lambda` dep.
- Lit + Vite UI (`AIVIBECLAW-CONTROL-CENTER/development/ui`) — Vite but no React.
- Python FastAPI (`.../development/aivaruna`), Swift (peekaboo, CodexBar), plain Node CLIs
  (mcporter, claude-code-mcp, clownfish).

Not applicable at all: Flutter, Next.js/Astro, Python-AWS-CDK, PHP/Laravel, Firebase, IoT, PayPal.

Fetch only rows whose condition is true right now.
