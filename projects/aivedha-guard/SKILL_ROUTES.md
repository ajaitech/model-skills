# AivedhA Guard — Skill Routes

| Condition | Skill URL |
|---|---|
| Editing the SPA in `src/` (React 18 / TypeScript / Vite / Tailwind / shadcn) | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/web-typescript-react-vite.md |
| Touching AWS surfaces — Lambda (zip or container), AppSync, API Gateway, S3/CloudFront, Cognito, Secrets Manager, ECR, CloudFormation, or anything in `deploy/` | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/aws-platform.md |
| Touching Razorpay orders, subscriptions, webhooks or plan IDs (`aws-lambda/razorpay-handler/`, `RAZORPAY_CONFIG`) | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/payments-india-razorpay.md |
| Touching `aws-lambda/paypal-handler/`, `createPayPalOrder`/`capturePayPalOrder`, or the `VITE_ENABLE_PAYPAL` path | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/payments-paypal.md |
| Building or reviewing UI surfaces — glass cards, blur panels, popups (`backdrop-blur`, the `glass` blur token in `tailwind.config.ts:90`) | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/design-liquid-glass.md |
| Writing SQL or schema against the shared Postgres RDS — `database/`, `shared/db_connection.py`, `shared/universal_db.py` | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/data-postgres-dynamo.md |
| Ad-hoc machine or system task, not product code | https://raw.githubusercontent.com/ajaitech/model-skills/main/machine/MACHINE_INDEX.md |

Fetch only rows whose condition is true right now.

No row for `backend-python-aws-cdk`: the Python Lambdas are real, but there is no CDK in this
repo — IaC is a single hand-written CloudFormation template, so AWS work routes to
`aws-platform`. No Firebase, Next/Astro, Flutter, Laravel, Go or serverless-framework code exists
here, so those guides have no row.
