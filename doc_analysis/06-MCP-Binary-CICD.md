# MCP Server, Binary Installer, and CI/CD

---

## MCP Server (`@serverless/mcp`)

The MCP (Model Context Protocol) server bridges AI-powered IDEs (Cursor, Windsurf, Claude Desktop) with deployed AWS infrastructure. It is a **read-only diagnostic gateway** — no write/modify capability.

### Entry Points

| Mode | File | Transport | Use Case |
|---|---|---|---|
| SSE | `src/server.js` | HTTP SSE on `127.0.0.1:3001` | Remote/network access |
| stdio | `src/stdio-server.js` | stdin/stdout | Local IDE integration |

### Connection to Engine and Framework

```javascript
// Deep imports from @serverless/engine for AWS clients
import { AwsCloudformationService } from '@serverless/engine/src/lib/aws/cloudformation.js'
import { AwsLambdaClient } from '@serverless/engine/src/lib/aws/lambda.js'
// ... 18 more engine client imports

// From @serverless/framework for config parsing
import { readConfig } from '@serverless/framework/lib/configuration/read.js'
```

### 15 MCP Tools

**Project Discovery:**
| Tool | Purpose |
|---|---|
| `list-projects` | Scans workspace for Serverless/CFN/SAM projects. Requires `userConfirmed=true`. |
| `list-resources` | Lists deployed AWS resources via CloudFormation. Requires prior `list-projects` call. |
| `service-summary` | One-call consolidated view of multiple resource types. Supports auto-discovery. |
| `deployment-history` | CloudFormation stack events (last 7 days), grouped by day. |

**AWS Resource Diagnostics (7 tools):**
| Tool | What it fetches |
|---|---|
| `aws-lambda-info` | Config, metrics (Invocations, Errors, Throttles, Duration), error logs |
| `aws-iam-info` | Trust policies, managed/inline policies (full JSON) |
| `aws-sqs-info` | Queue attributes, metrics (ApproximateNumberOfMessages, Sent, Received) |
| `aws-s3-info` | Bucket config (ACL, policy, CORS, encryption, versioning, lifecycle, logging) |
| `aws-rest-api-gateway-info` | Stages, resources, methods, integrations, deployments, API keys |
| `aws-http-api-gateway-info` | Routes, integrations, stages, authorizers, deployments |
| `aws-dynamodb-info` | Table config (throughput, keys, GSI/LSI, TTL, PITR), metrics |

**Logs & Monitoring (4 tools):**
| Tool | Cost | Description |
|---|---|---|
| `aws-logs-search` | ~$0.005/GB scanned | CloudWatch Logs Insights SQL-like queries. Confirmation gate for >3h. |
| `aws-logs-tail` | Free | Recent logs via FilterLogEvents. Default 15min lookback. |
| `aws-errors-info` | Variable | Pattern-based error aggregation with similarity grouping. |
| `aws-cloudwatch-alarms` | Free | Alarm configs, states, state-change history. |

**Documentation:**
| Tool | Description |
|---|---|
| `docs` | Read Serverless Framework docs from `docs/sf/` and `docs/scf/` |

### Key Design Features

1. **Cost awareness** — CloudWatch Logs Insights queries have a confirmation gate: the tool returns a token, and the agent must get user approval before proceeding. Approved once per session.

2. **Workflow enforcement** — `list-resources` refuses to run unless `list-projects` was called first (module-level flag).

3. **User confirmation gates** — `list-projects` requires explicit `userConfirmed=true`.

4. **Parallel data fetching** — All resource-info modules fetch config + metrics + error logs concurrently via `Promise.all`.

5. **Period optimization** — `parameter-validator.js` auto-adjusts CloudWatch metric periods (60s for short ranges → 1 day for very long ranges) to keep data points under 300.

6. **Error pattern grouping** — `errors-info-patterns.js` uses similarity matching to group related errors for pattern-level analysis.

7. **DNS-rebinding protection** — SSE server validates Host header against `localhost`, `127.0.0.1`, `[::1]`.

8. **Credential error handling** — `aws-credentials-error-handler.js` produces structured messages for 7 error categories (expired tokens, missing creds, access denied, invalid tokens, region missing, profile not found, SSO failures).

---

## Shared Utilities (`@serverless/util`)

Provides cross-package plumbing:

| Module | Purpose |
|---|---|
| `log` | Logging with level support (debug, info, warn, error) |
| `progress` | Spinner/progress bar management |
| `style` | Terminal styling (chalk-based) |
| `errors` | Error formatting and presentation |
| `aws` | AWS proxy config (`addProxyToAwsClient()`) |
| `docker` | Dockerode-based Docker client operations |
| `state` | State store utilities |
| `uuid` | UUID generation |
| `random` | Random string generation |

---

## Standards (`@serverlessinc/standards`)

Shared ESLint and Prettier configurations:

- **ESLint** (`src/eslint.js`) — Extends `@eslint/js` recommended config, adds `eslint-config-prettier` for integration.
- **Prettier** (`src/prettier.js`) — No semicolons, single quotes, 2-space indent, trailing commas, print width 120, LF line endings.

---

## Binary Installer (`binary-installer/main.go`)

The Go binary is the end-user `serverless` CLI entry point. It is a **version manager + launcher**.

### Responsibilities

1. **First-run setup** — Creates `~/.serverless/binaries/` directory.

2. **Config resolution** — Scans cwd for config files: `serverless.yml`, `serverless-compose.yml`, `serverless.containers.yml`, `serverless.ai.yml`, plus `.js/.ts/.json` variants.

3. **Version management** — Determines which framework version to run:
   - Reads `frameworkVersion` from config.
   - Checks for updates via `update` command or `SERVERLESS_FRAMEWORK_FORCE_UPDATE`.
   - Downloads correct framework tarball from S3.

4. **Self-update** — `serverless update` command downloads new binary from S3 (`install.serverless.com/installer-builds/`), replaces itself atomically: `.new` → current file, with `.old` cleanup.

5. **V3 backward compatibility** — Detects `node_modules/serverless` v3 and delegates to the legacy Node.js runner.

6. **Child process management** — Downloads the correct framework tarball, extracts to `package/`, then spawns `node package/dist/sf-core.js`.

7. **Platform detection** — Supports `amd64` and `arm64` on Linux/macOS; Windows supports `amd64` only. Requires Node.js 18+.

### Conditional Compilation

Two build variants controlled by Go build tags:

| File | Build Tag | Base URL |
|---|---|---|
| `install_base_url.go` | (default) | `install.serverless.com` |
| `install_base_url_canary.go` | `canary` | `install.serverless-dev.com` |

Users opt into canary via `frameworkVersion: canary` in `serverless.yml`.

### Installation

Users can install via:
```bash
curl -o- -L https://install.serverless.com | bash
# or
npm i serverless -g
```

The npm package (`sf-core-installer`) is a thin wrapper that downloads the Go binary on `postinstall`.

---

## Release Process (`release-scripts/`)

Two scripts manage release metadata:

| Script | File | Purpose |
|---|---|---|
| `publish:release` | `scripts/publishReleaseToMongo.js` | Inserts release record + updates `supportedVersions` in MongoDB |
| `publish:release-metadata` | `scripts/publishReleaseMetadataToS3.js` | Fetches `versions.json` from S3, appends new version, writes back |

Dependencies: `@aws-sdk/client-s3`, `mongodb`.

### Release Pipeline (GitHub Actions)

**Trigger:** Push to `main` touching `packages/`

1. **Engine tests** — Unit tests for `@serverless/engine`.
2. **Cross-platform integration tests** — Linux, Windows, ARM.
3. **Canary release** — Upload tarballs to canary S3 (`install.serverless-dev.com`) with Git SHA version. Tags repo if new SemVer detected.
4. **Production release** — Upload to production S3 (`install.serverless.com`), update MongoDB metadata, invalidate CloudFront.
5. **npm publish** — Publish `sf-core-installer` to npm.

---

## CI/CD Workflows

| Workflow File | Trigger | What It Does |
|---|---|---|
| `check-cla.yml` | PR comments, PR events | Verifies CLA signatures via `serverlessinc/cla` |
| `ci-engine.yml` | PRs to `main` touching `packages/engine/` | Engine unit tests (Ubuntu, Node.js 24.x) |
| `ci-framework.yml` | PRs to `main` (excluding docs) | ESLint + Prettier → engine unit tests → framework unit + integration + resolver tests |
| `ci-binary-installer.yml` | PRs touching `binary-installer/` | Go test + build |
| `ci-python.yml` | PRs touching Python plugin code | Python requirements bundling (Ubuntu + Windows, Python 3.13) |
| `release-framework.yml` | Push to `main` touching `packages/` | Full release: test → canary → production → npm |
| `release-binary-installer.yml` | Manual (`workflow_dispatch`) | Build Go binary for all platforms, upload to production S3 |

### IAM Roles Used

| Role | Purpose |
|---|---|
| `arn:aws:iam::762003938904:role/GithubActionsDeploymentRole` | CI testing (integration + resolver tests) |
| `arn:aws:iam::377024778620:role/GithubActionsPublicServerlessRepoAccessRole` | Canary releases |
| `arn:aws:iam::802587217904:role/GithubActionsPublicServerlessRepoAccessRole` | Production releases + binary installer |

---

## Versioning and Security

| Policy | Details |
|---|---|
| **Versioning** | Strict SemVer. PATCH = bug fixes, MINOR = new features, MAJOR = any breaking change. |
| **Node.js support** | ≥18.0 for `@serverless/framework`, ≥18.17 for installer. |
| **Security reports** | Via GitHub Security Advisories (not public issues). Triage within 3 business days. |
| **V.4 support** | All issues addressed promptly. |
| **V.3 support** | Critical issues only through end of 2024. |
| **V.2 and earlier** | No longer supported. |

---

## Testing Strategy

| Layer | Scope | Requirements | Run In |
|---|---|---|---|
| **Unit tests** | Isolated module logic | None | Local + CI |
| **Resolver tests** | Variable resolution (golden files) | None | Local + CI |
| **Integration tests** | Full deploy/remove/invoke cycles | AWS credentials, Dashboard access, Terraform Cloud | CI only |
| **MCP unit tests** | Tool handlers, AWS resource info | None | Local + CI |
| **MCP e2e tests** | End-to-end with live AWS | AWS credentials | CI only |
| **Engine unit tests** | Zod schemas, AWS client wrappers | None | Local + CI |
| **Engine e2e tests** | Full container deploy/remove | AWS credentials | CI only |

### Test Configuration

- **Jest** with `--experimental-vm-modules` for native ESM support.
- **No transforms** (`transform: {}` in Jest config).
- **Integration test timeout:** 600 seconds (10 minutes).
- **Mocking pattern:** `jest.unstable_mockModule('@serverless/util', () => ({...}))` for ES module mocking.
- **AWS SDK mocking:** `jest.fn()` with `.memoized` property for the v2 request layer; `mockSdkClient.send.mockResolvedValueOnce(...)` for v3.
