---
name: temporal-serverless
description: 'Deploy and operate Temporal Workers on serverless compute (AWS Lambda) using Worker Cloud Invocations (WCI). Covers concepts, deployment, SDK configuration, observability, and troubleshooting for serverless Temporal Workers. Use when the user mentions: "serverless worker", "Temporal serverless", "Worker Cloud Invocation", "WCI", "deploy Temporal worker on Lambda", "Lambda packaging", "serverless compute", "Lambda timeout", "ADOT layer", "ADOT", "X-Ray tracing", "OpenTelemetry Lambda", "Lambda not invoked", "workflows not progressing", "WCI inspection", "Lambda-tuned defaults", "serverless worker config", "CloudFormation Temporal", "IAM Temporal Lambda", "Worker Deployment Version Lambda", "TOML config serverless", "serverless Temporal worker Go", "serverless Temporal worker Python", "serverless Temporal worker TypeScript", "15-minute Activity limit", "Eager Activities not supported", "Pre-release serverless". Does NOT cover general SDK development or Workflow/Activity patterns (temporal-developer), traditional Worker tuning such as slot suppliers or poller autoscaling (temporal-workertuning), Temporal Cloud administration like namespaces or billing (temporal-ops), or CLI command reference beyond serverless-specific flags (temporal-cli).'
version: 0.1.0
---

# Skill: temporal-serverless

## Overview

This skill covers deploying and operating Temporal Workers on serverless compute (currently AWS Lambda). It spans concepts (WCI, lifecycle, autoscaling, failure handling), deployment (IAM, CloudFormation, CLI commands, Lambda packaging), per-SDK configuration (Go, Python, TypeScript), observability (OpenTelemetry/ADOT integration), and troubleshooting. Serverless Workers are in Pre-release, require Worker Versioning, and are supported for Go, Python, and TypeScript SDKs on AWS Lambda.

## Intent Decision Table

Use this table to find the right reference file for the user's question.

| User intent | Reference file |
|---|---|
| What is a Serverless Worker? How does invocation work? What is the WCI? How does autoscaling work? What are the constraints? When should I use Serverless Workers vs. long-lived Workers? | `references/concepts.md` |
| How do I deploy a Serverless Worker on AWS Lambda? How do I write Worker code, package it, deploy it, set up IAM, create a Worker Deployment Version? | `references/aws-lambda-deployment.md` |
| What are the SDK-specific configuration options? What are the Lambda-tuned defaults? What package do I import? How do I configure versioning behavior? How does connection config work (TOML, env vars)? | `references/sdk-configuration.md` |
| How do I add OpenTelemetry observability? What ADOT layers do I need? What env var do I set for the collector config? How do I enable X-Ray tracing? | `references/observability.md` |
| My Lambda is not being invoked. My Workflows are not progressing. How do I diagnose a Serverless Worker issue? How do I inspect the WCI? | `references/troubleshooting.md` |

## Key constraints

- Serverless Workers are in Pre-release and available to select Temporal Cloud customers.
- Worker Versioning is required. Each Workflow must have an `AutoUpgrade` or `Pinned` versioning behavior.
- Activity duration must complete within the compute provider's invocation limit (minus shutdown deadline buffer). For AWS Lambda, the maximum is 15 minutes.
- Workflow duration has no limit. Workflows can span multiple invocations.
- Each invocation is independent. No connection reuse or shared state across invocations.
- Eager Activities are not supported (always disabled).
- Self-hosted Temporal Service requires v1.31.0 or later.

## Out of scope (handled by sibling skills)

- **General SDK development patterns** (Workflows, Activities, Workers, signals, queries, Worker Versioning concepts): see `skill-temporal-developer`.
- **Traditional Worker tuning** (slot suppliers, tuners, poller autoscaling, resource-based tuning): see `skill-temporal-workertuning`.
- **Temporal Cloud administration** (Namespaces, users, certificates, billing): see `skill-temporal-ops`.
- **CLI command reference** (beyond the serverless-specific flags): see `skill-temporal-cli`.
