# AWS Lambda Deployment

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx
  docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx
  docs/production-deployment/worker-deployments/serverless-workers/index.mdx
-->

## Prerequisites

- A Temporal Cloud account with an AWS-hosted Namespace, or a self-hosted Temporal Service v1.31.0 or later. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:37 -->
- The Namespace's cloud provider must match the serverless compute provider. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:37-38 -->
- For self-hosted deployments, complete the self-hosted setup before following the deployment guide. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:39-41 -->
- Every Workflow must declare a versioning behavior, or the Worker must set a default versioning behavior. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:42-43 -->
- An AWS account with permissions to create and invoke Lambda functions and create IAM roles. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:44 -->
- The AWS-specific steps require the `aws` CLI installed and configured with your AWS credentials. You may also use the AWS Console or the AWS SDKs. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:45-46 -->
- The Go SDK, Python SDK, or TypeScript SDK, depending on your language. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:48-49 -->

Sample projects:
- Go: [Go Lambda Worker sample](https://github.com/temporalio/samples-go/tree/main/lambda-worker) <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:54 -->
- Python: [Python Lambda Worker sample](https://github.com/temporalio/samples-python/tree/main/lambda_worker) <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:55 -->
- TypeScript: [TypeScript Lambda Worker sample](https://github.com/temporalio/samples-typescript/tree/main/lambda-worker) <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:56 -->

## Step 1: Write Worker code

The Worker handles the per-invocation lifecycle: connecting to Temporal, polling for tasks, and gracefully shutting down before the invocation deadline. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:62-63 -->

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
| `--role` | ARN of the Lambda execution role, which grants the function permission to run (trusted principal: `lambda.amazonaws.com`). This is separate from the role Temporal uses to invoke the function. The role must have at least the `AWSLambdaBasicExecutionRole` managed policy attached. |
| `--zip-file` | Path to your packaged deployment zip. |
| `--timeout` | Invocation deadline in seconds. Maximum time each Lambda invocation can run before AWS terminates it. Set high enough for the Worker to start, process Tasks, and shut down gracefully. |
| `--memory-size` | Memory in MB allocated to each invocation. |

**Caution:** AWS Lambda functions default to a 3-second timeout, which is too short for the Worker to start, connect to Temporal, and register the Task Queue. If the first invocation times out before the Worker polls, the Task Queue binding is never created and the Lambda is never invoked again. Always set `--timeout` high enough for the Worker to start, process Tasks, and shut down gracefully. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:328-337 -->

### Environment variables

<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:321-326 -->

| Variable | Description |
|---|---|
| `TEMPORAL_ADDRESS` | Temporal frontend address (e.g., `<namespace>.<account>.tmprl.cloud:7233`). |
| `TEMPORAL_NAMESPACE` | Temporal Namespace. |
| `TEMPORAL_TASK_QUEUE` | Task Queue name. Overrides the value set in code. |
| `TEMPORAL_TLS_CLIENT_CERT_PATH` | Path to the TLS client certificate file for mTLS authentication. |
| `TEMPORAL_TLS_CLIENT_KEY_PATH` | Path to the TLS client key file for mTLS authentication. |
| `TEMPORAL_API_KEY` | API key for API key authentication. |

The serverless Worker packages read environment variables and configuration files automatically at startup. For the full list of supported environment variables, config file format, and profiles, see the Environment configuration docs (`/develop/environment-configuration`). <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:339-341 -->

Sensitive values like TLS keys and API keys should be encrypted at rest. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:343-344 -->

### Update existing function

```bash
aws lambda update-function-code \
  --function-name my-temporal-worker \
  --zip-file fileb://function.zip
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:348-352 -->

After updating, increment the Build ID in your Worker code and publish a new Lambda function version. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:354-355 -->

### Publish a Lambda function version

For production, create an immutable snapshot of your Lambda code after creating the function and after each `update-function-code`, and maintain a one-to-one mapping between each Lambda function version and each Temporal Worker Deployment Version Build ID. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:359,371 -->

```bash
aws lambda publish-version \
  --function-name my-temporal-worker \
  --description "Build ID build-5"
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:362-365 -->

The command prints the `FunctionArn` for the new version, for example `arn:aws:lambda:us-east-1:123456789012:function:my-temporal-worker:5`. Use this qualified versioned ARN when you create the Worker Deployment Version. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:367-369 -->

For development or non-critical workloads, you can skip `publish-version` and use an unqualified ARN to iterate faster. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:372 -->

To roll back, revert the Temporal Current Version with `temporal worker deployment set-current-version`. The previous Worker Deployment Version still points at its original Lambda function version and is ready to receive traffic again. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:375-377 -->

## Step 3: Configure IAM for Temporal invocation

### Temporal Cloud

This section applies to Temporal Cloud. For self-hosted, see the self-hosted section below. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:379-383 -->

Temporal needs permission to invoke your Lambda function and check its status. The Temporal server assumes an IAM role in your AWS account with a handful of Lambda permissions scoped to your Worker functions. The trust policy on the role includes an External ID condition to prevent confused deputy attacks. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:385-388 -->

#### CloudFormation template parameters

<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:394-398 -->

| Parameter | Description |
|---|---|
| `AssumeRoleExternalId` | A string you choose to prevent confused deputy attacks. Can be any value. Use the same value when creating the Worker Deployment Version. |
| `LambdaFunctionARNs` | Comma-separated list of Lambda function ARNs that Temporal may invoke. To allow invocation of any published version of a function, use a wildcard suffix (for example, `arn:aws:lambda:...:function:my-temporal-worker:*`). One role can authorize multiple Worker Lambdas. |
| `RoleName` | Base name for the created IAM role. Defaults to `Temporal-Cloud-Serverless-Worker`. Provide a new role name if creating more than one stack. |

#### Trust policy principals

The Cloud template trusts five Temporal Cloud AWS account IDs with the role `wci-lambda-invoke`: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:457-464 -->

- `arn:aws:iam::902542641901:role/wci-lambda-invoke` <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:459 -->
- `arn:aws:iam::160190466495:role/wci-lambda-invoke` <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:460 -->
- `arn:aws:iam::819232936619:role/wci-lambda-invoke` <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:461 -->
- `arn:aws:iam::829909441867:role/wci-lambda-invoke` <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:462 -->
- `arn:aws:iam::354116250941:role/wci-lambda-invoke` <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:463 -->

The IAM policy grants `lambda:InvokeFunction` and `lambda:GetFunction` on the specified Lambda function ARNs. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:481-484 -->

#### Deploy the CloudFormation stack

This skill ships the complete, ready-to-deploy template at `assets/temporal-cloud-serverless-worker-role.yaml` (transcribed verbatim from the docs). Copy it into your working directory, or point `--template-body` at the skill's copy — no need to author it by hand. <!-- field note: template shipped as skill asset; verbatim from docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:404-501 -->

```bash
aws cloudformation create-stack \
  --stack-name <STACK_NAME> \
  --template-body file://temporal-cloud-serverless-worker-role.yaml \
  --parameters \
    ParameterKey=AssumeRoleExternalId,ParameterValue=<EXTERNAL_ID> \
    ParameterKey=LambdaFunctionARNs,ParameterValue='"<LAMBDA_FUNCTION_ARN>"' \
  --capabilities CAPABILITY_NAMED_IAM \
  --region <AWS_REGION>
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:509-517 -->

Retrieve the IAM role ARN from the stack outputs: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:519 -->

```bash
aws cloudformation describe-stacks --stack-name <STACK_NAME> --query 'Stacks[0].Outputs[?OutputKey==`RoleARN`].OutputValue' --output text --region <AWS_REGION>
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:521-522 -->

### Self-hosted Temporal Service

Self-hosted Serverless Workers require Temporal Service v1.31.0 or later. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:28 -->

#### Network reachability

The Temporal Service frontend must be reachable from the Lambda execution environment. If the Temporal Service runs on a private network, you may need VPC access for Lambda, VPC peering, or a similar mechanism. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:44-47 -->

#### Enable the Worker Controller Instance (WCI)

WCI is disabled by default and must be enabled through dynamic configuration. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:52-53 -->

Add the following keys to your dynamic config file: <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:55 -->

```yaml
workercontroller.enabled:
  - value: true

workercontroller.compute_providers.enabled:
  - value:
      - aws-lambda

workercontroller.scaling_algorithms.enabled:
  - value:
      - no-sync
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:57-68 -->

To enable WCI for specific Namespaces instead of globally, add a `constraints` section with the Namespace name under `workercontroller.enabled`: <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:70-71 -->

```yaml
workercontroller.enabled:
  - value: true
    constraints:
      namespace: 'your-namespace'
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:73-78 -->

The Temporal Service watches the dynamic config file for changes and applies updates without a restart. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:80 -->

#### Configure AWS credentials

The Temporal Service needs AWS credentials to assume an IAM role that invokes Lambda functions. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:84 -->

**On AWS infrastructure (EC2, ECS, EKS):** The server uses the attached instance role, task role, or pod role automatically. No additional credential configuration is needed. The attached role must have `sts:AssumeRole` permission for the Lambda invocation role. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:87-89 -->

**Outside AWS:** Use IAM Roles Anywhere, or configure static AWS credentials in the server's environment (not recommended). These credentials must belong to an IAM user or role that has `sts:AssumeRole` permission for the Lambda invocation role. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:91-101 -->

```
AWS_ACCESS_KEY_ID=<access-key>
AWS_SECRET_ACCESS_KEY=<secret-key>
AWS_REGION=<region>
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:94-98 -->

#### Create the Lambda invocation role (self-hosted)

Temporal invokes Lambda functions by assuming an IAM role in your AWS account. This role needs `lambda:GetFunction` and `lambda:InvokeFunction` permission on your Worker Lambda functions, and a trust policy that allows the Temporal server's identity to assume it. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:105-107 -->

This skill ships the complete self-hosted template at `assets/temporal-self-hosted-serverless-worker-role.yaml` (verbatim from the docs). Copy it locally or point `--template-body` at the skill's copy. <!-- field note: template shipped as skill asset; verbatim from docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:136-209 -->

```bash
aws cloudformation create-stack \
  --stack-name temporal-serverless-worker \
  --template-body file://temporal-self-hosted-serverless-worker-role.yaml \
  --parameters \
    ParameterKey=TemporalIamRoleArn,ParameterValue=<TEMPORAL_SERVER_ROLE_ARN> \
    ParameterKey=AssumeRoleExternalId,ParameterValue=<EXTERNAL_ID> \
    ParameterKey=LambdaFunctionARNs,ParameterValue='"<LAMBDA_FUNCTION_ARN>"' \
  --capabilities CAPABILITY_NAMED_IAM \
  --region <AWS_REGION>
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:113-123 -->

| Parameter | Description |
|---|---|
| `TemporalIamRoleArn` | ARN of the IAM role or user that the Temporal Service runs as (the identity used to call `sts:AssumeRole`). Run `aws sts get-caller-identity` in the server's environment to find it. |
| `AssumeRoleExternalId` | A unique string to prevent confused deputy attacks. Use the same value when creating the Worker Deployment Version. |
| `LambdaFunctionARNs` | Comma-separated list of Lambda function ARNs that Temporal may invoke. To allow any published version, use a wildcard suffix (for example, `arn:aws:lambda:...:function:my-temporal-worker:*`). |
| `RoleName` | Base name for the created IAM role. Defaults to `Temporal-Serverless-Worker`. Provide a new role name if creating more than one stack. |
<!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:125-130 -->

Retrieve the role ARN: <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:214 -->

```bash
aws cloudformation describe-stacks \
  --stack-name temporal-serverless-worker \
  --query 'Stacks[0].Outputs[?OutputKey==`RoleARN`].OutputValue' \
  --output text \
  --region <AWS_REGION>
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:216-222 -->

**Key distinction:** The Lambda execution role (trusted by `lambda.amazonaws.com`) is separate from the Temporal invocation role (trusted by Temporal's `wci-lambda-invoke` principals for Cloud, or the Temporal Service's own IAM identity for self-hosted). The execution role grants the function permission to run. The invocation role grants Temporal permission to invoke the function. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:317,548,586 -->

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

If the Workflow does not progress or the Lambda is not invoked, see Troubleshooting Serverless Workers. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:632-633 -->
