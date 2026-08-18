# Serverless Video Processing Pipeline

## 1. Overview

The system uses a **serverless Fleet Manager** to control ephemeral EC2 workers.

---

- media-worker are the as it is mention on the serverfull-v1.md

---

# 2. Dynamic Monitoring Schedule

The Fleet Manager does **not** run continuously.

After starting the EC2 worker, it calculates when the next monitoring check should happen.

Example:

```text
Video:
10 GB

Estimated processing time:
10 minutes
```

The first monitoring schedule can be:

```text
10 minutes × 50%
        ↓
5 minutes
```

So:

```text
Video uploaded
      ↓
Start EC2
      ↓
Estimated time = 10 minutes
      ↓
Schedule Fleet Manager
      ↓
Run after 5 minutes
```

---

# 3. EventBridge Scheduler

The Fleet Manager creates a **one-time EventBridge Scheduler schedule** for its next execution.

```text
Fleet Manager
      ↓
Calculate next check time
      ↓
Create one-time schedule
      ↓
Lambda exits
```

At the scheduled time:

```text
EventBridge Scheduler
      ↓
Fleet Manager
```

The schedule is deleted after execution.

---

# 4. Monitoring Check

When the Fleet Manager runs, it checks:

```text
1. Job status
2. Video progress
3. Worker heartbeat
4. EC2 instance status
5. Processing errors
```

Example:

```text
Current progress:
48%

Heartbeat:
Healthy

EC2:
Running

Job:
Processing
```

The Fleet Manager then calculates the next monitoring time.

---

# 5. Dynamic Rescheduling

The next schedule is calculated from the current state.

```text
Fleet Manager
      ↓
Check progress
      ↓
Check heartbeat
      ↓
Check EC2
      ↓
Check errors
      ↓
Calculate next check
      ↓
Create new schedule
      ↓
Exit
```

Example:

```text
Worker starting
    ↓
Check again in 10 seconds
```

```text
Worker healthy
    ↓
Check again in 60 seconds
```

```text
Worker appears unhealthy
    ↓
Check again quickly
```

The schedule is therefore **dynamic**, rather than always running at a fixed interval.

---

# 6 Successful Completion

When processing finishes:

```text
FFmpeg
   ↓
HLS completed
   ↓
Upload completed
   ↓
Progress = 100%
   ↓
Status = completed
```

The Fleet Manager detects:

```text
status = completed
```

Then:

```text
Do not create another schedule
```

The monitoring cycle ends.

---


# 7. Failure Detection

If the Fleet Manager detects a problem:

```text
Worker heartbeat stopped
        ↓
Fleet Manager detects failure
        ↓
Check EC2
        ↓
Check job
```

Possible failures:

* EC2 instance stopped
* Worker process crashed
* Heartbeat stopped
* Processing is stuck
* FFmpeg failed
* Job entered `failed` state

---

# 8. Failure Recovery

When a worker fails:

```text
Failure detected
      ↓
Mark worker failed
      ↓
Check job state
      ↓
Start replacement EC2
      ↓
Retry job
      ↓
Create monitoring schedule
```

The new worker continues the processing flow.

---

# 9. New Video

After a video completes, the Fleet Manager does not keep scheduling itself.

A new video creates a new cycle:

```text
User uploads new video
        ↓
New video job
        ↓
Trigger Fleet Manager
        ↓
Start/reuse EC2
        ↓
Create first monitoring schedule
        ↓
Monitor
        ↓
Complete
        ↓
Stop scheduling
```

---

# 10. Complete Flow

```text
User
 ↓
Upload video
 ↓
Backend
 ↓
Create job
 ↓
pg-boss
 ↓
Trigger Fleet Manager
 ↓
Check workers
 ↓
Start EC2
 ↓
Estimate processing time
 ↓
Create first dynamic schedule
 ↓
Fleet Manager exits
 ↓
      ┌─────────────────────┐
      │ Scheduled execution │
      └──────────┬──────────┘
                 ↓
          Fleet Manager
                 ↓
       Check progress
       Check heartbeat
       Check EC2
       Check errors
                 ↓
          Job completed?
           /          \
         YES           NO
          ↓             ↓
   Stop scheduling   Check health
                        ↓
                 Worker healthy?
                   /        \
                 YES         NO
                  ↓           ↓
             Calculate      Recover
             next check     worker
                  ↓           ↓
                  └─────┬─────┘
                        ↓
                 Create schedule
                        ↓
                 Lambda exits
```

---

# 12. Architecture

```text
                    CONTROL PLANE

                         Backend
                            │
                            ▼
                    PostgreSQL + pg-boss
                            │
                            ▼
                  Video Upload Event
                            │
                            ▼
                   Fleet Manager Lambda
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
        Check Jobs      Check Workers   Start EC2
             │              │              │
             └──────────────┴──────────────┘
                            │
                            ▼
                  Dynamic Schedule
                            │
                            ▼
                 EventBridge Scheduler
                            │
                            └──────────────┐
                                           │
                                           ▼
                                  Fleet Manager Lambda


                    DATA PLANE

                    EC2 Worker
                        │
                        ├── Consume Job
                        ├── FFmpeg
                        ├── Generate HLS
                        ├── Upload HLS
                        ├── Update Progress
                        ├── Update Heartbeat
                        │
                        └── 30s Idle
                              ↓
                        Self Terminate
```

## 13. Purpose of the Dynamic Schedule

The dynamic schedule is used for **monitoring and recovery**, not for video processing.

It allows the serverless Fleet Manager to:

* Check worker heartbeat
* Check video progress
* Check EC2 health
* Detect processing errors
* Detect stuck workers
* Restart failed workers
* Stop monitoring completed jobs

The cycle continues only while the video is being processed.

```text
Processing
    ↓
Schedule
    ↓
Check
    ↓
Schedule again
    ↓
Check
    ↓
...
    ↓
Completed
    ↓
Stop scheduling
```
