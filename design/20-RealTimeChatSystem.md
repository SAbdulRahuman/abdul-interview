# Design a Real-Time Chat System

Examples: WhatsApp, Slack, Discord, Telegram

---

## 1. Requirements

### Functional
- 1:1 messaging and group chats (up to 100K members)
- Message delivery with read receipts (sent → delivered → read)
- Online/offline presence indicators
- Typing indicators
- Media sharing (images, files)
- Message history and search
- Push notifications for offline users

### Non-Functional
- 500M DAU, 50B messages/day
- Sub-200ms message delivery latency
- Messages must never be lost (at-least-once delivery)
- Message ordering within a conversation
- End-to-end encryption support

---

## 2. Scale Estimation

```
DAU:              500M
Messages/day:     50B → ~580K messages/sec
Peak:             ~2M messages/sec
Connections:      500M concurrent WebSocket connections
Message size:     ~1 KB average
Storage/day:      50B × 1 KB = 50 TB/day
Annual storage:   ~18 PB/year
```

---

## 3. High-Level Architecture

```
  ┌──────────┐  WebSocket  ┌──────────────────┐
  │  Client A │────────────▶│  Chat Gateway    │
  └──────────┘             │  (WS connection  │
                           │   manager)       │
  ┌──────────┐  WebSocket  │                  │
  │  Client B │────────────▶│  Server Pool    │
  └──────────┘             └────────┬─────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
             ┌──────▼──────┐ ┌─────▼──────┐ ┌──────▼──────┐
             │  Message    │ │  Presence  │ │  Group      │
             │  Service    │ │  Service   │ │  Service    │
             └──────┬──────┘ └────────────┘ └─────────────┘
                    │
         ┌──────────┼──────────┐
         │          │          │
  ┌──────▼──────┐  ┌▼────────┐ ┌▼────────────┐
  │  Message    │  │ Message │ │ Push         │
  │  Queue      │  │ Store   │ │ Notification │
  │  (Kafka)    │  │ (Cass.) │ │ Service      │
  └─────────────┘  └─────────┘ └──────────────┘
```

---

## 4. Connection Management

```
  WebSocket connection lifecycle:

  Client ──────────────────────────── Gateway Server
    │                                      │
    │  1. HTTP Upgrade: WebSocket           │
    │  ──────────────────────────────▶     │
    │                                      │
    │  2. Auth (JWT token)                 │
    │  ◀──────────────────────────────     │
    │                                      │
    │  3. Connection established           │
    │  ◀ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─▶     │
    │                                      │
    │  4. Register in connection registry: │
    │     Redis: user_id → {server_id,     │
    │                       conn_id}       │
    │                                      │
    │  5. Heartbeat (ping/pong every 30s)  │
    │  ◀ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─▶     │
    │                                      │

  Connection registry (Redis):
  ┌──────────────────────────────────────┐
  │  Key: user:conn:{user_id}            │
  │  Value: {                            │
  │    "server_id": "gw-12",             │
  │    "connected_at": 1710000000,       │
  │    "last_heartbeat": 1710000060      │
  │  }                                   │
  │                                      │
  │  500M entries × 100 bytes = ~50 GB   │
  │  Sharded Redis cluster               │
  └──────────────────────────────────────┘
```

---

## 5. Message Delivery Flow

```
  1:1 Message: Alice → Bob

  Alice's          Gateway       Message        Gateway       Bob's
  Client           Server A      Service        Server B      Client
    │                │              │              │             │
    │  send(msg)     │              │              │             │
    │───────────────▶│              │              │             │
    │                │  publish     │              │             │
    │                │─────────────▶│              │             │
    │                │              │              │             │
    │                │              │  1. Store in │             │
    │                │              │     Cassandra│             │
    │                │              │              │             │
    │                │              │  2. Lookup Bob│            │
    │                │              │     in conn   │            │
    │                │              │     registry  │            │
    │                │              │              │             │
    │                │              │  Bob online? │             │
    │                │              │──YES────────▶│             │
    │                │              │              │  deliver    │
    │                │              │              │────────────▶│
    │                │              │              │             │
    │                │              │              │  ack        │
    │                │              │              │◀────────────│
    │                │              │              │             │
    │                │              │  Bob offline?│             │
    │                │              │──push notif──▶ APNs/FCM   │
    │                │              │              │             │
    │  ack (sent)    │              │              │             │
    │◀───────────────│              │              │             │
    │                │              │              │             │

  Message states:
  ┌──────┐     ┌───────────┐     ┌───────────┐
  │ Sent │────▶│ Delivered │────▶│   Read    │
  └──────┘     └───────────┘     └───────────┘
   (server      (arrived at       (recipient
    received)    recipient's      opened chat)
                 device)
```

---

## 6. Group Messaging

```
  Group: 500 members

  Sender posts message to group:
  ┌──────────────────────────────────────────────┐
  │  1. Store message in group message table     │
  │     Partition key: group_id                  │
  │     Clustering key: message_id (time-ordered)│
  │                                              │
  │  2. Fan-out to online members:               │
  │     Lookup connection registry for 500 users │
  │     ~200 online → send via WebSocket         │
  │     ~300 offline → push notification          │
  │                                              │
  │  Optimization for large groups (>10K):       │
  │  Don't fan-out on write.                     │
  │  Members pull new messages when they open     │
  │  the group (fan-out on read).                │
  └──────────────────────────────────────────────┘

  Data model (Cassandra):
  ┌──────────────────────────────────────┐
  │  messages_by_conversation            │
  │  ┌──────────────────────────────┐    │
  │  │ conversation_id (PK)  │ UUID │    │
  │  │ message_id (CK, DESC) │ UUID │    │
  │  │ sender_id              │ UUID │    │
  │  │ content                │ TEXT │    │
  │  │ content_type           │ ENUM │    │
  │  │ created_at             │ TS   │    │
  │  └──────────────────────────────┘    │
  │                                      │
  │  Query: SELECT * FROM messages       │
  │         WHERE conversation_id = ?    │
  │         ORDER BY message_id DESC     │
  │         LIMIT 50                     │
  └──────────────────────────────────────┘
```

---

## 7. Presence & Typing Indicators

```
  Presence (online/offline):
  ┌──────────────────────────────────────────────┐
  │  On connect:  SET user:presence:{id} "online"│
  │               EXPIRE 60s                     │
  │  Heartbeat:   EXPIRE user:presence:{id} 60s  │
  │  On disconnect: DEL user:presence:{id}       │
  │  On timeout:  Key expires → user is offline  │
  │                                              │
  │  Broadcast: publish to friends' connections  │
  │  Throttle: update at most every 30s          │
  └──────────────────────────────────────────────┘

  Typing indicator:
  ┌──────────────────────────────────────────────┐
  │  Client sends "typing" event                 │
  │  Server forwards to other party              │
  │  Auto-expires after 5s if no update          │
  │  Don't persist — purely ephemeral            │
  └──────────────────────────────────────────────┘
```

---

## 8. Offline Message Handling

```
  Bob comes back online after 2 days:
  ┌──────────────────────────────────────────────┐
  │  1. Bob connects via WebSocket               │
  │  2. Server queries undelivered messages:     │
  │     SELECT * FROM messages                   │
  │     WHERE conversation_id IN (Bob's convos)  │
  │     AND message_id > Bob's last_seen_id      │
  │     ORDER BY message_id ASC                  │
  │                                              │
  │  3. Deliver messages in order                │
  │  4. Client acks each message → mark delivered│
  │  5. Sync read receipts for messages Bob read │
  └──────────────────────────────────────────────┘
```

---

## 9. End-to-End Encryption

```
  Signal Protocol (used by WhatsApp):
  ┌──────────────────────────────────────────────┐
  │  Key exchange: X3DH (Extended Triple DH)     │
  │  Ratchet: Double Ratchet Algorithm           │
  │                                              │
  │  Alice                    Bob                │
  │  ┌──────┐                ┌──────┐            │
  │  │ Gen  │                │ Gen  │            │
  │  │ keys │                │ keys │            │
  │  └──┬───┘                └──┬───┘            │
  │     │  Upload public key    │                │
  │     │──────────▶ Server ◀───│                │
  │     │                       │                │
  │     │  Fetch Bob's key      │                │
  │     │──────────▶ Server     │                │
  │     │◀──── Bob's pub key    │                │
  │     │                       │                │
  │     │  Derive shared secret │                │
  │     │  cipher = AES(msg, secret)             │
  │     │────ciphertext────────▶│                │
  │     │                  decrypt               │
  │                                              │
  │  Server sees only ciphertext — cannot read   │
  └──────────────────────────────────────────────┘
```

---

## 10. Fault Tolerance

- **Gateway failure:** Client reconnects to another gateway; resume from last ack'd message
- **Message store failure:** Cassandra's built-in replication (RF=3); no single point of failure
- **Message ordering:** Per-conversation sequence number; client reorders if needed
- **Duplicate messages:** Idempotency key per message; deduplicate on server and client
- **Network partition:** Buffer messages locally on client; sync when reconnected

---

---

## 11. Low-Level Design (LLD)

### API & Protocol Contract
```
  REST API:
  POST   /api/v1/conversations            Create 1:1 or group
  GET    /api/v1/conversations/{id}/messages?before=cursor&limit=50
  POST   /api/v1/conversations/{id}/messages
         { "content": "Hello", "type": "text", "client_msg_id": "uuid" }
  PUT    /api/v1/conversations/{id}/read   Mark as read (seq_num)
  GET    /api/v1/conversations?updated_after=timestamp

  WebSocket Protocol:
  ┌────────────────────────────────────────────────────────┐
  │  Client → Server:                                      │
  │  { "type": "send", "conv_id": "c1", "content": "Hi",  │
  │    "client_msg_id": "uuid-abc" }                       │
  │  { "type": "typing", "conv_id": "c1" }                │
  │  { "type": "ack", "msg_id": "m123", "status": "read" }│
  │  { "type": "ping" }                                    │
  │                                                        │
  │  Server → Client:                                      │
  │  { "type": "message", "conv_id": "c1", "msg_id": "m1",│
  │    "from": "user_456", "content": "Hi", "seq": 42,    │
  │    "timestamp": "2024-03-13T10:00:00Z" }               │
  │  { "type": "typing", "conv_id": "c1", "user": "u456" }│
  │  { "type": "receipt", "msg_id": "m1",                  │
  │    "status": "delivered" | "read" }                     │
  │  { "type": "pong" }                                    │
  └────────────────────────────────────────────────────────┘
```

### Message Storage Schema
```
  Messages (Cassandra):
  ┌────────────────────────────────────────────────────────┐
  │  PK: (conversation_id, bucket)   — bucket = YYYYMMDD  │
  │  CK: (sequence_number)           — per-conversation    │
  │                                                        │
  │  msg_id          TIMEUUID                              │
  │  sender_id       BIGINT                                │
  │  content         BLOB         (encrypted)              │
  │  content_type    TEXT         (text/image/file/system)  │
  │  client_msg_id   UUID         (for dedup)              │
  │  created_at      TIMESTAMP                             │
  │  deleted_at      TIMESTAMP    (nullable, soft delete)  │
  └────────────────────────────────────────────────────────┘

  Conversations (Cassandra):
  ┌────────────────────────────────────────────────────────┐
  │  Table: user_conversations                             │
  │  PK: (user_id)                                         │
  │  CK: (last_activity DESC, conversation_id)            │
  │                                                        │
  │  conversation_id  UUID                                 │
  │  conv_type        TEXT    (direct / group / channel)    │
  │  last_message     TEXT                                 │
  │  unread_count     INT                                  │
  │  muted            BOOLEAN                              │
  └────────────────────────────────────────────────────────┘

  Connection Registry (Redis):
  ┌────────────────────────────────────────────────────────┐
  │  Key: conn:{user_id}                                   │
  │  Value: { "gateway": "gw-03", "ws_id": "ws-abc",      │
  │           "connected_at": "...", "device": "ios" }     │
  │  TTL: heartbeat-based (expire if no ping in 30s)      │
  │                                                        │
  │  Multi-device: conn:{user_id}:{device_id}              │
  │  Message delivered to ALL active devices               │
  └────────────────────────────────────────────────────────┘
```

### Message Delivery Flow (Detailed)
```
  ┌────────────────────────────────────────────────────────┐
  │  1. Client sends message via WebSocket                 │
  │  2. Gateway: validate auth, assign sequence number     │
  │  3. Write to Kafka: topic=messages, key=conv_id        │
  │     (ensures ordering per conversation)                │
  │  4. ACK to sender: { "type": "ack", "status": "sent" }│
  │  5. Message consumer:                                   │
  │     a. Write to Cassandra                              │
  │     b. Update last_activity in user_conversations      │
  │     c. Increment unread_count for recipients           │
  │  6. Delivery router:                                    │
  │     a. Lookup recipient in Redis connection registry   │
  │     b. If ONLINE: route to gateway → push via WS      │
  │        → wait for delivery ACK → update status         │
  │     c. If OFFLINE: push notification (FCM/APNs)       │
  │        → store in offline queue                        │
  │  7. Recipient comes online:                            │
  │     a. Sync: GET /messages?after=last_seen_seq         │
  │     b. Drain offline queue                              │
  └────────────────────────────────────────────────────────┘
```

---

## 12. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Connection Scaling:                                    │
  │  • Each gateway: ~100K concurrent WebSockets            │
  │  • 10M concurrent → 100 gateway servers                 │
  │  • Consistent hashing: user → gateway assignment       │
  │  • Multi-device: same user on multiple gateways OK     │
  │                                                         │
  │  Message Throughput:                                    │
  │  • Kafka: partition by conversation_id                  │
  │  • 100 partitions → 100K msg/sec                        │
  │  • Consumers: auto-scale based on lag                   │
  │                                                         │
  │  Storage:                                               │
  │  • Cassandra: partition by (conv_id, day_bucket)        │
  │  • Each partition: ~1000 messages (manageable)          │
  │  • 50Bn messages/month: ~50 TB (Cassandra handles well) │
  │                                                         │
  │  Group Chats:                                           │
  │  • Small groups (< 256): fan-out on write               │
  │  • Channels (> 256): fan-out on read                    │
  │  • Typing indicators: only first 5 users broadcasted   │
  └─────────────────────────────────────────────────────────┘
```

---

## 13. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Message Durability:                                    │
  │  • Kafka: acks=all, RF=3, min.insync.replicas=2        │
  │  • Cassandra: RF=3, QUORUM writes                       │
  │  • Message persisted before ACK to sender               │
  │  • Client retry with client_msg_id (deduplication)     │
  │                                                         │
  │  Delivery Guarantee:                                    │
  │  • At-least-once delivery to all recipients             │
  │  • Dedup at client: client_msg_id + local DB            │
  │  • Offline queue: messages buffered until device online │
  │  • Multi-device sync: sequence numbers per conversation │
  │                                                         │
  │  End-to-End Encryption:                                 │
  │  • Server cannot read messages (only encrypted blobs)  │
  │  • Key exchange: Signal Protocol (Double Ratchet)       │
  │  • If device lost: messages aren't recoverable on new  │
  │    device (security trade-off)                          │
  │                                                         │
  │  Backup:                                                │
  │  • Cassandra: snapshot + incremental backup to S3       │
  │  • Redis (connection state): ephemeral, no backup needed│
  └─────────────────────────────────────────────────────────┘
```

---

## 14. Latency

```
  Latency Targets:
  ┌───────────────────────────────┬──────────┬──────────┐
  │ Operation                    │ p50      │ p99      │
  ├───────────────────────────────┼──────────┼──────────┤
  │ Message send → delivered     │ 100 ms   │ 500 ms  │
  │ Message send → server ACK    │ 30 ms    │ 100 ms  │
  │ Typing indicator             │ 50 ms    │ 200 ms  │
  │ Presence update              │ 1 s      │ 5 s     │
  │ Message history load (50 msg)│ 50 ms    │ 200 ms  │
  │ WebSocket connect            │ 100 ms   │ 500 ms  │
  └───────────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • Direct gateway-to-gateway for same-server recipients│
  │  • Connection registry (Redis): <1ms lookup             │
  │  • Kafka: batch at 5ms linger for throughput            │
  │  • Client-side optimistic rendering (show immediately) │
  │  • Binary WebSocket frames (MessagePack vs JSON, 30%   │
  │    smaller)                                             │
  │  • Typing: throttle to 1 event per second              │
  │  • Presence: heartbeat aggregation (batch 100ms window)│
  └─────────────────────────────────────────────────────────┘
```

---

## 15. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Gateway crash          │ Client auto-reconnect (exp.     │
  │                        │ backoff); resume from last seq  │
  ├────────────────────────┼─────────────────────────────────┤
  │ Network disconnection  │ Offline queue on device; sync   │
  │ (mobile user)          │ on reconnect; push notification │
  │                        │ as fallback                     │
  ├────────────────────────┼─────────────────────────────────┤
  │ Message ordering       │ Per-conversation sequence       │
  │ violation              │ numbers; client reorders if     │
  │                        │ received out-of-order            │
  ├────────────────────────┼─────────────────────────────────┤
  │ Kafka consumer lag     │ Auto-scale consumers; alert if  │
  │                        │ delivery delay > 5s              │
  ├────────────────────────┼─────────────────────────────────┤
  │ Push notification      │ Retry 3 times; fallback to      │
  │ failure (FCM/APNs)     │ in-app sync on next connection  │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 16. Availability

```
  Target: 99.99% for message delivery; 99.9% for history queries

  ┌─────────────────────────────────────────────────────────┐
  │  Connection Layer:                                      │
  │  • Gateway fleet: N+2 redundancy across 3 AZs           │
  │  • Client reconnect: automatic with exp. backoff        │
  │  • NLB: health check every 5s, remove unhealthy         │
  │                                                         │
  │  Message Pipeline:                                      │
  │  • Kafka: 3 brokers × 3 AZs                            │
  │  • If Kafka down: gateway buffers ~100K messages/server │
  │  • Cassandra: multi-AZ, LOCAL_QUORUM                    │
  │                                                         │
  │  Offline Resilience:                                    │
  │  • Messages queue server-side until recipient online   │
  │  • Push notifications as backup delivery channel        │
  │  • Client stores messages locally (SQLite)              │
  │  • Sync protocol: reconcile client-server state        │
  │                                                         │
  │  Multi-Region:                                          │
  │  • Active-active regions with user-affinity routing     │
  │  • Cross-region: message forwarded to recipient's region│
  └─────────────────────────────────────────────────────────┘
```

---

## 17. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  End-to-End Encryption (E2EE):                          │
  │  • Signal Protocol: Double Ratchet + X3DH key exchange │
  │  • Server only sees encrypted blobs                     │
  │  • Per-message key rotation (forward secrecy)           │
  │  • Group: Sender Keys (each member has group key)       │
  │                                                         │
  │  Transport Security:                                    │
  │  • TLS 1.3 for WebSocket and REST API                   │
  │  • Certificate pinning on mobile clients                │
  │  • Noise Protocol for additional transport security     │
  │                                                         │
  │  Authentication:                                        │
  │  • Phone number verification (OTP)                      │
  │  • Session tokens: rotate on each app restart           │
  │  • Multi-device: separate keys per device               │
  │  • Remote device logout                                 │
  │                                                         │
  │  Anti-Abuse:                                            │
  │  • Spam detection: ML model on message patterns         │
  │  • Rate limiting: 100 messages/min per user             │
  │  • Block/report: user-level blocking                    │
  │  • Metadata minimization: no IP logging after delivery  │
  │                                                         │
  │  Compliance:                                            │
  │  • GDPR: data export, account deletion                  │
  │  • HIPAA (healthcare): audit logs, access controls      │
  │  • Disappearing messages: auto-delete after N hours     │
  └─────────────────────────────────────────────────────────┘
```

---

## 18. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  At Scale (100M DAU, 10M concurrent):                    │
  │                                                         │
  │  Gateway Servers (WebSocket):                           │
  │  • 100 × c5.2xlarge: ~$25K/month                        │
  │  • 100K connections per server                           │
  │                                                         │
  │  Message Pipeline:                                      │
  │  • Kafka (6 brokers): ~$5K/month                        │
  │  • Consumer workers (20 × c5.xl): ~$3K/month           │
  │                                                         │
  │  Storage:                                               │
  │  • Cassandra (30 × i3.2xl, 90d retention): ~$30K/month │
  │  • Redis (connection registry, 10 nodes): ~$5K/month   │
  │                                                         │
  │  Push Notifications:                                    │
  │  • FCM/APNs: free (Google/Apple infrastructure)         │
  │  • SNS: $0.50 per 1M → ~$500/month                     │
  │                                                         │
  │  Total: ~$70K/month                                     │
  │                                                         │
  │  Cost Optimization:                                     │
  │  • Binary protocol: 30% less bandwidth vs JSON          │
  │  • Message compression: zstd for stored messages        │
  │  • Shorter retention: 90d default, archive to S3        │
  │  • Presence coalescing: reduce heartbeat frequency      │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **WebSocket vs long polling?** — WebSocket: full-duplex, lower latency, less overhead. Long polling: works through firewalls but higher latency
2. **How to handle 500M concurrent connections?** — Horizontally scaled gateway fleet; each server handles ~100K connections; connection registry in Redis
3. **Message ordering guarantees?** — Per-conversation ordering with sequence numbers; not global ordering
4. **How to scale group messaging?** — Small groups: fan-out on write. Large groups/channels: fan-out on read
5. **How to handle message retention?** — TTL-based deletion for ephemeral messages; archive to cold storage for persistence; compliance needs
