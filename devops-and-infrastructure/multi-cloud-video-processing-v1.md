# Multi-Cloud Video Processing Pipeline

| Status | Version | Date |
|---|---:|---|
| Proposed | 1.0 | 15 August 2026 |

*This document covers system behavior — the pipeline, sizing formulas, worker lifecycle, queues, and data model. For the repo layout, package responsibilities, and the `CloudDriver` contract, see [`code-structure.md`](./code-structure.md).*

# 1. Overview

This document defines the video-processing pipeline for an on-demand, cost-optimized, reusable EC2 worker fleet.

The Fleet Manager can run in two deployment modes:

- **Serverless Fleet Manager** — AWS Lambda-based orchestration.
- **Serverful Fleet Manager** — a continuously running Node.js service/daemon that performs the same orchestration logic.

Both modes use the same Fleet Manager core, sizing formulas, worker-pool rules, PostgreSQL state, and pg-boss queue. Only the execution model of the control-plane scheduler changes.

The pipeline has four main ideas:

1. The user can request any supported output quality from **144p to 4320p**.
2. A worker first analyzes the source video and creates **no-reencode chunks**. The first worker then immediately becomes a normal encoding worker.
3. One encoding job represents **one source chunk**, and that worker generates **all requested output qualities** for that chunk.
4. EC2 instances are a **shared reusable worker pool**. Workers are not permanently assigned to one video. After completing a chunk, a worker immediately takes another pending chunk from any video.

The system uses:

- **AWS Lambda** for fleet reconciliation/orchestration.
- **EC2 c5.2xlarge** as the default worker type.
- **CloudDriver plugin architecture** (Ports & Adapters) so the same Fleet Manager core can launch/terminate workers on AWS, GCP, or other providers — see `code-structure.md`.
- **PostgreSQL + pg-boss** for the encoding queue.
- **S3** for source chunks and final media.
- **Native Node.js** for worker orchestration.
- **Native FFmpeg** spawned directly by Node.js.
- **No Docker/container layer** on the media workers.

---

# 2. Supported Output Qualities

The pipeline supports the following target heights:

| Quality | Height |
|---|---:|
| 144p | 144 |
| 240p | 240 |
| 360p | 360 |
| 480p | 480 |
| 540p | 540 |
| 720p | 720 |
| 900p | 900 |
| 1080p | 1080 |
| 1440p | 1440 |
| 2160p | 2160 |
| 4320p | 4320 |

A job can request one quality or multiple qualities.

Examples:

```text
240p
```

or:

```text
240p + 360p + 480p + 720p + 1080p
```

The worker assigned to a chunk generates every requested rendition for that chunk.

For example:

```text
chunk-001
├── 240p
├── 360p
├── 480p
├── 720p
└── 1080p
```

A separate worker is **not** created for each quality.

---

# 3. High-Level Architecture

```text
                         ┌──────────────────────┐
                         │       API / Client   │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────────────┐
                         │        Fleet Manager          │
                         │                              │
                         │ Serverless: AWS Lambda       │
                         │ Serverful: Node.js daemon    │
                         └──────────────┬───────────────┘
                                    │
                                    ▼
                    ┌────────────────────────────────┐
                    │       Shared EC2 Worker Pool   │
                    │                                │
                    │  c5.2xlarge (default)          │
                    │                                │
                    │  Worker 1   Worker 2   Worker 3│
                    │  Worker 4   Worker 5   ...     │
                    └───────────────┬────────────────┘
                                    │
                                    ▼
                             ┌────────────┐
                             │ PostgreSQL │
                             │  pg-boss   │
                             └─────┬──────┘
                                   │
                     ┌─────────────┼─────────────┐
                     │             │             │
                     ▼             ▼             ▼
                  Video A       Video B       Video C
                  Chunk jobs    Chunk jobs    Chunk jobs
                     │             │             │
                     └─────────────┼─────────────┘
                                   │
                                   ▼
                                FFmpeg
                                   │
                                   ▼
                              HLS / DASH
                                   │
                                   ▼
                                  S3
```

---

# 4. Fleet Manager Deployment Modes

The Fleet Manager is the control plane responsible for:

- creating and tracking video jobs
- calculating chunk duration
- calculating required workers
- checking existing EC2 capacity
- launching EC2 instances when capacity is missing
- reusing already-running workers
- monitoring worker heartbeats
- detecting completed jobs
- terminating workers when no useful queue work remains
- coordinating final manifest generation

## 4.1 Serverless Fleet Manager

In serverless mode, the Fleet Manager runs as AWS Lambda.

Lambda is invoked when an event requires orchestration, for example:

```text
New video job
Worker registration
Chunk completion
Worker heartbeat
Job completion
Scheduled reconciliation
```

Lambda performs the required reconciliation and exits.

It does not need to remain running continuously.

Conceptually:

```text
Event
  ↓
AWS Lambda
  ↓
Reconcile fleet
  ↓
Launch/reuse/terminate workers
  ↓
Exit
```

## 4.2 Serverful Fleet Manager

In serverful mode, the same Fleet Manager core runs as a continuously running Node.js service.

```text
Serverful Fleet Manager
        ↓
PostgreSQL + pg-boss
        ↓
Worker fleet
```

The serverful process can continuously monitor:

- pg-boss pending jobs
- active workers
- worker heartbeats
- job progress
- worker idle time
- video completion
- fleet capacity

The business logic should remain independent of whether it is running inside Lambda or a persistent server.

## 4.3 Shared Fleet Manager Core

Both deployment modes run the same `fleet-core` business logic (sizing engine, chunk calculator, worker calculator, fleet reconciliation, worker lifecycle rules, job state rules) behind a thin entrypoint adapter — an AWS Lambda handler for serverless, a Node.js daemon loop for serverful:

```text
Serverless
AWS Lambda
    ↓
fleet-core

Serverful
Node.js server
    ↓
fleet-core
```

The sizing and scheduling rules are therefore never duplicated between the two modes. See `code-structure.md` for the package/folder breakdown of `fleet-core` and the two entrypoints.

---

# 5. Core Pipeline

The complete lifecycle of a video is:

```text
1. API creates video-processing job
        ↓
2. Fleet Manager receives/reconciles the job
   (Serverless Lambda or Serverful service)
        ↓
3. Fleet Manager provisions or reuses an EC2 worker
        ↓
4. Worker downloads/analyzes the source video
        ↓
5. Worker calculates dynamic chunk duration
        ↓
6. Worker splits the source WITHOUT encoding
        ↓
7. Worker uploads chunks to S3
        ↓
8. Worker creates pg-boss jobs for the chunks
        ↓
9. The same worker becomes an encoding worker
        ↓
10. All workers consume chunk jobs
        ↓
11. One worker processes one chunk at a time
        ↓
12. Worker generates ALL requested qualities for that chunk
        ↓
13. Worker uploads HLS/DASH output to S3
        ↓
14. Worker immediately takes the next pending chunk
        ↓
15. Next chunk may belong to the same or another video
        ↓
16. Finalizer creates the final manifests
        ↓
17. Video becomes COMPLETE
        ↓
18. Worker checks pg-boss for another pending task
        ↓
19. If another task exists, worker processes it
        ↓
20. If no task exists, worker is marked IDLE
        ↓
21. Fleet Manager evaluates whether the idle worker is still needed
        ↓
22. Unneeded idle workers are terminated
        ↓
23. Remaining workers stay reusable
```

---

# 6. Important Worker Rule

The first EC2 is **not a dedicated splitter machine**.

It performs:

```text
Analyze
   ↓
Split
   ↓
Upload chunks
   ↓
Create queue jobs
   ↓
Become normal encoding worker
```

Therefore:

```text
EC2 #1
```

does not terminate after splitting.

It immediately processes a chunk.

Example:

```text
Video A
60 minutes
chunk duration = 20 minutes

chunks:
1. 0–20
2. 20–40
3. 40–60
```

The first worker can do:

```text
EC2 #1
├── split video
├── upload chunks
├── enqueue chunk jobs
└── process chunk-001
```

Other workers process:

```text
EC2 #2 → chunk-002
EC2 #3 → chunk-003
```

---

# 7. Formula A — Dynamic No-Reencode Chunk Duration

The first sizing formula answers:

> How many minutes should each no-reencode chunk contain?

The chunk duration must become smaller when the requested encoding workload becomes larger.

It must become larger when the requested encoding workload becomes smaller.

## 7.1 Quality Weight

Use a configurable quality-weight table.

Recommended initial values:

| Quality | Weight |
|---|---:|
| 144p | 0.6 |
| 240p | 0.8 |
| 360p | 1.0 |
| 480p | 1.2 |
| 540p | 1.4 |
| 720p | 2.0 |
| 900p | 2.4 |
| 1080p | 3.0 |
| 1440p | 4.0 |
| 2160p | 6.0 |
| 4320p | 10.0 |

These are **scheduler weights**, not bitrate specifications. They should be calibrated using real encoding benchmarks.

For requested qualities:

```text
QualityComplexity = Σ QualityWeight(q)
```

Example:

```text
240p + 360p + 480p + 720p

= 0.8 + 1.0 + 1.2 + 2.0

= 5.0
```

---

# 8. Source Complexity

The same target quality can require different amounts of CPU depending on the source.

Define:

```text
SourceComplexity =
    ResolutionFactor
    × FPSFactor
    × CodecFactor
```

A simple initial model can be:

```text
ResolutionFactor =
    SourcePixels / BaselinePixels
```

```text
FPSFactor =
    SourceFPS / BaselineFPS
```

Then apply codec-specific multipliers.

Example:

```text
BaselinePixels = 1920 × 1080
BaselineFPS = 30
```

The actual factors and codec weights should be calibrated using benchmark data.

The important design principle is:

```text
higher source complexity
        ↓
higher encoding workload
        ↓
smaller chunks
```

---

# 9. Encoding Workload

Define:

```text
EncodingWorkload =
    VideoDuration
    × QualityComplexity
    × SourceComplexity
```

For chunk sizing, calculate the estimated work per minute:

```text
WorkPerMinute =
    QualityComplexity
    × SourceComplexity
```

---

# 10. Target Chunk Work Budget

Define a configurable target:

```text
TARGET_CHUNK_WORK
```

Then:

```text
RawChunkDuration =
    TARGET_CHUNK_WORK
    /
    WorkPerMinute
```

Clamp the result:

```text
ChunkDuration =
    clamp(
        RawChunkDuration,
        MIN_CHUNK_DURATION,
        MAX_CHUNK_DURATION
    )
```

Recommended starting configuration:

```text
MIN_CHUNK_DURATION = 5 minutes
MAX_CHUNK_DURATION = 30 minutes
```

These values are configuration, not hard requirements.

They should be tuned using actual FFmpeg benchmarks.

---

# 11. Chunk Duration Behavior

The formula must behave like this:

```text
More requested qualities
        ↓
Higher QualityComplexity
        ↓
Higher WorkPerMinute
        ↓
Smaller ChunkDuration
        ↓
More chunks
        ↓
More possible parallelism
```

And:

```text
Fewer requested qualities
        ↓
Lower QualityComplexity
        ↓
Lower WorkPerMinute
        ↓
Larger ChunkDuration
        ↓
Fewer chunks
        ↓
Less required parallelism
```

This is the main relationship between quality and chunk size.

---

# 12. Example — 60 Minute Video at 240p

Suppose the calibrated sizing engine determines:

```text
ChunkDuration = 20 minutes
```

Then:

```text
VideoDuration = 60 minutes

ChunkCount =
    ceil(60 / 20)

    = 3
```

The chunks are:

```text
chunk-001 = 0–20 minutes
chunk-002 = 20–40 minutes
chunk-003 = 40–60 minutes
```

Therefore the maximum number of workers for this video is:

```text
3
```

Even if 20 EC2 machines are available globally, this video must not consume more than 3 workers.

---

# 13. No-Reencode Splitting

The preparation split must use FFmpeg stream copy whenever the source/container allows it.

Conceptually:

```bash
ffmpeg -i input.mp4 \
  -ss 00:00:00 \
  -t 00:20:00 \
  -map 0 \
  -c copy \
  chunk-001.mp4
```

`-c copy` performs stream copy rather than decoding and re-encoding, so this stage does not perform the expensive quality conversion. FFmpeg documents streamcopy as copying packets without decoding or encoding. citeturn0search1

The exact splitting implementation must account for keyframes. FFmpeg's segment documentation notes that segment boundaries depend on keyframes and that accurate arbitrary split points may require suitable input keyframes. citeturn0search0

Therefore the splitter should prefer keyframe-safe boundaries.

---

# 14. Chunk Metadata

Every chunk should have metadata similar to:

```json
{
  "chunkId": "chunk-001",
  "videoId": "video-123",
  "chunkIndex": 0,
  "startSeconds": 0,
  "durationSeconds": 1200,
  "sourceKey": "jobs/video-123/chunks/chunk-001.mp4",
  "status": "PENDING"
}
```

For a 60-minute video split into 20-minute chunks:

```text
chunk-001
start = 0
duration = 1200

chunk-002
start = 1200
duration = 1200

chunk-003
start = 2400
duration = 1200
```

---

# 15. pg-boss Queue

After the no-reencode chunks are uploaded to S3, the worker creates one pg-boss job per chunk.

Example:

```text
video-123
│
├── encode-video-chunk-001
├── encode-video-chunk-002
└── encode-video-chunk-003
```

The queue payload should contain:

```json
{
  "videoId": "video-123",
  "chunkId": "chunk-001",
  "chunkIndex": 0,
  "sourceKey": "jobs/video-123/chunks/chunk-001.mp4",
  "startSeconds": 0,
  "durationSeconds": 1200,
  "qualities": [240, 360, 480, 720, 1080]
}
```

---

# 16. One Worker = One Chunk at a Time

A worker must not take multiple encoding chunks simultaneously.

The rule is:

```text
ONE WORKER
    ↓
ONE ACTIVE CHUNK
    ↓
ALL REQUESTED QUALITIES
```

For example:

```text
EC2 #1
    │
    └── chunk-001
          ├── 240p
          ├── 360p
          ├── 480p
          ├── 720p
          └── 1080p
```

After completion:

```text
EC2 #1
    ↓
ACK chunk-001
    ↓
Take next pg-boss job
```

The next job can belong to:

```text
Video A
```

or:

```text
Video B
```

or:

```text
Video C
```

The worker is not permanently associated with a video.

---

# 17. Formula B — Maximum Workers for One Video

The second formula answers:

> How many EC2 workers can this video use?

The fundamental rule is:

```text
MaxWorkersForVideo = ChunkCount
```

Therefore:

```text
WorkersAssignedToVideo <= ChunkCount
```

If:

```text
ChunkCount = 3
```

then:

```text
Maximum workers = 3
```

If:

```text
ChunkCount = 5
```

and:

```text
MAX_WORKERS_PER_VIDEO = 4
```

then:

```text
Maximum workers = min(5, 4)

                 = 4
```

So:

```text
MaxWorkersForVideo =
    min(
        ChunkCount,
        MAX_WORKERS_PER_VIDEO
    )
```

Recommended initial value:

```text
MAX_WORKERS_PER_VIDEO = 4
```

This is a safety limit.

---

# 18. Why Chunk Count Limits Workers

Suppose:

```text
60-minute video
20-minute chunks
```

There are only:

```text
3 chunks
```

Therefore:

```text
chunk-001
chunk-002
chunk-003
```

There is no useful fourth encoding job for that video.

Running:

```text
EC2 #4 → Video A
```

would create no additional parallel work.

Therefore:

```text
MaxWorkersForVideo = 3
```

---

# 19. Reusable EC2 Worker Pool

EC2 instances belong to the global fleet, not to individual videos.

Example:

```text
EC2 #1
EC2 #2
EC2 #3
EC2 #4
EC2 #5
```

These machines can process:

```text
Video A
Video B
Video C
Video D
```

over their lifetime.

After a worker finishes a chunk:

```text
Worker
   ↓
ACK current job
   ↓
pg-boss
   ↓
Take next pending job
```

It does not wait for the same video.

---

# 20. Reuse Existing Workers Before Launching EC2

This is a critical fleet rule.

EC2 startup takes time.

Therefore, when a new video arrives, the scheduler must first look at existing workers.

Worker states:

```text
IDLE
BUSY
NEAR_COMPLETE
STARTING
STOPPING
FAILED
```

Define:

```text
REUSE_THRESHOLD = 85%
```

A busy worker can be considered **near-complete** when:

```text
progress >= 85%
```

The scheduler can reserve that worker for upcoming work.

It should not start the next job until its current job has actually completed.

---

# 21. Reusable Capacity

Define:

```text
ReusableWorkers =
    IdleWorkers
    +
    NearCompleteWorkers
```

Then:

```text
RequiredWorkersForVideo =
    min(
        ChunkCount,
        MAX_WORKERS_PER_VIDEO
    )
```

And:

```text
WorkersToLaunch =
    max(
        0,
        RequiredWorkersForVideo
        - ReusableWorkers
    )
```

This is the primary worker-launch formula.

---

# 22. Example — Existing Workers at 85%+

Suppose:

```text
New Video
60 minutes

Chunk duration = 20 minutes

Chunk count = 3
```

Therefore:

```text
RequiredWorkers = 3
```

Existing fleet:

```text
EC2 #1 → 90%
EC2 #2 → 88%
EC2 #3 → 86%
EC2 #4 → 40%
```

Near-complete workers:

```text
#1
#2
#3
```

Therefore:

```text
ReusableWorkers = 3
```

Then:

```text
WorkersToLaunch =
    max(0, 3 - 3)

    = 0
```

No new EC2 instances are launched.

The three existing workers are reused when they finish their current jobs.

---

# 23. Example — Launch Only What Is Missing

Suppose the same video requires:

```text
3 workers
```

Existing fleet:

```text
EC2 #1 → 92%
EC2 #2 → 89%
EC2 #3 → 40%
EC2 #4 → 30%
```

Near-complete workers:

```text
#1
#2
```

Therefore:

```text
ReusableWorkers = 2
```

Required:

```text
3
```

So:

```text
WorkersToLaunch =
    max(0, 3 - 2)

    = 1
```

Launch only one new EC2.

The scheduler does not launch three.

---

# 24. Existing Idle Workers

Idle workers are even better than near-complete workers.

Suppose:

```text
RequiredWorkers = 3

IdleWorkers = 2
NearCompleteWorkers = 1
```

Then:

```text
ReusableWorkers = 2 + 1

                 = 3
```

Therefore:

```text
WorkersToLaunch = 0
```

The existing fleet is sufficient.

---

# 25. Worker Boot-Time Optimization

The 85% rule can be improved with estimated remaining time.

Store:

```text
progressPercent
estimatedRemainingSeconds
workerState
```

The scheduler can define:

```text
REUSE_THRESHOLD = 85%
EC2_BOOT_TIME = measured average startup time
BOOT_BUFFER = safety margin
```

A worker is considered near-complete when:

```text
progressPercent >= 85%
```

and its estimated remaining time is reasonably close to or below:

```text
EC2_BOOT_TIME + BOOT_BUFFER
```

This prevents unnecessary EC2 launches.

The exact threshold should be tuned using real fleet startup measurements.

---

# 26. Global Fleet Capacity

There is a difference between:

```text
Maximum workers for one video
```

and:

```text
Maximum workers for the entire fleet
```

Example:

```text
MAX_WORKERS_PER_VIDEO = 4
MAX_TOTAL_WORKERS = 20
```

You can therefore have:

```text
Video A → max 4
Video B → max 4
Video C → max 4
Video D → max 4
Video E → max 4
```

while the fleet maximum remains:

```text
20 EC2s
```

The fleet scheduler must enforce both limits.

---

# 27. Global Launch Formula

Let:

```text
GlobalRunningWorkers
GlobalIdleWorkers
GlobalNearCompleteWorkers
MAX_TOTAL_WORKERS
```

Then:

```text
GlobalLaunchCapacity =
    MAX_TOTAL_WORKERS
    - GlobalRunningWorkers
```

For a specific video:

```text
VideoRequiredWorkers =
    min(
        ChunkCount,
        MAX_WORKERS_PER_VIDEO
    )
```

Then:

```text
VideoReusableWorkers =
    IdleWorkers
    +
    NearCompleteWorkers
```

Then:

```text
VideoWorkersToLaunch =
    max(
        0,
        VideoRequiredWorkers
        - VideoReusableWorkers
    )
```

Finally:

```text
WorkersToLaunch =
    min(
        VideoWorkersToLaunch,
        GlobalLaunchCapacity
    )
```

---

# 28. Scheduler Example With Multiple Videos

Suppose:

```text
MAX_TOTAL_WORKERS = 10
MAX_WORKERS_PER_VIDEO = 4
```

Current fleet:

```text
EC2 #1 → Video X → 90%
EC2 #2 → Video X → 88%
EC2 #3 → Video Y → 40%
EC2 #4 → Video Y → 20%
EC2 #5 → IDLE
EC2 #6 → IDLE
```

Now Video A arrives.

Video A:

```text
60 min
20-minute chunks

ChunkCount = 3
```

Therefore:

```text
VideoRequiredWorkers = 3
```

Available immediately:

```text
Idle = 2
```

Near complete:

```text
#1
#2
```

The scheduler has enough reusable capacity.

Therefore:

```text
WorkersToLaunch = 0
```

When workers finish:

```text
#1 → Video A
#2 → Video A
#5 → Video A
```

No new EC2 is required.

---

# 29. Worker Lifecycle

Every worker follows this lifecycle:

```text
PROVISIONING
     ↓
BOOTING
     ↓
REGISTERING
     ↓
IDLE
     ↓
PROCESSING
     ↓
UPLOADING
     ↓
IDLE
     ↓
PROCESSING
     ↓
...
```

The worker can remain alive for many videos.

Eventually:

```text
IDLE
  ↓
Idle timeout
  ↓
Fleet no longer needs capacity
  ↓
STOPPING
  ↓
TERMINATED
```

---

# 30. Work Stealing / Queue Consumption

The queue should be global.

Do not create permanently isolated queues like:

```text
video-1-queue
video-2-queue
video-3-queue
```

Prefer:

```text
encode-video
```

with jobs containing:

```text
videoId
chunkId
chunkIndex
sourceKey
qualities
```

Then any idle worker can consume the next available job.

This gives the fleet natural work sharing.

---

# 31. Quality Generation

For one chunk:

```text
chunk-001
```

the worker generates:

```text
240p
360p
480p
540p
720p
900p
1080p
1440p
2160p
4320p
```

only when requested.

If the user requests:

```text
360p
720p
1080p
```

the worker generates only:

```text
360p
720p
1080p
```

There is no reason to encode unused qualities.

---

# 32. Worker Completion, Termination, and Queue Check

After a worker completes an encoding chunk, it must **not automatically terminate**, and it must **not decide to terminate itself** — both decisions belong to the Fleet Manager.

The lifecycle is:

```text
Encoding chunk
    ↓
Upload output to S3
    ↓
Mark chunk COMPLETE
    ↓
ACK pg-boss job
    ↓
Check pg-boss for another pending job
    │
    ├── Task exists
    │      ↓
    │   Take next task
    │      ↓
    │   Process it
    │
    └── No task exists
           ↓
        Worker → Fleet Manager: "NO_WORK"
           ↓
        Fleet Manager re-verifies pg-boss
           │
           ├── Task exists → keep worker, assign next job
           │
           └── No task     → terminate worker
```

The next task can belong to the same video or a different one (§19) — a worker is never permanently associated with the video that created it, including when it just finished that video's last chunk.

## 32.1 Worker Does Not Self-Terminate

The worker never calls the cloud provider's termination API directly. Responsibility is separated:

```text
Media Worker
    │
    ├── process media
    ├── upload output
    ├── report progress
    └── report NO_WORK
             │
             ▼
      Fleet Manager
             │
             ├── verify queue state
             ├── verify worker state
             └── terminate worker (via CloudDriver — see code-structure.md)
```

This keeps cloud-provider operations inside the Fleet Manager and its `CloudDriver` adapters, and prevents a race condition where a worker would terminate itself while a new job is being assigned to it.

## 32.2 Race-Safe Termination

A new job can arrive in the gap between the worker's queue check and the Fleet Manager's termination call. To close that window, the Fleet Manager always re-checks pg-boss after receiving `NO_WORK`, before terminating anything:

```text
Worker checks pg-boss → No task → Worker sends NO_WORK
        ↓
Fleet Manager re-checks pg-boss
        │
        ├── Task exists → keep worker, assign it
        └── No task     → terminate worker
```

## 32.3 Serverless vs Serverful Termination

The `NO_WORK` event is handled identically in both deployment modes — only the entry point differs:

```text
Serverless: Worker → AWS Lambda        → verify → CloudDriver → terminated
Serverful:  Worker → Fleet Manager API → verify → CloudDriver → terminated
```

Worker behavior is the same regardless of which mode the Fleet Manager runs in.

---

# 33. PostgreSQL State

Recommended `video_jobs` fields:

```text
id
status
source_key
duration_seconds
source_width
source_height
source_fps
source_codec

requested_qualities
quality_complexity
source_complexity

chunk_duration_seconds
chunk_count

required_workers
active_workers
completed_chunks

created_at
started_at
completed_at
```

Recommended `video_chunks` fields:

```text
id
video_id
chunk_index

start_seconds
duration_seconds

source_key

status
worker_id

started_at
completed_at

retry_count
error
```

Recommended `workers` fields:

```text
id
instance_id
provider
instance_type

state

current_job_id
current_video_id

progress_percent
estimated_remaining_seconds

last_heartbeat_at
started_at
idle_since
```

---

# 34. Worker Heartbeat

Every worker should periodically report:

```text
workerId
jobId
videoId
progressPercent
fps
frames
estimatedRemainingSeconds
cpuUsage
memoryUsage
state
```

This data is used by the Fleet Scheduler.

The scheduler should not depend only on a worker saying:

```text
"busy"
```

It needs enough information to decide whether an existing worker is close enough to completion to reuse.

---

# 35. Failure Handling

If an EC2 dies:

```text
Worker
   ↓
heartbeat timeout
   ↓
mark worker FAILED
   ↓
job becomes retryable
   ↓
pg-boss retry
   ↓
another worker picks the chunk
```

The chunk must not become permanently lost because its EC2 disappeared.

---

# 36. Job Idempotency

Every chunk encoding job should be idempotent.

Use a unique identifier:

```text
videoId + chunkId + encodingVersion
```

Before starting, the worker checks whether the output already exists or whether the database already marks the chunk complete.

This prevents duplicate output if a worker crashes after uploading but before acknowledging the pg-boss job.

---

# 37. S3 Layout, Chunk Naming, and Cleanup

Temporary processing assets and final playback assets are stored separately.

```text
s3://video-processing-source/
└── videos/
    └── {videoId}/
        ├── source/
        │   └── original.mp4
        │
        └── chunks/
            ├── chunk-00001.mp4
            ├── chunk-00002.mp4
            └── chunk-00003.mp4
```

The no-reencode splitter creates deterministic names:

```text
chunk-{chunkIndex}.mp4

Examples:
chunk-00001.mp4
chunk-00002.mp4
chunk-00003.mp4
```

The generated S3 key is stored with the chunk record/job.

Final playback assets are stored separately:

```text
s3://video-processing-output/
└── videos/
    └── {videoId}/
        ├── master.m3u8
        ├── 240p/
        │   ├── playlist.m3u8
        │   └── segments...
        ├── 360p/
        │   ├── playlist.m3u8
        │   └── segments...
        ├── 720p/
        │   ├── playlist.m3u8
        │   └── segments...
        └── 1080p/
            ├── playlist.m3u8
            └── segments...
```

The application stores the final playback entry point:

```text
/videos/{videoId}/master.m3u8
```

`master.m3u8` is the single entry point for the player. It is a manifest, not one physical video file.

## 37.1 Temporary Chunk Cleanup

No-reencode chunks are temporary.

Only after:

```text
all chunk jobs COMPLETE
        ↓
Finalizer successfully creates final manifests
        ↓
video status = COMPLETE
```

the cleanup process deletes:

```text
s3://video-processing-source/videos/{videoId}/chunks/*
```

The final HLS/DASH assets remain available for playback.

The original source is deleted separately according to the application's source-retention policy.

Cleanup must happen **after successful finalization**, so failed processing can still be retried from the temporary chunks.


# 38. Two pg-boss Queues

The system uses **two separate pg-boss jobs** with different responsibilities.

## Queue 1 — Video Processing Job

The backend creates the initial video-level job.

Payload:

```json
{
  "jobId": "job-123",
  "videoId": "video-123",
  "videoPath": "s3://video-processing-source/videos/video-123/source/original.mp4",
  "qualities": [240, 360, 720, 1080]
}
```

This job means:

```text
Process this video.
```

The video-level worker performs:

```text
read source
    ↓
analyze metadata
    ↓
calculate source complexity
    ↓
calculate requested-quality workload
    ↓
calculate no-reencode chunk duration
    ↓
split source without encoding
    ↓
upload chunks to S3
    ↓
create Queue 2 jobs
```

Queue 1 contains the high-level video task. It does not contain every chunk.

## Queue 2 — Chunk Encoding Jobs

After the no-reencode split, one pg-boss job is created for every chunk.

Example:

```text
video-123
├── chunk-00001
├── chunk-00002
└── chunk-00003
```

Payload:

```json
{
  "videoId": "video-123",
  "chunkId": "chunk-00001",
  "chunkPath": "s3://video-processing-source/videos/video-123/chunks/chunk-00001.mp4",
  "chunkIndex": 0,
  "startSeconds": 0,
  "durationSeconds": 1200,
  "qualities": [240, 360, 720, 1080]
}
```

Queue 2 means:

```text
Encode this specific chunk.
```

One worker takes **one Queue 2 job at a time** and generates all requested qualities for that chunk.

After completing it, the worker checks Queue 2 again:

```text
Task exists
    ↓
take next task immediately

No task
    ↓
send NO_WORK to Fleet Manager
```

The next task can belong to the same video or another video.

## 38.1 Queue Responsibilities

```text
┌────────────────────────────────────────────┐
│ Queue 1: video-processing                  │
│                                            │
│ Input: jobId + videoPath                   │
│                                            │
│ Responsibility:                            │
│ analyze → size → split → create chunks    │
└──────────────────────┬─────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────┐
│ Queue 2: video-chunk-encoding              │
│                                            │
│ Input: chunkPath + chunk metadata          │
│                                            │
│ Responsibility:                            │
│ encode → all requested qualities → S3      │
└────────────────────────────────────────────┘
```

---

# 39. Complete Example

## Input

```text
Video:
60 minutes

Source:
1920x1080
30 FPS

Requested qualities:
240p
360p
480p
720p
1080p
```

Sizing engine:

```text
QualityComplexity
    = 0.8 + 1.0 + 1.2 + 2.0 + 3.0
    = 8.0
```

Assume the calibrated source complexity is:

```text
SourceComplexity = 1.0
```

The target work budget produces:

```text
ChunkDuration = 12 minutes
```

Therefore:

```text
ChunkCount =
    ceil(60 / 12)

    = 5
```

But:

```text
MAX_WORKERS_PER_VIDEO = 4
```

Therefore:

```text
VideoRequiredWorkers =
    min(5, 4)

    = 4
```

The chunks are:

```text
1 → 0–12
2 → 12–24
3 → 24–36
4 → 36–48
5 → 48–60
```

Four workers can process the first four chunks:

```text
EC2 #1 → chunk 1
EC2 #2 → chunk 2
EC2 #3 → chunk 3
EC2 #4 → chunk 4
```

When EC2 #2 finishes:

```text
EC2 #2 → chunk 5
```

No fifth EC2 is required.

---

# 40. Example With Reusable Fleet

Current fleet:

```text
EC2 #1 → Video X → 90%
EC2 #2 → Video X → 88%
EC2 #3 → Video Y → 40%
EC2 #4 → Video Y → 30%
EC2 #5 → IDLE
```

New Video Z requires:

```text
4 workers
```

Reusable capacity:

```text
#1 = near complete
#2 = near complete
#5 = idle

ReusableWorkers = 3
```

Therefore:

```text
WorkersToLaunch = 4 - 3

                 = 1
```

The scheduler launches exactly one new EC2.

As workers finish:

```text
#1 → Video Z chunk
#2 → Video Z chunk
#5 → Video Z chunk
#6 → Video Z chunk
```

This avoids unnecessary EC2 boot time.

---

# 41. Important Scheduling Principle

The system should optimize for:

```text
minimum required EC2 launches
+
maximum worker utilization
+
maximum useful parallelism
```

It should **not** optimize for:

```text
one video = fixed number of EC2s
```

The actual model is:

```text
Video
  ↓
Dynamic chunk duration
  ↓
Chunk count
  ↓
Maximum possible workers for that video
  ↓
Check reusable fleet capacity
  ↓
Launch only missing workers
  ↓
Workers consume global pg-boss queue
```

---

# 42. Final Formulas

## Formula A — Chunk Duration

```text
QualityComplexity =
    Σ QualityWeight(q)

SourceComplexity =
    ResolutionFactor
    × FPSFactor
    × CodecFactor

WorkPerMinute =
    QualityComplexity
    × SourceComplexity

RawChunkDuration =
    TARGET_CHUNK_WORK
    /
    WorkPerMinute

ChunkDuration =
    clamp(
        RawChunkDuration,
        MIN_CHUNK_DURATION,
        MAX_CHUNK_DURATION
    )

ChunkCount =
    ceil(
        VideoDuration / ChunkDuration
    )
```

The relationship is:

```text
More quality / complexity
        ↓
Smaller chunk duration
        ↓
More chunks
        ↓
More possible parallelism
```

and:

```text
Less quality / complexity
        ↓
Larger chunk duration
        ↓
Fewer chunks
        ↓
Less required parallelism
```

---

## Formula B — Workers Required for a Video

```text
VideoRequiredWorkers =
    min(
        ChunkCount,
        MAX_WORKERS_PER_VIDEO
    )
```

with:

```text
MAX_WORKERS_PER_VIDEO = 4
```

Then:

```text
ReusableWorkers =
    IdleWorkers
    +
    NearCompleteWorkers
```

where:

```text
NearCompleteWorker =
    progress >= 85%
```

Then:

```text
VideoWorkersToLaunch =
    max(
        0,
        VideoRequiredWorkers
        - ReusableWorkers
    )
```

Finally enforce the global fleet limit:

```text
WorkersToLaunch =
    min(
        VideoWorkersToLaunch,
        MAX_TOTAL_WORKERS - GlobalRunningWorkers
    )
```

---

# 43. Final Architecture Rules

1. **Supported qualities:** 144p, 240p, 360p, 480p, 540p, 720p, 900p, 1080p, 1440p, 2160p, 4320p.
2. Quality is user-selected.
3. One worker generates all requested qualities for its assigned chunk.
4. The first worker analyzes and splits the source.
5. Splitting uses stream copy/no-reencode whenever technically possible.
6. The first worker then becomes a normal encoding worker.
7. One worker processes one chunk at a time.
8. After finishing a chunk, the worker immediately takes another pending chunk.
9. The next chunk may belong to the same or another video.
10. Workers are reusable across videos.
11. Chunk duration is dynamically calculated from encoding workload.
12. Higher quality/complexity produces smaller processing chunks.
13. Lower quality/complexity produces larger processing chunks.
14. `ChunkCount` determines the maximum possible parallelism for that video.
15. A video cannot use more workers than its chunk count.
16. `MAX_WORKERS_PER_VIDEO = 4` provides an additional safety ceiling.
17. Existing idle workers are reused first.
18. Workers at or above the reuse threshold can be treated as near-complete reusable capacity.
19. New EC2s are launched only for capacity that is actually missing.
20. The global fleet has its own maximum worker limit.
21. Completed workers remain alive for reuse until the fleet scheduler decides they are excess.
22. Idle workers can eventually be terminated after an idle timeout.
23. pg-boss is the global work queue.
24. Workers must be idempotent and retry-safe.
25. Final HLS/DASH manifests are generated after all required chunks are complete.

---

# 44. Initial Configuration

```yaml
worker:
  default_instance_type: c5.2xlarge
  max_workers_per_video: 4
  max_total_workers: 20

reuse:
  progress_threshold: 85
  idle_timeout_seconds: 300

chunk:
  min_duration_seconds: 300
  max_duration_seconds: 1800
  target_work_budget: configurable

quality_weights:
  144: 0.6
  240: 0.8
  360: 1.0
  480: 1.2
  540: 1.4
  720: 2.0
  900: 2.4
  1080: 3.0
  1440: 4.0
  2160: 6.0
  4320: 10.0

queue:
  provider: postgresql
  library: pg-boss

storage:
  source: s3
  chunks: s3
  output: s3
```

The quality weights and chunk-work budget should be treated as **calibration parameters**, not permanent constants. Benchmark real FFmpeg jobs on the chosen `c5.2xlarge` and tune them using observed encoding time.

---

# 45. Final Mental Model

The complete system can be reduced to:

```text
USER VIDEO
    │
    ▼
QUALITY SELECTION
    │
    ▼
WORKLOAD CALCULATION
    │
    ▼
DYNAMIC CHUNK DURATION
    │
    ▼
NO-REENCODE SPLIT
    │
    ▼
CHUNK COUNT
    │
    ▼
MAX WORKERS FOR VIDEO
    │
    ▼
CHECK EXISTING EC2 FLEET
    │
    ├── idle workers
    ├── near-complete workers
    └── running workers
    │
    ▼
LAUNCH ONLY MISSING CAPACITY
    │
    ▼
PG-BOSS
    │
    ▼
SHARED WORKER POOL
    │
    ├── Worker → Chunk A → all qualities
    ├── Worker → Chunk B → all qualities
    ├── Worker → Chunk C → all qualities
    └── Worker → next pending chunk
    │
    ▼
S3 HLS/DASH
    │
    ▼
FINAL MANIFEST
    │
    ▼
VIDEO COMPLETE
```

The most important invariant is:

```text
MAX_WORKERS_FOR_VIDEO
    <=
NUMBER_OF_CHUNKS_FOR_VIDEO
    <=
GLOBAL_WORKER_CAPACITY
```

This prevents the system from creating useless EC2 instances while still allowing high-complexity videos to gain more parallelism through smaller no-reencode chunks.
