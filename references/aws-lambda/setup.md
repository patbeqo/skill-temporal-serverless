# AWS Lambda — Setup (happy path)

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx
  docs/production-deployment/worker-deployments/serverless-workers/index.mdx
  docs/develop/environment-configuration.mdx
-->

This is the end-to-end golden path: connect, write the Worker, package and deploy, register a Worker Deployment Version, set it current, and verify. For the operator permissions and preflight, execution/invocation roles, and CloudFormation, see `iam.md`. For production build versioning (`publish-version`, qualified ARNs, rollback), see `versioning.md`. For self-hosted server enablement, see `self-hosted.md`. If it doesn't work, see `diagnostics.md`.

## Prerequisites

- A Temporal Cloud account with an AWS-hosted Namespace, or a self-hosted Temporal Service v1.31.0 or later. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:37 -->
- The Namespace's cloud provider must match the serverless compute provider. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:37-38 -->
- For self-hosted deployments, complete the self-hosted setup before following the deployment guide. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:39-41 -->
- Every Workflow must declare a versioning behavior, or the Worker must set a default versioning behavior. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:42-43 -->
- An AWS account with permissions to create and invoke Lambda functions and create IAM roles. For the exact operator actions and a preflight check, see `iam.md`. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:44 -->
- The AWS-specific steps require the `aws` CLI installed and configured with your AWS credentials. You may also use the AWS Console or the AWS SDKs. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:45-46 -->
- The Go SDK, Python SDK, or TypeScript SDK, depending on your language. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:48-49 -->
- The `temporal` CLI, authenticated to the target Temporal Service — Steps 4–6 and the CLI troubleshooting paths use it. See "Temporal CLI and Cloud connection" below. <!-- field note: not in upstream prerequisites; the deployment-version and verify steps require the CLI -->

Sample projects:
- Go: [Go Lambda Worker sample](https://github.com/temporalio/samples-go/tree/main/lambda-worker) <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:54 -->
- Python: [Python Lambda Worker sample](https://github.com/temporalio/samples-python/tree/main/lambda_worker) <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:55 -->
- TypeScript: [TypeScript Lambda Worker sample](https://github.com/temporalio/samples-typescript/tree/main/lambda-worker) <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:56 -->

## Temporal CLI and Cloud connection

Steps 4–6 and the CLI troubleshooting paths use the `temporal` CLI. Install it and authenticate it to the target Temporal Service before those steps, or commands default to `localhost:7233` and fail against Temporal Cloud. The serverless `worker deployment create-version` subcommand and its `--aws-lambda-*` flags also require a recent CLI build — see the field note in Step 4. <!-- field note: connection setup not covered in upstream serverless docs; see docs/develop/environment-configuration.mdx -->

**Authenticate to Temporal Cloud (API key).** Export environment variables (the CLI and the serverless Worker packages both read these):

```bash
export TEMPORAL_ADDRESS="<namespace_id>.<account_id>.tmprl.cloud:7233"
export TEMPORAL_NAMESPACE="<namespace_id>.<account_id>"
export TEMPORAL_API_KEY="<your-api-key>"
```

or configure a profile and pass `--profile prod` on each command:

```bash
temporal --profile prod config set --prop address --value "<namespace_id>.<account_id>.tmprl.cloud:7233"
temporal --profile prod config set --prop namespace --value "<namespace_id>.<account_id>"
temporal --profile prod config set --prop api_key --value "<your-api-key>"
```
<!-- docs/develop/environment-configuration.mdx:122-131 -->

- For Temporal Cloud the Namespace is the fully-qualified `<namespace_id>.<account_id>`, not the bare name. <!-- docs/develop/environment-configuration.mdx:128-129 -->
- Supplying an API key auto-enables TLS; no cert flags are needed for API-key auth. <!-- docs/develop/environment-configuration.mdx:70 -->
- The `temporal ...` commands in Steps 4–6 assume this is configured. To create an API key, see `skill-temporal-ops`.

**Temporal-side preflight.** Confirm the CLI can reach the Namespace before deploying — this is the Temporal side of the pre-deploy access check. It should list (empty is fine) without an auth or connection error:

```bash
temporal worker deployment list
```

## Step 1: Write Worker code

The Worker handles the per-invocation lifecycle: connecting to Temporal, polling for tasks, and gracefully shutting down before the invocation deadline. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:62-63 -->

**Fastest path:** start from the language sample linked in Prerequisites — it has a working Worker, Workflow, and Activity already wired together. The handler examples below import the Workflow and Activity from separate modules (`my_workflows`, `my_activities`). When writing from scratch, create those modules with at least one registered Workflow (declaring a versioning behavior) and one Activity, and name the entry-point file to match the `--handler` you deploy (for example, `lambda_function.py` → `--handler lambda_function.lambda_handler`). <!-- field note: sample-first path and handler/file-name matching not spelled out upstream -->

### Go

Use the Go SDK's `lambdaworker` package. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:68 -->

```go
package main

import (
    lambdaworker "go.temporal.io/sdk/contrib/aws/lambdaworker"
    "go.temporal.io/sdk/worker"
    "go.temporal.io/sdk/workflow"
)

func main() {
    lambdaworker.RunWorker(worker.WorkerDeploymentVersion{
        DeploymentName: "my-app",
        BuildID:        "build-1",
    }, func(opts *lambdaworker.Options) error {
        opts.TaskQueue = "my-task-queue"

        opts.RegisterWorkflowWithOptions(MyWorkflow, workflow.RegisterOptions{
            VersioningBehavior: workflow.VersioningBehaviorPinned,
        })
        opts.RegisterActivity(MyActivity)

        return nil
    })
}
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:70-94 -->

Versioning behavior: set per-Workflow at registration time with `workflow.VersioningBehaviorPinned` or `workflow.VersioningBehaviorAutoUpgrade`, or set a Worker-level default with `DefaultVersioningBehavior` in `DeploymentOptions`. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:96-98 -->

### Python

Use the Python SDK's `lambda_worker` contrib package. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:106 -->

```python
from temporalio.common import WorkerDeploymentVersion
from temporalio.contrib.aws.lambda_worker import LambdaWorkerConfig, run_worker

from my_workflows import MyWorkflow
from my_activities import my_activity


def configure(config: LambdaWorkerConfig) -> None:
    config.worker_config["task_queue"] = "my-task-queue"
    config.worker_config["workflows"] = [MyWorkflow]
    config.worker_config["activities"] = [my_activity]


lambda_handler = run_worker(
    WorkerDeploymentVersion(
        deployment_name="my-app",
        build_id="build-1",
    ),
    configure,
)
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:108-129 -->

Versioning behavior: set per-Workflow in the `@workflow.defn` decorator with `VersioningBehavior.PINNED` or `VersioningBehavior.AUTO_UPGRADE`, or set a Worker-level default with `default_versioning_behavior` in the worker config. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:131-133 -->

```python
from temporalio import workflow
from temporalio.common import VersioningBehavior


@workflow.defn(versioning_behavior=VersioningBehavior.PINNED)
class MyWorkflow:
    @workflow.run
    async def run(self, input: str) -> str:
        ...
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:135-145 -->

### TypeScript

Use the `@temporalio/lambda-worker` package. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:153 -->

```typescript
import { runWorker } from '@temporalio/lambda-worker';
import * as activities from './activities';

export const handler = runWorker({ deploymentName: 'my-app', buildId: 'build-1' }, (config) => {
  config.workerOptions.taskQueue = 'my-task-queue';
  config.workerOptions.workflowBundle = {
    codePath: require.resolve('./workflow-bundle.js'),
  };
  config.workerOptions.activities = activities;
  config.workerOptions.workerDeploymentOptions!.defaultVersioningBehavior = 'PINNED';
});
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:155-167 -->

Use `workflowBundle` with pre-bundled code instead of `workflowsPath` to avoid webpack bundling overhead on Lambda cold starts. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:169-170 -->

Versioning behavior: set per-Workflow with `setWorkflowOptions` in the Workflow file, or set a default for all Workflows with `defaultVersioningBehavior` in the configure callback. Values are `'AUTO_UPGRADE'` or `'PINNED'`. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:172-174 -->

## Step 2: Deploy Lambda function

### Build and package

#### Go

Cross-compile for Lambda's Linux runtime: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:191 -->

```bash
GOOS=linux GOARCH=amd64 go build -tags lambda.norpc -o bootstrap ./worker
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:194 -->

Package the binary into a zip file: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:197 -->

```bash
zip function.zip bootstrap
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:200 -->

#### Python

Install dependencies into a local directory for packaging, using `--platform` for Linux-compatible binaries: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:206-207 -->

```bash
pip install --target ./package --platform manylinux2014_x86_64 --only-binary=:all: temporalio
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:210 -->

**Field note:** Pin the download to the Lambda runtime's Python version and architecture, not your local interpreter's. If they differ (e.g. local `3.14` vs the function's `python3.13`), add `--python-version 3.13` alongside `--only-binary=:all:` so pip fetches runtime-matching wheels, and keep `--platform` (`manylinux2014_x86_64` for `x86_64`, `manylinux2014_aarch64` for `arm64`) consistent with the function's `--architectures`. Mismatches surface as import errors only at invocation time, not at package time. <!-- field note: 2026-07 serverless deployment test session; not in docs -->

To include OpenTelemetry support, install `temporalio[lambda-worker-otel]` instead. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:213-214 -->

Package dependencies and application code: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:216 -->

```bash
cd package && zip -r ../function.zip . && cd ..
zip function.zip lambda_function.py my_workflows.py my_activities.py
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:218-220 -->

#### TypeScript

Build the Workflow bundle and compile the project: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:226 -->

```bash
npx ts-node src/scripts/build-workflow-bundle.ts
npx tsc
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:229-230 -->

Install production dependencies and package everything: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:233 -->

```bash
npm install --omit=dev
zip -r function.zip lib/ node_modules/ workflow-bundle.js
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:236-237 -->

### Deploy the Lambda function

**Field note:** A freshly created execution role may not be assumable immediately — `create-function` can fail with an assume-role / "cannot be assumed by Lambda" error due to IAM propagation delay. Wait a few seconds and retry; it is not a policy error. <!-- field note: 2026-07 serverless deployment test session; not in docs -->

#### Go

```bash
aws lambda create-function \
  --function-name my-temporal-worker \
  --runtime provided.al2023 \
  --handler bootstrap \
  --role <EXECUTION_ROLE_ARN> \
  --zip-file fileb://function.zip \
  --timeout 600 \
  --memory-size 256 \
  --environment '{"Variables":{"HOME":"/tmp","TEMPORAL_ADDRESS":"<your-temporal-address>:7233","TEMPORAL_NAMESPACE":"<your-namespace>","TEMPORAL_API_KEY":"<your-api-key>"}}'
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:250-260 -->

- `--runtime`: `provided.al2023` for custom Go binaries. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:265 -->
- `--handler`: `bootstrap` when using the `provided.al2023` custom runtime. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:266 -->

#### Python

```bash
aws lambda create-function \
  --function-name my-temporal-worker \
  --runtime python3.13 \
  --handler lambda_function.lambda_handler \
  --role <EXECUTION_ROLE_ARN> \
  --zip-file fileb://function.zip \
  --timeout 600 \
  --memory-size 256 \
  --environment '{"Variables":{"TEMPORAL_ADDRESS":"<your-temporal-address>:7233","TEMPORAL_NAMESPACE":"<your-namespace>","TEMPORAL_API_KEY":"<your-api-key>"}}'
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:271-281 -->

- `--runtime`: `python3.13` (or another supported Python version). <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:286 -->
- `--handler`: `lambda_function.lambda_handler` (entry point in `module.function` format, must point to the handler returned by `run_worker`). <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:287 -->

#### TypeScript

```bash
aws lambda create-function \
  --function-name my-temporal-worker \
  --runtime nodejs22.x \
  --handler lib/index.handler \
  --role <EXECUTION_ROLE_ARN> \
  --zip-file fileb://function.zip \
  --timeout 600 \
  --memory-size 256 \
  --environment '{"Variables":{"HOME":"/tmp","TEMPORAL_ADDRESS":"<your-temporal-address>:7233","TEMPORAL_NAMESPACE":"<your-namespace>","TEMPORAL_API_KEY":"<your-api-key>"}}'
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:292-302 -->

- `--runtime`: `nodejs22.x` (or another supported Node.js version, 20+). <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:307 -->
- `--handler`: `lib/index.handler` (entry point in `module.export` format, must point to the handler exported by `runWorker`). <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:308 -->

### Common parameters (all SDKs)

<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:315-320 -->

| Parameter | Description |
|---|---|
| `--role` | ARN of the Lambda execution role, which grants the function permission to run (trusted principal: `lambda.amazonaws.com`). This is separate from the role Temporal uses to invoke the function. The role must have at least the `AWSLambdaBasicExecutionRole` managed policy attached. (See `iam.md` for the execution role.) |
| `--zip-file` | Path to your packaged deployment zip. |
| `--timeout` | Invocation deadline in seconds. Maximum time each Lambda invocation can run before AWS terminates it. Set high enough for the Worker to start, process Tasks, and shut down gracefully. |
| `--memory-size` | Memory in MB allocated to each invocation. |

**Caution:** AWS Lambda functions default to a 3-second timeout, which is too short for the Worker to start, connect to Temporal, and register the Task Queue. If the first invocation times out before the Worker polls, the Task Queue binding is never created and the Lambda is never invoked again. Always set `--timeout` high enough for the Worker to start, process Tasks, and shut down gracefully. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:328-337 -->

### Environment variables

<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:321-326 -->

| Variable | Description |
|---|---|
| `TEMPORAL_ADDRESS` | Temporal frontend address (e.g., `<namespace>.<account>.tmprl.cloud:7233`). |
| `TEMPORAL_NAMESPACE` | Temporal Namespace. For Temporal Cloud, the fully-qualified `<namespace_id>.<account_id>`, not the bare name. |
| `TEMPORAL_TASK_QUEUE` | Task Queue name. Overrides the value set in code. |
| `TEMPORAL_TLS_CLIENT_CERT_PATH` | Path to the TLS client certificate file for mTLS authentication. |
| `TEMPORAL_TLS_CLIENT_KEY_PATH` | Path to the TLS client key file for mTLS authentication. |
| `TEMPORAL_API_KEY` | API key for API key authentication. Supplying it auto-enables TLS; mTLS cert paths are not needed. |

The serverless Worker packages read environment variables and configuration files automatically at startup. For the full list of supported environment variables, config file format, and profiles, see the Environment configuration docs (`/develop/environment-configuration`). <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:339-341 -->

Sensitive values like TLS keys and API keys should be encrypted at rest. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:343-344 -->

The `--environment` examples above pass `TEMPORAL_API_KEY` inline for brevity — **that is acceptable for development only.** For production, store the API key (or TLS private key) in AWS Secrets Manager or SSM Parameter Store, grant the *execution* role `secretsmanager:GetSecretValue` (or `ssm:GetParameter`), and load it at cold start before the Worker initializes — for example, at module scope in the handler file, fetch the secret and set `os.environ["TEMPORAL_API_KEY"]` so the serverless Worker package reads it at startup. Do not commit key values into the `--environment` block for production functions. <!-- field note: Secrets Manager wiring not covered upstream; SKILL.md Step 4 forbids plaintext secrets -->

For updating the function code and publishing immutable versions, see `versioning.md`.

## Step 3: Configure IAM for Temporal invocation

Step 3 (execution role, Temporal invocation role, and CloudFormation for Temporal Cloud and self-hosted) lives in `iam.md`. Complete it before Step 4.

## Step 4: Create Worker Deployment Version

Create a Worker Deployment Version with a compute provider that points to your Lambda function. The compute configuration tells Temporal how to invoke your Worker: the provider type (`aws-lambda`), the Lambda function ARN, and the IAM role to assume. The deployment name and build ID must match the values in your Worker code. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:529-532 -->

### Using Temporal UI

<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:539-553 -->

1. In the Temporal UI, open your Namespace.
2. In the left pane, select **Workers**.
3. Click **Create Worker Deployment** in the upper right corner.
4. Under **Configuration**, enter a **Name** and **Build ID** (must match `DeploymentName` and `BuildID` in your Worker code).
5. Under **Compute**, select **AWS Lambda** and provide:
   - **Lambda ARN**: the ARN of your Lambda function.
   - **IAM Role ARN**: the ARN of the role Temporal assumes to invoke your Lambda function (the `RoleARN` output from the CloudFormation stack). This is not the Lambda execution role or your own IAM user/role.
   - **External ID**: the same value passed to the CloudFormation template.
6. Click **Save**.

When you create a version through the UI, the version is automatically set as current. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:552 -->

### Using Temporal CLI

Use the CLI for manual setup, shell scripts, and CI/CD pipelines. When you create a version through the CLI, you must set the version as current as a separate step. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:558-559 -->

**Field note — check your CLI version first.** The `worker deployment create-version` subcommand and its `--aws-lambda-*` flags only exist in recent Temporal CLI builds. Run `temporal --version` and `temporal worker deployment create-version --help`; if the subcommand or flags are missing, upgrade. In testing, a Homebrew-installed v1.5.0 lacked `create-version` entirely, while a standalone v1.8.0 had the serverless flags. You can install a current standalone build without disturbing a packaged one. <!-- field note: 2026-07 serverless deployment test session; not in docs. Exact minimum version unconfirmed against the CLI changelog — treat 1.5.0=absent / 1.8.0=present as observed bounds. -->

First, create the Worker Deployment if it does not already exist: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:561 -->

```bash
temporal worker deployment create \
  --namespace <YOUR_NAMESPACE> \
  --name my-app
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:563-567 -->

Then create the version with the compute provider configuration: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:569 -->

```bash
temporal worker deployment create-version \
  --namespace <YOUR_NAMESPACE> \
  --deployment-name my-app \
  --build-id build-1 \
  --aws-lambda-function-arn <LAMBDA_FUNCTION_ARN> \
  --aws-lambda-assume-role-arn <INVOCATION_ROLE_ARN> \
  --aws-lambda-assume-role-external-id <EXTERNAL_ID>
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:571-579 -->

| Flag | Description |
|---|---|
| `--deployment-name` | Worker Deployment name. Must match `DeploymentName` in your Worker code. |
| `--build-id` | Worker Deployment Version build ID. Must match `BuildID` in your Worker code. |
| `--aws-lambda-function-arn` | Qualified versioned ARN of the Lambda function Temporal invokes for this version (for example, `function:my-worker:5`). An unqualified ARN is also accepted for development. |
| `--aws-lambda-assume-role-arn` | IAM role Temporal assumes to invoke the function. This is the `RoleARN` output from the CloudFormation stack. This is not the Lambda execution role or your own IAM user/role. |
| `--aws-lambda-assume-role-external-id` | External ID configured in the IAM role trust policy. |
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:581-587 -->

### Validate connection

Go to **Workers** > **Deployments** > select your deployment > open the **Actions** menu on the version and click **Validate Connection**. This checks that Temporal can assume the IAM role and invoke the function. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:592-594 -->

## Step 5: Set version as current

If you created the version through the Temporal UI, the version is already current — skip this step. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:598 -->

If you used the CLI, set the version as current. Without this step, tasks on the Task Queue will not route to the version, and Temporal will not invoke the Lambda function. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:600-601 -->

```bash
temporal worker deployment set-current-version \
  --deployment-name my-app \
  --build-id build-1
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:603-607 -->

## Step 6: Verify deployment

Start a Workflow on the same Task Queue to confirm that Temporal invokes your Lambda Worker. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:611 -->

```bash
temporal workflow start \
  --task-queue my-task-queue \
  --type MyWorkflow \
  --input '"Hello, serverless!"'
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:613-618 -->

Verify the invocation by checking: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:623 -->

- **Temporal UI:** The Workflow execution should show task completions in the event history. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:625 -->
- **AWS CloudWatch Logs:** The Lambda function's log group (`/aws/lambda/my-temporal-worker`) should show invocation logs with the Worker startup, task processing, and graceful shutdown. Requires the execution role to have CloudWatch Logs permissions (included in `AWSLambdaBasicExecutionRole`). <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:626-630 -->

If the Workflow does not progress or the Lambda is not invoked, see `diagnostics.md`. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:632-633 -->

## Teardown

To remove a serverless Worker deployment (for example, after a Pre-release trial), tear down in this order so nothing is left invoking or being invoked: <!-- field note: teardown not covered upstream -->

1. Delete the Worker Deployment Version. This stops its WCI (one WCI runs per version with a compute provider). If other versions exist, set another current first with `set-current-version`.
   ```bash
   temporal worker deployment delete-version --deployment-name my-app --build-id build-1
   ```
2. Delete the CloudFormation stack that created the Temporal invocation role:
   ```bash
   aws cloudformation delete-stack --stack-name <STACK_NAME> --region <AWS_REGION>
   ```
3. Delete the Lambda function (removes all published versions) and, if you created a dedicated execution role, delete that role:
   ```bash
   aws lambda delete-function --function-name my-temporal-worker
   ```
4. Revoke the Temporal Cloud API key if it was created only for this deployment (see `skill-temporal-ops`).
