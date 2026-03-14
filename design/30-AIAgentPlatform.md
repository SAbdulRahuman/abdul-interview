# Design an AI Agent Platform

## Overview
Design a platform that enables AI agents to autonomously execute multi-step tasks by orchestrating LLM reasoning, tool execution, memory retrieval, and planning — similar to LangChain, CrewAI, or AutoGPT production deployments.

## 1. Requirements

**Functional:**
- Agent receives a goal, decomposes into subtasks, executes tools, and returns results
- Support tool orchestration (APIs, code execution, web search, file operations)
- Short-term memory (conversation context) and long-term memory (vector store)
- Planning engine: ReAct, function calling, chain-of-thought
- Multi-agent collaboration (team of specialized agents)
- Human-in-the-loop approval for high-risk actions

**Non-Functional:**
- Task completion latency: <30s for simple tasks, <5min for complex workflows
- Support 10K concurrent agent sessions
- Fault-tolerant execution (resume from checkpoint on failure)
- Audit trail for every action taken

## 2. Scale Estimation

```
Concurrent sessions:     10,000
Avg steps per task:      8
LLM calls per step:      1-2
Tool calls per step:      1
Total LLM calls/sec:     ~2,000
Tool executions/sec:      ~1,000
Memory lookups/sec:       ~5,000
Avg task duration:        30s - 5min
```

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        API Gateway                           │
│               (REST + WebSocket for streaming)               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Agent        │  │  Planning     │  │  Memory           │  │
│  │  Orchestrator │  │  Engine       │  │  Service          │  │
│  │              │  │  (ReAct/CoT)  │  │  (Short+Long term)│  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────────┘  │
│         │                 │                  │              │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────────┐  │
│  │  Tool         │  │  LLM          │  │  Vector           │  │
│  │  Registry &   │  │  Gateway      │  │  Database         │  │
│  │  Executor     │  │  (Multi-model)│  │  (Qdrant/Pinecone)│  │
│  └──────┬───────┘  └──────────────┘  └──────────────────┘  │
│         │                                                    │
│  ┌──────┴──────────────────────────────────────────────┐    │
│  │              Tool Sandbox (isolated execution)       │    │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────────┐   │    │
│  │  │Web API │ │Code    │ │File    │ │Database    │   │    │
│  │  │Caller  │ │Runner  │ │Manager │ │Query       │   │    │
│  │  └────────┘ └────────┘ └────────┘ └────────────┘   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐      │
│  │           Workflow State Store (PostgreSQL)         │      │
│  │    Task states, checkpoints, audit logs             │      │
│  └────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

## 4. Agent Execution Loop (ReAct Pattern)

```
ReAct (Reasoning + Acting) Loop:

User Goal: "Find the cheapest flight from SFO to NYC next Friday"

Step 1 — THINK:
  "I need to search for flights. Let me use the flight search tool."
Step 1 — ACT:
  Tool: flight_search(from="SFO", to="NYC", date="2024-01-19")
Step 1 — OBSERVE:
  Result: [{airline: "United", price: $250}, {airline: "Delta", price: $220}]

Step 2 — THINK:
  "Delta at $220 is cheapest. Let me check if there are budget airlines too."
Step 2 — ACT:
  Tool: flight_search(from="SFO", to="NYC", date="2024-01-19", carriers=["Spirit","Frontier"])
Step 2 — OBSERVE:
  Result: [{airline: "Spirit", price: $89, layover: "DEN"}]

Step 3 — THINK:
  "Spirit at $89 is cheapest but has a layover. I should present both options."
Step 3 — ACT:
  Tool: respond_to_user(message="Found options: Spirit $89 (1 stop) or Delta $220 (nonstop)")

State Machine:
  PLANNING ──think──→ REASONING ──act──→ EXECUTING ──observe──→ REASONING
       │                                                            │
       │                                                   (goal met)
       │                                                            ▼
       └────────────────────────────────────────────────────── COMPLETED
```

## 5. Tool Registry & Execution

```
Tool Definition Schema:
{
  "name": "web_search",
  "description": "Search the web for current information",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {"type": "string", "description": "Search query"},
      "max_results": {"type": "integer", "default": 5}
    },
    "required": ["query"]
  },
  "execution": {
    "type": "http",
    "endpoint": "https://api.search.com/v1/search",
    "method": "GET",
    "timeout_ms": 5000
  },
  "safety": {
    "requires_approval": false,
    "max_calls_per_task": 10,
    "cost_per_call": 0.001
  }
}

Tool Execution Sandbox:
┌────────────────────────────────────────────────┐
│ Tool Executor (isolated container per session) │
│                                                │
│  Security:                                     │
│  - No network access to internal services      │
│  - CPU: 1 core, Memory: 512MB, Timeout: 30s   │
│  - No filesystem persistence                   │
│  - Outbound allowlist per tool                 │
│                                                │
│  Code execution (Python sandbox):              │
│  - gVisor container isolation                  │
│  - No subprocess/os/sys imports                │
│  - Output captured: stdout + return value      │
│  - Resource limits enforced by cgroups         │
└────────────────────────────────────────────────┘
```

## 6. Memory Architecture

```
Memory Hierarchy:
┌─────────────────────────────────────────────────┐
│ Working Memory (per-session, in Redis)          │
│  - Current conversation turns                   │
│  - Tool results from current task               │
│  - Planning state (current step, remaining)     │
│  - Token window: last 8K tokens                 │
│  TTL: session lifetime                          │
├─────────────────────────────────────────────────┤
│ Short-Term Memory (per-task, PostgreSQL)        │
│  - Full task execution log                      │
│  - All tool inputs/outputs                      │
│  - Intermediate reasoning steps                 │
│  - Checkpoints for resume                       │
│  TTL: 7 days                                    │
├─────────────────────────────────────────────────┤
│ Long-Term Memory (per-user, Vector DB)          │
│  - User preferences and past interactions       │
│  - Learned facts ("user prefers window seats")  │
│  - Embeddings of past task summaries            │
│  - Retrieved via semantic search at task start  │
│  TTL: indefinite (user-controlled)              │
└─────────────────────────────────────────────────┘

Memory Retrieval at Each Step:
  1. Query long-term memory: embed(current_goal) → top-5 relevant memories
  2. Inject into system prompt: "User context: {memories}"
  3. Append working memory: last N conversation turns
  4. Total context: system + memories + conversation + current step
```

## 7. Multi-Agent Collaboration

```
Multi-Agent Architecture:
┌──────────────────────────────────────────────────┐
│              Supervisor Agent                    │
│  (Decomposes task, assigns to specialists)       │
│                                                  │
│  Task: "Write a blog post about AI trends        │
│         with data from recent papers"            │
│                                                  │
│  Plan:                                           │
│  1. Researcher agent → find recent papers        │
│  2. Analyst agent → extract key trends           │
│  3. Writer agent → draft blog post               │
│  4. Editor agent → review and polish             │
├─────────┬─────────┬─────────┬────────────────────┤
│Researcher│ Analyst │ Writer  │ Editor             │
│ Tools:   │ Tools:  │ Tools:  │ Tools:             │
│ -search  │ -analyze│ -draft  │ -grammar_check     │
│ -pdf_read│ -chart  │ -format │ -fact_check        │
└─────────┴─────────┴─────────┴────────────────────┘

Communication:
  - Message passing via Kafka topics (agent_{id}_inbox)
  - Shared context in Redis (task-level scratchpad)
  - Supervisor monitors progress, can reassign or retry
```

## 8. Low-Level Design (LLD)

### API Contracts

```
# Create Agent Task
POST /api/v1/tasks
{
  "agent_type": "general",       // general | researcher | coder | analyst
  "goal": "Find the cheapest flight from SFO to NYC next Friday",
  "tools": ["web_search", "flight_search", "calendar"],
  "max_steps": 15,
  "max_cost_usd": 0.50,
  "stream": true,                // stream reasoning steps
  "approval_required": ["book_flight", "send_email"]
}
Response: {
  "task_id": "task_abc123",
  "status": "planning",
  "ws_url": "wss://api.example.com/tasks/task_abc123/stream"
}

# Stream (WebSocket)
→ {"type": "thinking", "content": "I need to search for flights..."}
→ {"type": "tool_call", "tool": "flight_search", "args": {...}}
→ {"type": "tool_result", "tool": "flight_search", "result": {...}}
→ {"type": "approval_needed", "action": "book_flight", "details": {...}}
← {"type": "approval", "approved": true}
→ {"type": "result", "content": "Booked Spirit Airlines $89...", "steps": 5}

# Get Task Status
GET /api/v1/tasks/{task_id}
Response: {
  "task_id": "task_abc123",
  "status": "completed",        // planning|executing|waiting_approval|completed|failed
  "steps": [...],
  "total_cost_usd": 0.12,
  "total_tokens": 4500,
  "duration_sec": 28
}
```

### Workflow State Schema

```sql
CREATE TABLE tasks (
    task_id         UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    agent_type      VARCHAR(50),
    goal            TEXT NOT NULL,
    status          task_status NOT NULL,
    current_step    INT DEFAULT 0,
    max_steps       INT DEFAULT 15,
    total_tokens    BIGINT DEFAULT 0,
    total_cost_usd  DECIMAL(10,6) DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    INDEX idx_user_tasks (user_id, created_at DESC)
);

CREATE TABLE task_steps (
    step_id         BIGSERIAL PRIMARY KEY,
    task_id         UUID REFERENCES tasks,
    step_number     INT NOT NULL,
    step_type       VARCHAR(20),    -- think | act | observe | approve
    content         JSONB NOT NULL, -- reasoning text or tool call details
    tool_name       VARCHAR(100),
    tool_input      JSONB,
    tool_output     JSONB,
    tokens_used     INT,
    duration_ms     INT,
    created_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (task_id, step_number)
);

CREATE TABLE agent_memories (
    memory_id       UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    content         TEXT NOT NULL,
    embedding       VECTOR(1536),
    memory_type     VARCHAR(20),    -- fact | preference | experience
    source_task_id  UUID,
    created_at      TIMESTAMPTZ,
    INDEX idx_user_memories (user_id)
);
```

## 9. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Concurrent Sessions** | Stateless orchestrator; session state in Redis | 10K concurrent |
| **LLM Throughput** | Multi-model gateway with failover (see file 28) | 2K LLM calls/sec |
| **Tool Execution** | Containerized tool sandboxes; Kubernetes auto-scaling | 5K tool exec/sec |
| **Memory Retrieval** | Vector DB sharded by user_id; cached frequent queries | 10K queries/sec |
| **Multi-Agent** | Each agent is a stateless worker; Kafka for message passing | 100 agents per task |

## 10. No Data Loss

| Component | Protection Mechanism |
|-----------|---------------------|
| **Task State** | PostgreSQL with synchronous replication; every step persisted before execution |
| **Checkpoints** | After each step, full state saved; task resumes from last checkpoint on failure |
| **Tool Results** | Logged to task_steps before proceeding to next step |
| **Memory** | Vector DB with WAL + Raft replication; PostgreSQL backup for metadata |
| **Audit Trail** | Append-only log of all agent actions; immutable S3 archival |

## 11. Latency

| Operation | p50 | p99 | Target |
|-----------|-----|-----|--------|
| LLM reasoning step | 1.5s | 5s | <10s |
| Tool execution | 200ms | 3s | <5s |
| Memory retrieval | 10ms | 50ms | <100ms |
| End-to-end simple task (3 steps) | 8s | 25s | <30s |
| End-to-end complex task (10 steps) | 45s | 3min | <5min |

## 12. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **LLM timeout** | Step stuck | Retry with exponential backoff; fallback to simpler model |
| **Tool failure** | Missing information | Retry 2×; agent reasons about failure and tries alternative approach |
| **Infinite loop** | Agent stuck in cycle | Max step limit (15); loop detection (same tool+args repeated 3×) |
| **Bad reasoning** | Wrong actions taken | Guardrails: validate tool args; human approval for destructive actions |
| **Session crash** | Progress lost | Resume from checkpoint in PostgreSQL; idempotent tool calls |

## 13. Availability

**Target: 99.9% for task submission, 99% for task completion**

```
Graceful Degradation:
  1. LLM unavailable → Queue task, notify user of delay
  2. Tool unavailable → Skip tool, agent adapts plan
  3. Memory DB down → Proceed without long-term context
  4. Heavy load → Queue with priority (paid > free)
```

## 14. Security

| Layer | Mechanism |
|-------|-----------|
| **Tool Sandbox** | gVisor containers; no internal network; resource limits; allowlisted outbound |
| **Code Execution** | Isolated Python sandbox; blocked imports (os, subprocess); 30s timeout |
| **Prompt Injection** | Input sanitization; instruction hierarchy (system > user > tool output) |
| **Data Access** | Per-user memory isolation; no cross-tenant data access |
| **Human-in-Loop** | Configurable approval gates for high-risk actions (payments, emails, deletions) |
| **Cost Control** | Per-task cost ceiling; per-user daily spending limit |

## 15. Cost Constraints

**Estimated Cost (10K concurrent sessions, 1M tasks/day):**

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| **LLM API costs** | 500M tokens/day × $0.002/1K avg | $30,000 |
| **GPU (self-hosted models)** | 4× A100 nodes for fast models | $40,000 |
| **Orchestrator** | 20× c6g.xlarge (stateless) | $7,000 |
| **Tool Sandbox (K8s)** | 50× m6g.large (auto-scaling) | $5,000 |
| **PostgreSQL** | RDS r6g.2xlarge Multi-AZ | $4,000 |
| **Vector DB** | Qdrant 3-node cluster | $2,000 |
| **Redis** | ElastiCache r6g.xlarge | $1,500 |
| **Total** | | **~$89,500/month** |

**Cost per task: ~$0.003 (simple) to $0.05 (complex)**

## Key Interview Discussion Points

1. **ReAct vs function calling?** — ReAct gives visible reasoning chain; function calling is more structured. ReAct better for debugging/audit; function calling more reliable
2. **How to prevent infinite loops?** — Max step limit, loop detection (repeated actions), cost ceiling, timeout per task
3. **How to handle tool failures?** — Agent should reason about failure and try alternatives; not just retry blindly
4. **How to scale multi-agent systems?** — Kafka-based message passing; each agent is stateless worker; supervisor monitors progress
5. **Memory management?** — Sliding window for working memory; vector search for long-term; summarize old conversations to compress
