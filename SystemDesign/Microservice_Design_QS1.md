# Microservice Design Interview QS 1: Log Forwarded Service

## Question

Develop a service S on some cloud of your choice (like AWS, Azure, GCP, OCI). Functional requirements are following. Do High level and Low level design of it.

**Input:**

Reading log messages:

It has a location in an object store (for example S3) where lots of files are stored.
**Each log file is immutable (read only), it's not getting modified by any other service.**
It picks one log file at a time and then starts reading log messages from that file. It reads one log messages at a time.
Each log message starts with a new line and has format:
`<timestamp>-<service-name>-<uniqueId>-<category>:   <message-content>`
The log message header `<timestamp>-<service-name>-<uniqueId>-<category>` makes every log message unique across all files.

**Output:**

Service S has output in two paths:

Sending one single log messages (read by reader) to an external service A. (HealthCheck & POST)
It also accumulates log messages of 1 hr, computing some statistics and send it to external service B. (HealthCheck & POST)

**Important Requirement:**

The Service S should be scalable and fault tolerant.
If External services A or B slow down or goes offline, service S should adjust accordingly.
S should not incur data loss. No log message or 1-hour summary should be lost.

## High-Level Design (HLD)

### Architecture Overview
The HLD focuses on the overall architecture, data flow, and major components. It's cloud-native, using AWS for scalability and fault tolerance.

### Data Flow
1. New log files uploaded to S3 trigger an SQS message.
2. Workers (EC2/Lambda) dequeue, read file from S3, parse messages.
3. Each message: Sent to A via queue/buffer. Also added to accumulator.
4. At 1-hour boundaries: Compute stats, send to B via queue/buffer.

### Major Components
- **S3**: Immutable storage for logs.
- **SQS**: Decouples file discovery from processing; FIFO for ordering.
- **Workers**: Stateless, auto-scaling (EC2 ASG or Lambda).
- **Queues/Buffers**: SQS/Kinesis for A/B to handle backpressure.
- **Accumulator**: Redis for fast, in-memory aggregation.
- **State Store**: DynamoDB for checkpoints (e.g., last processed offset per file).
- **Monitoring**: CloudWatch for metrics/alerts.

### Diagram (Mermaid)
```mermaid
graph TD
    A[S3 Bucket (Log Files)] --> B[SQS Queue (File Processing)]
    B --> C[Auto-Scaling Workers (EC2/Lambda)]
    C --> D[Message Processor]
    D --> E[Queue to A]
    D --> F[In-Memory Accumulator (Redis)]
    E --> G[External Service A]
    F --> H[Queue to B]
    H --> I[External Service B]
```

### Scaling Strategy
- Horizontal scaling via ASG/Lambda. Partition by file hash if needed.

### Fault Tolerance
- Workers are stateless; state in DynamoDB. Queues buffer during outages. Circuit breakers for A/B.

## Low-Level Design (LLD)

### Component Details

1. **File Reader**:
   - **Implementation**: Use AWS SDK (boto3) to stream S3 objects. Read line-by-line using `boto3.s3.Object().get()['Body'].iter_lines()`.
   - **Checkpointing**: Store last processed byte offset in DynamoDB (key: file path; attributes: offset, lastTimestamp). Update after each message.
   - **Error Handling**: If read fails, retry with backoff. If file corrupted, log and skip (but alert).
   - **Concurrency**: One worker per file; no intra-file parallelism to maintain order.

2. **Message Parser**:
   - **Algorithm**: Split line by first `: ` to get header and content. Parse header with regex: `^(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}Z)-([^-]+)-([^-]+)-([^:]+): (.+)$`.
   - **Validation**: Check timestamp format, non-empty fields. Discard invalid messages (log to CloudWatch).
   - **Data Structure**: Use a dict: `{'timestamp': str, 'serviceName': str, 'uniqueId': str, 'category': str, 'content': str}`.

3. **Sender to A**:
   - **Implementation**: HTTP client (requests library) with retries (up to 3, exponential backoff). Use circuit breaker (e.g., pybreaker) to open if >50% failures in 1 min.
   - **Idempotency**: Include uniqueId in POST body; assume A handles duplicates.
   - **Buffering**: If A is slow, buffer in SQS (dead-letter queue for failures).
   - **Health Check**: Poll `/health` every 30s; disable sending if unhealthy.

4. **Accumulator for B**:
   - **Data Structure**: Redis hash per hour window (key: `stats:<windowStart>`; fields: `total`, `categories:<cat>`, etc.).
   - **Algorithm**: On message receipt, increment counters in Redis. Use sorted sets for timestamps if needed.
   - **Window Management**: Use a scheduler (e.g., APScheduler) to trigger every hour: Compute final stats, send to B, clear Redis.
   - **Fault Tolerance**: Persist partial aggregates to DynamoDB if Redis fails.

5. **Sender to B**:
   - **Implementation**: Similar to A: HTTP client with retries/circuit breaker. Buffer in SQS if B is down.
   - **Stats Computation**: Aggregate from Redis: Total messages, category counts, etc.
   - **Idempotency**: Include windowStart/windowEnd; assume B handles duplicates.

6. **State Management**:
   - **DynamoDB Table**: `ProcessingState` (partition key: filePath; sort key: component; attributes: offset, lastProcessed).
   - **Consistency**: Use strongly consistent reads for checkpoints.

7. **Error Handling & Fault Tolerance**:
   - **Retries**: Exponential backoff for all external calls.
   - **Circuit Breaker**: For A/B; fallback to queue.
   - **Dead Letters**: SQS DLQ for unprocessable messages.
   - **Monitoring**: CloudWatch alarms for queue depth, error rates. Logs to CloudWatch Logs.

8. **Security & Deployment**:
   - **IAM**: Least-privilege roles for S3/DynamoDB access.
   - **Encryption**: TLS for all HTTP; S3 SSE.
   - **Deployment**: Terraform/CDK for infra. Docker for workers if EC2.

This design ensures scalability (auto-scale workers), fault tolerance (stateless workers, durable queues), and no data loss (checkpoints, buffers). If external services fail, messages queue up without blocking.

## Recommendations for Preparation
- **Books**: "Designing Data-Intensive Applications" by Martin Kleppmann; "System Design Interview – An Insider's Guide" by Alex Xu.
- **Tutorials**: Grokking the System Design Interview (Educative); System Design Primer (GitHub).
- **Practice**: Draw diagrams, explain trade-offs aloud.