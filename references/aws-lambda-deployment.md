# AWS Lambda Deployment

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx
  docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx
  docs/production-deployment/worker-deployments/serverless-workers/index.mdx
-->

## Prerequisites

- A Temporal Cloud account with an AWS-hosted Namespace, or a self-hosted Temporal Service v1.31.0 or later. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:38 -->
- The Namespace's cloud provider must match the serverless compute provider. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:39 -->
- For self-hosted deployments, complete the self-hosted setup before following the deployment guide. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:40-42 -->
- Every Workflow must declare a versioning behavior, or the Worker must set a default versioning behavior. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:43-44 -->
- An AWS account with permissions to create and invoke Lambda functions and create IAM roles. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:45 -->
- The `aws` CLI installed and configured with your AWS credentials. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:46-47 -->
- The Go SDK, Python SDK, or TypeScript SDK, depending on your language. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:49-50 -->

Sample projects:
- Go: [Go Lambda Worker sample](https://github.com/temporalio/samples-go/tree/main/lambda-worker) <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:55 -->
- Python: [Python Lambda Worker sample](https://github.com/temporalio/samples-python/tree/main/lambda_worker) <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:56 -->
- TypeScript: [TypeScript Lambda Worker sample](https://github.com/temporalio/samples-typescript/tree/main/lambda-worker) <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:57 -->

## Step 1: Write Worker code

The Worker handles the per-invocation lifecycle: connecting to Temporal, polling for tasks, and gracefully shutting down before the invocation deadline. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:63-64 -->

### Go

Use the Go SDK's `lambdaworker` package. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:69 -->

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
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:71-95 -->

Versioning behavior: set per-Workflow at registration time with `workflow.VersioningBehaviorPinned` or `workflow.VersioningBehaviorAutoUpgrade`, or set a Worker-level default with `DefaultVersioningBehavior` in `DeploymentOptions`. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:97-99 -->

### Python

Use the Python SDK's `lambda_worker` contrib package. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:107 -->

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
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:109-130 -->

Versioning behavior: set per-Workflow in the `@workflow.defn` decorator with `VersioningBehavior.PINNED` or `VersioningBehavior.AUTO_UPGRADE`, or set a Worker-level default with `default_versioning_behavior` in the worker config. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:132-134 -->

```python
@workflow.defn(versioning_behavior=VersioningBehavior.PINNED)
class MyWorkflow:
    @workflow.run
    async def run(self, input: str) -> str:
        ...
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:136-146 -->

### TypeScript

Use the `@temporalio/lambda-worker` package. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:154 -->

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
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:156-168 -->

Use `workflowBundle` with pre-bundled code instead of `workflowsPath` to avoid webpack bundling overhead on Lambda cold starts. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:170-171 -->

Versioning behavior: set per-Workflow with `setWorkflowOptions` in the Workflow file, or set a default for all Workflows with `defaultVersioningBehavior` in the configure callback. Values are `'AUTO_UPGRADE'` or `'PINNED'`. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:173-175 -->

## Step 2: Deploy Lambda function

### Build and package

#### Go

Cross-compile for Lambda's Linux runtime: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:194 -->

```bash
GOOS=linux GOARCH=amd64 go build -tags lambda.norpc -o bootstrap ./worker
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:195 -->

Package the binary into a zip file: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:199 -->

```bash
zip function.zip bootstrap
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:201 -->

#### Python

Install dependencies into a local directory for packaging, using `--platform` for Linux-compatible binaries: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:207-208 -->

```bash
pip install --target ./package --platform manylinux2014_x86_64 --only-binary=:all: temporalio
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:211 -->

To include OpenTelemetry support, install `temporalio[lambda-worker-otel]` instead. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:214-215 -->

Package dependencies and application code: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:217 -->

```bash
cd package && zip -r ../function.zip . && cd ..
zip function.zip lambda_function.py my_workflows.py my_activities.py
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:219-222 -->

#### TypeScript

Build the Workflow bundle and compile the project: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:228 -->

```bash
npx ts-node src/scripts/build-workflow-bundle.ts
npx tsc
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:229-231 -->

Install production dependencies and package everything: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:235 -->

```bash
npm install --omit=dev
zip -r function.zip lib/ node_modules/ workflow-bundle.js
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:237-239 -->

### Deploy the Lambda function

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
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:252-261 -->

- `--runtime`: `provided.al2023` for custom Go binaries. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:253,267 -->
- `--handler`: `bootstrap` when using the `provided.al2023` custom runtime. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:268 -->

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
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:273-282 -->

- `--runtime`: `python3.13` <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:275 -->
- `--handler`: `lambda_function.lambda_handler` (entry point in `module.function` format, must point to the handler returned by `run_worker`). <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:288 -->

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
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:294-303 -->

- `--runtime`: `nodejs22.x` (Node.js 20+). <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:299,308 -->
- `--handler`: `lib/index.handler` (entry point in `module.export` format, must point to the handler exported by `runWorker`). <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:309 -->

### Common parameters (all SDKs)

<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:316-328 -->

| Parameter | Description |
|---|---|
| `--role` | ARN of the Lambda execution role (trusted principal: `lambda.amazonaws.com`). This is separate from the role Temporal uses to invoke the function. The role must have at least the `AWSLambdaBasicExecutionRole` managed policy attached. |
| `--zip-file` | Path to your packaged deployment zip. |
| `--timeout` | Invocation deadline in seconds. Maximum time each Lambda invocation can run before AWS terminates it. Set high enough for the Worker to start, process Tasks, and shut down gracefully. |
| `--memory-size` | Memory in MB allocated to each invocation. |

### Environment variables

<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:322-328 -->

| Variable | Description |
|---|---|
| `TEMPORAL_ADDRESS` | Temporal frontend address (e.g., `<namespace>.<account>.tmprl.cloud:7233`). |
| `TEMPORAL_NAMESPACE` | Temporal Namespace. |
| `TEMPORAL_TASK_QUEUE` | Task Queue name. Overrides the value set in code. |
| `TEMPORAL_TLS_CLIENT_CERT_PATH` | Path to the TLS client certificate file for mTLS authentication. |
| `TEMPORAL_TLS_CLIENT_KEY_PATH` | Path to the TLS client key file for mTLS authentication. |
| `TEMPORAL_API_KEY` | API key for API key authentication. |

The serverless Worker packages read environment variables and configuration files automatically at startup. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:329-331 -->

Sensitive values like TLS keys and API keys should be encrypted at rest. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:333-334 -->

### Update existing function

```bash
aws lambda update-function-code \
  --function-name my-temporal-worker \
  --zip-file fileb://function.zip
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:338-342 -->

**Lambda versioning best practice:** Create a 1-to-1 mapping between each build ID in your Worker code and a Lambda function version. If you use an unversioned Lambda, do not change the Build Id in your Worker code without also creating a new Worker Deployment Version. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:344-351 -->

## Step 3: Configure IAM for Temporal invocation

### Temporal Cloud

Temporal needs permission to invoke your Lambda function. The Temporal server assumes an IAM role in your AWS account to call `lambda:InvokeFunction`. The trust policy on the role includes an External ID condition to prevent confused deputy attacks. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:359-361 -->

#### CloudFormation template parameters

<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:366-371 -->

| Parameter | Description |
|---|---|
| `AssumeRoleExternalId` | A string you choose to prevent confused deputy attacks. Can be any value. Use the same value when creating the Worker Deployment Version. |
| `LambdaFunctionARNs` | Comma-separated list of Lambda function ARNs that Temporal may invoke. One role can authorize multiple Worker Lambdas. |
| `RoleName` | Base name for the created IAM role. Defaults to `Temporal-Cloud-Serverless-Worker`. |

#### Trust policy principals

The Cloud template trusts five Temporal Cloud AWS account IDs with the role `wci-lambda-invoke`: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:429-436 -->

- `arn:aws:iam::902542641901:role/wci-lambda-invoke` <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:431 -->
- `arn:aws:iam::160190466495:role/wci-lambda-invoke` <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:432 -->
- `arn:aws:iam::819232936619:role/wci-lambda-invoke` <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:433 -->
- `arn:aws:iam::829909441867:role/wci-lambda-invoke` <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:434 -->
- `arn:aws:iam::354116250941:role/wci-lambda-invoke` <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:435 -->

The IAM policy grants `lambda:InvokeFunction` and `lambda:GetFunction` on the specified Lambda function ARNs. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:454-455 -->

#### Deploy the CloudFormation stack

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
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:481-489 -->

Retrieve the IAM role ARN from the stack outputs: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:491 -->

```bash
aws cloudformation describe-stacks --stack-name <STACK_NAME> --query 'Stacks[0].Outputs[?OutputKey==`RoleARN`].OutputValue' --output text --region <AWS_REGION>
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:493-494 -->

### Self-hosted Temporal Service

Self-hosted Serverless Workers require Temporal Service v1.31.0 or later. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:29 -->

#### Network reachability

The Temporal Service frontend must be reachable from the Lambda execution environment. If the Temporal Service runs on a private network, you may need VPC access for Lambda, VPC peering, or a similar mechanism. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:45-48 -->

#### Enable the Worker Controller Instance (WCI)

WCI is disabled by default and must be enabled through dynamic configuration. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:53-54 -->

Add the following keys to your dynamic config file: <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:56 -->

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
<!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:58-69 -->

To enable WCI for specific Namespaces instead of globally, add a `constraints` section with the Namespace name under `workercontroller.enabled`: <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:71-72 -->

```yaml
workercontroller.enabled:
  - value: true
    constraints:
      namespace: 'your-namespace'
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:74-79 -->

The Temporal Service watches the dynamic config file for changes and applies updates without a restart. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:81 -->

#### Configure AWS credentials

The Temporal Service needs AWS credentials to assume an IAM role that invokes Lambda functions. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:85 -->

**On AWS infrastructure (EC2, ECS, EKS):** The server uses the attached instance role, task role, or pod role automatically. No additional credential configuration is needed. The attached role must have `sts:AssumeRole` permission for the Lambda invocation role. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:88-90 -->

**Outside AWS:** Use IAM Roles Anywhere, or configure static AWS credentials in the server's environment (not recommended): <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:92-93 -->

```
AWS_ACCESS_KEY_ID=<access-key>
AWS_SECRET_ACCESS_KEY=<secret-key>
AWS_REGION=<region>
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:95-99 -->

#### Create the Lambda invocation role (self-hosted)

The self-hosted CloudFormation template creates a role that the Temporal Service can assume. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:106-108 -->

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
<!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:114-124 -->

| Parameter | Description |
|---|---|
| `TemporalIamRoleArn` | ARN of the IAM role or user that the Temporal Service runs as. Run `aws sts get-caller-identity` in the server's environment to find it. |
| `AssumeRoleExternalId` | A unique string to prevent confused deputy attacks. Use the same value when creating the Worker Deployment Version. |
| `LambdaFunctionARNs` | Comma-separated list of Lambda function ARNs that Temporal may invoke. |
| `RoleName` | Base name for the created IAM role. Defaults to `Temporal-Serverless-Worker`. |
<!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:126-132 -->

Retrieve the role ARN: <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:215 -->

```bash
aws cloudformation describe-stacks \
  --stack-name temporal-serverless-worker \
  --query 'Stacks[0].Outputs[?OutputKey==`RoleARN`].OutputValue' \
  --output text \
  --region <AWS_REGION>
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:217-223 -->

**Key distinction:** The Lambda execution role (trusted by `lambda.amazonaws.com`) is separate from the Temporal invocation role (trusted by Temporal's `wci-lambda-invoke` principals for Cloud, or the Temporal Service's own IAM identity for self-hosted). The execution role grants the function permission to run. The invocation role grants Temporal permission to invoke the function. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:318,519,558 -->

## Step 4: Create Worker Deployment Version

Create a Worker Deployment Version with a compute provider that points to your Lambda function. The deployment name and build ID must match the values in your Worker code. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:501-504 -->

### Using Temporal UI

<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:509-526 -->

1. In the Temporal UI, open your Namespace.
2. In the left pane, select **Workers**.
3. Click **Create Worker Deployment** in the upper right corner.
4. Under **Configuration**, enter a **Name** and **Build ID** (must match `DeploymentName` and `BuildID` in your Worker code).
5. Under **Compute**, select **AWS Lambda** and provide:
   - **Lambda ARN**: the ARN of your Lambda function.
   - **IAM Role ARN**: the ARN of the role Temporal assumes to invoke your Lambda function (the `RoleARN` output from the CloudFormation stack). This is not the Lambda execution role or your own IAM user/role.
   - **External ID**: the same value passed to the CloudFormation template.
6. Click **Save**.

When you create a version through the UI, the version is automatically set as current. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:524-525 -->

### Using Temporal CLI

First, create the Worker Deployment if it does not already exist: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:533 -->

```bash
temporal worker deployment create \
  --namespace <YOUR_NAMESPACE> \
  --name my-app
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:535-539 -->

Then create the version with the compute provider configuration: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:541 -->

```bash
temporal worker deployment create-version \
  --namespace <YOUR_NAMESPACE> \
  --deployment-name my-app \
  --build-id build-1 \
  --aws-lambda-function-arn <LAMBDA_FUNCTION_ARN> \
  --aws-lambda-assume-role-arn <INVOCATION_ROLE_ARN> \
  --aws-lambda-assume-role-external-id <EXTERNAL_ID>
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:543-551 -->

| Flag | Description |
|---|---|
| `--deployment-name` | Worker Deployment name. Must match `DeploymentName` in your Worker code. |
| `--build-id` | Worker Deployment Version build ID. Must match `BuildID` in your Worker code. |
| `--aws-lambda-function-arn` | ARN of the Lambda function Temporal invokes for this version. |
| `--aws-lambda-assume-role-arn` | IAM role Temporal assumes to invoke the function. This is the `RoleARN` output from the CloudFormation stack. This is not the Lambda execution role or your own IAM user/role. |
| `--aws-lambda-assume-role-external-id` | External ID configured in the IAM role trust policy. |
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:553-559 -->

### Validate connection

Go to **Workers** > **Deployments** > select your deployment > open the **Actions** menu on the version and click **Validate Connection**. This checks that Temporal can assume the IAM role and invoke the function. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:564-566 -->

## Step 5: Set version as current

If you created the version through the Temporal UI, the version is already current — skip this step. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:570 -->

If you used the CLI, set the version as current. Without this step, tasks on the Task Queue will not route to the version, and Temporal will not invoke the Lambda function. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:572-573 -->

```bash
temporal worker deployment set-current-version \
  --deployment-name my-app \
  --build-id build-1
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:575-579 -->

## Step 6: Verify deployment

Start a Workflow on the same Task Queue to confirm that Temporal invokes your Lambda Worker. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:583 -->

```bash
temporal workflow start \
  --task-queue my-task-queue \
  --type MyWorkflow \
  --input '"Hello, serverless!"'
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:585-590 -->

Verify the invocation by checking: <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:595 -->

- **Temporal UI:** The Workflow execution should show task completions in the event history. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:597 -->
- **AWS CloudWatch Logs:** The Lambda function's log group (`/aws/lambda/my-temporal-worker`) should show invocation logs with the Worker startup, task processing, and graceful shutdown. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:598-599 -->

If the Workflow does not progress or the Lambda is not invoked, see Troubleshooting Serverless Workers. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:601 -->
