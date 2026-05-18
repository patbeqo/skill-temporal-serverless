# SDK Configuration for Serverless Workers

<!-- Sources:
  docs/develop/go/workers/serverless-workers/aws-lambda.mdx
  docs/develop/python/workers/serverless-workers/aws-lambda.mdx
  docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx
-->

## Go SDK

### Package

Import: `lambdaworker "go.temporal.io/sdk/contrib/aws/lambdaworker"` <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:51 -->

### Entry point

`lambdaworker.RunWorker` — starts a Lambda-based Worker. Pass a `WorkerDeploymentVersion` and a callback that registers Workflows and Activities. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:40-41 -->

### Configure callback

The `Options` callback gives access to the same registration methods as a traditional Worker: `RegisterWorkflow`, `RegisterWorkflowWithOptions`, `RegisterActivity`, `RegisterActivityWithOptions`, and `RegisterNexusService`. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:82 -->

### Versioning behavior

Set per-Workflow at registration time with `workflow.VersioningBehaviorPinned` or `workflow.VersioningBehaviorAutoUpgrade`. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:79 -->
Or set a Worker-level default with `DefaultVersioningBehavior` in `DeploymentOptions`. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:80 -->

### Lambda-tuned defaults

<!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:104-116 -->

| Setting | Lambda default |
|---|---|
| `MaxConcurrentActivityExecutionSize` | 2 |
| `MaxConcurrentWorkflowTaskExecutionSize` | 10 |
| `MaxConcurrentLocalActivityExecutionSize` | 2 |
| `MaxConcurrentNexusTaskExecutionSize` | 5 |
| `MaxConcurrentActivityTaskPollers` | 1 |
| `MaxConcurrentWorkflowTaskPollers` | 2 |
| `MaxConcurrentNexusTaskPollers` | 1 |
| `WorkerStopTimeout` | 5 seconds |
| `DisableEagerActivities` | Always true |
| Sticky cache size | 100 |
| `ShutdownDeadlineBuffer` | 7 seconds |

These are the same `worker.Options` available to any Temporal Worker, just with lower values for Lambda's constrained environment. Except for `ShutdownDeadlineBuffer`, which is specific to the `lambdaworker` package. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:102,121 -->

`DisableEagerActivities` is always true and cannot be overridden. Eager Activities require a persistent connection, which Lambda invocations don't maintain. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:118-119 -->

`ShutdownDeadlineBuffer` controls how much time before the Lambda deadline the Worker begins its graceful shutdown. The default is `WorkerStopTimeout` + 2 seconds. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:121-123 -->

If your Worker handles long-running Activities, increase `WorkerStopTimeout`, `ShutdownDeadlineBuffer`, and the Lambda invocation deadline (`--timeout`) together. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:125-126 -->

### Connection configuration

The `lambdaworker` package automatically loads Temporal client configuration from a TOML config file and environment variables. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:86 -->

TOML config file resolution order: <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:88-93 -->

1. `TEMPORAL_CONFIG_FILE` environment variable, if set.
2. `temporal.toml` in `$LAMBDA_TASK_ROOT` (typically `/var/task`).
3. `temporal.toml` in the current working directory.

The file is optional. If absent, only environment variables are used. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:94 -->

---

## Python SDK

### Package

Import: `from temporalio.contrib.aws.lambda_worker import LambdaWorkerConfig, run_worker` <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:48 -->

### Entry point

`run_worker` — takes a `WorkerDeploymentVersion` and a configure callback, returns a Lambda handler. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:40,67 -->

### Configure callback

The `configure` callback receives a `LambdaWorkerConfig` dataclass with fields pre-populated with Lambda-appropriate defaults. Set the Task Queue, Workflows, and Activities through `worker_config`, which accepts the same keyword arguments as the `Worker` constructor. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:72-73 -->

### Versioning behavior

Set per-Workflow in the `@workflow.defn` decorator: `VersioningBehavior.PINNED` or `VersioningBehavior.AUTO_UPGRADE`. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:75 -->
Or set a Worker-level default with `default_versioning_behavior` in the worker config. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:76 -->

### Lambda-tuned defaults

<!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:109-121 -->

| Setting | Lambda default |
|---|---|
| `max_concurrent_activities` | 2 |
| `max_concurrent_workflow_tasks` | 10 |
| `max_concurrent_local_activities` | 2 |
| `max_concurrent_nexus_tasks` | 5 |
| `workflow_task_poller_behavior` | `SimpleMaximum(2)` |
| `activity_task_poller_behavior` | `SimpleMaximum(1)` |
| `nexus_task_poller_behavior` | `SimpleMaximum(1)` |
| `graceful_shutdown_timeout` | 5 seconds |
| `max_cached_workflows` | 30 |
| `disable_eager_activity_execution` | Always `True` |
| `shutdown_deadline_buffer` | 7 seconds |

`disable_eager_activity_execution` is always `True` and cannot be overridden. Eager Activities require a persistent connection, which Lambda invocations don't maintain. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:123-124 -->

`shutdown_deadline_buffer` is specific to the `lambda_worker` package. It controls how much time before the Lambda deadline the Worker begins its graceful shutdown. The default is `graceful_shutdown_timeout` + 2 seconds. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:126-128 -->

If your Worker handles long-running Activities, increase `graceful_shutdown_timeout`, `shutdown_deadline_buffer`, and the Lambda invocation deadline (`--timeout`) together. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:130-131 -->

### Connection configuration

The `lambda_worker` package automatically loads Temporal client configuration from a TOML config file and environment variables. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:92 -->

TOML config file resolution order: <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:94-99 -->

1. `TEMPORAL_CONFIG_FILE` environment variable, if set.
2. `temporal.toml` in `$LAMBDA_TASK_ROOT` (typically `/var/task`).
3. `temporal.toml` in the current working directory.

The file is optional. If absent, only environment variables are used. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:100 -->

---

## TypeScript SDK

### Package

Import: `import { runWorker } from '@temporalio/lambda-worker'` <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:138 -->

### Entry point

`runWorker` — creates a Lambda handler that runs a Temporal Worker. Pass a deployment version and a configure callback. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:40-41 -->

### Configure callback

Set Worker options via `config.workerOptions`. For Workflow code, use `workflowBundle` with pre-bundled code instead of `workflowsPath` to avoid webpack bundling overhead on Lambda cold starts. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:69-70 -->

### Pre-bundling Workflow code

Build the bundle as a separate build step: <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:72 -->

```typescript
import { bundleWorkflowCode } from '@temporalio/worker';
import { writeFile } from 'fs/promises';

const { code } = await bundleWorkflowCode({
  workflowsPath: require.resolve('./workflows'),
});
await writeFile('./workflow-bundle.js', code);
```
<!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:74-82 -->

Then reference the bundle in your handler with `workflowBundle: { codePath: require.resolve('./workflow-bundle.js') }`. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:84 -->

### Versioning behavior

Set per-Workflow with `setWorkflowOptions` in the Workflow file, or set a default for all Workflows with `defaultVersioningBehavior` in the configure callback. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:64-65 -->
Values are `'PINNED'` or `'AUTO_UPGRADE'`. The default versioning behavior is `PINNED`. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:64-65 -->

Access via: `config.workerOptions.workerDeploymentOptions!.defaultVersioningBehavior = 'PINNED'` <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:51 (from production deploy page: aws-lambda.mdx:166) -->

### Lambda-tuned defaults

<!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:105-116 -->

| Setting | Lambda default |
|---|---|
| `maxConcurrentActivityTaskExecutions` | 2 |
| `maxConcurrentWorkflowTaskExecutions` | 10 |
| `maxConcurrentLocalActivityExecutions` | 2 |
| `maxConcurrentNexusTaskExecutions` | 5 |
| `workflowTaskPollerBehavior` | `SimpleMaximum(2)` |
| `activityTaskPollerBehavior` | `SimpleMaximum(1)` |
| `nexusTaskPollerBehavior` | `SimpleMaximum(1)` |
| `shutdownGraceTime` | 5 seconds |
| `maxCachedWorkflows` | 30 |
| `shutdownDeadlineBufferMs` | 7000 |

Eager Activities are not supported. Lambda invocations don't maintain persistent connections. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:118 -->

`shutdownDeadlineBufferMs` is specific to the `@temporalio/lambda-worker` package. It controls how much time before the Lambda deadline the Worker begins its graceful shutdown. The default is `shutdownGraceTime` (5s) + 2s. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:120-122 -->

If your Worker handles long-running Activities, increase `shutdownGraceTime`, `shutdownDeadlineBufferMs`, and the Lambda invocation deadline (`--timeout`) together. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:124-125 -->

### Connection configuration

The `@temporalio/lambda-worker` package automatically loads Temporal client configuration from a TOML config file and environment variables. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:88 -->

TOML config file resolution order: <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:90-95 -->

1. `TEMPORAL_CONFIG_FILE` environment variable, if set.
2. `temporal.toml` in `$LAMBDA_TASK_ROOT` (typically `/var/task`).
3. `temporal.toml` in the current working directory.

The file is optional. If absent, only environment variables are used. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:96 -->

---

## Cross-SDK comparison: Lambda-tuned defaults

| Concept | Go | Python | TypeScript |
|---|---|---|---|
| Max concurrent activities | `MaxConcurrentActivityExecutionSize` = 2 | `max_concurrent_activities` = 2 | `maxConcurrentActivityTaskExecutions` = 2 |
| Max concurrent workflow tasks | `MaxConcurrentWorkflowTaskExecutionSize` = 10 | `max_concurrent_workflow_tasks` = 10 | `maxConcurrentWorkflowTaskExecutions` = 10 |
| Sticky cache size | 100 | `max_cached_workflows` = 30 | `maxCachedWorkflows` = 30 |
| Worker stop timeout | `WorkerStopTimeout` = 5s | `graceful_shutdown_timeout` = 5s | `shutdownGraceTime` = 5s |
| Shutdown deadline buffer | `ShutdownDeadlineBuffer` = 7s | `shutdown_deadline_buffer` = 7s | `shutdownDeadlineBufferMs` = 7000 |
| Eager activities | `DisableEagerActivities` always true | `disable_eager_activity_execution` always `True` | Not supported |

<!-- Go: docs/develop/go/workers/serverless-workers/aws-lambda.mdx:104-116 -->
<!-- Python: docs/develop/python/workers/serverless-workers/aws-lambda.mdx:109-121 -->
<!-- TypeScript: docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:105-116 -->

Note: Go sticky cache size is 100, while Python and TypeScript are 30. These values come from each SDK's own docs and are not interchangeable.
