# Backend: Node.js Serverless (Lambda)

## Applies when
`serverless.yml`/`serverless.ts` exists, OR `package.json` lists `aws-lambda`/`serverless-http` with no `cdk.json` present.

## Authoritative sources
| Source | URL |
|---|---|
| Serverless Framework docs | https://www.serverless.com/framework/docs |
| Serverless Framework repo | https://github.com/serverless/serverless |
| AWS Lambda docs | https://docs.aws.amazon.com/lambda |
| Node.js docs | https://nodejs.org/api |

## Non-obvious rules
- Module system is decided by `package.json`'s `"type"` field plus the handler file's extension, not by the bundler config alone. Mixing `require()` and `import` in the same deployed bundle fails at cold-start invocation time, not at build time — the bundler often doesn't catch it.
- AWS SDK v2 ships pre-installed inside the Lambda Node.js runtime image, so it doesn't need to be packaged — but AWS SDK v3 does NOT ship in the runtime and must be included in the deployment artifact. Assuming v3 behaves like v2 here causes a "works locally, missing module in Lambda" failure.
- Cold start latency scales with bundle size and the amount of code executed at module scope (import-time work), not just total dependency count. Tree-shake with esbuild/webpack and prefer modular `@aws-sdk/client-*` v3 imports over importing the whole SDK.
- Lambda response streaming (`awslambda.streamifyResponse`) requires a Function URL configured with `InvokeMode: RESPONSE_STREAM`, or a compatible invoke path — it is not available transparently behind API Gateway REST or the older HTTP API integration the same way.
- Execution context reuse means module-scope state (DB connections, SDK clients) persists across warm invocations of the same container — initialize expensive clients outside the handler for reuse, but the handler must still work correctly on a cold start where that state doesn't exist yet.
- `provider.environment` values in `serverless.yml` are baked in at deploy time; they are not a live runtime config store — changing a value requires a redeploy, not just an environment update.
- Timeout and memory are independent knobs with real cost/perf tradeoffs: raising memory also raises allocated CPU proportionally in Lambda, so a timeout issue is sometimes fixed by more memory, not a longer timeout.

## Production checklist
- [ ] Bundle minified and tree-shaken (esbuild or equivalent), verified under the Lambda package size limit
- [ ] `package.json` `"type"` and handler file extension consistent (no mixed CJS/ESM in one function)
- [ ] AWS SDK v3 modular clients used where v3 semantics are needed; v2 not assumed to be present unless explicitly bundled
- [ ] Memory and timeout tuned per function based on measured invocations, not left at framework defaults
- [ ] Tracing (X-Ray or equivalent) enabled on production functions
- [ ] Dead-letter queue or on-failure destination configured for async-invoked functions
- [ ] Secrets sourced from Secrets Manager/SSM Parameter Store, not plaintext `environment:` values

## Never
- Never mix `require()` and `import` within the same deployed Lambda bundle.
- Never assume module-scope state exists on every invocation — code must be correct on a cold start too.
- Never package the entire AWS SDK v2 when only v3 modular clients are used.
- Never leave a production function at the framework's default timeout/memory without measuring actual usage.
