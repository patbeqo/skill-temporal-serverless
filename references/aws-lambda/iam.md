# AWS Lambda — IAM & permissions

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx
  docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx
-->

This file covers three distinct identities: the **operator** (whose credentials run the deploy commands), the **execution role** (grants the function permission to run), and the **Temporal invocation role** (grants Temporal permission to invoke the function). The deploy steps themselves are in `setup.md`; self-hosted server enablement is in `self-hosted.md`.

## Operator AWS permissions

The identity whose credentials run the `aws`/CloudFormation commands (local profile, EC2/ECS instance role, CI role, or CloudShell console identity) needs the actions below. These are separate from the execution and invocation roles created in Step 3. <!-- field note: derived from the commands in this guide; not enumerated in upstream docs -->

| Deployment step | Operator actions required |
|---|---|
| Deploy / update the Lambda (`setup.md`, Step 2) | `lambda:CreateFunction`, `lambda:UpdateFunctionCode`, `lambda:GetFunction`, `lambda:PublishVersion`, `lambda:InvokeFunction`, and **`iam:PassRole`** on the execution role. |
| Create the execution role directly (if not using CloudFormation) | `iam:CreateRole`, `iam:AttachRolePolicy`. |
| Create the invocation role via CloudFormation (Step 3) | `cloudformation:CreateStack`, `cloudformation:DescribeStacks`, plus IAM write (`iam:CreateRole`, `iam:PutRolePolicy`/`iam:AttachRolePolicy`, `iam:GetRole`, `iam:DeleteRole` for rollback) — hence `--capabilities CAPABILITY_NAMED_IAM`. |
| Read Worker logs (`setup.md` verify + `diagnostics.md`) | `logs:FilterLogEvents`, `logs:GetLogEvents`, `logs:DescribeLogGroups`, `logs:DescribeLogStreams`. |

`iam:PassRole` is required separately from `lambda:CreateFunction`: `create-function` attaches the execution role to the function, and without `iam:PassRole` on that role the deploy fails with `AccessDenied ... not authorized to perform: iam:PassRole` even when `CreateFunction` is allowed. Scope it to the execution role ARN.

If the operator cannot have IAM write permissions, have an administrator run the Step 3 CloudFormation stack once and hand back the `RoleARN` output (and pre-create the execution role if needed). The operator then needs only the Lambda actions, `iam:PassRole` on the pre-created execution role, and CloudWatch Logs read.

### Preflight check

Run before any command that creates or modifies AWS resources. None should print `DENIED`:

```bash
aws sts get-caller-identity
aws lambda list-functions --max-items 1 >/dev/null 2>&1 && echo "lambda: ok" || echo "lambda: DENIED"
aws cloudformation describe-stacks >/dev/null 2>&1 && echo "cloudformation: ok" || echo "cloudformation: DENIED"
aws iam list-roles --max-items 1 >/dev/null 2>&1 && echo "iam read: ok" || echo "iam: limited (role creation may be blocked)"
aws logs describe-log-groups --limit 1 >/dev/null 2>&1 && echo "logs: ok" || echo "logs: DENIED"
```

The calls above cannot exercise `iam:PassRole`. To check it (and any other specific action) authoritatively, use the policy simulator against the caller's ARN:

```bash
CALLER_ARN=$(aws sts get-caller-identity --query Arn --output text)
aws iam simulate-principal-policy \
  --policy-source-arn "$CALLER_ARN" \
  --action-names lambda:CreateFunction iam:PassRole cloudformation:CreateStack iam:CreateRole \
  --query 'EvaluationResults[].{action:EvalActionName,decision:EvalDecision}' --output table
```

For an assumed-role session, rewrite the ARN (`arn:aws:sts::…:assumed-role/Name/session`) to the underlying role ARN (`arn:aws:iam::…:role/Name`) for `simulate-principal-policy`.

## Execution role

The Lambda execution role grants the function permission to run. It is trusted by `lambda.amazonaws.com` and must have at least the `AWSLambdaBasicExecutionRole` managed policy attached (which includes the CloudWatch Logs permissions the Worker needs). This is separate from the Temporal invocation role below. Pass its ARN as `--role` when you create the function (`setup.md`, Step 2). <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:317 -->

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

### Self-hosted invocation role

For self-hosted server enablement (dynamic config, WCI, and the server's own AWS credentials), see `self-hosted.md`. The invocation role itself is below.

Self-hosted Serverless Workers require Temporal Service v1.31.0 or later. <!-- docs/production-deployment/worker-deployments/serverless-workers/self-hosted-setup.mdx:28 -->

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
