# Serverless Framework V.4 — Project Overview

## What It Is

The **Serverless Framework V.4** is a CLI tool for deploying serverless applications to AWS Lambda and other managed cloud services. Users define infrastructure in YAML (`serverless.yml`), and the framework compiles that into CloudFormation templates, packages code, uploads artifacts to S3, and orchestrates the deployment lifecycle.

V.4 introduced major features: Managed Instances, Durable Functions, built-in AppSync & Prune plugins, AWS SSO support, native TypeScript with esbuild, HTTP response streaming, per-function IAM roles, and an MCP server for AI-powered IDEs.

---

## Repository Structure

```
serverless/                         # Monorepo root (npm workspaces)
├── packages/
│   ├── sf-core/                    # Main CLI framework (wraps everything)
│   │   ├── bin/sf-core.js          # CLI entry point
│   │   ├── src/                    # Runner system, resolver, auth, frameworks
│   │   └── tests/                  # Unit + integration tests
│   ├── serverless/                 # Core framework (legacy plugin engine)
│   │   ├── lib/serverless.js       # Serverless class entry point
│   │   ├── lib/plugins/            # ~55 bundled plugins (deploy, invoke, logs, etc.)
│   │   └── test/                   # Unit tests
│   ├── engine/                     # Container framework engine
│   │   ├── src/index.js            # ServerlessEngine class
│   │   ├── src/lib/deploymentTypes/ # Strategy pattern (aws, awsApi, sfaiAws)
│   │   └── test/                   # Unit + integration tests
│   ├── mcp/                        # MCP server for AI IDEs
│   │   ├── src/server.js           # SSE server
│   │   ├── src/stdio-server.js     # stdio server
│   │   └── src/tools/              # 15 MCP tools (lambda, logs, iam, etc.)
│   ├── util/                       # Shared utilities (logging, progress, AWS proxy)
│   ├── standards/                  # ESLint + Prettier configs
│   ├── sf-core-installer/          # npm wrapper that downloads the binary
│   ├── framework-dist/             # Built distribution artifacts (excluded from workspaces)
│   └── serverless/                 # Separately published npm wrapper (v4.38.1)
├── binary-installer/               # Go binary that manages framework versions
├── release-scripts/                # MongoDB + S3 release metadata publishing
├── docs/                           # User-facing documentation
└── .github/workflows/              # CI/CD: 7 workflow files
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Runtime** | Node.js 18+ (ES Modules everywhere) |
| **CLI Parsing** | yargs |
| **Package Manager** | npm workspaces |
| **Testing** | Jest (native ESM with `--experimental-vm-modules`) |
| **Linting** | ESLint + Prettier (no semicolons, single quotes, 2-space indent) |
| **Build** | esbuild (bundles CLI into single file) |
| **Schema Validation** | Zod v4 (engine configs), AJV v8 (serverless configs), Joi (legacy) |
| **Graph/Resolution** | @dagrejs/graphlib (DAG-based variable resolution) |
| **AWS SDK** | Dual-path: v2 (`aws-sdk`) and v3 (`@aws-sdk/*` at 3.1057.x) |
| **CloudFormation Diff** | @aws-cdk/cloudformation-diff |
| **Binary Installer** | Go (platform detection, self-update, version management) |

---

## Package Dependency Graph

```
binary-installer (Go)
    └── spawns ──> sf-core (built artifact)
                       ├── @serverless/framework (traditional)
                       ├── @serverless/engine (container/AI framework)
                       ├── @serverless/mcp
                       └── @serverless/util
```

- **`sf-core`** is the orchestrator: it reads the config file, selects the right runner, resolves variables, handles auth, and delegates to the appropriate engine.
- **`@serverless/framework`** is used when the config is a traditional `serverless.yml` (the legacy plugin architecture).
- **`@serverless/engine`** is used when the config is `serverless.containers.yml` (SCF) or `serverless.ai.yml` (SAI).
- **`@serverless/mcp`** is a standalone SSE/stdio server that provides AWS resource diagnostics to AI agents.
- **`@serverless/util`** provides shared plumbing: logging, progress spinners, AWS proxy config, Docker client, state store utilities.

---

## Key Architectural Patterns

1. **Strategy Pattern (Runners):** The CLI router selects among 6 runner classes based on config file detection. Each runner encapsulates a different deployment workflow (traditional, compose, SAM/CFN, SCF, SAI).

2. **Dependency Graph Resolution:** All `${source:key}` variables in config files are resolved as a DAG. Sink nodes (no dependents) are processed first in parallel, and the graph unwinds until all variables are resolved.

3. **Plugin Lifecycle Composition (Traditional):** The legacy framework uses a layered lifecycle — outer commands (`deploy:deploy`) spawn inner AWS-specific lifecycles (`aws:deploy:deploy:createStack`) via `pluginManager.spawn()`.

4. **Dual AWS SDK Support:** Both SDK v2 (`aws-sdk` v2) and v3 (`@aws-sdk/*`) are supported, selected at runtime via `SLS_AWS_SDK=3`.

5. **Observability as Pluggable Provider:** Observability (logging/metrics/tracing for deployed functions) can be Axiom, Serverless Dashboard, or disabled — selected via config.

6. **Container Framework State Machine:** The engine tracks deployment state (VPC, ALB, ECS, Lambda, CloudFront) with incremental updates and zero-downtime compute migration (Lambda ↔ Fargate).

---

## License Notes

V.4 introduced a license change. The npm module contains some proprietary-licensed software. The repository will be restructured to separate MIT-licensed open-source from proprietary components. Authentication is required within the CLI for V.4. Non-AWS providers have been deprecated.
