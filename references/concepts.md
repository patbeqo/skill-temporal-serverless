# Serverless Workers — Concepts

<!-- Sources: docs/encyclopedia/workers/serverless-workers.mdx, docs/evaluate/development-production-features/serverless-workers/index.mdx -->

## Pre-release status

Serverless Workers are in Pre-release. <!-- docs/encyclopedia/workers/serverless-workers.mdx:24 -->
APIs are experimental and may be subject to backwards-incompatible changes. <!-- docs/encyclopedia/workers/serverless-workers.mdx:26 -->

## What is a Serverless Worker?

A Serverless Worker is a Temporal Worker that runs on serverless compute instead of a long-lived process. <!-- docs/encyclopedia/workers/serverless-workers.mdx:44 -->
There is no always-on infrastructure to provision or scale. Temporal invokes the Worker when Tasks arrive on a Task Queue, and the Worker shuts down when the work is done. <!-- docs/encyclopedia/workers/serverless-workers.mdx:45-46 -->

A Serverless Worker uses the same Temporal SDKs as a traditional long-lived Worker. It registers Workflows and Activities the same way. The difference is in the lifecycle: instead of the Worker starting and polling continuously, Temporal invokes the Serverless Worker on demand, the Worker starts, processes available Tasks, and then shuts down. <!-- docs/encyclopedia/workers/serverless-workers.mdx:48-50 -->

Serverless Workers require Worker Versioning. Each Serverless Worker must be associated with a Worker Deployment Version that has a compute provider configured. <!-- docs/encyclopedia/workers/serverless-workers.mdx:52-53 -->

Each Workflow must have an `AutoUpgrade` or `Pinned` versioning behavior, set per-Workflow or as a Worker-level default. <!-- docs/encyclopedia/workers/serverless-workers.mdx:246 -->

## How Serverless invocation works

With long-lived Workers, the Worker process starts, connects to Temporal, and polls a Task Queue for work. Temporal does not need to know anything about the Worker's infrastructure. <!-- docs/encyclopedia/workers/serverless-workers.mdx:60-61 -->

With Serverless Workers, Temporal starts the Worker. <!-- docs/encyclopedia/workers/serverless-workers.mdx:63 -->

### Worker Controller Instance (WCI)

The Worker Controller Instance (WCI) is a system Workflow that scales Serverless Workers based on Task Queue conditions. <!-- docs/encyclopedia/workers/serverless-workers.mdx:67 -->
One WCI Workflow runs per Worker Deployment Version that has a compute provider configured. The WCI runs in the same Namespace as your Worker Deployment. <!-- docs/encyclopedia/workers/serverless-workers.mdx:68-69 -->

The WCI responds to two triggers: sync match failures and Task Queue backlog. When either trigger fires, the WCI produces a scaling action, such as invoking the configured compute provider (for example, calling AWS Lambda's `InvokeFunction` API) to start new Workers. <!-- docs/encyclopedia/workers/serverless-workers.mdx:71-73 -->

You can list WCI Workflows in your Namespace: <!-- docs/encyclopedia/workers/serverless-workers.mdx:76 -->

```bash
temporal workflow list \
  --namespace <NAMESPACE> \
  --query 'TemporalNamespaceDivision = "TemporalWorkerControllerInstance"'
```
<!-- docs/encyclopedia/workers/serverless-workers.mdx:78-82 -->

WCI Workflow IDs follow the pattern `temporal-sys-worker-controller-instance:<deployment-name>:<build-id>`. <!-- docs/encyclopedia/workers/serverless-workers.mdx:84 -->

You can inspect a WCI Workflow's history to see its recent Activity results: <!-- docs/encyclopedia/workers/serverless-workers.mdx:85 -->

```bash
temporal workflow show \
  --namespace <NAMESPACE> \
  --workflow-id 'temporal-sys-worker-controller-instance:<DEPLOYMENT_NAME>:<BUILD_ID>'
```
<!-- docs/encyclopedia/workers/serverless-workers.mdx:87-91 -->

### Invocation flow

The invocation flow works as follows: <!-- docs/encyclopedia/workers/serverless-workers.mdx:102 -->

1. A Task is submitted (for example, `StartWorkflow` or `ScheduleActivity`). <!-- docs/encyclopedia/workers/serverless-workers.mdx:104 -->
2. The Matching Service attempts to route the Task directly to an available Worker (a sync match). <!-- docs/encyclopedia/workers/serverless-workers.mdx:105-106 -->
3. If a Worker is available, the Task is routed to that Worker. <!-- docs/encyclopedia/workers/serverless-workers.mdx:107 -->
4. If no Worker is available (sync match fails), the Matching Service pushes a signal to the WCI, and the WCI invokes the configured compute provider. <!-- docs/encyclopedia/workers/serverless-workers.mdx:108-109 -->
5. The Serverless Worker starts, creates a Temporal Client, and begins polling the Task Queue. <!-- docs/encyclopedia/workers/serverless-workers.mdx:110 -->
6. The Worker processes available Tasks until it exits (see Worker lifecycle). <!-- docs/encyclopedia/workers/serverless-workers.mdx:111 -->

Each invocation is independent. The Worker creates a fresh client connection on every invocation. There is no connection reuse or shared state across invocations. <!-- docs/encyclopedia/workers/serverless-workers.mdx:113-114 -->

## Autoscaling

The WCI automatically scales Serverless Workers based on Task Queue signals. When Tasks arrive and no Worker is available, the WCI invokes new Workers. When the Tasks are done, Workers exit and scale to zero. <!-- docs/encyclopedia/workers/serverless-workers.mdx:118-119 -->

The WCI uses two signals to decide when to invoke new Workers: <!-- docs/encyclopedia/workers/serverless-workers.mdx:121 -->

### Sync match failure

When a Task is submitted, the Matching Service attempts to route it directly to an available Worker. If no Worker is available, the sync match fails, and the Matching Service pushes a signal to the WCI. The WCI then invokes a new Worker. This is the primary scaling path. <!-- docs/encyclopedia/workers/serverless-workers.mdx:125-127 -->

Because the Matching Service pushes match failures to the WCI as they happen rather than the WCI polling on a timer, latency stays low and scaling is responsive. <!-- docs/encyclopedia/workers/serverless-workers.mdx:128-129 -->

### Task Queue backlog

The WCI monitors Task Queue metadata to determine whether pending Tasks exist without enough Workers to process them. If there are Tasks on the queue and not enough Workers, the WCI invokes additional Workers. <!-- docs/encyclopedia/workers/serverless-workers.mdx:133-134 -->

## Scaling with long-lived Workers

Serverless Workers can share a Task Queue with long-lived Workers. Because Serverless Workers are only invoked on sync match failure, Serverless Workers only pick up Tasks that no long-lived Worker was available to handle. In practice, the Serverless Workers act as spillover capacity for the long-lived fleet. <!-- docs/encyclopedia/workers/serverless-workers.mdx:138-141 -->

**Warning:** If you configure Serverless and long-lived Workers on the same Task Queue, do not enable dynamic scaling on the long-lived Workers. The two groups cannot coordinate their scaling behavior. If both scale dynamically, the long-lived Workers may scale up to handle the same Tasks that Temporal is simultaneously invoking Serverless Workers for, leading to unnecessary invocations and unpredictable scaling. <!-- docs/encyclopedia/workers/serverless-workers.mdx:144-147 -->

## Worker lifecycle

A single Serverless Worker invocation has three phases: init, work, and shutdown. <!-- docs/encyclopedia/workers/serverless-workers.mdx:153 -->

### Init phase

The Worker initializes and establishes a client connection to Temporal. <!-- docs/encyclopedia/workers/serverless-workers.mdx:162 -->

### Work phase

The Worker polls the Task Queue and processes Tasks. <!-- docs/encyclopedia/workers/serverless-workers.mdx:164 -->

### Shutdown phase

The Worker stops polling, waits for in-flight Tasks to finish, and runs any shutdown hooks (for example, OpenTelemetry telemetry flushes). Shutdown begins before the invocation deadline so the Worker can exit cleanly before the compute provider forcibly terminates the execution environment. <!-- docs/encyclopedia/workers/serverless-workers.mdx:166-168 -->

### Tuning for long-running Activities

If your Worker handles long-running Activities, set these three values together: <!-- docs/encyclopedia/workers/serverless-workers.mdx:172 -->

- **Worker stop timeout > longest Activity runtime.** Gives in-flight Activities enough time to finish after polling stops. <!-- docs/encyclopedia/workers/serverless-workers.mdx:174-175 -->
- **Shutdown deadline buffer > Worker stop timeout + shutdown hook time.** Ensures the drain and any shutdown hooks complete before the compute provider terminates the environment. <!-- docs/encyclopedia/workers/serverless-workers.mdx:176-177 -->
- **Invocation deadline > longest Activity runtime + shutdown deadline buffer.** Set on the compute provider to give each invocation enough total runtime. <!-- docs/encyclopedia/workers/serverless-workers.mdx:178-179 -->

If your longest-running Activity runs longer than half the maximum invocation deadline, use Activity Heartbeats to record the state of the Activity execution so that the next retry can pick up where it left off. <!-- docs/encyclopedia/workers/serverless-workers.mdx:183-186 -->

Example: if your longest Activity runtime is 5 minutes, and your shutdown hooks take 3 seconds, set the Worker stop timeout to more than 5 minutes, and the shutdown deadline buffer to more than 303 seconds (5 minutes + 3 seconds). Set your invocation deadline to at least 10 minutes and 3 seconds. <!-- docs/encyclopedia/workers/serverless-workers.mdx:190-192 -->

The Worker stop timeout controls how long the Worker waits for in-flight Tasks to finish after it stops polling. The shutdown deadline buffer controls how much time before the invocation deadline the Worker stops polling for Tasks. <!-- docs/encyclopedia/workers/serverless-workers.mdx:194-195 -->

Raising only the shutdown deadline buffer makes the Worker stop polling earlier, but does not give in-flight Tasks any more time to complete. <!-- docs/encyclopedia/workers/serverless-workers.mdx:197-198 -->

Raising only the Worker stop timeout does not make the Worker stop polling earlier, which means the compute provider might terminate the Worker before the full stop timeout completes. <!-- docs/encyclopedia/workers/serverless-workers.mdx:200-201 -->

## Failure handling

Serverless Workers rely on Temporal's standard retry and timeout semantics to recover from failures. <!-- docs/encyclopedia/workers/serverless-workers.mdx:206 -->

### Worker crash

If a Worker invocation crashes (out of memory, unhandled exception, etc.): <!-- docs/encyclopedia/workers/serverless-workers.mdx:211 -->

- The Activity Timeout fires after the configured duration. <!-- docs/encyclopedia/workers/serverless-workers.mdx:213 -->
- Temporal retries the Activity on a different Worker invocation. <!-- docs/encyclopedia/workers/serverless-workers.mdx:214 -->
- No manual intervention is required. <!-- docs/encyclopedia/workers/serverless-workers.mdx:215 -->

### Provider concurrency limit

If the compute provider's concurrency limit is reached (for example, AWS Lambda account concurrency): <!-- docs/encyclopedia/workers/serverless-workers.mdx:220 -->

- Further invocations from the WCI fail. <!-- docs/encyclopedia/workers/serverless-workers.mdx:222 -->
- Tasks remain in the Task Queue backlog. No data loss occurs. <!-- docs/encyclopedia/workers/serverless-workers.mdx:223 -->
- Processing slows until concurrency frees up. <!-- docs/encyclopedia/workers/serverless-workers.mdx:224 -->

### Resource exhaustion across Activity slots

By default, a single Worker invocation may run multiple Activity slots. A crash or resource exhaustion in one Activity can affect other Activities running in the same invocation. <!-- docs/encyclopedia/workers/serverless-workers.mdx:228-230 -->

To isolate Activities from each other: <!-- docs/encyclopedia/workers/serverless-workers.mdx:232 -->

- Split Workflow and Activity Workers into separate compute functions. <!-- docs/encyclopedia/workers/serverless-workers.mdx:234 -->
- Set Activity slots to 1 per invocation. <!-- docs/encyclopedia/workers/serverless-workers.mdx:235 -->

With single-slot configuration, each Activity gets a dedicated execution environment. <!-- docs/encyclopedia/workers/serverless-workers.mdx:237 -->

## Constraints

<!-- docs/encyclopedia/workers/serverless-workers.mdx:240-247 -->

| Constraint | Detail |
|---|---|
| Activity duration | Must complete within the compute provider's invocation limit (minus shutdown deadline buffer). For AWS Lambda, the maximum is 15 minutes. |
| Workflow duration | No limit. Workflows of any duration work, regardless of the invocation timeout. A Workflow runs across as many invocations as needed. |
| Worker code | Same Temporal SDK Worker code, using the serverless Worker package for your SDK. |
| Versioning | Worker Versioning is required. Each Workflow must have an `AutoUpgrade` or `Pinned` behavior, set per-Workflow or as a Worker-level default. |

## Compute providers

A compute provider is the configuration that tells Temporal how to invoke a Serverless Worker. The compute provider is set on a Worker Deployment Version and specifies the provider type, the invocation target, and the credentials Temporal needs to trigger the invocation. <!-- docs/encyclopedia/workers/serverless-workers.mdx:250-252 -->

For example, an AWS Lambda compute provider includes the Lambda function ARN and the IAM role that Temporal assumes to invoke the function. <!-- docs/encyclopedia/workers/serverless-workers.mdx:254-255 -->

### Supported providers

<!-- docs/encyclopedia/workers/serverless-workers.mdx:261-263 -->

| Provider | Description |
|---|---|
| AWS Lambda | Temporal assumes an IAM role in your AWS account to invoke a Lambda function. |

## Why use Serverless Workers?

<!-- docs/evaluate/development-production-features/serverless-workers/index.mdx:34-48 -->

- **Reduce operational overhead.** No always-on infrastructure to manage and no autoscaling policies to tune. Temporal and the compute provider handle invocation and scaling.
- **Get started faster.** Deploying a Worker is as simple as deploying a function. No Kubernetes, container orchestration, or scaling strategy required.
- **Scale automatically.** The compute provider handles scaling natively. When traffic drops, instances scale down. When there is no work, there is no compute running.
- **Pay only for what you use.** Workers run only when Tasks are available. For low or intermittent volume workloads, this pay-per-invocation model can significantly reduce compute costs.

## When to use Serverless Workers

<!-- docs/evaluate/development-production-features/serverless-workers/index.mdx:73-87 -->

Good fit when:

- Workloads are bursty or event-driven (order processing, notifications, webhook handlers).
- Traffic is low or intermittent.
- You want a simpler getting-started path.
- Your organization has standardized on serverless.
- You serve multiple tenants with infrequent workloads.

May not be ideal when:

<!-- docs/evaluate/development-production-features/serverless-workers/index.mdx:88-97 -->

- Activities are long-running and cannot be interrupted. AWS Lambda has a 15-minute execution limit. Activities that run longer and cannot be broken into smaller steps need a different hosting strategy or a provider with longer limits.
- Workloads require sustained high throughput. Long-lived Workers on dedicated compute may be more cost-effective and performant.
- You need persistent connections. Some features require a persistent connection between the Worker and Temporal, which serverless invocations do not maintain.

## How Serverless Workers compare to long-lived Workers

<!-- docs/evaluate/development-production-features/serverless-workers/index.mdx:101-106 -->

|                | Long-lived Worker | Serverless Worker |
|---|---|---|
| **Lifecycle** | Long-lived process that runs continuously. | Invoked on demand. Starts and stops per invocation. |
| **Scaling** | You manage scaling (Kubernetes HPA, instance count, etc.). | Temporal invokes additional instances as needed, within the compute provider's concurrency limits. |
| **Connection** | Persistent connection to Temporal. | Fresh connection on each invocation. |
