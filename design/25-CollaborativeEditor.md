# Design a Collaborative Editing System (Google Docs)

Examples: Google Docs, Notion, Figma

---

## 1. Requirements

### Functional
- Multiple users edit the same document simultaneously
- Real-time cursor and selection visibility
- Presence indicators (who is viewing/editing)
- Undo/redo per user
- Document version history with restore
- Comments and suggestions mode

### Non-Functional
- Sub-100ms real-time sync between collaborators
- Support 1000+ concurrent editors on a single document
- No data loss even during network partitions
- Conflict-free concurrent edits
- Offline editing with sync on reconnect

---

## 2. High-Level Architecture

```
  ┌───────────┐  WebSocket  ┌──────────────────┐
  │  Editor A  │────────────▶│  Collaboration   │
  └───────────┘             │  Server          │
  ┌───────────┐  WebSocket  │  (per-document   │
  │  Editor B  │────────────▶│   session)       │
  └───────────┘             └────────┬─────────┘
  ┌───────────┐  WebSocket           │
  │  Editor C  │────────────▶        │
  └───────────┘                      │
                          ┌──────────┼──────────┐
                          │          │          │
                   ┌──────▼──────┐  ┌▼────────┐ ┌▼──────────┐
                   │  OT/CRDT   │  │Document │ │Presence   │
                   │  Transform │  │Store    │ │Service    │
                   │  Engine    │  │(versions│ │(cursors,  │
                   │            │  │ history)│ │ users)    │
                   └────────────┘  └─────────┘ └───────────┘
```

---

## 3. Conflict Resolution: OT vs CRDT

```
  THE PROBLEM:
  Two users type at the same time:

  Original text: "ABCD"
  User A: Insert "X" at position 1 → "AXBCD"
  User B: Delete char at position 2 → "ABD"

  If applied naively in wrong order → corruption!

  ═══════════════════════════════════════════════

  OPTION A: Operational Transformation (OT)
  ┌──────────────────────────────────────────────┐
  │  Used by: Google Docs                        │
  │                                              │
  │  Idea: Transform operations against each     │
  │  other so they remain correct regardless     │
  │  of arrival order.                           │
  │                                              │
  │  Operation: { type, position, char, userId } │
  │                                              │
  │  User A: insert("X", pos=1)                  │
  │  User B: delete(pos=2)                       │
  │                                              │
  │  Transform(A, B):                            │
  │  Since A inserts before B's position:        │
  │  B' = delete(pos=3)  (shift position by 1)   │
  │                                              │
  │  Apply: A then B' → "AXBD" ✓                │
  │  Apply: B then A  → "AXBD" ✓ (same result!) │
  │                                              │
  │  Requires: central server to order operations│
  │  Pro: Well-understood, battle-tested         │
  │  Con: Complex transform functions, N² pairs  │
  │       Server is single point of ordering     │
  └──────────────────────────────────────────────┘

  OPTION B: CRDT (Conflict-free Replicated Data Type)
  ┌──────────────────────────────────────────────┐
  │  Used by: Figma, Notion (partially)          │
  │                                              │
  │  Idea: Data structure that mathematically    │
  │  guarantees convergence without central      │
  │  coordination.                               │
  │                                              │
  │  Each character has a unique ID:             │
  │  (siteId, counter)                           │
  │                                              │
  │  "ABCD" → [A(1,0), B(1,1), C(1,2), D(1,3)] │
  │                                              │
  │  User A inserts "X" between A and B:         │
  │  X gets ID (2,0), positioned between (1,0)   │
  │  and (1,1)                                   │
  │                                              │
  │  User B deletes C (1,2):                     │
  │  Mark (1,2) as tombstone                     │
  │                                              │
  │  Both operations commute → same result       │
  │                                              │
  │  Pro: No central server needed               │
  │       Works offline; peer-to-peer possible   │
  │  Con: Higher memory (tombstones, unique IDs) │
  │       Harder to implement undo correctly     │
  └──────────────────────────────────────────────┘
```

---

## 4. Server-Side OT Architecture

```
  Central server as "single source of truth":

  Client A                 Server               Client B
    │                        │                      │
    │  op1: ins("X", 1)      │                      │
    │───────────────────────▶│                      │
    │                        │  assign revision r5  │
    │                        │                      │
    │                        │  op2: del(2)         │
    │                        │◀─────────────────────│
    │                        │  assign revision r6  │
    │                        │                      │
    │                        │  Transform:          │
    │                        │  op2' = transform    │
    │                        │    (op2, op1)         │
    │                        │                      │
    │  broadcast: op2'       │  broadcast: op1      │
    │◀───────────────────────│─────────────────────▶│
    │  apply op2'            │                      │
    │  (adjusted position)   │          apply op1   │
    │                        │                      │

  Server maintains:
  ┌──────────────────────────────────────────────┐
  │  Document state: current text                │
  │  Operation log: [op1, op2, op3, ...]         │
  │  Revision counter: monotonically increasing  │
  │  Per-client: last acknowledged revision      │
  │                                              │
  │  When client sends op with base_revision=r4: │
  │  Server transforms op against all ops since  │
  │  r4 (i.e., op5, op6, ...) → op'             │
  │  Apply op' to document                       │
  │  Broadcast op' to all other clients          │
  └──────────────────────────────────────────────┘
```

---

## 5. Presence and Cursor Tracking

```
  Each client broadcasts cursor position:

  ┌──────────────────────────────────────────────┐
  │  {                                           │
  │    "user_id": "alice",                       │
  │    "cursor_position": 42,                    │
  │    "selection_start": 42,                    │
  │    "selection_end": 55,                      │
  │    "color": "#FF5733",                       │
  │    "name": "Alice"                           │
  │  }                                           │
  └──────────────────────────────────────────────┘

  Cursor positions are adjusted when ops applied:
  If an insert happens before the cursor position → shift right
  If a delete happens before the cursor position → shift left

  Throttle: send cursor updates at most every 50ms
  (not every keystroke)

  Presence:
  • Show avatar/name on document
  • Update on connect/disconnect
  • Heartbeat every 30s to detect stale sessions
```

---

## 6. Document Storage & Versioning

```
  Two storage strategies:

  Option A: Store full snapshots
  ┌──────────────────────────────────────────────┐
  │  v1: "Hello"                                 │
  │  v2: "Hello World"                           │
  │  v3: "Hello Beautiful World"                 │
  │                                              │
  │  Pro: Fast to load any version               │
  │  Con: Storage-heavy for large documents      │
  └──────────────────────────────────────────────┘

  Option B: Snapshot + Op log (recommended)
  ┌──────────────────────────────────────────────┐
  │  Snapshot every N operations:                │
  │  Snapshot@v100: "Hello World..."             │
  │  Ops: [v101: ins(X,5), v102: del(3), ...]   │
  │  Snapshot@v200: "Hello Beautiful World..."   │
  │                                              │
  │  To load v150:                               │
  │  1. Load snapshot@v100                       │
  │  2. Replay ops v101–v150                     │
  │                                              │
  │  Pro: Balance between speed and storage      │
  │  Con: Replay required for non-snapshot revs  │
  └──────────────────────────────────────────────┘

  Version history UI:
  ┌──────────────────────────────────────────────┐
  │  Show named versions (auto or manual):       │
  │  • "March 15, 10:30 AM — Alice"             │
  │  • "March 15, 11:00 AM — Bob"               │
  │  • "Final version — Alice (named)"          │
  │  User can restore any version               │
  └──────────────────────────────────────────────┘
```

---

## 7. Undo in Collaborative Context

```
  Problem: User A types "Hello", then User B types "World".
  User A presses Undo. What happens?

  ┌──────────────────────────────────────────────┐
  │  Global undo: Undo last operation (B's)      │
  │  → BAD: A would undo B's work!              │
  │                                              │
  │  Per-user undo (correct):                    │
  │  Undo only A's last operation                │
  │                                              │
  │  Implementation:                             │
  │  Each client maintains own undo stack        │
  │  On undo: generate inverse operation         │
  │    insert("Hello", 0) → delete(0, 5)        │
  │  Transform inverse against all subsequent ops│
  │  Send transformed inverse to server          │
  │                                              │
  │  This is one of the hardest parts of OT!     │
  └──────────────────────────────────────────────┘
```

---

## 8. Offline Editing

```
  User goes offline, continues editing:

  ┌──────────────────────────────────────────────┐
  │  Client buffers ops locally:                 │
  │  [op1, op2, op3, ...] (queued, not sent)     │
  │                                              │
  │  Meanwhile, server processes other users' ops│
  │                                              │
  │  On reconnect:                               │
  │  1. Client sends buffered ops with           │
  │     base_revision = last known revision      │
  │  2. Server sends all ops since that revision │
  │  3. Client transforms its ops against        │
  │     server ops                               │
  │  4. Server transforms server ops against     │
  │     client ops                               │
  │  5. Both converge to same state              │
  │                                              │
  │  If offline too long → prompt for manual     │
  │  merge (like git conflicts)                  │
  └──────────────────────────────────────────────┘
```

---

## 9. Scaling to 1000+ Concurrent Editors

```
  Challenges:
  ┌──────────────────────────────────────────────┐
  │  1000 users × 5 ops/sec = 5000 ops/sec      │
  │  Each op broadcast to 999 others             │
  │  = ~5M messages/sec for one document!        │
  │                                              │
  │  Solutions:                                  │
  │  • Batch ops: coalesce 50ms window of ops    │
  │    into single message                       │
  │  • Delta compression: send only diffs        │
  │  • Hierarchical broadcast: server → region   │
  │    relay → clients                           │
  │  • Partition document into blocks (sections) │
  │    Only sync block that's being edited       │
  │  • Rate limit cursor broadcasts to 50ms      │
  └──────────────────────────────────────────────┘
```

---

## 10. Fault Tolerance

- **Server crash:** Op log persisted to DB; new server replays from last snapshot
- **Client disconnect:** Ops buffered locally; sync on reconnect
- **Network partition:** Each partition continues editing independently; reconcile on heal
- **Data corruption:** Checksum on document state; rebuild from op log if mismatch

---

## 11. Low-Level Design (LLD)

### API Contracts

```
# WebSocket Connection (real-time editing)
WS /api/v1/docs/{doc_id}/collaborate
Headers: Authorization: Bearer <token>

# Client → Server Messages
{
  "type": "operation",
  "client_id": "c_abc123",
  "revision": 42,              // client's last known server revision
  "ops": [
    {"type": "retain", "count": 15},
    {"type": "insert", "text": "Hello "},
    {"type": "retain", "count": 230},
    {"type": "delete", "count": 5}
  ]
}

{
  "type": "cursor",
  "client_id": "c_abc123",
  "position": 156,
  "selection_end": 180         // null if no selection
}

# Server → Client Messages
{
  "type": "ack",
  "revision": 43               // new server revision after applying op
}

{
  "type": "operation",          // broadcast to other clients
  "client_id": "c_def456",     // who made the change
  "revision": 43,
  "ops": [...]                 // transformed operation
}

{
  "type": "presence",
  "users": [
    {"client_id": "c_def456", "name": "Alice", "color": "#FF5733", 
     "cursor": 42, "selection_end": null}
  ]
}

# REST API
POST /api/v1/docs                        # Create document
GET  /api/v1/docs/{doc_id}               # Get document content + metadata
GET  /api/v1/docs/{doc_id}/history       # Revision history
POST /api/v1/docs/{doc_id}/snapshot      # Force snapshot
PUT  /api/v1/docs/{doc_id}/permissions   # Update sharing
```

### OT Server Internal Architecture

```
OT Server Processing (per document):
┌──────────────────────────────────────────────────────────┐
│ Document Session (in-memory, one per active document)    │
│                                                          │
│ State:                                                   │
│   current_revision: 43                                   │
│   document_text: "Hello world... (full text)"            │
│   pending_ops: []         // ops awaiting transform      │
│   history: [op1, op2, ..., op43]  // operation log       │
│   connected_clients: {c_abc: rev42, c_def: rev43}       │
│                                                          │
│ On Receive Operation(client_op, client_rev):             │
│   1. Fetch server_ops = history[client_rev..current_rev] │
│   2. For each server_op in server_ops:                   │
│        client_op = transform(client_op, server_op)       │
│   3. Apply transformed client_op to document_text        │
│   4. current_revision += 1                               │
│   5. Append to history                                   │
│   6. Send ACK to originating client                      │
│   7. Broadcast transformed op to all other clients       │
├──────────────────────────────────────────────────────────┤
│ Transform Function (core OT):                           │
│                                                          │
│ transform(op_a, op_b) → (op_a', op_b')                  │
│                                                          │
│ Example: Two concurrent inserts                          │
│   op_a: insert("X", pos=5)  (from client A)             │
│   op_b: insert("Y", pos=3)  (from client B, applied 1st)│
│                                                          │
│   Since op_b inserts at pos 3 (before 5):                │
│   op_a' = insert("X", pos=6)  // shift right by 1       │
│   op_b' = insert("Y", pos=3)  // unchanged              │
└──────────────────────────────────────────────────────────┘
```

### Document Storage Schema

```sql
-- Documents table
CREATE TABLE documents (
    doc_id          UUID PRIMARY KEY,
    title           VARCHAR(500),
    owner_id        UUID NOT NULL,
    content_type    VARCHAR(50) DEFAULT 'richtext',
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- Snapshots (periodic full document state)
CREATE TABLE snapshots (
    doc_id          UUID REFERENCES documents,
    revision        BIGINT,
    content         JSONB NOT NULL,         -- full document state
    created_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (doc_id, revision)
);

-- Operations log (append-only)
CREATE TABLE operations (
    doc_id          UUID REFERENCES documents,
    revision        BIGINT,
    client_id       VARCHAR(100),
    ops             JSONB NOT NULL,         -- operation array
    created_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (doc_id, revision)
);
-- Snapshot every 100 operations; replay from nearest snapshot

-- Permissions
CREATE TABLE doc_permissions (
    doc_id          UUID REFERENCES documents,
    user_id         UUID,
    role            VARCHAR(20),            -- owner|editor|commenter|viewer
    PRIMARY KEY (doc_id, user_id)
);
```

## 12. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Concurrent Editors** | Single document: 1 OT server process (serialized ops) | 50 concurrent editors per doc |
| **1000+ Editors** | Document partitioning: split into sections, each with own OT server | 1000+ per doc |
| **Total Documents** | Shard by doc_id; route WebSocket to correct server via consistent hashing | 100M documents |
| **Operation History** | Snapshot every 100 ops; compact old ops; archive to cold storage | Unlimited revisions |
| **Presence Updates** | Throttle cursor broadcasts to 10/sec; batch presence updates | 1000 cursors per doc |

**Scaling Architecture:**
```
Document Routing:
  doc_id → hash → OT Server assignment
  
  OT Server Pool:
    Server 1: docs [A-F hash range]
    Server 2: docs [G-M hash range]  
    Server 3: docs [N-Z hash range]
  
  When doc becomes hot (>50 editors):
    1. Partition document into sections
    2. Each section gets own OT server
    3. Cross-section operations: two-phase coordination
    4. Merge results at document level

  Session migration (for scaling):
    1. Pause incoming ops (buffer at gateway)
    2. Serialize document state + op history  
    3. Transfer to new server
    4. Resume processing (clients reconnect)
```

## 13. No Data Loss

| Component | Protection Mechanism |
|-----------|---------------------|
| **Operations Log** | Append-only in PostgreSQL; WAL ensures crash safety |
| **Document State** | Snapshots every 100 ops; full document state preserved |
| **In-Memory State** | OT server persists ops to DB before broadcasting ACK |
| **Client Offline** | Client queues ops locally; replays on reconnect; server transforms against missed ops |
| **Version History** | Full operation log enables any historical version reconstruction |
| **Backups** | Hourly snapshots replicated cross-region; PITR from WAL |

**Recovery Scenarios:**
```
OT Server Crash:
  1. New server takes over document session
  2. Load latest snapshot from DB
  3. Replay operations since snapshot
  4. Rebuild in-memory document state
  5. Clients reconnect and re-sync (server sends missed ops)
  Recovery time: <5 seconds

Client Crash:
  1. Unacknowledged ops are in client local buffer
  2. On restart: client detects last ACKed revision
  3. Re-sends buffered ops
  4. Server transforms and applies (idempotent by revision check)
```

## 14. Latency

| Operation | p50 | p99 | Target |
|-----------|-----|-----|--------|
| Keystroke to remote display | 50ms | 150ms | <200ms |
| OT transform (server-side) | 0.1ms | 1ms | <5ms |
| Cursor update broadcast | 30ms | 100ms | <200ms |
| Document load (open) | 100ms | 500ms | <1s |
| Snapshot save | 5ms | 50ms | <100ms |

**Optimization Techniques:**
- **Optimistic rendering**: Apply ops locally immediately; transform if server response differs
- **Op batching**: Batch rapid keystrokes into single operation (50ms debounce)
- **Binary protocol**: Use MessagePack/Protobuf over WebSocket instead of JSON (30% smaller)
- **Presence throttling**: Cursor positions broadcast at max 10Hz (not per-keystroke)
- **Regional servers**: Route to nearest OT server for lowest WebSocket RTT
- **Snapshot caching**: Cache recent snapshots in Redis for fast document open

## 15. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **OT server crash** | Document editing paused | Automatic failover: new server loads from DB snapshot + op replay; <5s recovery |
| **WebSocket disconnect** | Client can't send ops | Auto-reconnect with exponential backoff; client buffers ops locally |
| **Transform conflict** | Unexpected text corruption | OT correctness proofs: convergence property guarantees all clients reach same state |
| **Clock skew** | Operation ordering issues | Server assigns monotonic revision numbers; client clock irrelevant |
| **Slow client** | Falls behind on ops | Server sends compressed catch-up batch; client shows "syncing" indicator |
| **Concurrent undo** | Per-user undo conflicts | Transform inverse operation against intervening ops from all users |

## 16. Availability

**Target: 99.99% for editing, 99.999% for document access (read-only)**

```
Availability Architecture:
┌─────────────────────────────────────────────────┐
│            WebSocket Gateway (N+2)              │
│    Health check: /healthz every 5s              │
│    Sticky routing by doc_id                     │
├──────────────┬──────────────┬───────────────────┤
│  OT Server 1 │  OT Server 2 │  OT Server 3     │
│  Active docs │  Active docs │  Active docs      │
│  [A-F]       │  [G-M]       │  [N-Z]           │
│  Standby:    │  Standby:    │  Standby:        │
│  OT Server 4 │  OT Server 5 │  OT Server 6    │
├──────────────┴──────────────┴───────────────────┤
│           PostgreSQL (snapshots + ops)          │
│           Primary + Sync Standby                │
└─────────────────────────────────────────────────┘

Graceful Degradation:
  1. OT server overloaded → New editors get read-only mode
  2. DB down → Editing continues in-memory; ops queued for persistence
  3. All OT servers down → Document available as last snapshot (read-only)
  4. Network partition → Clients edit locally; reconcile on reconnect
```

## 17. Security

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | OAuth 2.0 / Google SSO; JWT in WebSocket handshake |
| **Authorization** | Per-document ACL: owner > editor > commenter > viewer; checked on every operation |
| **Transport** | WSS (TLS 1.3) for all WebSocket connections |
| **Operation Validation** | Server validates every op: position bounds, content policy, size limits |
| **Content Safety** | Real-time content scanning for sensitive data (PII, secrets) |
| **Link Sharing** | Signed URLs with expiry; optional password protection |
| **Audit Trail** | Every operation logged with user_id + timestamp; tamper-evident log |
| **Data Residency** | Document stored in user's chosen region (GDPR compliance) |

## 18. Cost Constraints

**Estimated Cost (10M documents, 500K DAU, 50K concurrent editors):**

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| **OT Servers** | 20× c6g.xlarge (stateful, high CPU) | $7,000 |
| **WebSocket Gateways** | 10× c6g.large | $2,500 |
| **PostgreSQL** | RDS r6g.2xlarge Multi-AZ | $4,000 |
| **Redis (presence + cache)** | ElastiCache r6g.xlarge cluster | $2,000 |
| **Object Storage** | S3: document assets (images, etc.) 10TB | $230 |
| **Bandwidth** | WebSocket traffic: 50TB/month | $4,500 |
| **Total** | | **~$20,200/month** |

**Cost Optimization:**
- **Session lifecycle**: Unload inactive documents from OT servers after 5min idle (95% of docs are idle)
- **Op compression**: Delta encoding for operation logs — 80% storage reduction
- **Snapshot compaction**: Keep only every 1000th snapshot for old documents
- **WebSocket compression**: permessage-deflate extension — 60% bandwidth savings
- **Spot instances for dev/staging**: 70% savings on non-production OT servers

## Key Interview Discussion Points

1. **OT vs CRDT trade-offs?** — OT: simpler with central server, battle-tested (Google Docs). CRDT: works peer-to-peer, better for offline, higher memory overhead
2. **Why is per-user undo hard?** — Must transform inverse operation against all subsequent ops from all users
3. **How to handle 1000 concurrent editors?** — Batching, document partitioning, hierarchical broadcast
4. **How to handle images/tables in collaborative editing?** — Treat as atomic blocks; use block-level CRDT/OT; avoid character-level ops for non-text
5. **How to implement access control?** — Per-document permissions (owner, editor, commenter, viewer); check on every operation; propagate changes in real-time
