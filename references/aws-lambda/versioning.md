# AWS Lambda — Versioning, updates & rollback

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx
-->

For the conceptual model — Temporal Worker Deployment Versions vs. Lambda function versions, and Pinned vs. Auto-Upgrade behavior — see `../concepts.md` ("Worker Versioning with Serverless Workers").

## Update existing function

```bash
aws lambda update-function-code \
  --function-name my-temporal-worker \
  --zip-file fileb://function.zip
```
<!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:348-352 -->

After updating, increment the Build ID in your Worker code and publish a new Lambda function version. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:354-355 -->

## Publish a Lambda function version

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
