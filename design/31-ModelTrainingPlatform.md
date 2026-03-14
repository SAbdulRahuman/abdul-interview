# Design a Model Training Platform

## Overview
Design a distributed ML model training platform similar to Kubeflow, SageMaker, or Vertex AI — supporting distributed training across GPU clusters, dataset pipelines, experiment tracking, and model registry.

## 1. Requirements

**Functional:**
- Submit training jobs with code, data, and hyperparameters
- Distributed training (data parallel, model parallel, pipeline parallel)
- Dataset management and preprocessing pipelines
- Experiment tracking (metrics, parameters, artifacts)
- Hyperparameter tuning (grid, random, Bayesian)
- Model registry with versioning and promotion staging

**Non-Functional:**
- Support 100+ concurrent training jobs
- Efficient GPU utilization (>80% average)
- Fault-tolerant training (checkpoint and resume)
- Support models from 1M to 400B+ parameters

## 2. Scale Estimation

```
Concurrent training jobs:      100
GPUs in cluster:               500 (mix of A100, H100)
Dataset storage:               500TB (S3/GCS)
Experiments tracked/day:       1,000
Model versions registered/day: 50
Avg training job duration:     2-48 hours
GPU hours/month:               360,000
```

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     Training Platform                        │
│                                                              │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│  │ Job Scheduler  │  │ Experiment     │  │ Model Registry  │  │
│  │ (K8s + custom) │  │ Tracker        │  │ (MLflow/custom) │  │
│  └───────┬───────┘  │ (MLflow/W&B)   │  └─────────────────┘  │
│          │          └───────────────┘                        │
│  ┌───────┴──────────────────────────────────────────────┐   │
│  │              GPU Cluster Manager                      │   │
│  │   ┌──────────┐ ┌──────────┐ ┌──────────┐            │   │
│  │   │ Node 1   │ │ Node 2   │ │ Node N   │            │   │
│  │   │ 8×H100   │ │ 8×H100   │ │ 8×A100   │            │   │
│  │   │ NVLink   │ │ NVLink   │ │ NVLink   │            │   │
│  │   └──────────┘ └──────────┘ └──────────┘            │   │
│  │        InfiniBand / RoCE fabric (400Gbps)            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│  │ Data Pipeline  │  │ Feature Store  │  │ Artifact Store  │  │
│  │ (Spark/Ray)   │  │ (Feast)        │  │ (S3 + metadata) │  │
│  └───────────────┘  └───────────────┘  └─────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## 4. Distributed Training Strategies

```
Data Parallelism (most common):
┌──────────────────────────────────────────────────┐
│  Same model replicated across N GPUs             │
│                                                  │
│  GPU 0: model_copy + data_shard_0               │
│  GPU 1: model_copy + data_shard_1               │
│  GPU 2: model_copy + data_shard_2               │
│  GPU 3: model_copy + data_shard_3               │
│                                                  │
│  Each step:                                      │
│  1. Forward pass (independent per GPU)           │
│  2. Backward pass (independent per GPU)          │
│  3. All-reduce gradients (NCCL ring)             │
│  4. Update weights (identical on all GPUs)        │
│                                                  │
│  Effective batch = per_gpu_batch × N_gpus        │
│  Linear speedup up to ~256 GPUs                  │
└──────────────────────────────────────────────────┘

FSDP (Fully Sharded Data Parallel) for large models:
┌──────────────────────────────────────────────────┐
│  Model weights sharded across GPUs               │
│  Each GPU holds 1/N of parameters                │
│                                                  │
│  GPU 0: params[0:25%] + full data batch          │
│  GPU 1: params[25:50%] + full data batch         │
│  GPU 2: params[50:75%] + full data batch         │
│  GPU 3: params[75:100%] + full data batch        │
│                                                  │
│  Each layer:                                     │
│  1. All-gather params for current layer          │
│  2. Forward/backward on full layer               │
│  3. Discard non-owned params (reduce memory)     │
│  4. Reduce-scatter gradients                     │
│                                                  │
│  Memory: O(params/N) vs O(params) for DDP        │
│  Enables 70B model on 4×A100 (80GB each)         │
└──────────────────────────────────────────────────┘

Pipeline Parallelism (400B+ models):
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Node 1   │ →  │ Node 2   │ →  │ Node 3   │ →  │ Node 4   │
│ Layers   │    │ Layers   │    │ Layers   │    │ Layers   │
│ 0-19     │    │ 20-39    │    │ 40-59    │    │ 60-79    │
│ 8×H100   │    │ 8×H100   │    │ 8×H100   │    │ 8×H100   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
  Micro-batching: split batch into 4 micro-batches
  Pipeline bubble: ~25% overhead (1F1B schedule reduces to ~15%)
```

## 5. Training Job Lifecycle

```
Job Submission → Scheduling → Execution → Completion

┌──────────────────────────────────────────────────────────┐
│ 1. SUBMITTED                                            │
│    User submits: code, config, dataset ref, GPU request │
│    Validation: config schema, resource availability     │
├──────────────────────────────────────────────────────────┤
│ 2. QUEUED                                               │
│    Priority queue: sorted by priority × wait_time       │
│    Gang scheduling: all N GPUs must be available         │
│    Preemption: low-pri jobs preempted for high-pri       │
├──────────────────────────────────────────────────────────┤
│ 3. PROVISIONING                                         │
│    Allocate GPU nodes (K8s pod scheduling)               │
│    Pull container image (cached on nodes)                │
│    Mount dataset (S3 FUSE or pre-cached on NVMe)        │
│    Setup NCCL communication (discover peers)             │
├──────────────────────────────────────────────────────────┤
│ 4. RUNNING                                              │
│    Training loop: forward → backward → optimize          │
│    Report metrics every N steps (loss, lr, throughput)   │
│    Checkpoint every M steps (to S3)                      │
│    Health check: GPU utilization, loss NaN detection     │
├──────────────────────────────────────────────────────────┤
│ 5. COMPLETED / FAILED                                    │
│    Save final model artifact to Model Registry           │
│    Log final metrics to Experiment Tracker               │
│    Release GPU resources                                 │
│    If failed: keep last checkpoint for resume             │
└──────────────────────────────────────────────────────────┘
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Submit Training Job
POST /api/v1/jobs
{
  "name": "llama-finetune-v3",
  "image": "registry.example.com/training:v2.1",
  "config": {
    "model": "meta-llama/Llama-3.1-70B",
    "dataset": "s3://datasets/instruction-tuning-v5/",
    "epochs": 3,
    "batch_size_per_gpu": 4,
    "learning_rate": 2e-5,
    "strategy": "fsdp",          // ddp | fsdp | deepspeed_zero3 | pipeline
    "precision": "bf16"
  },
  "resources": {
    "gpu_type": "H100",
    "gpu_count": 32,
    "cpu_per_node": 96,
    "memory_gb_per_node": 512
  },
  "checkpoint_interval_steps": 500,
  "max_runtime_hours": 48,
  "priority": "high"
}
Response: {
  "job_id": "job_xyz789",
  "status": "queued",
  "estimated_start": "2024-01-15T14:00:00Z"
}

# Get Job Status
GET /api/v1/jobs/{job_id}
Response: {
  "job_id": "job_xyz789",
  "status": "running",
  "progress": {"epoch": 1, "step": 2450, "total_steps": 7500},
  "metrics": {"loss": 0.342, "lr": 1.8e-5, "tokens_per_sec": 125000},
  "gpu_utilization": 0.87,
  "elapsed_hours": 8.2,
  "checkpoints": ["s3://checkpoints/job_xyz789/step_2000/"]
}

# Register Model
POST /api/v1/models
{
  "name": "llama-3.1-70b-instruct-v3",
  "source_job_id": "job_xyz789",
  "artifact_path": "s3://models/llama-70b-v3/",
  "stage": "staging",           // staging | canary | production | archived
  "metadata": {
    "base_model": "meta-llama/Llama-3.1-70B",
    "training_tokens": 1000000000,
    "eval_metrics": {"mmlu": 0.82, "humaneval": 0.73}
  }
}
```

### Job Scheduler Internals

```
Gang Scheduler (GPU allocation):
┌──────────────────────────────────────────────────────────┐
│ Priority Queue:                                         │
│   [job_1: 32×H100, priority=HIGH, wait=2h]              │
│   [job_2: 8×A100, priority=MED, wait=30m]               │
│   [job_3: 4×H100, priority=LOW, wait=5m]                │
│                                                          │
│ Scheduling Algorithm:                                    │
│   1. Sort by: priority DESC, wait_time DESC              │
│   2. For top job: find N contiguous GPUs                 │
│      - Topology-aware: prefer same InfiniBand switch    │
│      - NUMA-aware: prefer GPUs on same NUMA node        │
│   3. If N GPUs available → ALLOCATE                      │
│   4. If not → check if preemption possible               │
│      Preempt job where: priority < requesting job        │
│      AND checkpointed within 10 min                     │
│   5. Preempted job: save checkpoint → re-queue          │
│                                                          │
│ Topology-Aware Placement:                                │
│   Prefer: same node > same switch > same rack            │
│   Reason: NVLink (900GB/s) >> InfiniBand (400Gb/s)      │
│           >> cross-rack (100Gb/s)                         │
└──────────────────────────────────────────────────────────┘
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **GPU Count** | K8s cluster auto-scaling; spot/preemptible GPU nodes | 1000+ GPUs |
| **Job Concurrency** | Gang scheduler with priority queues; preemption | 100+ concurrent |
| **Dataset Size** | Streaming from S3; no full download; ShardedDataLoader | Petabyte-scale |
| **Experiment Tracking** | Time-series DB for metrics (VictoriaMetrics); S3 for artifacts | Millions of runs |
| **Model Registry** | PostgreSQL + S3; versioned artifacts | 100K model versions |

## 8. No Data Loss

| Component | Protection Mechanism |
|-----------|---------------------|
| **Training State** | Periodic checkpoints to S3 (every 500 steps); includes optimizer state |
| **Model Weights** | Final model saved to S3 with versioning; SHA-256 checksums |
| **Experiment Metrics** | Streamed to time-series DB with WAL; buffered locally if DB unavailable |
| **Datasets** | Immutable in S3; versioned with DVC or Delta Lake |
| **Job Config** | Stored in PostgreSQL before execution; reproducible training runs |

## 9. Latency

| Operation | p50 | p99 | Target |
|-----------|-----|-----|--------|
| Job submission to start (queued) | 30s | 10min | Depends on GPU availability |
| Checkpoint save (70B model) | 2min | 5min | <10min |
| Checkpoint restore | 3min | 8min | <10min |
| Metric reporting | 100ms | 500ms | <1s |
| Model registration | 200ms | 1s | <5s |

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **GPU failure (mid-training)** | Job crashes | Resume from last checkpoint on replacement GPU; elastic training (reduce world_size) |
| **NaN loss** | Training diverged | Auto-detect NaN; rollback to last good checkpoint; reduce learning rate |
| **Node eviction (spot)** | Job interrupted | Checkpoint on SIGTERM (30s grace period); re-queue with priority boost |
| **NCCL timeout** | Distributed training stuck | Watchdog: kill and restart if no progress for 10min |
| **Dataset corruption** | Bad training data | Checksum validation; data validation pipeline before training |

## 11. Availability

**Target: 99.9% for job submission, 99% for GPU availability**

```
Cluster Resilience:
  - Multi-AZ GPU nodes (survive AZ failure)
  - Spot + on-demand mix (70% spot + 30% on-demand for baseline)
  - Preemption for high-priority jobs
  - Job queue persists if scheduler restarts (PostgreSQL-backed)
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Multi-tenancy** | Namespace isolation per team; GPU quotas per namespace |
| **Data Access** | IAM roles per job; least-privilege S3 access |
| **Model Protection** | Encrypted model artifacts; access control on model registry |
| **Code Execution** | Container isolation; no host access; read-only code volumes |
| **Network** | GPU cluster on private network; no public ingress to training nodes |
| **Secrets** | Training API keys via K8s secrets + Vault; never in job config |

## 13. Cost Constraints

**Estimated Cost (500 GPUs, mix H100/A100):**

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| **On-demand H100** | 100× H100 (p5.48xlarge) | $800,000 |
| **Spot H100** | 200× H100 (70% discount) | $480,000 |
| **Spot A100** | 200× A100 (70% discount) | $240,000 |
| **Storage (S3)** | 500TB datasets + checkpoints | $11,500 |
| **Networking** | InfiniBand fabric | Included |
| **Platform Services** | Scheduler, tracker, registry | $5,000 |
| **Total** | | **~$1,536,500/month** |

**Cost Optimization:**
- **Spot instances**: 70% savings; checkpoint frequently for spot tolerance
- **Elastic training**: Reduce GPU count mid-job if spots reclaimed (instead of crash)
- **Efficient checkpointing**: Async checkpoint to S3; don't block training loop
- **Mixed precision (bf16)**: 2× throughput, same quality → halve GPU hours
- **Data loading optimization**: Avoid GPU idle time waiting for data (pre-fetch + cache)

## Key Interview Discussion Points

1. **DDP vs FSDP vs DeepSpeed?** — DDP: replicate model on each GPU (simple, model must fit). FSDP: shard model params (memory efficient). DeepSpeed ZeRO: progressive sharding (Stage 1/2/3)
2. **How to handle GPU failures?** — Checkpoint frequently; elastic training (adjust world_size); SIGTERM handlers for spot preemption
3. **Gang scheduling challenges?** — Must allocate all GPUs atomically; fragmentation wastes resources; topology-aware placement critical for communication
4. **How to optimize data loading?** — Streaming from S3; multi-worker pre-fetch; cache on NVMe; overlap data loading with compute
5. **Model registry workflow?** — staging → canary (shadow traffic eval) → production → archived; automated promotion based on eval metrics
