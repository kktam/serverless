# Traditional Serverless Framework (`@serverless/framework`)

The `@serverless/framework` package implements the classic Serverless Framework plugin architecture. It is invoked by the `TraditionalRunner` in sf-core when a `serverless.yml` config is detected.

---

## Core Class: `Serverless` (`lib/serverless.js`)

The `Serverless` class is the top-level orchestrator:

```javascript
const serverless = new Serverless({
  commands: ['deploy'],
  options: { stage: 'prod', region: 'us-east-1' },
  service: null,  // populated by sf-core
  configuration: { /* parsed serverless.yml */ },
  appConfig: { orgId: '...', userId: '...' },
  credentialProviders: { /* from sf-core auth */ },
})
```

### Lifecycle

1. **`init()`** — Creates CLI, loads service config, loads all plugins.
2. **`run()`** — Validates command and service config, triggers the plugin lifecycle via `PluginManager.run()`.

---

## Plugin System Architecture

### PluginManager (`lib/classes/plugin-manager.js`, 1072 lines)

The PluginManager is the backbone. It:

1. **Loads ~55 internal (bundled) plugins** — all statically imported at the top of the file.
2. **Loads external plugins** — from the service's `node_modules` or local paths. Supports TypeScript plugins via `tsx`.
3. **Registers commands** from all plugins into a command map.
4. **Registers hooks** — lifecycle event → handler function maps.
5. **Runs the lifecycle** when a command is invoked.

### Plugin Structure

Each plugin is a class with:

```javascript
class MyPlugin {
  constructor(serverless, options) {
    this.serverless = serverless
    this.options = options

    this.commands = {
      myCommand: {
        usage: 'Description',
        lifecycleEvents: ['event1', 'event2'],
        options: { /* yargs-like option definitions */ },
      },
    }

    this.hooks = {
      'before:myCommand:event1': () => { /* ... */ },
      'myCommand:event1': () => { /* ... */ },
      'after:myCommand:event1': () => { /* ... */ },
    }

    this.provider = 'aws'  // only load for AWS provider
  }
}
```

### Lifecycle Execution

When `PluginManager.run(commandsArray)` is called:

1. Run all `initialize` hooks.
2. For each command:
   - Fire `before:<cmd>:<event>` hooks.
   - Fire `<cmd>:<event>` hooks ("at" hooks, the default).
   - Fire `after:<cmd>:<event>` hooks.
3. On error → fire all `error` hooks.
4. Always → fire all `finalize` hooks.

**Nested lifecycles:** Plugins use `this.serverless.pluginManager.spawn('aws:deploy:deploy')` to invoke sub-lifecycles. The `deploy` command spawns the `aws:deploy:deploy` lifecycle, which itself has nested steps: `createStack`, `checkForChanges`, `uploadArtifacts`, `validateTemplate`, `updateStack`.

### Commands Hierarchy

Commands can be nested:
```
deploy
├── deploy function
└── deploy list
    └── deploy list functions
```

**Entrypoint commands** (`type: 'entrypoint'`) are invisible to the CLI but callable via `spawn()`. This is how internal AWS lifecycle steps are structured.

### Built-in Plugin Overrides

Several formerly-community plugins are now built-in. The framework skips the internal version if the service explicitly lists the community version in `plugins`:

- `serverless-python-requirements` → built-in `python` plugin
- `serverless-appsync-plugin` → built-in `appsync` plugin
- `serverless-apigateway-service-proxy` → built-in
- `serverless-prune-plugin` → built-in `prune` plugin

---

## AWS Provider (`lib/plugins/aws/provider.js`, 4232 lines)

The `AwsProvider` is registered via `serverless.setProvider('aws', this)`.

### Responsibilities

| Area | Details |
|---|---|
| **Credentials** | Resolves via `credentialProviders.aws` (delegated from sf-core) |
| **SDK Management** | Provides `this.sdk` (AWS SDK v2) for legacy plugin compatibility, plus v3 via conditional path |
| **Config Schema** | Defines the entire AWS-specific JSON Schema via `configSchemaHandler.defineProvider()` |
| **Naming** | Delegates to `lib/plugins/aws/lib/naming.js` (964 lines) — generates all CloudFormation logical IDs |
| **Service Methods** | `request()`, `getCredentials()`, `getStage()`, `getRegion()`, `getAccountId()`, `getServerlessDeploymentBucketName()` |

### Dual SDK Request System (`lib/aws/request.js`)

Both SDK implementations are maintained and selected at runtime via `SLS_AWS_SDK=3`:

| Aspect | v2 Path (`lib/aws/v2/`) | v3 Path (`lib/aws/v3/`) |
|---|---|---|
| SDK | `aws-sdk` v2 | `@aws-sdk/*` modular |
| Client creation | Global SDK config | Dynamic factory per service |
| Retry | Exponential backoff (4 retries max) | `@smithy/util-retry` |
| Concurrency | Request queue (limit 2) | N/A (handled by callers) |
| Proxy | HTTPS_PROXY, TLS certs | Via `@smithy/node-http-handler` |
| Memoization | Yes | No (command pattern) |
| S3 Transfer Acceleration | Yes | N/A |

### Config Schema Registration

`AwsProvider.constructor()` calls `configSchemaHandler.defineProvider('aws', {...})` with a massive JSON Schema that covers:

- Lambda function configuration (handler, runtime, memory, timeout, VPC, environment, etc.)
- IAM role definitions (managed policies, statements, permissions boundary)
- API Gateway (REST, HTTP, WebSocket)
- Event source mappings (23 types)
- Custom domains, CloudFront, ALB
- CloudFormation intrinsic functions (`Fn::GetAtt`, `Ref`, `Fn::Sub`, etc.)
- IAM policy document structure

Individual event plugins further register their schemas via `configSchemaHandler.defineFunctionEvent('aws', 'schedule', {...})`.

---

## Deploy Pipeline

The deploy lifecycle (`lib/plugins/aws/deploy/index.js`) orchestrates:

```
before:deploy:deploy
  ├── aws:common:validate — validates config, sets up temp dirs
  ├── setup observability integration (Axiom/Dashboard)
  └── optionally spawns "package" lifecycle if not already packaged

deploy:deploy → spawns aws:deploy:deploy:
  ├── createStack — check stack existence, create if missing
  ├── checkForChanges — compare local artifact hashes vs S3 objects
  ├── uploadArtifacts — upload CloudFormation template, state, zip files
  ├── validateTemplate — validate via AWS API
  ├── updateStack — update (or fallback to create) via direct or change-set method
  └── monitorStack — poll describeStackEvents, show progress

deploy:finalize → spawns aws:deploy:finalize:
  └── cleanup — S3 bucket cleanup
```

**Two deployment methods:**
- **Direct** (default) — `createStack` / `updateStack` API calls.
- **Change Sets** — `createChangeSet` → poll for readiness → `executeChangeSet`. Supports custom stack policies.

**Deployment skipping:** If no changes detected (same artifact hashes), the update step is skipped entirely.

---

## Package Compile Pipeline

The AWS package plugin (`lib/plugins/aws/package/index.js`) compiles the service config into CloudFormation resources:

```
package:cleanup
package:initialize
package:setupProviderConfiguration
package:createDeploymentArtifacts
package:compileLayers         → compile/layers.js
package:compileFunctions      → compile/functions.js (1473 lines)
package:compileEvents         → 23 individual event compile plugins
package:compileCapacityProviders → capacity-providers.js
package:finalize
```

### Event Compile Plugins

Each handles one event source type, reads function events from the service config, and generates CloudFormation resources:

| Plugin | AWS Resource |
|---|---|
| `schedule.js` | `AWS::Events::Rule` or `AWS::Scheduler::Schedule` |
| `s3/` | `AWS::S3::Bucket` + Lambda notification config |
| `api-gateway/` | `AWS::ApiGateway::RestApi`, resources, methods, integrations |
| `websockets/` | `AWS::ApiGatewayV2::Api` |
| `http-api.js` | `AWS::ApiGatewayV2::Api` (HTTP API variant) |
| `sns.js` | `AWS::SNS::Topic` + subscription |
| `sqs.js` | `AWS::SQS::Queue` (event source mapping) |
| `stream.js` | DynamoDB/Kinesis stream event source mapping |
| `kafka.js` | MSK/Self-managed Kafka event source |
| `activemq.js`, `rabbitmq.js` | MQ broker event sources |
| `msk/` | MSK cluster event source |
| `alb/` | ALB target group + listener rules |
| `event-bridge/` | EventBridge event bus rules |
| `cloud-watch-event.js` | CloudWatch Events rule |
| `cloud-watch-log.js` | CloudWatch Logs subscription filter |
| `cognito-user-pool.js` | Cognito user pool trigger |
| `cloud-front.js` | CloudFront distribution + behaviors |
| `iot.js` | IoT rule |
| `iot-fleet-provisioning.js` | IoT fleet provisioning |
| `alexa-skill.js`, `alexa-smart-home.js` | Alexa skill triggers |

Each compile plugin:
1. **Iterates** over `this.serverless.service.getFunction(functionName).events`.
2. **Validates** config against registered JSON Schema.
3. **Generates** CloudFormation resource entries into `this.serverless.service.provider.compiledCloudFormationTemplate.Resources`.

---

## Built-in Esbuild Plugin (`lib/plugins/esbuild/index.js`, 1234 lines)

The default TypeScript/JS bundler hooks into:

- `before:package:createDeploymentArtifacts` — builds and packages
- `before:deploy:function:packageFunction` — builds single function
- `before:invoke:local:invoke` — builds for local invocation
- `before:dev-build:build` — builds for dev mode
- `before:offline:start` — builds for serverless-offline

Conflicts with `serverless-webpack`, `serverless-plugin-typescript`, `serverless-bundle`, or `serverless-esbuild` (throws error unless `build.esbuild: false`).

---

## Configuration Model (`lib/classes/service.js`)

The `Service` class manages the user's config:

**Loading flow:**
1. `configuration/read.js` parses config (supports `.yml`, `.yaml`, `.json`, `.toml`, `.ts`, `.js`, `.cjs`, `.mjs`). TS files use `tsx` for transpilation.
2. `Service.load()` populates: `service`, `provider`, `custom`, `plugins`, `functions`, `resources`, `package`, `layers`, `outputs`, `configValidationMode`.
3. `Service.reloadServiceFileParam()` re-reads after variable resolution.
4. `Service.validate()` runs AJV-based validation.

**Key properties:**
- `service` — service name
- `provider` — provider config (name, stage, region, stack tags, etc.)
- `functions` — map of function name → { handler, events, environment, layers, etc. }
- `resources` — raw CloudFormation resource definitions (extensions)
- `layers` — Lambda Layer definitions
- `custom` — user-defined variable namespace
- `package` — packaging config (patterns, artifact, individually)

---

## CLI Commands

| Command | Lifecycle Events | Service Required |
|---|---|---|
| `print` | `[print]` | Yes |
| `help` | `[]` | Optional |
| `package` | `[cleanup, initialize, setupProviderConfiguration, createDeploymentArtifacts, compileLayers, compileFunctions, compileEvents, finalize]` | Yes |
| `deploy` | `[deploy, finalize]` | Yes |
| `deploy function` | `[initialize, packageFunction, deploy]` | Yes |
| `deploy list` | `[log]` | Yes |
| `deploy list functions` | `[log]` | Yes |
| `info` | `[info]` | Yes |
| `diff` | `[diff]` | Yes |
| `dev` | `[dev]` | Yes |
| `invoke` | `[invoke]` | Yes |
| `invoke local` | `[loadEnvVars, invoke]` | Yes |
| `logs` | `[logs]` | Yes |
| `metrics` | `[metrics]` | Yes |
| `remove` | `[remove]` | Yes |
| `rollback` | `[initialize, rollback]` | Yes |
| `rollback function` | `[rollback]` | Yes |
| `prune` | `[prune]` | Yes |

---

## Key Design Patterns

1. **Object.assign() composition** — Many AWS plugin classes use `Object.assign(this, { ...libModule })` to mix in behavior from separate lib files. Keeps individual files small but is non-idiomatic class design.

2. **Plugin lifecycle composition** — Outer commands spawn inner provider-specific lifecycles via `spawn()`. This allows provider-agnostic command definitions to coexist with provider-specific implementations.

3. **AJV-based config validation** — The `config-schema-handler` builds a cumulative JSON Schema from all provider and event plugin registrations, then validates the user's config against it.

4. **CloudFormation resource compilation** — The framework generates a complete CloudFormation template from the service config, adding Lambda functions, IAM roles, event sources, log groups, and custom resources. The user can extend via `resources` in their config.
