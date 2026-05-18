---
name: temporal-serverless
description: 'Deploy and operate Temporal Workers on serverless compute (AWS Lambda) using Worker Cloud Invocations (WCI). Covers concepts, deployment, SDK configuration, observability, and troubleshooting for serverless Temporal Workers. Use when the user mentions: "serverless worker", "Temporal serverless", "Worker Cloud Invocation", "WCI", "deploy Temporal worker on Lambda", "Lambda packaging", "serverless compute", "Lambda timeout", "ADOT layer", "ADOT", "X-Ray tracing", "OpenTelemetry Lambda", "Lambda not invoked", "workflows not progressing", "WCI inspection", "Lambda-tuned defaults", "serverless worker config", "CloudFormation Temporal", "IAM Temporal Lambda", "Worker Deployment Version Lambda", "TOML config serverless", "serverless Temporal worker Go", "serverless Temporal worker Python", "serverless Temporal worker TypeScript", "15-minute Activity limit", "Eager Activities not supported", "Pre-release serverless". Does NOT cover general SDK development or Workflow/Activity patterns (temporal-developer), traditional Worker tuning such as slot suppliers or poller autoscaling (temporal-workertuning), Temporal Cloud administration like namespaces or billing (temporal-ops), or CLI command reference beyond serverless-specific flags (temporal-cli).'
version: 0.1.0
---

# Skill: temporal-serverless

## Overview

This skill helps users deploy and operate Temporal Workers on serverless compute (currently AWS Lambda). It produces Worker code, CloudFormation templates, IAM policies, TOML connection configs, Lambda packaging scripts, and OTel collector configs. It also walks users through troubleshooting when their serverless Workers aren't picking up tasks.

Serverless Workers are in **Pre-release** — always surface this early. APIs may change, access is limited to select Temporal Cloud customers, and users should confirm eligibility with Temporal before building production workloads on serverless Workers.

## How to Use This Skill

### Check available credentials and adapt

At the start of a session, check what tools you have access to by running `aws sts get-caller-identity` and `temporal env list` (or checking for `TEMPORAL_ADDRESS` / `TEMPORAL_API_KEY` env vars). Your behavior changes based on what's available:

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

### Troubleshooting

If you have AWS CLI access, run diagnostic commands directly (`aws lambda get-function`, `aws logs filter-log-events`, etc.). If you have Temporal CLI access, inspect the WCI Workflow and deployment versions. Otherwise, walk the user through the diagnostic tree — tell them what to check (CloudWatch logs, WCI state in the UI, deployment version status) and interpret what they report back.

## Key Facts

- **Pre-release** — available to select Temporal Cloud customers only.
- Supported SDKs: Go, Python, TypeScript. Supported compute: AWS Lambda only.
- Worker Versioning required — each Workflow needs `AutoUpgrade` or `Pinned` behavior.
- Activity limit: 15 minutes on Lambda (invocation limit minus shutdown deadline buffer).
- Shutdown deadline buffer default: WorkerStopTimeout + 2 seconds.
- Workflow duration has no limit — Workflows can span multiple invocations.
- Each invocation is independent — no connection reuse or shared state.
- Eager Activities always disabled.
- Self-hosted requires Temporal Server v1.31.0+.
- WCI Workflow ID pattern: `temporal-sys-worker-controller-instance:<deployment-name>:<build-id>`.
- Lambda execution role and Temporal invocation role are different — do not confuse them.
- OTel env var divergence: Python uses `OPENTELEMETRY_COLLECTOR_CONFIG_FILE`; Go and TypeScript use `_URI`.

## Intent Decision Table

Use this table to find the right reference file for the user's question.

| User intent | Reference file |
|---|---|
| What is a Serverless Worker? How does invocation work? What is the WCI? How does autoscaling work? What are the constraints? When should I use Serverless Workers vs. long-lived Workers? | `references/concepts.md` |
| How do I deploy a Serverless Worker on AWS Lambda? How do I write Worker code, package it, deploy it, set up IAM, create a Worker Deployment Version? | `references/aws-lambda-deployment.md` |
| What are the SDK-specific configuration options? What are the Lambda-tuned defaults? What package do I import? How do I configure versioning behavior? How does connection config work (TOML, env vars)? | `references/sdk-configuration.md` |
| How do I add OpenTelemetry observability? What ADOT layers do I need? What env var do I set for the collector config? How do I enable X-Ray tracing? | `references/observability.md` |
| My Lambda is not being invoked. My Workflows are not progressing. How do I diagnose a Serverless Worker issue? How do I inspect the WCI? | `references/troubleshooting.md` |

## Out of Scope

- **General SDK development patterns** (Workflows, Activities, Workers, signals, queries, Worker Versioning concepts): see `skill-temporal-developer`.
- **Traditional Worker tuning** (slot suppliers, tuners, poller autoscaling, resource-based tuning): see `skill-temporal-workertuning`.
- **Temporal Cloud administration** (Namespaces, users, certificates, billing): see `skill-temporal-ops`.
- **CLI command reference** (beyond the serverless-specific flags): see `skill-temporal-cli`.
