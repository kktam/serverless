# Serverless Container Framework Engine (`@serverless/engine`)

The engine is a **reusable, container-focused deployment orchestrator** extracted from the Serverless Container Framework (SCF). It is the deployment backend for the SCF and SAI runners in sf-core.

---

## Architecture Layers

```
src/
├── index.js                          # ServerlessEngine — public API
├── types.js                          # Zod schemas for config + state (1604 lines)
├── types/
│   ├── aws.js                       # AWS infrastructure schemas (412 lines)
│   ├── containers.js                # Container config schemas
│   └── integrations.js              # Slack, EventBridge schemas
├── lib/
│   ├── deploymentTypes/             # Strategy pattern
│   │   ├── aws/                     # Full deployment (CloudFront, Slack, AI)
│   │   ├── awsApi/                  # Simpler ALB-only API deployment
│   │   └── sfaiAws/                 # AI Framework variant
│   ├── devMode/                     # Local Docker dev mode
│   ├── aws/                         # 22 AWS client wrapper classes
│   ├── integrations/                # Slack integration
│   └── utils/                       # Shared utilities
└── test/                            # Unit tests
```

---

## ServerlessEngine Class (`src/index.js`)

### Constructor

```javascript
new ServerlessEngine({
  stateStore: { load, save },   // persistence backend
  projectConfig,                 // parsed serverless.containers.yml
  projectPath,                   // filesystem path
  stage,                         // deployment stage
  provider: 'aws',               // cloud provider (only aws supported)
})
```

### Validation

- `stateStore` must have `load()` and `save()` methods
- `stage` must be 3–12 characters
- `provider` must be `"aws"`

### Dispatch Logic

Based on `projectConfig.deployment.type`:

| `deployment.type` | Class | Features |
|---|---|---|
| `"aws@1.0"` | `DeploymentTypeAws` | Full: VPC, ALB, CloudFront, ECS, Lambda, Slack, EventBridge, AI detection |
| `"awsApi@1.0"` | `DeploymentTypeAwsApi` | Simpler: ALB-only, no CloudFront/Slack/AI |
| `"sfaiAws@1.0"` | `DeploymentTypeSfaiAws` | AI Framework: Slack OAuth, EventBridge schedules, AI manifest, ES testing |

### Lifecycle Methods

| Method | Description |
|---|---|
| `deploy({ force })` | Load state → delegate to deployment type → save state with `isDeployed=true` |
| `remove({ all, force })` | Load state → delegate → reset state to undeployed |
| `dev({ proxyPort, controlPort, onStart, onLogStdOut, onLogStdErr })` | Start local Docker-based dev mode |
| `getDeploymentState()` | Return current state from state store |
| `executeCommand(name, options)` | Route `deploy/dev/remove/info` or delegate custom commands |

---

## Deployment Type Strategy Pattern

Each deployment type class implements the same interface:

```typescript
interface DeploymentType {
  constructor(args: { state, projectConfig, projectPath, stage, provider, resourceNameBase })
  deploy({ force }): Promise<void>
  remove({ all, force }): Promise<void>
  dev(...): Promise<void>  // shared via ServerlessEngineDevMode
  executeCustomCommand(name, options): Promise<any>
}
```

### `aws@1.0` — Full Deployment

**Deploy orchestration** (`src/lib/deploymentTypes/aws/deploy.js`, 1143 lines):

```
1. Check Docker is running
2. Initialize 11 AWS clients (VPC, ALB, ECS, IAM, Lambda, ACM, ECR, Route53, CloudFront, CloudWatch, Autoscaling)
3. Generate scfForwardToken for routing
4. Deploy foundational infrastructure:
   ├── VPC (2 public + 2 private subnets, NAT gateways, route tables)
   ├── ALB (internet-facing, security groups, listeners)
   ├── ECS cluster (with capacity providers)
   ├── IAM roles (task execution, task, Lambda execution)
   ├── CloudFront distribution (origin = ALB, path-based forwarding)
   └── ACM certificate (request + validate)
5. Deploy containers (one at a time — concurrency 1 due to state races):
   ├── Build Docker image via `docker buildx`
   ├── Push to ECR
   ├── Route to Lambda (deployAwsLambda.js) OR
   ├── Route to Fargate ECS (deployAwsFargateEcs.js)
   └── Handle zero-downtime compute type migration if needed
6. Apply dynamic routing (CloudFront → ALB path-based rules)
7. Integrate Slack (channels, webhooks)
8. Integrate EventBridge (rules, targets)
```

**Compute types:**
- **Lambda** — Deploy as Lambda function with ALB trigger, supports provisioned concurrency
- **Fargate ECS** — Deploy as ECS service with ALB target group, supports autoscaling

**Zero-downtime compute migration:**
When switching a container from Lambda to Fargate (or vice versa), the engine:
1. Deploys the new compute type with `deprioritized` ALB routing priority
2. Verifies the new target is healthy
3. Swaps routing priority to the new target
4. Removes the old compute type

**AI Framework Detection** (`detector.js`):
Reads `package.json` dependencies to detect AI frameworks. Currently only supports Mastra.

### `awsApi@1.0` — Simplified API Deployment

Lightweight path without CloudFront, Slack, or AI integration. Deploys Lambda and/or Fargate ECS behind an ALB. Used for simpler API services.

### `sfaiAws@1.0` — Serverless AI Framework

Extends the AWS deployment with:
- **Slack app manifest** generation and OAuth installation flow
- **Socket-mode Slack** integration for dev mode
- **EventBridge integration** management (create, reconcile, schedules)
- **ES testing** — deployment testing feature
- **AI manifest validation**

---

## Dev Mode (`src/lib/devMode/index.js`, 1492 lines)

The `ServerlessEngineDevMode` class orchestrates local Docker-based development:

### Components

1. **Proxy container** (`serverless-dev-mode-proxy`) — Express-based proxy that routes HTTP requests to local runtime containers. Handles path-based routing, health checks, and hot reload.

2. **Runtime containers** — Per-language containers (Node.js 20, Python 3.12) with:
   - Hot-reload on code change (via `chokidar` file watcher)
   - Configurable image versions
   - Environment variable injection

3. **Shared Docker network** (`serverless-dev-mode-proxy-network`) — All containers communicate over this network.

4. **Control server** — HTTP server for lifecycle management (start/stop/restart individual containers).

### Flow

```
1. docker network create serverless-dev-mode-proxy-network
2. docker run serverless-dev-mode-proxy (port 8080)
3. For each local container:
   ├── docker build (if Dockerfile exists) or pull runtime image
   ├── docker run with mounted source code
   └── register route in proxy
4. chokidar watches for file changes → restarts affected containers
5. On exit: docker stop + docker rm all containers
```

---

## AWS Client Layer (`src/lib/aws/`)

22 thin wrapper classes around AWS SDK v3, each following the same pattern:

| Client | File | Key Methods |
|---|---|---|
| `AwsVpcClient` | `vpc.js` | `getOrCreateVpc()`, `deleteVpc()` |
| `AwsAlbClient` | `alb.js` | `getOrCreateLoadBalancer()`, `getOrCreateTargetGroup()`, `getOrCreateListenerRule()` |
| `AwsAutoscalingClient` | `autoscaling.js` | `registerScalableTarget()`, `putScalingPolicy()` |
| `AwsCloudformationClient` | `cloudformation.js` | `createStack()`, `describeStacks()`, `deleteStack()` |
| `AwsCloudfrontClient` | `cloudfront.js` | `getOrCreateDistribution()`, `updateDistribution()`, `deleteDistribution()` |
| `AwsCloudfrontFunctionClient` | `cloudfront-function.js` | Manage CloudFront Functions for routing |
| `AwsCloudfrontKvClient` | `cloudfront-kv.js` | CloudFront KeyValueStore for routing |
| `AwsCloudwatchClient` | `cloudwatch.js` | Metrics, alarms |
| `AwsDynamodbClient` | `dynamodb.js` | Table operations |
| `AwsEcrClient` | `ecr.js` | `getOrCreateRepository()`, `getAuthorizationToken()` |
| `AwsEcsClient` | `ecs.js` (845 lines) | `getOrCreateCluster()`, `createOrUpdateService()`, `registerTaskDefinition()` |
| `AwsEventbridgeClient` | `eventbridge.js` | Event rules, targets |
| `AwsHttpApiGatewayClient` | `http-api-gateway.js` | HTTP API management |
| `AwsIamClient` | `iam.js` | Roles, policies |
| `AwsLambdaClient` | `lambda.js` | Functions, versions, aliases, provisioned concurrency |
| `AwsRestApiGatewayClient` | `rest-api-gateway.js` | REST API management |
| `AwsRoute53Client` | `route53.js` | DNS records |
| `AwsS3Client` | `s3.js` | Bucket operations |
| `AwsSqsClient` | `sqs.js` | Queue operations |
| `AwsSsmClient` | `ssm.js` | Parameter store |
| `AwsCloudfrontRoutingFunction` | `cloudfront-routing-function.js` | CloudFront Function for request routing |
| `AwsCloudwatchLogsClient` | `cloudwatch-logs.js` | Log groups, log streams |

Each client:
- Accepts an `awsConfig` object (region + credentials) in the constructor.
- Has proxy support via `addProxyToAwsClient()` from `@serverless/util`.
- Uses the `@smithy/node-http-handler` for HTTP.
- Follows consistent method naming: `getOrCreate*()`, `deploy*()`, `remove*()`, `ensure*()`.

---

## State Management

State is persisted via the `stateStore` (provided by sf-core's runner, backed by the Serverless Dashboard API).

### State Shape

```typescript
{
  name: string
  stage: string
  deploymentType: 'aws@1.0' | 'awsApi@1.0' | 'sfaiAws@1.0'
  isDeployed: boolean
  timeCreated: string (ISO)
  timeLastUpdated: string (ISO)
  timeLastDeployed: string (ISO)
  timeLastRemoved: string (ISO)
  config: object  // project config (sensitive keys obfuscated)
  containers: {
    [containerName: string]: {
      computeType: 'lambda' | 'fargateEcs'
      imageUri: string
      routingConfig: { priority, pathPattern, ... }
      integrations?: { [name: string]: any }
      lastDeployed: string (ISO)
    }
  }
  // Infrastructure state
  vpc?: { vpcId, subnetIds, securityGroupIds }
  loadBalancer?: { arn, dnsName, listenerArn }
  cloudFront?: { distributionId, domainName }
  ecsCluster?: { clusterArn }
  // ... per-resource tracking
}
```

### Key Features

- **Incremental deployments** — Compares previous and current state to determine what changed; only deploys what's necessary (e.g., skip VPC creation if it already exists).
- **Obfuscation** — Sensitive config keys (environment variables, secrets) are obfuscated before saving state.
- **Zod validation** — State is validated with `safeParseAsync` before every save.

---

## Configuration Validation (Zod)

All config schemas use **Zod v4**.

### Config Schema Categories (`types.js`, 1604 lines)

- **Essential:** `name`, `stage`, `org`, `frameworkVersion`, `stages`
- **Deployment:** `deployment.type` (aws@1.0, awsApi@1.0, sfaiAws@1.0) + type-specific sub-schemas for VPC, IAM, ALB, ECS, Lambda, CloudFront
- **Container:** compute type (lambda | fargateEcs), routing (path pattern, priority), scaling (min/max, target tracking, step scaling), build (Dockerfile path, context, args), environment
- **Integrations:** Slack (channels, webhooks), EventBridge (rules, schedules, targets)

### Cross-Field Validation

Complex interdependent rules:
- Scaling min < max
- Target tracking and step scaling are mutually exclusive
- Desired count conflicts with target tracking
- Max is required when autoscaling is enabled
- Step scaling adjustments must have valid step boundaries

---

## Dependency Graph

```
@serverless/engine
├── @serverless/util          — logging, progress, Docker, AWS proxy
├── @serverlessinc/sf-core    — utility imports (hashing, diff, obfuscation)
├── @aws-sdk/* (25+ pkgs)     — AWS SDK v3 modular clients
├── zod                       — config + state validation
├── lodash                    — deep cloning, object manipulation
├── chokidar                  — file watching (dev mode)
├── eventsource               — SSE for deployment monitoring
├── p-limit                   — concurrency control
├── @slack/web-api            — Slack integration
└── date-fns                  — date formatting
```
