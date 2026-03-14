# Design a Job Scheduling System

Examples: Kubernetes Scheduler, Apache Airflow, Temporal, Celery

---

## 1. Requirements

### Functional
- Submit jobs with execution requirements (CPU, memory, dependencies)
- Schedule jobs based on time (cron), events, or dependencies (DAG)
- Execute jobs on distributed workers
- Retry failed jobs with configurable policies
- Track job status (pending, running, succeeded, failed)
- Support job priorities and preemption

### Non-Functional
- Exactly-once execution (no duplicate runs)
- Sub-second scheduling latency for high-priority jobs
- Scalable to millions of jobs per day
- Fault tolerant (scheduler and worker failures)
- Observability (job history, logs, metrics)

---

## 2. High-Level Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                    Control Plane                          │
  │                                                          │
  │  ┌──────────┐    ┌──────────────┐    ┌──────────────┐   │
  │  │  API      │    │  Scheduler   │    │  Job State   │   │
  │  │  Server   │───▶│  Engine      │───▶│  Store       │   │
  │  │          │    │  (priority   │    │  (PostgreSQL │   │
  │  │  submit, │    │   queue,     │    │   or etcd)   │   │
  │  │  cancel, │    │   DAG eval)  │    │              │   │
  │  │  status  │    └──────┬───────┘    └──────────────┘   │
  │  └──────────┘           │                                │
  │                         │                                │
  └─────────────────────────┼────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Worker 1 │  │ Worker 2 │  │ Worker 3 │
        │ (CPU)    │  │ (GPU)    │  │ (CPU)    │
        │          │  │          │  │          │
        │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │
        │ │Job A │ │  │ │Job C │ │  │ │Job E │ │
        │ │Job B │ │  │ │Job D │ │  │ │Job F │ │
        │ └──────┘ │  │ └──────┘ │  │ └──────┘ │
        └──────────┘  └──────────┘  └──────────┘
              │             │             │
              └─────────────┼─────────────┘
                            │
                   ┌────────▼────────┐
                   │  Message Queue   │
                   │  (Kafka / Redis) │
                   │  (job events)    │
                   └─────────────────┘
```

---

## 3. DAG-Based Job Dependencies

```
  Job DAG (Directed Acyclic Graph):

        ┌─────┐
        │ ETL │  (start)
        │Extract│
        └──┬──┘
           │
     ┌─────┴─────┐
     ▼           ▼
  ┌──────┐   ┌──────┐
  │Clean │   │Clean │
  │Data A│   │Data B│
  └──┬───┘   └──┬───┘
     │           │
     └─────┬─────┘
           ▼
     ┌──────────┐
     │Transform │
     │& Merge   │
     └────┬─────┘
          │
     ┌────┴────┐
     ▼         ▼
  ┌──────┐  ┌──────┐
  │Load  │  │Send  │
  │to DW │  │Report│
  └──────┘  └──────┘

  Scheduling rule:
  • A job runs only when ALL upstream dependencies succeed
  • If "Clean Data A" fails → "Transform" is blocked
  • Retry "Clean Data A" up to 3 times before marking DAG as failed
```

---

## 4. Scheduler Engine Design

```
  ┌────────────────────────────────────────────────┐
  │              Scheduler Loop                     │
  │                                                │
  │  while true:                                   │
  │    1. Check cron triggers → enqueue due jobs   │
  │    2. Check event triggers → enqueue matching  │
  │    3. Evaluate DAGs → find ready tasks         │
  │    4. Priority queue → pick highest priority   │
  │    5. Resource matching:                       │
  │       job.requirements ≤ worker.available?     │
  │    6. Assign job to worker                     │
  │    7. Update state: PENDING → SCHEDULED        │
  │    8. Sleep(100ms)                             │
  │                                                │
  │  Resource matching (bin-packing):              │
  │  ┌───────────────────────────────┐             │
  │  │ Job needs: 2 CPU, 4GB RAM    │             │
  │  │ Worker 1:  free 1 CPU, 8GB   │ ✗           │
  │  │ Worker 2:  free 4 CPU, 16GB  │ ✓ (pick)    │
  │  │ Worker 3:  free 2 CPU, 2GB   │ ✗           │
  │  └───────────────────────────────┘             │
  └────────────────────────────────────────────────┘
```

---

## 5. Job Lifecycle

```
  ┌──────────┐
  │ SUBMITTED│
  └────┬─────┘
       │
  ┌────▼─────┐    dependencies met?
  │ PENDING  │◀─── no ──┐
  └────┬─────┘          │
       │ yes             │
  ┌────▼─────┐          │
  │SCHEDULED │          │
  └────┬─────┘          │
       │ worker picks up │
  ┌────▼─────┐          │
  │ RUNNING  │          │
  └────┬─────┘          │
       │                │
  ┌────┴──────────┐     │
  ▼               ▼     │
┌────────┐  ┌────────┐  │
│SUCCEEDED│  │ FAILED │──┘ (retry? → back to PENDING)
└────────┘  └────┬───┘
                 │ max retries exceeded
            ┌────▼────┐
            │  DEAD   │
            └─────────┘
```

---

## 6. Retry Policies

```
  ┌──────────────────────────────────────────────┐
  │  Retry Configuration per Job:                │
  │                                              │
  │  max_retries: 3                              │
  │  retry_delay: exponential backoff            │
  │    attempt 1: wait 1s                        │
  │    attempt 2: wait 4s                        │
  │    attempt 3: wait 16s                       │
  │  retry_on: [TIMEOUT, OOM, TRANSIENT_ERROR]   │
  │  no_retry_on: [INVALID_INPUT, AUTH_FAILURE]  │
  │                                              │
  │  Dead Letter Queue:                          │
  │  After max_retries → move to DLQ for manual  │
  │  investigation                               │
  └──────────────────────────────────────────────┘
```

---

## 7. Exactly-Once Execution

```
  Problem: Scheduler assigns job to Worker A
           Worker A crashes mid-execution
           Scheduler reassigns to Worker B
           Worker A recovers → both running the same job!

  Solution: Fencing tokens + lease-based locking

  ┌──────────────────────────────────────────────┐
  │  1. Scheduler assigns job with fence_token=5 │
  │  2. Worker acquires lease (30s TTL)          │
  │  3. Worker sends heartbeats to extend lease  │
  │  4. If no heartbeat for 30s → lease expires  │
  │  5. Scheduler reassigns with fence_token=6   │
  │  6. Worker A (token=5) tries to write result │
  │     → State store rejects (token < current)  │
  │  7. Only Worker B (token=6) can write        │
  └──────────────────────────────────────────────┘
```

---

## 8. Scheduler High Availability

```
  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
  │ Scheduler 1 │    │ Scheduler 2 │    │ Scheduler 3 │
  │  (LEADER)   │    │ (STANDBY)   │    │ (STANDBY)   │
  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
         │                  │                  │
         └──────────┬───────┘──────────────────┘
                    │
              ┌─────▼─────┐
              │   etcd     │  ← leader election
              │  (Raft)    │  ← job state
              └────────────┘

  • Only the leader scheduler processes the queue
  • Standby instances monitor leader health
  • On leader failure → new leader elected (< 5s)
  • All state in etcd → new leader picks up seamlessly
```

---

## 9. Scaling Strategy

- **Worker scaling:** Auto-scale worker pool based on queue depth
- **Scheduler sharding:** Shard by job namespace or tenant
- **Queue partitioning:** Separate queues for different priorities (critical, default, batch)
- **Multi-cluster:** Regional job execution with global orchestration
- **Resource pools:** Dedicated pools for GPU, high-memory, or latency-sensitive jobs

---

## 10. Observability

- **Metrics:** Job throughput, queue depth, scheduling latency, execution duration, retry rate, failure rate
- **Dashboards:** DAG visualization, job timeline (Gantt chart), resource utilization
- **Alerts:** Job failure rate > threshold, queue depth growing, scheduler lag
- **Audit log:** Who submitted what job, when, with what parameters

---

---

## 11. Low-Level Design (LLD)

### API Contract
```
  Job Management:
  POST   /api/v1/jobs
         Body: { "name": str, "image": str, "command": [str],
                 "resources": { "cpu": 2, "memory_gb": 4, "gpu": 0 },
                 "schedule": "0 */6 * * *" | null,
                 "dependencies": ["job-id-1", "job-id-2"],
                 "retry_policy": { "max_retries": 3, "backoff": "exponential" },
                 "priority": "critical|high|default|low",
                 "timeout_seconds": 3600 }
         Response: { "job_id": uuid, "status": "SUBMITTED" }

  GET    /api/v1/jobs/{job_id}
         Response: { "job_id": uuid, "status": str, "attempts": int,
                     "started_at": ts, "finished_at": ts, "worker_id": str }

  DELETE /api/v1/jobs/{job_id}   (cancel)
  POST   /api/v1/jobs/{job_id}/retry  (manual retry)

  DAG Management:
  POST   /api/v1/dags
         Body: { "name": str, "tasks": [...], "edges": [[from,to],...] }
  GET    /api/v1/dags/{dag_id}/status
         Response: { "dag_id": str, "state": str, "tasks": [{ status per task }] }
```

### Database Schema
```
  ┌────────────────────────────────────────────────────────┐
  │  jobs:                                                 │
  │    id (PK), name, image, command, resources (JSON),    │
  │    schedule (cron), priority, status, dag_id (FK),     │
  │    fence_token, created_at, updated_at                 │
  │                                                        │
  │  job_attempts:                                         │
  │    id (PK), job_id (FK), attempt_num, worker_id,       │
  │    started_at, finished_at, exit_code, error_msg       │
  │                                                        │
  │  dag_edges:                                            │
  │    dag_id, from_job_id, to_job_id                      │
  │                                                        │
  │  worker_leases:                                        │
  │    worker_id (PK), job_id, fence_token,                │
  │    lease_expiry, heartbeat_at                          │
  └────────────────────────────────────────────────────────┘
```

### Scheduler Loop (detailed)
```
  Every 100ms:
  1. Cron evaluator: check all cron-scheduled jobs
     → if due: insert new job instance (status=PENDING)
  2. DAG evaluator: for each active DAG
     → topological sort, check if all parents SUCCEEDED
     → mark ready tasks as PENDING
  3. Priority queue dequeue:
     → SELECT jobs WHERE status='PENDING'
       ORDER BY priority DESC, created_at ASC
       LIMIT 100 FOR UPDATE SKIP LOCKED
  4. Resource matching:
     → for each pending job, find worker with
       available_cpu ≥ job.cpu AND available_mem ≥ job.mem
  5. Assign: update job status=SCHEDULED, set worker_id, fence_token++
  6. Dispatch: send job spec to worker via gRPC
```

---

## 12. Scalability

```
  ┌────────────────────────────────────────────────────────┐
  │  Scheduling Throughput:                                │
  │  • Single scheduler: ~10K jobs/min scheduling rate     │
  │  • Sharded schedulers: partition by namespace/tenant   │
  │  • 10 scheduler shards → 100K jobs/min                │
  │                                                        │
  │  Worker Scaling:                                       │
  │  • Auto-scale based on pending queue depth             │
  │  • Queue depth > 100 for 5 min → add workers          │
  │  • Queue depth = 0 for 10 min → remove workers        │
  │  • Kubernetes HPA or cloud auto-scaling groups         │
  │  • Spot/preemptible instances for fault-tolerant jobs  │
  │                                                        │
  │  State Store Scaling:                                  │
  │  • PostgreSQL: partition job table by created_at       │
  │  • Archive completed jobs after 30 days                │
  │  • Index on (status, priority) for scheduler queries   │
  │  • Read replicas for status queries / dashboards       │
  │                                                        │
  │  Multi-Region:                                         │
  │  • Regional scheduler + workers for data locality      │
  │  • Global orchestrator for cross-region DAGs           │
  │  • Follow-the-sun scheduling for time-zone jobs        │
  └────────────────────────────────────────────────────────┘
```

---

## 13. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Job State Durability:                                  │
  │  • All state changes persisted to PostgreSQL/etcd       │
  │    BEFORE acknowledging to client                       │
  │  • WAL-based replication to standby DB (sync)           │
  │  • Every state transition is atomic:                    │
  │    BEGIN; UPDATE job SET status='RUNNING'; INSERT attempt; COMMIT;│
  │                                                         │
  │  Job Output Preservation:                               │
  │  • Worker streams stdout/stderr to object store (S3)    │
  │  • Job artifacts uploaded before reporting completion   │
  │  • Checksummed uploads for integrity                    │
  │                                                         │
  │  Crash Recovery Scenarios:                              │
  │  • Scheduler crash: new leader reads state from DB;     │
  │    re-evaluates all PENDING/SCHEDULED jobs              │
  │  • Worker crash mid-job: lease expires → scheduler      │
  │    re-queues with new fence_token                       │
  │  • DB crash: failover to synchronous standby;           │
  │    zero data loss (RPO=0)                               │
  │                                                         │
  │  Idempotent Job Design:                                 │
  │  • Jobs should be re-runnable (write to temp, then move)│
  │  • Job ID embedded in output path for dedup             │
  │  • Fence token prevents stale worker writes             │
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Latency

```
  Latency Targets:
  ┌─────────────────────────┬──────────┬──────────┐
  │ Operation               │ p50      │ p99      │
  ├─────────────────────────┼──────────┼──────────┤
  │ Job submission          │ 10 ms    │ 50 ms    │
  │ Scheduling (to assign)  │ 100 ms   │ 500 ms   │
  │ Job start (on worker)   │ 1 s      │ 5 s      │
  │ Status query            │ 5 ms     │ 20 ms    │
  └─────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  Fast Scheduling:                                       │
  │  • In-memory priority queue for hot path                │
  │  • Pre-computed resource availability per worker        │
  │  • Skip DB poll: event-driven (new job → notify sched) │
  │                                                         │
  │  Fast Job Start:                                        │
  │  • Pre-pulled container images on workers               │
  │  • Warm worker pools for common job types               │
  │  • Pre-forked worker processes (avoid cold start)       │
  │                                                         │
  │  Fast DAG Evaluation:                                   │
  │  • Cache DAG topology in memory                         │
  │  • Incremental evaluation (only re-check affected tasks)│
  │  • Batch ready-task determination per DAG               │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Scheduler crash        │ Leader election (etcd/Raft);    │
  │                        │ standby takes over in < 5s      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Worker crash mid-job   │ Lease timeout → reschedule with │
  │                        │ fence_token; retry on new worker│
  ├────────────────────────┼─────────────────────────────────┤
  │ Worker OOM             │ cgroups memory limits; kill job │
  │                        │ → retry on larger worker        │
  ├────────────────────────┼─────────────────────────────────┤
  │ Cascading DAG failure  │ Configurable: fail-fast (stop   │
  │                        │ DAG) or continue-on-error       │
  ├────────────────────────┼─────────────────────────────────┤
  │ Poison job (always     │ Max retries → DLQ; alert ops;   │
  │ crashes)               │ skip and continue DAG           │
  ├────────────────────────┼─────────────────────────────────┤
  │ Clock skew (cron)      │ Use UTC everywhere; NTP sync;   │
  │                        │ scheduler uses monotonic clock  │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 16. Availability

```
  Target: 99.95% (scheduler) — jobs are durable, brief unavail OK

  ┌─────────────────────────────────────────────────────────┐
  │  Scheduler HA:                                          │
  │  • 3 scheduler instances (1 active, 2 standby)          │
  │  • Leader election via etcd lease                       │
  │  • Failover time: < 5 seconds                           │
  │  • No jobs lost: all state in persistent store           │
  │                                                         │
  │  Worker Pool HA:                                        │
  │  • Workers spread across multiple AZs                    │
  │  • Min pool size maintained even during scale-down       │
  │  • Spot instance preemption → graceful job migration     │
  │  • Health check: CPU/mem/disk; unhealthy → drain & evict│
  │                                                         │
  │  State Store HA:                                        │
  │  • PostgreSQL primary + sync standby (zero RPO failover)│
  │  • etcd: 3-node or 5-node cluster (Raft consensus)      │
  │  • Automated failover with < 10s RTO                     │
  │                                                         │
  │  Graceful Degradation:                                  │
  │  • Scheduler down → pending jobs queue, no data lost     │
  │  • Workers down → jobs re-queued to surviving workers    │
  │  • DB read replica down → use primary for reads          │
  └─────────────────────────────────────────────────────────┘
```

---

## 17. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Authentication & Authorization:                        │
  │  • mTLS between scheduler ↔ workers                    │
  │  • RBAC: who can submit, cancel, view jobs              │
  │  • Namespace isolation: team A can't see team B's jobs  │
  │  • Service accounts for automated DAG submissions       │
  │                                                         │
  │  Job Isolation:                                         │
  │  • Container-based execution (namespace, cgroup)         │
  │  • Network policies: restrict job network access         │
  │  • Read-only filesystem (except designated output dir)   │
  │  • No privileged containers by default                   │
  │  • Resource limits enforced (CPU, memory, disk)          │
  │                                                         │
  │  Secrets Management:                                     │
  │  • Job secrets injected via Vault / KMS at runtime       │
  │  • Never stored in job spec or logs                      │
  │  • Ephemeral credentials rotated per job execution       │
  │                                                         │
  │  Audit:                                                  │
  │  • Full audit trail: who submitted what, when, outcome   │
  │  • Tamper-proof audit log (append-only, signed)          │
  └─────────────────────────────────────────────────────────┘
```

---

## 18. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Compute (largest cost):                                │
  │  • Spot instances for batch/fault-tolerant jobs: 70% off│
  │  • Right-size: match worker type to job requirements    │
  │  • Bin-packing: maximize utilization per worker          │
  │  • Idle workers auto-terminated after timeout            │
  │                                                         │
  │  Scheduling Intelligence:                               │
  │  • Priority-based preemption: low-pri on cheap instances│
  │  • Time-of-day scheduling: run batch jobs off-peak      │
  │  • Resource quotas per team/namespace                    │
  │                                                         │
  │  Storage:                                                │
  │  • Job logs: S3 (cheap) with lifecycle policy            │
  │  • Job state: PostgreSQL (small, < 100 GB)               │
  │  • Archive old job records after 90 days                  │
  │                                                         │
  │  Cost Estimate (10K jobs/day):                           │
  │  • Workers: ~$5K/month (mix of spot + on-demand)        │
  │  • Scheduler: ~$500/month (3 small instances)            │
  │  • State store: ~$300/month (PostgreSQL RDS)             │
  │  • Total: ~$6K/month                                     │
  │  • With spot optimization: ~$3K/month (50% savings)     │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **How do you prevent duplicate execution?** — Fencing tokens + lease-based locking + idempotent job logic
2. **How do you handle a worker that's stuck?** — Heartbeat timeout → kill and reschedule; memory/CPU limits with cgroups
3. **How do you prioritize jobs?** — Multi-level priority queues; preemption for critical jobs (evict lower-priority)
4. **Airflow vs Kubernetes scheduler?** — Airflow: DAG-based workflows, data pipelines. K8s: container scheduling, resource bin-packing
5. **How do you scale to millions of jobs/day?** — Partition scheduler by namespace, scale workers elastically, use efficient state store
