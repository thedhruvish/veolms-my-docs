# Serverful Video Processing Pipeline

## 1. Overview

The system processes uploaded videos using temporary EC2 workers.

Components:

* Backend API
* PostgreSQL
* pg-boss
* Fleet Manager
* EC2 Workers
* Object Storage
* FFmpeg

---

## 2. Architecture

```text
                         ┌──────────────┐
                         │     User     │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │   Backend    │
                         └──────┬───────┘
                                │
                         Create Video Job
                                │
                                ▼
                     ┌────────────────────┐
                     │    PostgreSQL      │
                     │                    │
                     │   video_jobs       │
                     │   workers          │
                     └─────────┬──────────┘
                               │
                               ▼
                     ┌────────────────────┐
                     │      pg-boss       │
                     │                    │
                     │ video-processing   │
                     └─────────┬──────────┘
                               │
                               │ Job trigger
                               ▼
                     ┌────────────────────┐
                     │   Fleet Manager    │
                     │                    │
                     │ Check workers      │
                     │ Start EC2          │
                     └─────────┬──────────┘
                               │
                               ▼
                     ┌────────────────────┐
                     │    EC2 Worker      │
                     │                    │
                     │      FFmpeg        │
                     │      HLS           │
                     │      Progress      │
                     └─────────┬──────────┘
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
             ┌──────────────┐      ┌──────────────┐
             │  PostgreSQL  │      │    Storage   │
             │              │      │              │
             │ Progress     │      │ HLS          │
             │ Status       │      │ Master       │
             │ Worker       │      │ Segments     │
             └──────────────┘      └──────────────┘
```

---

# 3. Backend

The backend:

1. Receives the video upload request.
2. Stores the original video.
3. Creates a `video_jobs` record.
4. Adds the job to the `video-processing` pg-boss queue.

The backend does not process the video.

---

# 4. Queue

There is only **one queue**:

```text
video-processing
```

Example job:

```json
{
  "jobId": "job_123",
  "videoId": "video_123",
  "inputPath": "original/video_123.mp4",
  "quality":[720]
}
```

The queue is used for video-processing jobs.

---

# 5. Fleet Manager

The Fleet Manager is a continuously running serverful service.

It subscribes to the queue trigger.

```text
pg-boss
   ↓
new job
   ↓
Fleet Manager
```
- also it limit to the max n number of the worker are the run

When a job is triggered, Fleet Manager:

1. Checks available workers.
2. Checks currently processing workers.
3. Starts an EC2 worker if required.
4. Lets the EC2 worker process the queued job.

The Fleet Manager does not process videos.

---

# 6. EC2 Worker

Each EC2 worker runs:

```text
Node.js
FFmpeg
Worker application
```

The worker:

1. Starts.
2. Registers itself in PostgreSQL.
3. Consumes a job from `video-processing`.
4. Processes the video.
5. Generates HLS.
6. Uploads HLS to storage.
7. Updates progress.
8. Marks the job as completed.
9. Waits for another job.

---

# 7. One Worker = One Active Job

Each worker processes one video at a time.


```text
Worker 1 → Video A
Worker 2 → Video B
Worker 3 → Video C
```

If all workers are busy and another job arrives:

```text
Queue
  │
  ├── Job A → Worker 1
  ├── Job B → Worker 2
  ├── Job C → Worker 3
  └── Job D → Waiting
```

Fleet Manager starts another worker according to the configured worker limit.


---

# 9. Video Progress

Video progress is handled by the EC2 worker.

```text
FFmpeg
  ↓
Worker
  ↓
Calculate progress
  ↓
PostgreSQL
```

Example:

```text
0%
25%
50%
75%
100%
```

The worker updates:

```text
video_jobs.progress
video_jobs.status
video_jobs.current_stage
```

Stages:

```text
queued
downloading
transcoding
uploading
finalizing
completed
```

---

# 10. Worker Heartbeat

The worker periodically updates:

```text
last_heartbeat_at
```

Example:

```text
Worker
  ↓
DB heartbeat
  ↓
10 seconds
  ↓
DB heartbeat
  ↓
10 seconds
  ↓
DB heartbeat
```

Fleet Manager uses the heartbeat to detect failed workers.

---

# 11. Job Completion

After processing:

```text
FFmpeg complete
      ↓
HLS uploaded
      ↓
Master playlist created
      ↓
Progress = 100%
      ↓
Job = completed
      ↓
Worker = idle
```

The worker does not immediately terminate.

---

# 12. 30-Second Worker Wait

After completing a job:

```text
Worker
  ↓
idle
  ↓
wait 30 seconds
  ↓
check for next job
```

If a job exists:

```text
New job
  ↓
Process job
```

If no job exists:

```text
No job
  ↓
Update worker:
shutting_down
  ↓
Terminate EC2
```

---

# 13. Worker Self-Termination

The worker terminates itself after the 30-second idle period.

```text
Job completed
     ↓
Wait 30 seconds
     ↓
No new job
     ↓
Update DB:
shutting_down
     ↓
Terminate EC2
```

The database update happens before termination.

---

# 14. Worker States

```text
starting
   ↓
idle
   ↓
processing
   ↓
idle
   ↓
shutting_down
   ↓
terminated
```

Failure state:

```text
failed
```

---

# 15. Database

## video_jobs

```text
id
video_id
input_path
status
progress
current_stage
worker_id
quality
created_at
started_at
completed_at
failed_at
error
```

## workers

```text
id
instance_id
status
current_job_id
started_at
last_heartbeat_at
shutdown_at
terminated_at
```

## video_outputs

```text
id
video_id
master_playlist_path
created_at
```

---

# 17. Complete Flow

```text
User uploads video
        ↓
Backend creates video job
        ↓
Job added to pg-boss
        ↓
Fleet Manager is triggered
        ↓
Check available workers
        ↓
Start EC2 if required
        ↓
EC2 Worker starts
        ↓
Worker registers
        ↓
Worker consumes job
        ↓
Worker updates job = processing
        ↓
FFmpeg generates HLS
        ↓
Worker updates progress
        ↓
HLS uploaded
        ↓
Master playlist created
        ↓
Job = completed
        ↓
Worker = idle
        ↓
Wait 30 seconds
        ↓
Check for next job
        ↓
     ┌───────────────┐
     │  Job exists?  │
     └───────┬───────┘
             │
       ┌─────┴─────┐
       │           │
      YES          NO
       │           │
       ▼           ▼
 Process       shutting_down
 next job          ↓
               Terminate EC2
```

---
