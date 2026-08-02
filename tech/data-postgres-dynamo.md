# Data — Postgres vs DynamoDB

## Applies when
- `schema.sql`, `migrations/`, Prisma schema, or SQLAlchemy/Django models present (Postgres signal).
- DynamoDB table defs in CDK/CloudFormation/`serverless.yml`, or `boto3`/`@aws-sdk/client-dynamodb` calls.
- A design task requires choosing between the two, or reviewing an existing schema.

## Authoritative sources
| Need | URL |
|---|---|
| PostgreSQL documentation | https://www.postgresql.org/docs/ |
| DynamoDB developer guide | https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html |
| DynamoDB best practices | https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-table-design.html |
| DynamoDB data modeling | https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/data-modeling.html |
| Amazon RDS for PostgreSQL | https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html |

## Non-obvious rules
- **Access-pattern order is inverted.** Postgres normalizes for correctness first, indexes for patterns second. DynamoDB schema is derived from access patterns first — a table designed before the query list is final needs a redesign, not a migration.
- **Single-table design fits a bounded set of related entities**, not every workload: generic `PK`/`SK` names with type-prefixed values (`USER#123`, `ORDER#456`) turn joins into item collections retrievable in one Query.
- **Hot partitions throttle regardless of table-level capacity.** A low-cardinality key (single tenant, monotonic id) concentrates load on one partition — fix with a sharded key suffix.
- **GSIs are eventually consistent only, with their own capacity** — a GSI can throttle independently. Project only needed attributes; `ALL` on every GSI multiplies storage cost.
- **Items cap at 400KB total.** An unbounded list/map attribute eventually breaches this — model unbounded collections as separate items in the same partition.
- **Postgres DDL can lock the whole table.** A non-null-default column or plain `CREATE INDEX` takes `ACCESS EXCLUSIVE`, blocking reads/writes. Use `CREATE INDEX CONCURRENTLY` and nullable-then-backfill-then-constrain instead.
- **A foreign key constraint does not create an index** — joins and cascading deletes full-scan until one is added explicitly.
- **RLS is bypassed by the table owner and any `BYPASSRLS` role by default** — the app's runtime role must be neither, or isolation is decorative.
- **Choosing between them:** relational wins when patterns are unknown/evolving or reporting/multi-entity transactions matter. DynamoDB wins when patterns are fixed upfront and predictable single-digit-ms latency at scale is required.
- **DynamoDB has no `ALTER TABLE` equivalent** — "migrating" means an application-level dual-write/backfill/cutover script, not a DDL statement.

## Production checklist
- [ ] Postgres: every foreign key column has an explicit index
- [ ] Postgres: DDL on large/hot tables uses `CONCURRENTLY` or nullable-then-backfill
- [ ] Postgres: RLS-enforcing runtime role confirmed not table owner, not `BYPASSRLS`
- [ ] DynamoDB: access patterns documented and reviewed before table/GSI design
- [ ] DynamoDB: partition key cardinality checked against traffic; sharding applied where needed
- [ ] DynamoDB: GSI projections scoped to attributes actually needed
- [ ] DynamoDB: unbounded-growth data modeled as item collections
- [ ] Migration/backfill plan tested at production-scale volume before cutover

## Never
- Never design a DynamoDB table before the access-pattern list is final.
- Never key a high-traffic DynamoDB partition on a low-cardinality value without sharding.
- Never run a blocking `ALTER`/`CREATE INDEX` on a large hot Postgres table without `CONCURRENTLY`.
- Never assume a Postgres foreign key constraint indexed the referencing column.
- Never let unbounded data grow inside a single DynamoDB item attribute.
- Never give the app's runtime DB role `BYPASSRLS` or table ownership when RLS is the isolation mechanism.
