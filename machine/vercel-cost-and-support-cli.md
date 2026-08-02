# Vercel CLI — Usage, Billing & Cost Drivers

Model: `vercel` CLI for deploy/project management; usage and billing figures live primarily in the Vercel Dashboard, with some inspectable via CLI. Scope is per-project vs per-team — always confirm scope before drawing cost conclusions.

## Scoping
| Task | Command |
|---|---|
| Show current scope (user/team) | `vercel whoami` |
| List teams | `vercel teams list` |
| Switch scope | `vercel switch <team-slug>` |
| Link local dir to a project | `vercel link` |
| Show linked project info | `vercel project ls` |
A command run in the wrong scope silently reports the wrong team's numbers — check `vercel whoami` and `.vercel/project.json` before trusting a figure.

## Usage & billing surfaces
| Task | Where |
|---|---|
| Bandwidth, function invocations, build minutes, edge requests | Dashboard → Team/Project → Usage tab. Verify against current `vercel --help` for any newer `usage`/`billing` subcommand before assuming a CLI-only workflow exists |
| Deployment list + status | `vercel ls` |
| Inspect one deployment (region, config) | `vercel inspect <deployment-url>` |
| Logs for a deployment (surfaces invocation-heavy routes) | `vercel logs <deployment-url>` |
| Env vars (rule out a stray debug flag driving extra compute) | `vercel env ls` |

## Cost drivers to check first
| Driver | What inflates it | Where to look |
|---|---|---|
| Bandwidth | Large uncached assets, images bypassing Image Optimization, low CDN cache-hit rate | Usage tab bandwidth breakdown by project |
| Function invocations | A Serverless/Edge Function on a route that should be static/ISR; client polling an API route | `vercel logs`, per-route invocation count in Usage tab |
| Function duration (GB-hours) | Cold starts, heavy per-invocation compute, unbounded external-API wait inside the function | Function duration breakdown in Usage tab |
| Build minutes | Frequent pushes triggering full rebuilds, no build cache, monorepo rebuilding unrelated packages | Build logs — check cache-hit indicator |
| Image Optimization requests | Every unique width/quality variant of a source image counts separately | Image Optimization row in Usage tab |
Verify exact current pricing/limits/plan tiers on the Vercel pricing page — these change across plan revisions; don't quote a remembered number into a customer-facing estimate.

## Project vs team scoping gotcha
A project can transfer between a personal account and a team; historical usage before transfer may not follow it into the same view. Confirm which scope+project a usage number belongs to before comparing month-over-month.
