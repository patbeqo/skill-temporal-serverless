---
name: temporal-serverless
description: 'Deploy and operate Temporal Workers on serverless compute (currently AWS Lambda in Public Preview, and GCP Cloud Run in Pre-release) driven by the Worker Controller Instance (WCI). Use when the user mentions: "serverless worker", "Temporal serverless", "Worker Controller Instance", "WCI", "deploy Temporal worker on Lambda", "Lambda packaging", "Lambda timeout", "WCI inspection", "CloudFormation Temporal".'
version: 0.4.0
---

# Skill: temporal-serverless

## Overview

This skill helps users deploy and operate Temporal Workers on serverless compute. Instead of a long-lived process, Temporal invokes the Worker on demand through the Worker Controller Instance (WCI); the Worker processes available Tasks and shuts down, scaling to zero when idle. The skill produces Worker code, deployment configuration, connection configs, and packaging steps for the chosen SDK, and walks users through troubleshooting when serverless Workers aren't picking up Tasks.

AWS Lambda is the only compute provider this skill covers. (GCP Cloud Run exists but is still Pre-release and access-gated behind a manual request — if a user asks about it, say so and do not improvise a deployment from the Lambda material.) Provider-specific commands, templates, permissions, and defaults live in the reference files (see Routing) — this file stays at the workflow level so it can cover additional providers as they are added. When a step needs concrete commands, go to the reference file named at the end of that step.

Serverless Workers on **AWS Lambda are in Public Preview** as of July 30, 2026, and are **open to all Temporal Cloud customers** — no access request, no support ticket, no manual toggle. Do not tell a user to request access or confirm eligibility; they can select "AWS Lambda (Public Preview)" as the compute provider in the UI and deploy today. Public Preview is not GA: the APIs are still evolving and may change, so pin SDK and CLI versions for anything long-lived and read the installed package's actual API surface rather than writing from memory.

## Deployment workflow

Follow these steps in order. Each step is provider-neutral; the concrete commands, templates, and options live in the reference file named at the end of the step.

1. **Scope the task.** Identify the SDK language (Go, Python, or TypeScript), the deployment target (Temporal Cloud or self-hosted — self-hosted has its own server prerequisites), the compute provider, and whether this is a new setup, a configuration change, or troubleshooting. Confirm the deployment target is compatible with the chosen provider — for a Temporal Cloud Namespace, its cloud provider and region must match the compute provider, or the deployment fails only later at connection time. Ensure a Temporal client/CLI is available and authenticated to the target. Each changes the specifics. → `references/concepts.md` for what the user is building; `references/aws-lambda/setup.md` for the compatibility and client-setup details.

   **Agree a resource-naming prefix in this same batch of questions, and propose a default so the user can accept without thinking about it.** Assume the account and the Namespace are shared — unprefixed names like `temporal-serverless-worker` collide with, or quietly shadow, another team's deployment. Naming is not a late cosmetic choice you can patch on the provider side: the deployment name, build ID, and Task Queue are compiled into the Worker binary, so changing them after step 3 means editing code, rebuilding, repackaging, and cleaning up whatever was already created under the old names. Once agreed, apply the prefix to everything you create on both sides — compute unit, roles, infrastructure stacks, log groups, deployment name, and Task Queue.

2. **Confirm you can make the required changes — before making any.** Determine which credentials are available (for the compute provider and for Temporal) and confirm the active identity actually has permission to make the changes the task needs — creating or updating compute resources, creating roles, registering deployment versions. Verify *both* sides: the compute provider AND Temporal access. Do not run account-mutating commands and let them fail partway. **If access is missing or unconfirmed, stop and ask the user how they want to proceed** — extend their identity's permissions, have an administrator make the change and hand back the result, or generate the commands for the user to run under a privileged identity. Changing a user's cloud account is consequential; confirm authorization and the preferred method first. → `references/aws-lambda/iam.md` (exact permissions, AWS preflight) and `references/aws-lambda/setup.md` (Temporal connection preflight).

   Adapt to what is available — the skill is valuable at every level:

   | Compute-provider access | Temporal access | Behavior |
   |---|---|---|
   | Authenticated | Authenticated | Full workflow — run commands, verify results, register deployment versions. |
   | Authenticated | None | Deploy compute infrastructure; generate the Temporal commands for the user to run. |
   | None | Authenticated | Write Worker code and configs; generate compute/deploy commands for the user; run Temporal commands and verify WCI state. |
   | None | None | Write Worker code, deploy templates, permission policies, connection configs, packaging scripts; provide all commands with placeholder values. |

3. **Author the Worker.** *Install the SDK's serverless Worker package before writing any code* — it is often shipped separately from the main SDK, with its own version line, so having the base SDK installed does not mean it is importable. Then read the installed package's actual API surface and write against that; these are Public Preview APIs that drift between versions, and generating code from memory costs a build cycle. Every Workflow must declare a versioning behavior (`Pinned` or `AutoUpgrade`), per-Workflow or as a Worker-level default — code without it fails at runtime. → `references/sdk-configuration.md` (package, install, entry point, tuned defaults) and `references/aws-lambda/setup.md` (install commands, API-inspection recipes, handler shape).

4. **Package and deploy the compute unit.** Build and package per SDK, deploy the compute unit, and set the invocation deadline high enough for the Worker to start, connect, register the Task Queue, and shut down gracefully. Match the build's target architecture to the deployed compute unit's — a mismatch fails only at invocation time, not at build time. After a create or update, wait for the compute unit to reach a ready state before the next step; providers return from these calls while the unit is still settling. → `references/aws-lambda/setup.md`.

5. **Grant Temporal permission to invoke the Worker.** Configure the compute provider's access so Temporal can invoke and inspect the Worker. This access is separate from the compute unit's own execution role — do not confuse the two. Two things to get right before you create anything: (a) this grant is **shared, account-wide infrastructure** that a previous deployment may already have created — look for an existing one and extend it to cover your new Worker rather than creating a parallel copy, and never delete or repurpose one you did not create without asking; (b) scope the grant so that *future* immutable builds are covered, not just today's — a grant pinned to one build breaks the next release in a way that surfaces later as an unrelated-looking invocation failure. → `references/aws-lambda/iam.md`.

6. **Register the Worker Deployment Version, verify the validation invocation, then set it current.** Create the Worker Deployment Version with the compute provider configured; the deployment name and build ID must exactly match the values in the Worker code. Creating it triggers one validation invocation — **check that it bound the Task Queue before going further.** If the Task Queue is bound, the permission grant, package, config, and deadline are all provably correct, and any later failure is downstream; if it is not, setting the version current will not fix it. Then set it current: through the UI this happens automatically, through the CLI it is a separate step, without which Tasks never route to the version. → `references/aws-lambda/setup.md`.

7. **Verify.** Start a Workflow on the Task Queue and confirm Temporal invokes the Worker — check the Workflow history in the Temporal UI and the compute provider's logs. If it does not progress, → `references/aws-lambda/diagnostics.md`.

8. **Hand back an inventory.** Report the resources you created — compute unit and published build identifiers, roles, infrastructure stacks, region, deployment name and build ID — together with the teardown commands. These are live, billable resources whose names are only knowable from the run that created them, and reconstructing them later means scanning the user's account. → `references/aws-lambda/setup.md` (Teardown).

## Working practices

How to move through the workflow above. These are drawn from real deployments, where the failures were rarely conceptual — they came from acting on a recalled detail, or from chaining a step onto an unverified one.

- **Read the current state instead of recalling it.** Check the installed package's API, the CLI's own `--help` for the flags you are about to pass, the compute unit's reported state, and the CLI version. Every one of these has drifted or surprised in practice: a Public Preview SDK whose fields moved, a CLI too old to have the serverless subcommand at all, a resource that reports success while still settling. Reading costs one command; guessing costs a deploy cycle and can leave half-built resources behind.
- **Verify each step before building the next on top of it.** Compile the Worker before packaging it, confirm the package's target architecture before uploading, wait for the compute unit to be ready before publishing a build, and confirm the Task Queue is bound before shifting traffic. Deployment failures here surface far from their cause — an architecture or dependency mismatch appears only at first invocation, and a first-invocation failure appears as "the Worker is never invoked", several steps later.
- **When something fails, read the actual error before changing anything.** Fetch the failure reason from the provider (deployment events, logs, status fields) and fix that. Do not retry the same command with variations, and do not start editing permissions or trust policies on the theory that the problem might be access — most first-invocation failures are not permission problems, and some failures are on Temporal's side and will reproduce no matter what you change.
- **Treat the user's account as shared and pre-existing.** Assume other deployments, roles, and stacks are already there. Look before creating, extend rather than duplicate, and never delete or repurpose something you did not create without asking. When you do work around existing infrastructure — a different name, a reused role — say so explicitly in your summary rather than leaving it as a silent deviation.
- **Confirm the end state from two independent signals.** A Workflow that completes in the Temporal UI *and* the Worker's own logs showing startup, Task Queue registration, and Task execution. One signal alone can mislead: a system Workflow that exists and is running proves nothing about invocation health, and a command that exits zero may have done nothing at all if it was waiting on a confirmation prompt.
- **Account for what you created.** Keep the inventory as you go rather than reconstructing it at the end, say plainly that the resources are live and billable, and offer to tear them down (step 8).

## Never create or manage the WCI

Temporal creates the WCI automatically once a Worker Deployment Version has a compute provider. You never create, start, or manage it. A WCI that exists or is running is *not* evidence that invocation works — it continue-as-news and keeps running even while its Activities fail. Diagnose from Temporal's own signals: read the WCI Workflow history and look for Activity failures. Do not enumerate compute resources across regions or scan the account to reverse-engineer state. → `references/concepts.md`, `references/aws-lambda/diagnostics.md`.

## Provider-neutral principles

Surface these early — they apply regardless of compute provider:

- **Versioning behavior is mandatory.** Every Workflow needs `Pinned` or `AutoUpgrade`, or the Worker sets a default.
- **Deployment name and build ID must match exactly** between the Worker code and the Worker Deployment Version. A mismatch causes an invocation loop (Temporal invokes → Worker polls with the wrong version → Task not processed → invoke again). Signature: rapid repeated invocations with no Workflow progress.
- **Set the invocation deadline high enough.** Providers often default to a very short timeout. If the first invocation times out before the Worker registers the Task Queue, the binding is never created and the Worker is never invoked again. → `references/aws-lambda/setup.md` for the exact default.
- **Use an immutable, versioned build per Build ID in production.** Pointing the provider at a mutable "latest" target lets code change under in-flight Workflows and cause non-determinism errors, even for Pinned Workflows. Keep a 1-to-1 mapping between each Build ID and one immutable build. → `references/aws-lambda/versioning.md`.
- **Tune the timeout triple together for long-running Activities:** (1) worker stop timeout > longest Activity runtime, (2) shutdown deadline buffer > worker stop timeout + shutdown hook time, (3) invocation deadline > longest Activity runtime + shutdown deadline buffer. Raising one alone does not help. If the longest Activity exceeds half the maximum invocation deadline, recommend Activity Heartbeats. → `references/concepts.md`, `references/sdk-configuration.md`.
- **Eager Activities are always disabled** — serverless invocations don't maintain persistent connections. Don't suggest them as an optimization.
- **Activities are bounded by the invocation limit** (minus the shutdown deadline buffer); Workflow duration is unbounded and can span many invocations. Flag Activities that approach the provider's limit early. → `references/concepts.md`.
- **Mixed serverless + long-lived Workers on one Task Queue:** do not enable dynamic scaling on the long-lived Workers — the two groups can't coordinate scaling and will cause unnecessary invocations.
- **Secrets belong in a secret store**, not plaintext environment variables. Provider docs and quickstarts commonly pass the API key or TLS key as a plaintext environment variable; that is acceptable in a throwaway development walkthrough *only if you say so explicitly at the time*. Anything the user describes as production, shared, or long-lived gets the secret store, loaded at cold start. Either way, keep key material out of shell history and command echoes.
- **The CLI prompts for confirmation when shifting traffic.** Setting the current or ramping version asks interactively; run non-interactively without the confirmation flag, the command exits having done nothing, which reads as success. Pass the confirmation flag in scripts, CI, and agent shells, and confirm the resulting state rather than trusting the exit code. → `references/aws-lambda/setup.md`.

## Troubleshooting

Start by determining whether the Worker is being invoked at all. Then, in priority order: (1) **Validate Connection** in the Temporal UI (Workers > Deployments > select > Actions > Validate Connection) — checks credentials, role assumption, and reachability in one step; (2) check whether the version's **Task Queue is bound** — if it is, invocation and Worker startup provably work and the fault is downstream, which rules out most of the surface in one command; (3) confirm the version is **current** (CLI-created versions are not automatic, and a confirmation-prompted command may have silently done nothing); (4) check the compute provider's logs for connection, auth, or TLS errors; (5) if rapid repeated invocations show no progress, check the deployment name/build ID match. Distinguish a Temporal-side failure (reproduces no matter what you change on the provider side) from a genuine user-permission problem before editing anything. → `references/aws-lambda/diagnostics.md`, `references/concepts.md`.

## Common Pitfalls

High-impact mistakes — warn the user proactively. Each is a symptom → cause → fix.

1. **Deployment name / build ID mismatch → invocation loop.** *Symptom:* rapid, repeated invocations with no Workflow progress. *Cause:* the name or build ID in the Worker code doesn't match the Worker Deployment Version, so the Worker polls with the wrong version, the Task isn't processed, and Temporal invokes again. *Fix:* make the values in code exactly match the version configuration.
2. **Version not set as current.** A version created through the CLI is not automatically current; without it, Tasks don't route to the version and the Worker is never invoked. *Fix:* set it current as a separate step (the UI does this automatically).
3. **Failed first invocation.** When a version is created, the WCI invokes the Worker once to validate. If that invocation fails — missing env vars, bad TLS/auth config, missing dependencies, or an invocation deadline too short for the Worker to start and register the Task Queue — the Worker never connects, never polls, the binding is never created, and the Worker is never automatically invoked again. *Fix:* diagnose by manually invoking the compute unit, and confirm the invocation deadline is set high.
4. **Confusing the two roles.** The compute unit's execution role (grants the function permission to run) is separate from the access Temporal uses to invoke it. Never describe one as the other. → `references/aws-lambda/iam.md`.
5. **Timeout tuning mismatch.** Raising only the shutdown deadline buffer makes the Worker stop polling earlier but gives in-flight Activities no more time; raising only the worker stop timeout doesn't make it stop polling earlier, so the provider may terminate the Worker first. *Fix:* tune the three values together (see the timeout triple above).
6. **Mutable "latest" build reference in production.** Pointing the provider at a mutable/unqualified target means the code changes on every redeploy; deploying replay-unsafe code then causes non-determinism errors for in-flight Workflows, even Pinned ones. *Fix:* publish an immutable versioned build and keep a 1-to-1 mapping between each Build ID and one build. → `references/aws-lambda/versioning.md`.
7. **Re-creating shared permission infrastructure that already exists.** *Symptom:* the infrastructure deployment fails outright and rolls back, or it succeeds and leaves a second, redundant grant behind. *Cause:* the permission grant Temporal assumes is account-wide with a fixed default name, so a previous serverless deployment already owns it. *Fix:* check whether it exists and what owns it *before* creating; extend the existing one to cover the new Worker, and fall back to a distinctly named parallel one only when the existing infrastructure is not yours to change — saying why when you do. A failed-and-rolled-back deployment must be deleted before the name can be reused; a successful one is live infrastructure and must not be. → `references/aws-lambda/iam.md`.
8. **Invoke permission scoped to a single build.** *Symptom:* the deployment works, then the *next* release cannot be invoked, with an error that looks like a connection or configuration problem rather than a permissions one. *Cause:* the grant named one immutable build, and the new release is a different resource. *Fix:* scope the grant to cover the base resource and all its published builds. → `references/aws-lambda/iam.md`.

## Routing to reference files

Most questions need 2–3 reference files.

| User intent | Reference file(s) |
|---|---|
| What is a Serverless Worker / the WCI? How do invocation and autoscaling work? What are the constraints? Serverless vs long-lived Workers? | `references/concepts.md` |
| Deploy a Serverless Worker (happy path): write code, package, deploy, register + set-current version, verify, tear down. | `references/aws-lambda/setup.md` (+ `references/concepts.md`) |
| Operator permissions and preflight; execution role vs Temporal invocation role; CloudFormation (Cloud + self-hosted). | `references/aws-lambda/iam.md` |
| Update or redeploy; version the build, use a qualified ARN, roll back. | `references/aws-lambda/versioning.md` (+ `references/concepts.md`) |
| Self-hosted server enablement (dynamic config, WCI, server AWS credentials). | `references/aws-lambda/self-hosted.md` (+ `references/aws-lambda/iam.md`) |
| SDK-specific options and tuned defaults, which package to install and how it is distributed, imports, versioning-behavior configuration, connection config (TOML, env vars). Reduce cold start / pre-bundle Workflow code. | `references/sdk-configuration.md` |
| Add OpenTelemetry observability, collector config, tracing. | `references/aws-lambda/observability.md` |
| Worker not invoked, Workflows not progressing, inspect the WCI. | `references/aws-lambda/diagnostics.md` (+ `references/concepts.md`) |
| Long-running Activities and timeout relationships. Isolate Activities from resource exhaustion. | `references/concepts.md` (+ `references/sdk-configuration.md`) |

## Out of Scope

- **General SDK development patterns** (Workflows, Activities, signals, queries, Worker Versioning concepts): see `skill-temporal-developer`.
- **Traditional Worker tuning** (slot suppliers, tuners, poller autoscaling, resource-based tuning): see `skill-temporal-workertuning`.
- **Temporal Cloud administration** (Namespaces, users, certificates, billing): see `skill-temporal-ops`.
- **CLI command reference** (beyond the serverless-specific flags): see `skill-temporal-cli`.
