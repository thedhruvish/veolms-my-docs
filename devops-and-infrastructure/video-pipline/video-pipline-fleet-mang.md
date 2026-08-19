

| Status | Date | 
|---|---:|
| Proposed | 19 August 2026 | 

Counterproposal to @bhupeshb7 


## 1. pg-boss → Fleet Manager Lambda

* We don't need to put any specific cloud-provider logic, such as EventBridge, into the backend.
* pg-boss-related logic, or any queue-related logic, should not be included in the backend.

### Suggested Architecture

**Backend → Job Table → Fleet Manager**

* The backend adds the video job to the `job` table.
* In **serverless mode**, after the job is inserted, it triggers the Fleet Manager Lambda using either a public URL or the AWS SDK.
* In **serverful mode**, the backend does not need to trigger the Lambda. The Fleet Manager is already running and uses pg-boss to pick up the job.
* The Fleet Manager is responsible for all queue-related logic.

### Breakdown

1. **Backend**

   * Once the backend confirms that the video has been successfully uploaded, it adds the video URL/key to the `job` table.
   * The backend does not interact with pg-boss or perform any polling.

2. **Serverless mode**

   * After inserting the job into the database, the backend triggers the Fleet Manager Lambda.
   * The Lambda checks the job status and starts a worker if required.

3. **Fleet Manager Lambda**

   * When the Lambda is triggered for the first time, it checks whether the job has been completed.
   * If the job is not completed, the Lambda schedules another execution for **N seconds later**.
   * This acts like a heartbeat mechanism: the Fleet Manager periodically checks whether the work is still in progress.
   * If the job is completed, it does not schedule another execution.
   * If a worker fails or is terminated unexpectedly, the next Fleet Manager execution can detect the incomplete job and start another worker.

4. **Serverful mode**

   * The Fleet Manager is continuously running.
   * It uses pg-boss to pick up jobs from the queue.
   * It periodically checks the status of workers and jobs.
   * If a new job is added while the Fleet Manager is already running, it can immediately pick up the job through pg-boss.

5. **Worker**

   * The worker processes the video.
   * When processing is completed, the worker updates the job in the database with:

     * Completed status
     * Output file/path
   * After completing its work, the worker can terminate itself.

---

## 2. Should pg-boss logic be inside the backend?

**No.**

The backend should **not contain pg-boss logic** because:

* The backend does not need to poll the queue.
* The backend's responsibility is only to create/update the job in the database.
* pg-boss is responsible for queue management and job polling.
* Queue-related logic should remain inside the **Fleet Manager**.

### Responsibility

| Component     | Responsibility                                   |
| ------------- | ------------------------------------------------ |
| Backend       | Create/update job in DB                          |
| Fleet Manager | Queue polling, worker management, job monitoring |
| pg-boss       | Queue management                                 |
| Worker        | Process video and update job status              |
| Database      | Store job and processing state                   |

For **serverful mode**, pg-boss is used by the continuously running Fleet Manager.

For **serverless mode**, the Fleet Manager Lambda is triggered externally, checks the state, starts workers if required, and schedules its next execution when monitoring is still required.

---

## 3. EC2 Termination Status

After a worker finishes its task:

1. The worker checks the `job` table for pending jobs.
2. If another job is available, it can continue processing it.
3. If there are no pending jobs for **N seconds**, the worker prepares to terminate.
4. Before terminating, the worker updates its status in the database to `TERMINATING` or `TERMINATED`.
5. Once the database update succeeds, the worker terminates itself.

### Why don't we send an event to the Fleet Manager after termination?

We don't need to.

The worker's termination state is already stored in the database. The Fleet Manager can determine the worker's state by checking the database.

---

## 4. What if the Database Update or EC2 Termination Fails?

There are two possible failure cases:

### Case 1: Database update fails, but the EC2 instance terminates

The worker may terminate without successfully updating its termination status in the database.

The Fleet Manager's next monitoring cycle can detect that the worker is no longer available and determine whether the job was completed.

### Case 2: Database update succeeds, but EC2 termination fails

The database says that the worker should be terminated, but the EC2 instance is still running.

The Fleet Manager's next monitoring cycle can detect this state and terminate the worker if necessary.

Therefore, we don't need the worker to send a separate EventBridge event.

The **Fleet Manager is responsible for reconciliation**.

---

## 5. Final Flow

```text
                    ┌──────────────┐
                    │   Backend    │
                    └──────┬───────┘
                           │
                           │ Insert Job
                           ▼
                    ┌──────────────┐
                    │  Job Table   │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              │                         │
        Serverless                  Serverful
              │                         │
              ▼                         ▼
     ┌────────────────┐        ┌────────────────┐
     │ Fleet Manager   │        │ Fleet Manager  │
     │ Lambda          │        │ (Always Running)│
     └───────┬────────┘        └───────┬────────┘
             │                         │
             │ Schedule next check     │ pg-boss
             │ if required             │ polling
             ▼                         ▼
        ┌──────────────────────────────────┐
        │            Worker                │
        └────────────────┬─────────────────┘
                         │
                         │ Process Video
                         ▼
                  ┌──────────────┐
                  │  Job Table   │
                  │   Completed  │
                  │ Output File  │
                  └──────┬───────┘
                         │
                         ▼
                  Worker terminates
```

### Important note

**The worker does not trigger EventBridge.**

**The backend does not contain pg-boss logic.**

**The Fleet Manager owns queue management, worker management, and monitoring.**

**The database is the source of truth for job and worker state.**
