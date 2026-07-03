# Variable Resolution System

The resolver system handles `${source:key}` variable interpolation in YAML config files. This is one of the most architecturally complex subsystems — it processes a DAG of variable dependencies in parallel.

---

## Architecture Overview

```
src/lib/resolvers/
├── index.js          # createResolverManager() — factory + bootstrap
├── manager.js        # ResolverManager class (1357 lines)
├── graph.js          # processGraphInParallel() — DAG parallel executor
├── placeholders.js   # Parse ${...} placeholders from YAML
├── env.js            # .env file loading
├── validation.js     # Config validation
├── providers.js      # Provider abstraction layer
├── providers/        # Concrete provider implementations
│   ├── index.js           # AbstractProvider base class
│   ├── aws/aws.js         # AWS resolver (delegates to cf, s3, ssm)
│   ├── aws/cf.js          # CloudFormation stack output
│   ├── aws/s3.js          # S3 file content
│   ├── aws/ssm.js         # SSM Parameter Store
│   ├── aws/credentials.js # AWS credential resolution
│   ├── env/env.js         # Environment variables
│   ├── opt/opt.js         # CLI options
│   ├── file/file.js       # Local file content
│   ├── self/self.js       # Self-referencing (other config properties)
│   ├── sls/sls.js         # Framework built-in variables
│   ├── param/param.js     # Param store (stages)
│   ├── output/output.js   # Cross-service outputs (compose)
│   ├── git/git.js         # Git metadata
│   ├── str-to-bool/       # String-to-boolean conversion
│   ├── vault/vault.js     # Vault secrets
│   ├── terraform/         # Terraform state outputs
│   └── doppler/           # Doppler secrets
└── registry/         # Plugin-contributed resolver registration
```

---

## Bootstrap Sequence (`createResolverManager()`)

```
1. Validate config
   ├─ No params+stages together
   ├─ Valid resolver configs (unique names)
   └─ Config file exists

2. Create ResolverManager instance

3. Load placeholders from config
   └─ Build dependency graph (nodes = placeholders, edges = dependencies)

4. Resolve stage
   └─ From: CLI --stage option → env SLS_STAGE → provider.stage → "dev"

5. Load .env files
   └─ .env → .env.{stage} → .env.{stage}.local

6. Resolve provider.resolver key
   └─ The provider's own resolver configuration

7. Set credential resolver
   └─ Add the AWS credential resolver to the graph

8. Prune unused stages from config
   └─ Remove all stages except the active one

9. Reload placeholders with credential resolver edges
   └─ Add edges from every placeholder to the credential resolver

10. If ComposeRunner:
    └─ Load all required resolvers from referenced services
```

---

## Dependency Graph Processing (`processGraphInParallel()`)

Located in `src/lib/resolvers/graph.js`.

### Algorithm

```text
1. Build adjacency lists from the graphlib Graph
2. Find sink nodes = nodes with 0 outgoing edges (no one depends on them)
3. Process sink nodes in parallel (Promise.all)
4. When a node completes, remove its edges → new sinks may emerge
5. Repeat until all nodes resolved
```

- Uses a `Set` of Promises with notification-based wakeup.
- Concurrent processing of independent variables.
- The graph is directed: an edge from A→B means "A depends on B", so B must be resolved before A. Sinks are nodes that no one depends on (they have outgoing edges to their dependents).

### Example

```yaml
# serverless.yml
custom:
  bucketName: ${self:service}-${sls:stage}-bucket
  sourceCode: ${file(./config.js):exports.getCode}
  tableName: ${env:TABLE_PREFIX}-${self:service}
```

This produces a graph where:
- `env:TABLE_PREFIX` and `self:service` and `sls:stage` are sinks (no dependencies)
- `tableName` depends on `env:TABLE_PREFIX` and `self:service`
- `bucketName` depends on `self:service` and `sls:stage`
- `sourceCode` depends on `file(./config.js):exports.getCode`

Sinks `env:TABLE_PREFIX`, `self:service`, `sls:stage` resolve in parallel first, then the dependent expressions resolve.

---

## Provider System

### AbstractProvider Base Class (`providers/index.js`)

Every provider extends this base and implements:

| Method | Purpose |
|---|---|
| `resolve({ key, region, address })` | Resolve a single variable value |
| `writeable` (getter) | Whether this provider supports write operations |
| `write({ key, value, region, address })` | Write a value (if writeable) |

### Provider Hierarchy

- **Local (no auth needed):** `env`, `opt`, `file`, `self`, `sls`, `strToBool`, `git`, `param`
- **AWS (requires credentials):** `aws:ssm`, `aws:s3`, `aws:cf` (CloudFormation outputs)
- **Secrets (requires external auth):** `vault`, `terraform`, `doppler`
- **Cross-service:** `output` (Compose — resolved from other services' outputs)

### Limited Providers Set (pre-auth)

Before authentication is complete, only a safe subset of providers is available for "limited" resolution:

```
['env', 'opt', 'file', 'sls', 'strToBool', 'git', 'self', 'param']
```

These don't need network access or credentials to resolve. They're used to determine org, app, service, region, and provider profile — which are needed to authenticate.

### Plugin Registry

Custom resolvers from external plugins register via `providerRegistry.register()`. The registry accepts any class that conforms to the `AbstractProvider` interface, making the resolution system extensible.

---

## Placeholder Parsing (`placeholders.js`)

Parses `${source:key}` from resolved YAML:

- Handles nested placeholders: `${env:${self:custom.envPrefix}_TABLE}`
- Handles chained resolvers: `${aws:ssm:/path/to/key}`
- Handles default values: `${env:MY_VAR, 'default'}`
- Extracts all unique placeholder references for graph construction
- Stringifies and decycles the resolved config to handle circular references

---

## Key Design Decisions

1. **DAG-based parallel resolution** — Independent variables resolve concurrently, significantly reducing cold-start time for services with many variables.

2. **Two-phase resolution** — Early (limited) providers resolve before authentication; full resolution happens after. This allows the system to determine auth requirements from the config itself.

3. **Stage pruning** — When a config defines multiple stages but only one is active, the resolver prunes unused stage branches to reduce graph complexity and avoid resolving variables for inactive stages.

4. **Credential edge injection** — After the credential resolver is identified, an edge is injected from EVERY placeholder to the credential resolver. This ensures credentials are valid before any AWS variable is resolved, providing early failure with clear error messages.

5. **Compose cross-service resolution** — The `output` resolver allows one service in a compose deployment to reference outputs from another service. This creates cross-service graph dependencies that are resolved in topological order.
