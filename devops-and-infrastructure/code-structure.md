# Multi-Cloud Video Pipeline — Code & File Structure

| Status | Version | Date |
|---|---:|---|
| Proposed | 1.0 | 14 August 2026 |

*This document covers how the codebase is organized — the monorepo layout, what each package/app is responsible for, and the `CloudDriver` contract that keeps the Fleet Manager provider-agnostic. For system behavior — the pipeline, sizing formulas, worker lifecycle, queues, and data model — see [`multi-cloud-video-processing-v1.md`](./multi-cloud-video-processing-v1.md).*

---

# 1. Overview

To launch workers across AWS, GCP, DigitalOcean, or bare-metal providers without rewriting core scaling logic, the Fleet Manager uses a **Ports & Adapters (Hexagonal)** architecture within a monorepo. The Fleet Manager's business logic — sizing, scheduling, and queue rules — depends only on a `CloudDriver` interface, never on a specific cloud SDK.

---

# 2. Monorepo Layout

```text
/video-pipeline-repo
├── /packages
│   ├── /fleet-types          # Shared interfaces (CloudDriver, Job) used by plugins and apps
│   ├── /fleet-plugin-aws     # AWS SDK implementation (@aws-sdk/client-ec2)
│   └── /fleet-plugin-gcp     # GCP Compute implementation (@google-cloud/compute)
│
└── /apps
    ├── /fleet-manager        # fleet-core business logic & entry points (serverless/serverful)
    └── /media-worker         # Standalone Node.js worker codebase
```

---

# 3. Package & App Responsibilities

## 3.1 packages/fleet-types

Shared TypeScript interfaces — `CloudDriver`, `Job`, and related types — consumed by every plugin and app. No implementation logic lives here; it exists so plugins and apps compile against the same contracts.

## 3.2 packages/fleet-plugin-aws

AWS implementation of `CloudDriver`, built on `@aws-sdk/client-ec2`. Launches and terminates EC2 instances, injecting the `cloud-init` user-data script on boot.

## 3.3 packages/fleet-plugin-gcp

GCP implementation of `CloudDriver`, built on `@google-cloud/compute`. Same contract as `fleet-plugin-aws`, different provider — the two are interchangeable from the Fleet Manager's point of view.

## 3.4 apps/fleet-manager

Hosts `fleet-core` (the shared business logic) plus the two entrypoint adapters that run it:

```text
fleet-core
├── sizing engine
├── chunk calculator
├── worker calculator
├── fleet reconciliation
├── worker lifecycle rules
└── job state rules

serverless entrypoint
└── AWS Lambda adapter

serverful entrypoint
└── Node.js daemon adapter
```

Both entrypoints call into the same `fleet-core`, so sizing and scheduling rules are never duplicated between the serverless (AWS Lambda) and serverful (Node.js daemon) deployment modes. See the pipeline doc's *Fleet Manager Deployment Modes* section for how each mode is invoked and what it monitors.

## 3.5 apps/media-worker

Standalone Node.js worker codebase that runs on each provisioned machine. Responsible for:

- downloading and analyzing the source video
- spawning FFmpeg (directly, no container layer) to split and encode
- streaming progress heartbeats (FPS, frames, ETA) back to the Fleet Manager
- uploading chunk/output artifacts to S3
- reporting `NO_WORK` to the Fleet Manager when the queue has nothing left — it never decides to terminate itself

See the pipeline doc's *Worker Completion, Termination, and Queue Check* section for the full lifecycle this app implements.

---

# 4. CloudDriver Interface

Every provider plugin implements the same interface, so the Fleet Manager's scheduler can launch or terminate a worker without knowing which cloud it's running on:

```text
CloudDriver
├── launchWorker(spec)    → provisions a machine, injects the cloud-init user script
└── terminateWorker(id)   → shuts the machine down
```

The Fleet Manager calls `CloudDriver.launchWorker()` whenever the scheduler determines new capacity is needed (see the pipeline doc's *Global Launch Formula*), and `CloudDriver.terminateWorker()` once a worker reports `NO_WORK` and the Fleet Manager confirms no queue work remains (see *Worker Completion, Termination, and Queue Check*).

---

# 5. Adding a New Cloud Provider

Adding a provider — or running AWS and GCP side by side — means adding a new package under `/packages` that implements `CloudDriver`. `fleet-core` and `apps/media-worker` do not change.

```text
1. Create /packages/fleet-plugin-<provider>
2. Implement launchWorker(spec) and terminateWorker(id)
3. Register the plugin with fleet-manager's CloudDriver Plugin Adapter
4. No changes required to sizing, scheduling, or queue logic
```

The default implementation described throughout the pipeline doc targets **AWS EC2 (`c5.2xlarge`)**; the same sizing, queueing, and lifecycle logic applies unchanged to any provider that implements `CloudDriver`.
