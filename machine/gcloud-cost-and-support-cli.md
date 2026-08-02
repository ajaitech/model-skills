# gcloud Cost & Support CLI

Model: `gcloud` CLI against Cloud Billing, Budgets, BigQuery billing export, and GCP Support. Firebase projects bill through the underlying GCP project — Firebase Console spend and `gcloud billing` report the same account.

## Billing accounts & project linkage
| Task | Command |
|---|---|
| List billing accounts | `gcloud billing accounts list` |
| Show a project's linked billing account | `gcloud billing projects describe <project-id>` |
| Link a project to a billing account | `gcloud billing projects link <project-id> --billing-account=<ACCOUNT_ID>` |
No direct "list all projects under this billing account" command is guaranteed stable across CLI versions — verify against `gcloud billing --help` or use the Billing Console.

## Budgets
| Task | Command |
|---|---|
| List budgets | `gcloud billing budgets list --billing-account=<ACCOUNT_ID>` |
| Create budget | `gcloud billing budgets create --billing-account=<ACCOUNT_ID> --display-name="..." --budget-amount=<amount> ...` |
Verify threshold-rule and notification-channel flags against `gcloud billing budgets create --help` before authoring — field names shift across `beta`→GA promotion.

## BigQuery billing export
- Enabling export to BigQuery is done in the Billing Console (Billing export page) targeting a dataset you own — there is no dedicated `gcloud` "enable export" command; verify against the current console/API surface.
- Once flowing, this is the highest-fidelity cost source (per-SKU, per-label) — query with `bq query`:
```
bq query --use_legacy_sql=false "SELECT ... FROM \`project.dataset.gcp_billing_export_v1_XXXXXX\`"
```

## Support cases
| Task | Command |
|---|---|
| List support cases | `gcloud support cases list --project=<project-id>` |
| Create a case | `gcloud support cases create --display-name="..." --project=<project-id> ...` |
Requires the Cloud Support API enabled and a paid support plan (Standard/Enhanced/Premium) — Basic (free) tier has no API case-creation entitlement. Verify current command group (`gcloud support` vs `gcloud alpha support`) via `gcloud help support` — this surface has moved between release tracks.

## Quota inspection
| Task | Command |
|---|---|
| Compute-specific project quotas | `gcloud compute project-info describe --project=<project-id>` |
| General service-level quota | `gcloud services quota list --service=<service>.googleapis.com --consumer=projects/<project-id>` |
| Request a quota increase | Cloud Console Quotas page — no universal CLI grant command; verify per-service |

## Firebase-specific notes
- Spark (free) plan projects have no linked billing account — `gcloud billing projects describe` returns unlinked. Blaze plan is required for any billed usage (Cloud Functions beyond free tier, Gemini/Vertex AI calls, etc.).
- A feature stalled on a "spending cap" (e.g. Gemini image generation blocked) is a Billing Budget cap, not a code bug — check `gcloud billing budgets list` for an amount lower than expected spend, and the project's Blaze cap under Firebase Console → Usage & Billing.
