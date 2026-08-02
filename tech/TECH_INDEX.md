# Tech Skill Index

| Condition | Skill URL |
|---|---|
| `pubspec.yaml` exists at repo root | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/flutter-dart-mobile.md |
| `package.json` lists both `react` and `vite` under dependencies/devDependencies | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/web-typescript-react-vite.md |
| `package.json` lists `next` under dependencies, OR `astro.config.mjs`/`astro.config.ts` exists | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/web-nextjs-astro.md |
| `cdk.json` exists AND a `requirements.txt` or `pyproject.toml` exists | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/backend-python-aws-cdk.md |
| `serverless.yml`/`serverless.ts` exists, OR `package.json` lists `aws-lambda`/`serverless-http` with no `cdk.json` present | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/backend-node-serverless.md |
| `go.mod` exists at repo root | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/go-backend-services.md |
| `firebase.json` exists at repo root | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/firebase-platform.md |
| `cdk.json` exists, OR `template.yaml`/`samconfig.toml` exists, OR a `.aws/` directory exists | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/aws-platform.md |
| `platformio.ini` exists, OR any `*.ino` file exists, OR `package.json`/`requirements.txt` lists an `mqtt`/`coap`/`paho` library | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/iot-protocol-translation.md |
| `package.json`/`requirements.txt`/`pom.xml` lists a `razorpay` dependency, OR source references `RAZORPAY_KEY_ID` | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/payments-india-razorpay.md |
| `composer.json` exists AND requires `laravel/framework` | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/backend-php-laravel.md |
| `package.json`/`composer.json` lists `@paypal/paypal-server-sdk` or `paypal-rest-sdk`, OR a `services/paypal` directory or `paypal.ts`/`paypal.php` module exists | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/payments-paypal.md |
| Repo `CLAUDE.md` or a `/design` doc contains the string "Liquid Glass", OR a design-tokens file (e.g. `design/tokens.json`) defines glass/blur/translucency tokens | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/design-liquid-glass.md |
| `prisma/schema.prisma` exists, OR `requirements.txt`/`package.json` lists `psycopg2`/`pg`/`postgres`, OR a CDK/serverless template defines a DynamoDB table resource | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/data-postgres-dynamo.md |

Fetch ONLY the rows whose condition is true for the current project. Do not fetch the rest.
