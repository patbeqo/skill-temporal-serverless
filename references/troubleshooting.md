# Troubleshooting Serverless Workers

<!-- Sources:
  docs/troubleshooting/serverless-workers.mdx
  docs/encyclopedia/workers/serverless-workers.mdx
-->

## Invocation flow (when working correctly)

When a Serverless Worker invocation works correctly, the following sequence happens: <!-- docs/troubleshooting/serverless-workers.mdx:36 -->

1. You deploy the Worker function on Lambda. <!-- docs/troubleshooting/serverless-workers.mdx:38 -->
2. You configure a Worker Deployment Version with a compute provider. This starts a Worker Controller Instance (WCI) Workflow and a validation invocation of the Lambda function. <!-- docs/troubleshooting/serverless-workers.mdx:39 -->
3. The Lambda polls the Temporal Service successfully, binding the Task Queue configured on the Worker to the Worker Deployment Version. <!-- docs/troubleshooting/serverless-workers.mdx:40 -->
4. The WCI continuously monitors the associated Task Queue on a schedule. The Matching Service also notifies the WCI Workflow of sync match failures immediately as they happen. <!-- docs/troubleshooting/serverless-workers.mdx:41 -->
5. A Task arrives on the Task Queue and the WCI detects the backlog. <!-- docs/troubleshooting/serverless-workers.mdx:42 -->
6. The WCI invokes the Lambda function. <!-- docs/troubleshooting/serverless-workers.mdx:43 -->
7. The Lambda function starts, the Worker connects to Temporal and polls the Task Queue. <!-- docs/troubleshooting/serverless-workers.mdx:44 -->
8. The Worker processes Tasks and shuts down gracefully. <!-- docs/troubleshooting/serverless-workers.mdx:45 -->

## Diagnostic decision tree

Start by determining whether the Lambda function is being invoked at all. <!-- docs/troubleshooting/serverless-workers.mdx:47 -->

Check the Lambda function's CloudWatch metrics or invocation logs. In the AWS Console, go to **Lambda > Functions > your function > Monitor**. Look for recent invocations in the **Invocations** graph. You can also check **CloudWatch > Log groups > /aws/lambda/your-function-name** for execution logs. <!-- docs/troubleshooting/serverless-workers.mdx:51-55 -->

- If there are no invocations: see [Lambda is not being invoked](#lambda-not-invoked).
- If the Lambda is being invoked but Workflows are not progressing: see [Lambda is invoked but Tasks are not completing](#lambda-invoked-not-completing).

---

## Lambda is not being invoked {#lambda-not-invoked}

Work through the following checks in order. <!-- docs/troubleshooting/serverless-workers.mdx:64 -->

### 1. Validate the connection to Lambda

Go to **Workers > Deployments > select your deployment**, open the **Actions** menu on the version, and click **Validate Connection**. <!-- docs/troubleshooting/serverless-workers.mdx:68-69 -->

A successful validation confirms that: <!-- docs/troubleshooting/serverless-workers.mdx:69-71 -->
- The Worker Deployment Version has a compute provider configured.
- Temporal can assume the invocation role.
- The Lambda function can be invoked.

If validation fails: <!-- docs/troubleshooting/serverless-workers.mdx:73-76 -->
- Verify the Lambda function ARN and invocation role ARN in the Worker Deployment Version configuration are correct.
- Verify the invocation role was created using the CloudFormation template and that the External ID matches the value in the Worker Deployment Version configuration.

If the Worker Deployment Version does not have a compute provider configured, no WCI Workflow exists and the Lambda is never automatically invoked. <!-- docs/troubleshooting/serverless-workers.mdx:78-79 -->

**Common cause:** Manually invoking the Lambda function before creating the Worker Deployment Version in the UI or CLI. When the Lambda runs, the Worker connects to Temporal and polls the Task Queue. That polling registers the Worker Deployment Version and binds the Task Queue on the server, but the version has no compute provider. To fix, create or update the Worker Deployment Version with the compute provider flags. <!-- docs/troubleshooting/serverless-workers.mdx:80-84 -->

### 2. Check that the version is set as current

The Worker Deployment Version must be set as the current version for new Tasks to route to it. If you created the version through the CLI, you need to set it as current. <!-- docs/troubleshooting/serverless-workers.mdx:88-90 -->

Verify the current version with: <!-- docs/troubleshooting/serverless-workers.mdx:92 -->

```bash
temporal worker deployment describe
```

### 3. Check that the WCI is detecting Tasks

If the connection validates successfully but the Lambda is still not being invoked, the WCI may not be detecting Tasks on the Task Queue. <!-- docs/troubleshooting/serverless-workers.mdx:96-98 -->

Check which Task Queues are bound to the Worker Deployment Version and whether there is a backlog: <!-- docs/troubleshooting/serverless-workers.mdx:100 -->

```bash
temporal worker deployment describe-version \
  --namespace <NAMESPACE> \
  --deployment-name <DEPLOYMENT_NAME> \
  --build-id <BUILD_ID> \
  --report-task-queue-stats
```
<!-- docs/troubleshooting/serverless-workers.mdx:102-108 -->

If no Task Queues are listed, the binding has not been established. The server binds a Task Queue to a Worker Deployment Version when a Worker with that deployment version successfully connects and polls the Task Queue. <!-- docs/troubleshooting/serverless-workers.mdx:110-111 -->

### 4. Failed first invocation

A common cause of missing Task Queue bindings is a failed first invocation. When you create a Worker Deployment Version, the WCI invokes the Lambda to validate the configuration. If that first invocation fails (for example, due to missing environment variables, incorrect TLS configuration, or missing dependencies), the Worker never connects to Temporal and never polls. Without a successful poll, the Task Queue binding is never created. <!-- docs/troubleshooting/serverless-workers.mdx:113-116 -->

To diagnose: invoke the Lambda function manually from the AWS Console. The console displays the execution result and any errors directly, making it easier to identify configuration issues than searching through CloudWatch logs. Once the Lambda runs successfully and the Worker connects to Temporal, the Task Queue binding is established. <!-- docs/troubleshooting/serverless-workers.mdx:118-121 -->

---

## Lambda is invoked but Tasks are not completing {#lambda-invoked-not-completing}

If CloudWatch shows Lambda invocations but Workflows are not progressing, the problem is in the Worker's execution within the Lambda function. <!-- docs/troubleshooting/serverless-workers.mdx:125-126 -->

### Check Lambda execution logs

Check CloudWatch logs for errors during Worker startup. In the AWS Console, go to **CloudWatch > Log groups > /aws/lambda/your-function-name** and look for recent error messages. <!-- docs/troubleshooting/serverless-workers.mdx:130-131 -->

Common errors include: <!-- docs/troubleshooting/serverless-workers.mdx:133 -->

- **Connection failures**: The Worker cannot reach the Temporal Service. Check that the `TEMPORAL_ADDRESS` and `TEMPORAL_API_KEY` environment variables (or `temporal.toml` config file) are correctly set on the Lambda function. For self-hosted deployments, verify network reachability. <!-- docs/troubleshooting/serverless-workers.mdx:135-138 -->
- **TLS errors**: The TLS certificate or key is missing, expired, or does not match the Namespace. <!-- docs/troubleshooting/serverless-workers.mdx:139 -->
- **Authentication errors**: The API key is invalid or does not have access to the Namespace. <!-- docs/troubleshooting/serverless-workers.mdx:140 -->

### Check for Lambda timeout

If the Lambda function reaches its configured timeout before the Worker finishes processing, AWS terminates the invocation. <!-- docs/troubleshooting/serverless-workers.mdx:144-145 -->

The Worker begins graceful shutdown before the Lambda deadline. If Activities take longer than the available execution window, the Activities are abandoned mid-execution and retried on the next invocation. <!-- docs/troubleshooting/serverless-workers.mdx:147-148 -->

For long-running Activities, increase the Lambda timeout and the Worker's shutdown buffer together. <!-- docs/troubleshooting/serverless-workers.mdx:150-152 -->

### Check that the deployment name and build ID match

If CloudWatch shows rapid, repeated invocations with no Workflow progress, the deployment name or build ID in the Worker code may not match the Worker Deployment Version configuration. <!-- docs/troubleshooting/serverless-workers.mdx:156-157 -->

The deployment name and build ID in your Lambda function code must exactly match the values you used when creating the Worker Deployment Version. Compare the values in your code against the WCI Workflow ID (`temporal-sys-worker-controller-instance:<deployment-name>:<build-id>`) and the output of `temporal worker deployment describe`. <!-- docs/troubleshooting/serverless-workers.mdx:159-162 -->

A mismatch causes an invocation loop: the WCI invokes the Lambda, the Worker starts and polls with a different deployment version than the WCI expects, the Task is not processed, and the WCI invokes the Lambda again. <!-- docs/troubleshooting/serverless-workers.mdx:164-165 -->

To fix the loop, update the deployment name and build ID in the Worker code to match the Worker Deployment Version, then redeploy the Lambda function. <!-- docs/troubleshooting/serverless-workers.mdx:167-168 -->

---

## WCI Workflow inspection

List WCI Workflows in your Namespace: <!-- docs/encyclopedia/workers/serverless-workers.mdx:76 -->

```bash
temporal workflow list \
  --namespace <NAMESPACE> \
  --query 'TemporalNamespaceDivision = "TemporalWorkerControllerInstance"'
```
<!-- docs/encyclopedia/workers/serverless-workers.mdx:78-82 -->

WCI Workflow IDs follow the pattern `temporal-sys-worker-controller-instance:<deployment-name>:<build-id>`. <!-- docs/encyclopedia/workers/serverless-workers.mdx:84 -->

Inspect a WCI Workflow's history to see its recent Activity results: <!-- docs/encyclopedia/workers/serverless-workers.mdx:85 -->

```bash
temporal workflow show \
  --namespace <NAMESPACE> \
  --workflow-id 'temporal-sys-worker-controller-instance:<DEPLOYMENT_NAME>:<BUILD_ID>'
```
<!-- docs/encyclopedia/workers/serverless-workers.mdx:87-91 -->

Describe a Worker Deployment Version and check Task Queue stats: <!-- docs/troubleshooting/serverless-workers.mdx:102 -->

```bash
temporal worker deployment describe-version \
  --namespace <NAMESPACE> \
  --deployment-name <DEPLOYMENT_NAME> \
  --build-id <BUILD_ID> \
  --report-task-queue-stats
```
<!-- docs/troubleshooting/serverless-workers.mdx:102-108 -->
