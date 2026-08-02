# AWS Cost & Support CLI

Model: AWS CLI v2 (`aws`) against Cost Explorer (`ce`), Budgets, Cost Anomaly Detection, and Support. Requires credentials scoped to `ce:*`/`budgets:*`/support, or the management account for consolidated billing.

## Cost Explorer (`aws ce`)
| Task | Command shape |
|---|---|
| Cost/usage for a date range | `aws ce get-cost-and-usage --time-period Start=YYYY-MM-DD,End=YYYY-MM-DD --granularity MONTHLY --metrics UnblendedCost` |
| Group by service | add `--group-by Type=DIMENSION,Key=SERVICE` |
| Forecast | `aws ce get-cost-forecast --time-period ... --metric UNBLENDED_COST --granularity MONTHLY` |
| Rightsizing recommendations | `aws ce get-rightsizing-recommendation --service AmazonEC2` |
Verify exact required/optional flags via `aws ce get-cost-and-usage help` before running a real query — the filter/group-by JSON shape is easy to get subtly wrong.

## Budgets
| Task | Command shape |
|---|---|
| List budgets | `aws budgets describe-budgets --account-id <id>` |
| Create budget | `aws budgets create-budget --account-id <id> --budget file://budget.json --notifications-with-subscribers file://notify.json` |
Verify the budget/notification JSON schema against current `aws budgets create-budget help` before authoring the file — field names have shifted across CLI versions.

## Cost Anomaly Detection
| Task | Command shape |
|---|---|
| List monitors | `aws ce get-anomaly-monitors` |
| List detected anomalies | `aws ce get-anomalies --date-interval StartDate=YYYY-MM-DD,EndDate=YYYY-MM-DD` |
| Create a monitor | `aws ce create-anomaly-monitor --anomaly-monitor file://monitor.json` |

## Support (`aws support`)
Requires a **Business, Enterprise On-Ramp, or Enterprise** support plan — Basic/Developer plans get access-denied on the API, not an empty result. Confirm plan tier before assuming this path is viable; the Support Center console works on any plan, the API does not.
| Task | Command shape (Business+ only) |
|---|---|
| Open a case | `aws support create-case --subject "..." --service-code ... --category-code ... --communication-body "..."` |
| List open cases | `aws support describe-cases --include-resolved-cases false` |
Get `service-code`/`category-code` values from `aws support describe-services` first — these are account/catalog-specific, never guess them.

## Cost-allocation tagging
- Activate tags in the Billing Console (no `aws ce` activation API) before they appear in group-by/filter.
- Tag at resource creation — retroactive tagging doesn't backfill historical cost data.
- Untagged resources land in `NoTagKey`/`(not tagged)` buckets — a growing bucket signals tagging drift, not necessarily new spend.

## Common spend traps
| Trap | Why it bleeds money | Check |
|---|---|---|
| Idle NAT Gateway | Billed hourly + per-GB even near-zero traffic | Filter CE by `Service=NAT Gateway`; consider VPC endpoints instead |
| Provisioned Bedrock throughput left running | Billed per-hour regardless of usage, unlike on-demand | `aws bedrock list-provisioned-model-throughputs` — check for idle ones |
| Unbounded CloudWatch Logs retention | Default is "Never Expire" — storage grows forever | `aws logs describe-log-groups --query 'logGroups[?!retentionInDays]'` |
| Unattached EBS volumes / stale snapshots | Billed even when detached from any instance | `aws ec2 describe-volumes --filters Name=status,Values=available` |
| Idle/oversized RDS or EC2 | — | Use the rightsizing command above |
