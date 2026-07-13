---
name: temporal-serverless
description: 'Deploy and operate Temporal Workers on serverless compute (AWS Lambda) driven by the Worker Controller Instance (WCI). Use when the user mentions: "serverless worker", "Temporal serverless", "Worker Controller Instance", "WCI", "deploy Temporal worker on Lambda", "Lambda packaging", "Lambda timeout", "WCI inspection", "CloudFormation Temporal".'
version: 0.1.0
---

# Skill: temporal-serverless

## Overview

This skill helps users deploy and operate Temporal Workers on serverless compute (currently AWS Lambda). It produces Worker code, CloudFormation templates, IAM policies, TOML connection configs, Lambda packaging scripts, and OTel collector configs. It also walks users through troubleshooting when their serverless Workers aren't picking up tasks.

Ready-to-deploy CloudFormation templates for the Temporal invocation role ship in `assets/` (`temporal-cloud-serverless-worker-role.yaml` for Temporal Cloud, `temporal-self-hosted-serverless-worker-role.yaml` for self-hosted) — use them for the `--template-body` step instead of authoring the YAML by hand.

Serverless Workers are in **Pre-release** — always surface this early. APIs are experimental and may change; access is request-based during Pre-release (open a support ticket or contact your account team), and the feature has not yet reached Public Preview. Users should confirm eligibility with Temporal before building production workloads on serverless Workers.

## How to Use This Skill

### Check available credentials and adapt

At the start of a session, check what tools you have access to by running `aws sts get-caller-identity` and `temporal env list` (or checking for `TEMPORAL_ADDRESS` / `TEMPORAL_API_KEY` env vars). Also run `temporal --version`: the serverless `worker deployment create-version` subcommand and its `--aws-lambda-*` flags only exist in recent CLI builds (a Homebrew v1.5.0 lacked `create-version` entirely; v1.8.0 had the serverless flags). If the subcommand or flags are missing, install a current standalone CLI build without disturbing the packaged one. Your behavior changes based on what's available:

| AWS CLI | Temporal CLI | Agent behavior |
|---|---|---|
| Authenticated | Authenticated | Full deployment workflow — run commands directly, verify results, create deployment versions. |
| Authenticated | No credentials | Deploy Lambda infrastructure (IAM, packaging, CloudFormation). Generate `temporal` commands for the user to run for deployment versions and endpoint config. |
| No credentials | Authenticated | Write Worker code and configs. Generate `aws` commands and CloudFormation templates for the user to deploy. Run `temporal` commands to create deployment versions and verify WCI state. |
| No credentials | No credentials | Write Worker code, CloudFormation templates, IAM policies, TOML configs, packaging scripts. Provide all commands with placeholder values for the user to run. |

The skill is valuable at every level — even without any credentials, producing correct Lambda handler code, IAM policies, and deployment configs saves significant time.

When generating deployment configs, sensitive values (API keys, TLS private keys) should go in AWS Secrets Manager or Parameter Store, not Lambda environment variables in plaintext.

### Triage steps

Before consulting reference files, determine:

1. **SDK language** — Go, Python, or TypeScript. Each has different imports, option names, and Lambda-tuned defaults.
2. **Deployment target** — Temporal Cloud or self-hosted (requires Server v1.31.0+). This changes IAM setup, connection config, and available features.
3. **Task type** — new setup, configuration change, or troubleshooting an existing deployment.

### Multi-file routing for common questions

Most real questions require 2-3 reference files:

- **"Set up a serverless worker on Lambda"** → `aws-lambda-deployment.md` (end-to-end steps) + `sdk-configuration.md` (SDK-specific code and config) + `concepts.md` (WCI lifecycle, to understand what they're building).
- **"My serverless worker isn't picking up tasks"** → `troubleshooting.md` (diagnostic tree) + `concepts.md` (invocation flow) + `aws-lambda-deployment.md` (IAM and deployment version verification).
- **"Add observability to my Lambda worker"** → `observability.md` (ADOT layers, OTel config) + `sdk-configuration.md` (SDK-specific config callback for tracing).
- **"What are the Lambda-tuned defaults?"** → `sdk-configuration.md` (defaults table and per-SDK options) + `concepts.md` (why the defaults differ from long-lived Workers).
- **"Update my serverless Worker / deploy a new version"** → `aws-lambda-deployment.md` (update-function-code, publish a new Lambda function version, create new Worker Deployment Version, set as current) + `sdk-configuration.md` (update build ID in Worker code).
- **"How do I handle long-running Activities?"** → `concepts.md` (timeout tuning triple, heartbeat strategy) + `sdk-configuration.md` (SDK-specific timeout option names).
- **"Isolate Activities from resource exhaustion"** → `concepts.md` (split Workers into separate functions, or set Activity slots to 1).

### Troubleshooting

Always start by determining whether the Lambda is being invoked at all — check CloudWatch metrics or invocation logs first.

If you have AWS CLI access, run diagnostic commands directly (`aws lambda get-function`, `aws logs filter-log-events`, etc.). If you have Temporal CLI access, inspect the WCI Workflow and deployment versions (`temporal worker deployment describe-version --report-task-queue-stats`). Otherwise, walk the user through the diagnostic tree — tell them what to check and interpret what they report back.

**Diagnostic discipline.** You never create or manage the WCI — Temporal creates it automatically once a Worker Deployment Version has a compute provider. A WCI that exists or is running is *not* evidence invocation works: it continue-as-news and keeps running even while its Activities fail, so read its Workflow history (`temporal workflow show` on the WCI Workflow ID) and look for Activity failures to find the real error. Diagnose from Temporal's own signals first; do not enumerate Lambdas across regions or scan the AWS account to reverse-engineer state. And distinguish a Temporal-Cloud-side failure (e.g. Temporal cannot obtain its *own* base AWS credentials — reproduces no matter what you change on the AWS side) from a genuine user-IAM problem before you start editing IAM config.

Follow this priority order:
1. **Validate Connection** — the first step. In the Temporal UI: Workers > Deployments > select deployment > Actions > Validate Connection. This checks IAM, role assumption, and Lambda reachability in one step.
2. **Check version is current** — if created via CLI, the version may not be set as current.
3. **Check CloudWatch logs** — look for connection failures, TLS errors, or authentication errors in `/aws/lambda/<function-name>`.
4. **If rapid repeated invocations with no progress** — immediately check deployment name/build ID match. This is the signature of a version mismatch loop.

### Code generation guidelines

When generating Worker code, CloudFormation templates, or deployment scripts:

- **Always include versioning behavior.** Every Workflow must have `AutoUpgrade` or `Pinned` behavior, set per-Workflow or as a Worker-level default. Generated code that omits versioning behavior will fail at runtime.
- **TypeScript: always use `workflowBundle` with pre-bundled code.** Never use `workflowsPath` in Lambda — it triggers webpack bundling on every cold start.
- **Deployment name and build ID must exactly match the Worker Deployment Version.** A mismatch causes an invocation loop. When generating code, use clear placeholder names and remind the user these must match their `temporal worker deployment create-version` command.
- **Secrets belong in Secrets Manager or Parameter Store.** Never put API keys or TLS private keys in plaintext Lambda environment variables in generated CloudFormation templates.
- **Include both IAM roles.** The Lambda execution role (trusted by `lambda.amazonaws.com`) and the Temporal invocation role (trusted by Temporal's principals) are separate. Generated IAM resources must include both.
- **Use the correct OTel env var per SDK.** Go and TypeScript use `OPENTELEMETRY_COLLECTOR_CONFIG_URI`. Python uses `OPENTELEMETRY_COLLECTOR_CONFIG_FILE`. These are not interchangeable.
- **Use a qualified, versioned Lambda ARN for production.** Publish a Lambda function version (`aws lambda publish-version`) and configure the compute provider with the qualified versioned ARN (for example, `function:my-worker:5`), keeping a 1-to-1 mapping between each Build ID and one Lambda function version. An unqualified ARN points at `$LATEST`, which changes on every redeploy and can cause non-determinism errors for in-flight Workflows — even Pinned ones. Reserve unqualified ARNs for development.
- **Set `--timeout` well above Lambda's 3-second default.** The default 3-second timeout is too short for the Worker to start, connect to Temporal, and register the Task Queue. If the first invocation times out, the Task Queue binding is never created and the Lambda is never invoked again.
- **Verify the installed SDK's real signatures before generating code.** These are Pre-release APIs and can drift between versions. When the SDK is installed, introspect it (e.g. `python -c "import temporalio.contrib.aws.lambda_worker as m; help(m)"`) rather than trusting snippets in this skill or the docs — confirm the import path, entry-point function, and option names against the version actually in use, and pin a known-good SDK version in the packaging step.

### Proactive warnings

Surface these constraints early — don't wait for the user to discover them:

- **Pre-release status** — mention at the start of any serverless Worker session. APIs are experimental and may change; access is request-based (support ticket or account team) and the feature has not yet reached Public Preview.
- **15-minute Activity limit** — Activities must complete within the Lambda invocation limit minus the shutdown deadline buffer. If a user describes Activities that might approach this limit, flag it immediately.
- **Eager Activities always disabled** — Lambda invocations don't maintain persistent connections. Don't suggest Eager Activities as an optimization.
- **Lambda's 3-second default timeout is too short** — the function's default `--timeout` is 3 seconds, not enough for the first invocation to start the Worker, connect to Temporal, and register the Task Queue. If it times out, the Task Queue binding is never created and the Lambda is never invoked again. Always set `--timeout` high (the guide uses 600).
- **Unqualified `$LATEST` Lambda ARN** — for production, point the compute provider at a qualified versioned ARN (`publish-version`). An unqualified ARN tracks `$LATEST`, which changes on every redeploy and can trigger non-determinism errors for in-flight Workflows, even Pinned ones.
- **Timeout tuning triple** — If the user mentions long-running Activities, proactively calculate all three values together: (1) worker stop timeout > longest Activity runtime, (2) shutdown deadline buffer > worker stop timeout + shutdown hook time, (3) invocation deadline > longest Activity runtime + shutdown deadline buffer. Show the math with their specific durations.
- **Mixed serverless + long-lived Workers** — If the user wants to share a Task Queue, warn: do NOT enable dynamic scaling on the long-lived Workers. The two groups cannot coordinate scaling and will cause unnecessary invocations.
- **Activity heartbeats** — If the user's longest Activity runs longer than half the maximum invocation deadline (7.5 minutes for Lambda), recommend Activity Heartbeats to record state so the next retry can pick up where it left off.

## Key Facts

- **Pre-release** — request access via a support ticket or your account team; the feature has not yet reached Public Preview.
- WCI = **Worker Controller Instance**: a system Workflow (one per Worker Deployment Version with a compute provider) that scales Serverless Workers.
- Supported SDKs: Go, Python, TypeScript. Supported compute: AWS Lambda only.
- Worker Versioning required — each Workflow needs `AutoUpgrade` or `Pinned` behavior.
- Activity limit: 15 minutes on Lambda (invocation limit minus shutdown deadline buffer).
- Shutdown deadline buffer default: WorkerStopTimeout + 2 seconds.
- Workflow duration has no limit — Workflows can span multiple invocations.
- Each invocation is independent — no connection reuse or shared state.
- Eager Activities always disabled.
- Self-hosted requires Temporal Server v1.31.0+.
- Serverless CLI (`worker deployment create-version` + `--aws-lambda-*` flags) requires a recent Temporal CLI — check `temporal --version` (absent in v1.5.0, present by v1.8.0).
- Lambda runtimes: Go `provided.al2023`, Python `python3.13`, Node.js `nodejs22.x` (20+).
- Lambda default `--timeout` is 3 seconds — always raise it (the guide uses 600).
- Production: map each Build ID to one published Lambda function version and configure the compute provider with the qualified versioned ARN. Unqualified ARN = `$LATEST` (development only).
- WCI Workflow ID pattern: `temporal-sys-worker-controller-instance:<deployment-name>:<build-id>`.
- Lambda execution role and Temporal invocation role are different — do not confuse them.
- OTel env var divergence: Python uses `OPENTELEMETRY_COLLECTOR_CONFIG_FILE`; Go and TypeScript use `_URI`.

## Common Pitfalls

Watch for these high-impact mistakes and warn the user proactively:

1. **Deployment name/build ID mismatch** — If CloudWatch shows rapid, repeated invocations with no Workflow progress, the deployment name or build ID in the Worker code doesn't match the Worker Deployment Version. This causes an invocation loop: WCI invokes Lambda → Worker polls with wrong version → Task not processed → WCI invokes again. Fix: the values in code must exactly match the Worker Deployment Version configuration. Compare against the WCI Workflow ID (`temporal-sys-worker-controller-instance:<deployment-name>:<build-id>`).
2. **Version not set as current** — If you create a Worker Deployment Version via the CLI, it is not automatically set as current. Without this, Tasks on the Task Queue will not route to the version and the Lambda is never invoked. (The UI sets it automatically.)
3. **Failed first invocation** — When a Worker Deployment Version is created, the WCI invokes the Lambda to validate. If that invocation fails (missing env vars, bad TLS config, missing dependencies, or the Lambda's 3-second default timeout being too short for the Worker to start and register), the Worker never connects, never polls, and the Task Queue binding is never created. The Lambda is never automatically invoked again. Diagnose by manually invoking the Lambda from the AWS Console, and confirm `--timeout` is set high.
4. **Confusing execution role vs invocation role** — The Lambda execution role (trusted by `lambda.amazonaws.com`) is separate from the Temporal invocation role (trusted by Temporal's `wci-lambda-invoke` principals). The execution role grants the function permission to run. The invocation role grants Temporal permission to invoke the function. Never describe one as the other.
5. **Timeout tuning mismatch** — Raising only the shutdown deadline buffer makes the Worker stop polling earlier but does NOT give in-flight Activities more time. Raising only the Worker stop timeout does not make the Worker stop polling earlier, so the compute provider might terminate the Worker before the stop timeout completes. The three values (worker stop timeout, shutdown deadline buffer, invocation deadline) must be tuned together.
6. **Unqualified `$LATEST` Lambda ARN in production** — pointing the compute provider at an unqualified ARN tracks `$LATEST`, which changes on every redeploy. Deploying replay-unsafe code then causes non-determinism errors for in-flight Workflows, even Pinned ones. Fix: `aws lambda publish-version` after each `update-function-code`, keep a 1-to-1 mapping between each Build ID and one Lambda function version, and configure the compute provider with the qualified versioned ARN.

## Intent Decision Table

Use this table to find the right reference file for the user's question.

| User intent | Reference file |
|---|---|
| What is a Serverless Worker? How does invocation work? What is the WCI? How does autoscaling work? What are the constraints? When should I use Serverless Workers vs. long-lived Workers? | `references/concepts.md` |
| How do I deploy a Serverless Worker on AWS Lambda? How do I write Worker code, package it, deploy it, set up IAM, create a Worker Deployment Version? | `references/aws-lambda-deployment.md` |
| What are the SDK-specific configuration options? What are the Lambda-tuned defaults? What package do I import? How do I configure versioning behavior? How does connection config work (TOML, env vars)? | `references/sdk-configuration.md` |
| How do I add OpenTelemetry observability? What ADOT layers do I need? What env var do I set for the collector config? How do I enable X-Ray tracing? | `references/observability.md` |
| My Lambda is not being invoked. My Workflows are not progressing. How do I diagnose a Serverless Worker issue? How do I inspect the WCI? | `references/troubleshooting.md` |
| How do I update or redeploy my serverless Worker with a new version? | `references/aws-lambda-deployment.md` |
| How do I version my Lambda, map Build IDs to Lambda function versions, use a qualified ARN, or roll back a version? | `references/aws-lambda-deployment.md` + `references/concepts.md` |
| What IAM roles and permissions does Temporal need to invoke my Lambda? | `references/aws-lambda-deployment.md` |
| How do I handle long-running Activities on Lambda? What are the timeout relationships? | `references/concepts.md` + `references/sdk-configuration.md` |
| How do I reduce Lambda cold start time? How do I pre-bundle Workflow code? | `references/sdk-configuration.md` |
| How do I isolate Activities from each other to prevent resource exhaustion? | `references/concepts.md` |
| How do I set up a self-hosted Temporal Service for serverless Workers? | `references/aws-lambda-deployment.md` |

## Out of Scope

- **General SDK development patterns** (Workflows, Activities, Workers, signals, queries, Worker Versioning concepts): see `skill-temporal-developer`.
- **Traditional Worker tuning** (slot suppliers, tuners, poller autoscaling, resource-based tuning): see `skill-temporal-workertuning`.
- **Temporal Cloud administration** (Namespaces, users, certificates, billing): see `skill-temporal-ops`.
- **CLI command reference** (beyond the serverless-specific flags): see `skill-temporal-cli`.
