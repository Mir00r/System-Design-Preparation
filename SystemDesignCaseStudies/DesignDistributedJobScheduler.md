# 🎬 Design a Distributed Job Scheduler (Cron at Scale)

> *"Uber runs 10 MILLION scheduled jobs per day — from payment processing to surge pricing calculations to ETL pipelines. A single missed job at 2am could mean drivers don't get paid on time. A distributed job scheduler must guarantee: every job runs EXACTLY when scheduled, even if servers crash, and no job runs TWICE (imagine processing payroll twice!). This is the backbone of any large-scale system."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Distributed Locking](../../Microservices/DistributedLocking.md), [Message Queues](../../BuildingBlocks/MessageQueues.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Why Not Just Cron?](#-why-not-just-cron)
3. [High-Level Architecture](#-high-level-architecture)
4. [Core Components](#-core-components)
5. [Job Storage & Scheduling](#-job-storage--scheduling)
6. [Execution Engine](#-execution-engine)
7. [Failure Handling](#-failure-handling)
8. [Scalability Design](#-scalability-design)
9. [Java Implementation](#-java-implementation)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
✅ Schedule one-time jobs (run at specific timestamp)
✅ Schedule recurring jobs (cron expressions: "every 5 min")
✅ Job prioritization (critical jobs run first)
✅ Job dependencies (Job B runs after Job A completes)
✅ Retry on failure (configurable retry policy)
✅ Job monitoring & alerting (dashboards, failure alerts)
✅ API for CRUD operations on jobs
✅ Support millions of scheduled jobs
```

### Non-Functional Requirements
```
✅ Reliability: every job must execute (exactly-once semantics)
✅ No duplicate execution (CRITICAL for payments!)
✅ Scalable: handle 10M+ jobs/day
✅ Low scheduling latency: job starts within 5s of scheduled time
✅ Fault-tolerant: survive server crashes without losing jobs
✅ Observability: track job status, duration, failures
```

---

## ❌ Why Not Just Cron?

```
CRON LIMITATIONS:
  ┌──────────────────────────────────────────────────────────┐
  │  Problem           │  Why It's Bad                       │
  ├──────────────────────────────────────────────────────────┤
  │  Single machine    │  Machine dies → ALL jobs stop! 💀    │
  │  No failover       │  No automatic recovery               │
  │  No distribution   │  Can't spread across multiple nodes  │
  │  No retry logic    │  Failed job = failed forever         │
  │  No dependencies   │  Can't express "run B after A"      │
  │  No visibility     │  No dashboard, no alerting           │
  │  No exactly-once   │  Server restart → duplicates!        │
  │  Scale limit       │  Single machine = limited capacity   │
  └──────────────────────────────────────────────────────────┘
  
  CRON IS FINE FOR: Personal scripts, single-server apps
  NOT FINE FOR: Distributed systems, critical business processes
```

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       CLIENTS                                    │
│   📱 API Calls    💻 Dashboard    🔧 Internal Services          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                  ┌────────▼────────┐
                  │   API Service   │  CRUD for jobs
                  └────────┬────────┘
                           │
         ┌─────────────────┼─────────────────────────┐
         │                 │                         │
         ▼                 ▼                         ▼
┌──────────────┐  ┌─────────────────┐      ┌──────────────────┐
│  Job Store   │  │  Scheduler      │      │  Executor Pool   │
│  (Database)  │  │  (picks due     │      │  (runs jobs)     │
│              │  │   jobs)         │      │                  │
└──────────────┘  └────────┬────────┘      └────────┬─────────┘
                           │                        │
                           ▼                        ▼
                  ┌─────────────────┐      ┌──────────────────┐
                  │  Task Queue     │      │  Dead Letter Q   │
                  │  (Kafka/Redis)  │      │  (failed jobs)   │
                  └─────────────────┘      └──────────────────┘

FLOW:
  1. Client creates job (API → Job Store)
  2. Scheduler scans for due jobs (every 1-5 seconds)
  3. Due jobs → Task Queue (with distributed lock to prevent duplicates!)
  4. Executor workers pull from queue → execute job
  5. On success: mark complete. On failure: retry or dead-letter.
```

---

## 🧩 Core Components

### Job Store (Database)

```sql
CREATE TABLE scheduled_jobs (
    id UUID PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    job_type VARCHAR(50) NOT NULL,  -- HTTP_CALLBACK, KAFKA_EVENT, LAMBDA
    schedule_type VARCHAR(20),      -- ONE_TIME, RECURRING
    cron_expression VARCHAR(100),   -- "0 */5 * * *" (every 5 min)
    next_execution_time TIMESTAMP NOT NULL,  -- When to run next
    last_execution_time TIMESTAMP,
    status VARCHAR(20) DEFAULT 'SCHEDULED',  -- SCHEDULED, RUNNING, COMPLETED, FAILED
    payload JSONB,                  -- Job-specific data
    priority INT DEFAULT 5,        -- 1=highest, 10=lowest
    max_retries INT DEFAULT 3,
    retry_count INT DEFAULT 0,
    timeout_seconds INT DEFAULT 300,
    created_at TIMESTAMP DEFAULT NOW(),
    locked_by VARCHAR(100),        -- Which scheduler instance owns it
    locked_at TIMESTAMP            -- When it was locked (for stale lock detection)
);

-- Critical index: find due jobs efficiently!
CREATE INDEX idx_next_execution ON scheduled_jobs(next_execution_time)
    WHERE status = 'SCHEDULED';
```

### Scheduler Service

```
SCHEDULER LOOP (runs on multiple instances!):
  
  Every 1 second:
    1. Query: SELECT * FROM scheduled_jobs 
              WHERE next_execution_time <= NOW()
              AND status = 'SCHEDULED'
              ORDER BY priority, next_execution_time
              LIMIT 100;
              
    2. For each job:
       a. Try to LOCK it (atomic update with WHERE clause!)
       b. If lock acquired → enqueue to Task Queue
       c. If lock failed → another scheduler got it (skip!)
       
    3. For recurring jobs after completion:
       Calculate next_execution_time from cron expression
       Update record with new scheduled time

DISTRIBUTED LOCKING (prevent duplicate execution):
  
  UPDATE scheduled_jobs 
  SET status = 'RUNNING', 
      locked_by = 'scheduler-instance-3',
      locked_at = NOW()
  WHERE id = 'job-uuid-123'
    AND status = 'SCHEDULED'     -- Only if not already taken!
    AND next_execution_time <= NOW();
    
  If rows_affected = 1 → I got the lock! Enqueue it!
  If rows_affected = 0 → Someone else got it! Skip!
```

---

## ⚡ Execution Engine

```
EXECUTOR WORKERS:
  Multiple worker instances pulling from Task Queue (Kafka/SQS)
  
  Worker loop:
    1. Pull job from queue
    2. Execute based on job_type:
       • HTTP_CALLBACK: POST to configured URL with payload
       • KAFKA_EVENT: Publish event to configured topic
       • LAMBDA: Invoke serverless function
       • CONTAINER: Spin up container with job logic
    3. Track execution time (timeout if exceeds limit)
    4. On success: mark COMPLETED, trigger dependent jobs
    5. On failure: retry or dead-letter

EXECUTION TYPES:
  ┌─────────────────────────────────────────────────────────────┐
  │  Type           │  How It Works                              │
  ├─────────────────────────────────────────────────────────────┤
  │  HTTP Callback  │  POST https://service.com/webhook          │
  │                 │  Payload: job data. Expect 2xx response.   │
  ├─────────────────────────────────────────────────────────────┤
  │  Message Queue  │  Publish message to Kafka/SQS topic        │
  │                 │  Consumer processes asynchronously          │
  ├─────────────────────────────────────────────────────────────┤
  │  Function       │  Invoke Lambda/Cloud Function              │
  │                 │  Isolated execution, auto-scaled           │
  ├─────────────────────────────────────────────────────────────┤
  │  Container      │  Spin up Docker container with job code    │
  │                 │  For heavy jobs (ETL, ML training)         │
  └─────────────────────────────────────────────────────────────┘
```

---

## 🔄 Failure Handling

```
RETRY STRATEGY (Exponential Backoff):
  
  Attempt 1: Execute immediately
  Attempt 2: Wait 30 seconds, retry
  Attempt 3: Wait 2 minutes, retry
  Attempt 4: Wait 10 minutes, retry (if max_retries allows)
  All failed: Move to Dead Letter Queue + Alert!
  
  retry_delay = base_delay × 2^(attempt - 1) + random_jitter
  
TIMEOUT HANDLING:
  Job running too long?
    • Worker sets timeout alarm when starting job
    • If timeout fires: kill execution, mark as TIMED_OUT
    • Retry with same policy (timeout ≠ permanent failure)
    
STALE LOCK DETECTION:
  What if a scheduler crashes AFTER locking a job but BEFORE enqueuing?
  
  Background job (every 30s):
    SELECT * FROM scheduled_jobs 
    WHERE status = 'RUNNING' 
    AND locked_at < NOW() - INTERVAL '5 minutes';
    
  These jobs were "locked" 5+ minutes ago but never completed!
  → Reset status to 'SCHEDULED' → will be picked up again!

EXACTLY-ONCE GUARANTEE:
  Problem: network failure AFTER job executes but BEFORE ACK
  
  Solution: Idempotency!
    • Each job execution gets unique execution_id
    • Job handler checks: "Have I processed execution_id X?"
    • If yes → skip (idempotent)
    • If no → process + record execution_id
```

---

## 📈 Scalability Design

```
SCALING THE SCHEDULER:
  
  Problem: Multiple schedulers scanning same table = lock contention!
  
  Solution 1: PARTITION by job_id hash
    Scheduler 1: handles jobs where hash(id) % 4 == 0
    Scheduler 2: handles jobs where hash(id) % 4 == 1
    Scheduler 3: handles jobs where hash(id) % 4 == 2
    Scheduler 4: handles jobs where hash(id) % 4 == 3
    
    No contention! Each scheduler owns a partition!
    
  Solution 2: TIME-BASED SHARDING
    Database sharded by next_execution_time:
    Shard 1: jobs due in next 1 minute
    Shard 2: jobs due in 1-5 minutes
    Shard 3: jobs due in 5-60 minutes
    
    Hot shard (due now) → more schedulers assigned

SCALING THE EXECUTORS:
  Simple! Just add more workers pulling from the Task Queue.
  Kafka/SQS naturally distributes work across consumers.
  
  Auto-scale based on queue depth:
    Queue depth > 10,000 → scale up workers
    Queue depth < 100 → scale down workers

HANDLING MILLIONS OF JOBS:
  10M jobs/day = ~115 jobs/second average
  But they're NOT evenly distributed! 
  "Every midnight UTC" = millions of jobs at 00:00:00! 😱
  
  Solution: Jitter
    Instead of exact midnight: midnight + random(0, 60 seconds)
    Spreads the "midnight spike" across 60 seconds
```

---

## 💻 Java Implementation

### Scheduler Service

```java
@Service
public class JobSchedulerService {
    
    @Autowired private JobRepository jobRepository;
    @Autowired private KafkaTemplate<String, JobExecution> taskQueue;
    
    @Value("${scheduler.instance-id}")
    private String instanceId;
    
    @Scheduled(fixedRate = 1000) // Check every second
    public void scanForDueJobs() {
        // Find jobs that are due (with partition assignment)
        List<ScheduledJob> dueJobs = jobRepository.findDueJobs(
            Instant.now(), 
            getMyPartition(), // Only my assigned partition!
            100 // Batch size
        );
        
        for (ScheduledJob job : dueJobs) {
            if (tryLockJob(job)) {
                enqueueForExecution(job);
            }
        }
    }
    
    private boolean tryLockJob(ScheduledJob job) {
        // Atomic lock via optimistic update
        int updated = jobRepository.tryLock(
            job.getId(), 
            instanceId, 
            Instant.now(),
            "SCHEDULED" // Only lock if still SCHEDULED!
        );
        return updated == 1; // Got the lock!
    }
    
    private void enqueueForExecution(ScheduledJob job) {
        JobExecution execution = JobExecution.builder()
            .executionId(UUID.randomUUID().toString())
            .jobId(job.getId())
            .payload(job.getPayload())
            .jobType(job.getJobType())
            .attempt(job.getRetryCount() + 1)
            .timeout(job.getTimeoutSeconds())
            .build();
        
        taskQueue.send("job-executions", job.getId(), execution);
        
        // For recurring jobs: schedule next occurrence
        if (job.getScheduleType() == ScheduleType.RECURRING) {
            Instant nextRun = CronExpression.parse(job.getCronExpression())
                .next(Instant.now());
            jobRepository.scheduleNext(job.getId(), nextRun);
        }
    }
}
```

### Executor Worker

```java
@Service
public class JobExecutorWorker {
    
    @KafkaListener(topics = "job-executions", groupId = "executors")
    public void executeJob(JobExecution execution) {
        log.info("Executing job {} (attempt {})", 
            execution.getJobId(), execution.getAttempt());
        
        try {
            // Execute with timeout
            CompletableFuture<Void> future = CompletableFuture.runAsync(
                () -> doExecute(execution));
            
            future.get(execution.getTimeout(), TimeUnit.SECONDS);
            
            // Success!
            markCompleted(execution);
            
        } catch (TimeoutException e) {
            handleTimeout(execution);
        } catch (Exception e) {
            handleFailure(execution, e);
        }
    }
    
    private void doExecute(JobExecution execution) {
        switch (execution.getJobType()) {
            case HTTP_CALLBACK -> executeHttpCallback(execution);
            case KAFKA_EVENT -> publishKafkaEvent(execution);
            case LAMBDA -> invokeLambda(execution);
        }
    }
    
    private void handleFailure(JobExecution execution, Exception e) {
        ScheduledJob job = jobRepository.findById(execution.getJobId());
        
        if (job.getRetryCount() < job.getMaxRetries()) {
            // Schedule retry with exponential backoff
            Duration delay = Duration.ofSeconds(
                (long) (30 * Math.pow(2, job.getRetryCount())));
            
            jobRepository.scheduleRetry(job.getId(), 
                Instant.now().plus(delay),
                job.getRetryCount() + 1);
        } else {
            // Max retries exceeded → Dead Letter Queue + Alert!
            deadLetterService.send(execution, e);
            alertService.triggerJobFailureAlert(job);
        }
    }
}
```

---

## 🎮 Mini Challenge

### 🧩 Design: Job Dependencies (DAG Execution)

Design a system where Job C depends on both Job A AND Job B completing successfully.

```
    Job A ──┐
            ├──► Job C (runs only when both A and B are done!)
    Job B ──┘
```

<details>
<summary>🔑 Answer</summary>

**Data model:**
```sql
CREATE TABLE job_dependencies (
    job_id UUID,
    depends_on_job_id UUID,
    PRIMARY KEY (job_id, depends_on_job_id)
);
-- Job C depends on A and B:
INSERT INTO job_dependencies VALUES ('C', 'A');
INSERT INTO job_dependencies VALUES ('C', 'B');
```

**Execution flow:**
1. Job A completes → check: "Are there jobs that depend on A?"
2. Find Job C. Check all of C's dependencies: A=✅, B=?
3. B not complete yet → don't trigger C
4. Later: Job B completes → check dependents
5. Find Job C. Check: A=✅, B=✅ → ALL dependencies met!
6. Trigger Job C → enqueue for execution!

**Implementation:**
```java
public void onJobCompleted(String jobId) {
    List<String> dependentJobs = getDependents(jobId);
    for (String dependent : dependentJobs) {
        if (allDependenciesMet(dependent)) {
            enqueueJob(dependent); // All deps done → run!
        }
    }
}
```

**Handling failures:** If Job A fails → Job C will never trigger. Options: (1) Cascade failure to C, (2) Alert and wait for manual retry of A, (3) Auto-retry A, if succeeds → re-check C's deps.
</details>

---

## ❓ Interview Q&A

**Q1: How do you prevent duplicate job execution in a distributed scheduler?**
> Use database-level atomic locking: UPDATE job SET status='RUNNING', locked_by=? WHERE id=? AND status='SCHEDULED'. Only one instance can successfully update (others get 0 rows affected). For extra safety: idempotency keys on the execution side — even if a job somehow runs twice, the handler checks if execution_id was already processed.

**Q2: How do you handle scheduler instance failures?**
> Stale lock detection: background process checks for jobs in 'RUNNING' state with locked_at older than timeout (e.g., 5 minutes). These are presumed crashed — reset to 'SCHEDULED' for re-pickup. Multiple scheduler instances ensure redundancy. If one dies, others continue scanning their partitions. Leadership election can reassign dead instance's partition.

**Q3: How do you scale to 10 million jobs per day?**
> (1) Partition jobs across scheduler instances (by hash or time range), (2) Queue-based execution (Kafka distributes work to executor pool), (3) Auto-scale executors based on queue depth, (4) Jitter for "thundering herd" (millions at midnight → add random 0-60s offset), (5) Shard job database by time (hot partition for "due now" jobs).

**Q4: How do you handle timezone-aware scheduling?**
> Store next_execution_time in UTC always. When user creates job with "8am EST daily": convert to UTC cron, calculate next UTC execution time. Handle DST transitions: "2am daily" might not exist on spring-forward day or happen twice on fall-back day. Strategy: skip if time doesn't exist, run once if time repeats.

**Q5: Compare this design to existing tools (Quartz, Airflow, Temporal).**
> Quartz: Java library, good for single-JVM, limited distributed support. Airflow: DAG-based, great for data pipelines, Python-centric. Temporal: Workflow engine with durable execution, excellent fault tolerance, heavier. Our design: purpose-built for high-throughput simple job scheduling with exactly-once guarantees. Choose based on: simple scheduled tasks → our design, complex workflows → Temporal, data pipelines → Airflow.

---

## 🔗 Related Topics
- [Distributed Locking](../../Microservices/DistributedLocking.md) — Preventing duplicate execution
- [Message Queues](../../BuildingBlocks/MessageQueues.md) — Task queue infrastructure
- [Idempotency](../../APIs/Idempotency.md) — Exactly-once execution guarantee
- [Rate Limiting](../../BuildingBlocks/RateLimiting.md) — Throttling job execution rate

---

*"A job scheduler is easy to build and impossible to build correctly. The difference between 'runs at midnight' and 'runs exactly once at midnight even when servers crash' is about 10,000 lines of defensive code." — Distributed Systems Engineering* 🎬
