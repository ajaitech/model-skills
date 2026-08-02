# gcloud Cost & Support CLI

Model: `gcloud` CLI against Cloud Billing, Budgets, BigQuery billing export, and the Cloud Support API (this machine: gcloud 578.0.0 at `~/google-cloud-sdk/bin`; `alpha`/`beta` components are NOT installed — `gcloud components install alpha` first). Firebase projects bill through the underlying GCP project — Firebase Console spend and `gcloud billing` report the same account.

## Billing accounts & project linkage
| Task | Command |
|---|---|
| List billing accounts | `gcloud billing accounts list` |
| Show a project's linked billing account | `gcloud billing projects describe <project-id>` |
| Link a project to a billing account | `gcloud billing projects link <project-id> --billing-account=<ACCOUNT_ID>` |
| List all active projects on a billing account | `gcloud billing projects list --billing-account=<ACCOUNT_ID>` |
| Unlink | `gcloud billing projects unlink <project-id>` |
All of the above are GA (verified against `gcloud billing --help`, 578.0.0).

## Budgets
| Task | Command |
|---|---|
| List budgets | `gcloud billing budgets list --billing-account=<ACCOUNT_ID>` |
| Create budget | `gcloud billing budgets create --billing-account=<ID> --display-name="..." --budget-amount=100.75USD --threshold-rule=percent=0.50 --threshold-rule=percent=0.75,basis=forecasted-spend` |
GA group, verified against `gcloud billing budgets create --help` (578.0.0). `--threshold-rule` repeats, `percent` is a FRACTION (0.50 = 50%), `basis` is `current-spend` (default) or `forecasted-spend`, and the amount carries its currency inline (`100.75USD`). Scope with `--filter-projects`/`--filter-services`/`--filter-labels`; route alerts with `--notifications-rule-pubsub-topic` or `--notifications-rule-monitoring-notification-channels`. Mutually exclusive: `--budget-amount` vs `--last-period-amount`.

## BigQuery billing export
- Enabling export to BigQuery is done in the Billing Console (Billing export page) targeting a dataset you own — there is no dedicated `gcloud` "enable export" command; verify against the current console/API surface.
- Once flowing, this is the highest-fidelity cost source (per-SKU, per-label) — query with `bq query`:
```
bq query --use_legacy_sql=false "SELECT ... FROM \`project.dataset.gcp_billing_export_v1_XXXXXX\`"
```

## Support cases
There is **no `gcloud support` command group** — `gcloud support` returns `Invalid choice` (verified on 578.0.0) and there is no such page in the gcloud reference. Use the Cloud Customer Care REST API v2 directly, or the Console.
| Task | How |
|---|---|
| Enable the API | `gcloud services enable cloudsupport.googleapis.com --project=<project-id>` |
| List cases under a project | `GET https://cloudsupport.googleapis.com/v2/{parent}/cases` with `parent=projects/<project-id>`; auth `Authorization: Bearer $(gcloud auth print-access-token)` |
| Search across an org and its projects | `cases.search` — plain `cases.list` returns only cases parented DIRECTLY at that resource, not descendants |
OAuth scope: `https://www.googleapis.com/auth/cloudsupport` (or `cloud-platform`). Requires a paid support plan (Standard/Enhanced/Premium) — Basic (free) has no API case-creation entitlement. Docs root: https://cloud.google.com/support/docs/reference/rest

## Quota inspection
| Task | Command |
|---|---|
| Compute-specific project quotas | `gcloud compute project-info describe --project=<project-id>` |
| General service-level quota | `gcloud alpha services quota list --service=<service>.googleapis.com --consumer=projects/<PROJECT_NUMBER>` — **alpha only** (`gcloud services quota` does not exist in GA; requires `gcloud components install alpha`). Both flags are required; `--consumer` also accepts `folders/<id>` / `organizations/<id>` |
| Request a quota increase | Cloud Console Quotas page — no universal CLI grant command; verify per-service |

## Firebase-specific notes
- Spark (free) plan projects have no linked billing account — `gcloud billing projects describe` returns unlinked. Blaze plan is required for any billed usage (Cloud Functions beyond free tier, Gemini/Vertex AI calls, etc.).
- A feature stalled on a "spending cap" (e.g. Gemini image generation blocked) is a Billing Budget cap, not a code bug — check `gcloud billing budgets list` for an amount lower than expected spend, and the project's Blaze cap under Firebase Console → Usage & Billing.
