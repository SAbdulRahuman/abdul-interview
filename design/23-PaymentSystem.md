# Design a Payment System

Examples: Stripe, PayPal, Square, Adyen

---

## 1. Requirements

### Functional
- Process payments (charge, authorize, capture, refund)
- Support multiple payment methods (card, bank, wallet)
- Ledger for double-entry bookkeeping
- Webhook notifications for payment events
- Fraud detection
- Multi-currency support

### Non-Functional
- 99.999% availability for payment processing
- Exactly-once payment semantics (no double charges)
- PCI-DSS compliance
- Transaction latency < 1 second
- Audit trail for every operation

---

## 2. High-Level Architecture

```
  ┌──────────┐     ┌──────────────┐     ┌──────────────────┐
  │  Merchant │────▶│  API Gateway │────▶│  Payment Service │
  │  App      │     │  (auth, rate │     │  (orchestrator)  │
  └──────────┘     │   limit)     │     └────────┬─────────┘
                   └──────────────┘              │
                          ┌──────────────────────┼──────────────────┐
                          │                      │                  │
                   ┌──────▼──────┐        ┌──────▼──────┐   ┌──────▼──────┐
                   │  Fraud      │        │  Ledger     │   │  Payment    │
                   │  Detection  │        │  Service    │   │  Gateway    │
                   │  Engine     │        │  (double-   │   │  Router     │
                   └─────────────┘        │   entry)    │   └──────┬──────┘
                                          └─────────────┘          │
                                                          ┌────────┼────────┐
                                                          ▼        ▼        ▼
                                                     ┌──────┐ ┌──────┐ ┌──────┐
                                                     │ Visa │ │Master│ │ Bank │
                                                     │      │ │ Card │ │ ACH  │
                                                     └──────┘ └──────┘ └──────┘
```

---

## 3. Payment Flow (Idempotent)

```
  Merchant: POST /v1/payments
  Body: { amount: 100.00, currency: "USD",
          idempotency_key: "ord_12345_pay_1" }

  ┌──────────────────────────────────────────────────────────┐
  │  Step 1: Idempotency check                              │
  │  ┌──────────────────────────────────────────────┐       │
  │  │  Lookup idempotency_key in DB                │       │
  │  │  If exists → return cached result            │       │
  │  │  If not → create record with status=PENDING  │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                          │
  │  Step 2: Fraud check                                     │
  │  ┌──────────────────────────────────────────────┐       │
  │  │  Rules engine: velocity, amount, geo          │       │
  │  │  ML model: anomaly score                      │       │
  │  │  → APPROVE / REVIEW / DECLINE                │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                          │
  │  Step 3: Authorize with card network                     │
  │  ┌──────────────────────────────────────────────┐       │
  │  │  Route to payment processor (Visa/Mastercard) │       │
  │  │  Send: card token, amount, merchant ID        │       │
  │  │  Receive: auth_code or decline_reason         │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                          │
  │  Step 4: Record in ledger                                │
  │  ┌──────────────────────────────────────────────┐       │
  │  │  DEBIT  customer_account  $100.00             │       │
  │  │  CREDIT merchant_account  $100.00             │       │
  │  │  (always two entries that sum to zero)        │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                          │
  │  Step 5: Update status → AUTHORIZED                      │
  │  Step 6: Send webhook to merchant                        │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Payment State Machine

```
  ┌──────────┐   authorize   ┌────────────┐   capture    ┌──────────┐
  │ CREATED  │──────────────▶│ AUTHORIZED │─────────────▶│ CAPTURED │
  └──────────┘               └────────────┘              └──────────┘
       │                          │                           │
       │ decline                  │ void                      │ refund
       ▼                          ▼                           ▼
  ┌──────────┐              ┌──────────┐               ┌──────────┐
  │ DECLINED │              │  VOIDED  │               │ REFUNDED │
  └──────────┘              └──────────┘               └──────────┘
                                                            │
                                                            │ partial
                                                            ▼
                                                     ┌──────────────┐
                                                     │ PARTIALLY    │
                                                     │ REFUNDED     │
                                                     └──────────────┘

  Settlement (end of day):
  CAPTURED → SETTLED (funds transferred to merchant bank)
```

---

## 5. Ledger Design (Double-Entry Bookkeeping)

```
  Every financial movement has TWO entries:

  ┌──────────────────────────────────────────────────┐
  │  Transaction: Payment of $100 from Customer     │
  │                                                  │
  │  Entry 1: DEBIT  Accounts Receivable  $100.00   │
  │  Entry 2: CREDIT Revenue             $100.00    │
  │                                                  │
  │  Rule: Sum of all debits = Sum of all credits   │
  │        Balance ALWAYS = 0                        │
  └──────────────────────────────────────────────────┘

  Ledger table:
  ┌────────────────────────────────────────────┐
  │  entry_id        │ BIGINT (PK)            │
  │  transaction_id  │ UUID                    │
  │  account_id      │ BIGINT                  │
  │  amount          │ DECIMAL(19,4)           │
  │  direction       │ DEBIT / CREDIT          │
  │  currency        │ VARCHAR(3)              │
  │  created_at      │ TIMESTAMP               │
  │  description     │ TEXT                    │
  └────────────────────────────────────────────┘

  Important:
  • Use DECIMAL, never FLOAT for money
  • Integer cents ($100.00 → 10000) to avoid rounding
  • Immutable entries — never update, only append
  • Reconcile daily: sum(debits) must equal sum(credits)
```

---

## 6. Handling Network Failures

```
  Problem: Merchant sends payment request, network times out.
  Did the payment go through?

  ┌──────────────────────────────────────────────────┐
  │  Solution: Idempotency Key                       │
  │                                                  │
  │  Merchant retries with SAME idempotency_key      │
  │  Server checks:                                  │
  │    Key exists + status=COMPLETED → return result │
  │    Key exists + status=PENDING   → wait/retry    │
  │    Key not found → process normally              │
  │                                                  │
  │  Implementation (PostgreSQL):                    │
  │  INSERT INTO idempotency_keys                    │
  │    (key, status, response)                       │
  │  ON CONFLICT (key) DO NOTHING;                   │
  │                                                  │
  │  If INSERT succeeds → new request → process      │
  │  If INSERT fails (conflict) → duplicate          │
  │    → return stored response                      │
  └──────────────────────────────────────────────────┘

  Reconciliation for edge cases:
  ┌──────────────────────────────────────────────────┐
  │  Daily batch job:                                │
  │  1. Fetch all our AUTHORIZED transactions       │
  │  2. Fetch settlement report from card network    │
  │  3. Compare → find mismatches                   │
  │  4. Auto-resolve or flag for manual review       │
  └──────────────────────────────────────────────────┘
```

---

## 7. Fraud Detection

```
  Two-layer approach:

  Layer 1: Rules Engine (real-time, < 10ms)
  ┌──────────────────────────────────────────────┐
  │  • Transaction > $10,000 → FLAG              │
  │  • > 5 transactions in 1 minute → FLAG       │
  │  • Card used in 2 countries in 1 hour → FLAG │
  │  • First-time merchant + high amount → FLAG  │
  │  • BIN/IP on blocklist → DECLINE             │
  └──────────────────────────────────────────────┘

  Layer 2: ML Model (real-time, < 50ms)
  ┌──────────────────────────────────────────────┐
  │  Features:                                   │
  │  • Transaction amount, time of day           │
  │  • Customer history (avg amount, frequency)  │
  │  • Device fingerprint                        │
  │  • IP geolocation vs billing address          │
  │  • Merchant category code                    │
  │                                              │
  │  Model: Gradient Boosted Trees (XGBoost)     │
  │  Output: fraud_score 0.0 - 1.0               │
  │  Threshold: > 0.8 → DECLINE                  │
  │             0.5 - 0.8 → REVIEW               │
  │             < 0.5 → APPROVE                  │
  └──────────────────────────────────────────────┘
```

---

## 8. PCI-DSS Compliance

```
  Tokenization:
  ┌──────────────────────────────────────────────┐
  │  Card number never stored in payment service │
  │                                              │
  │  Client → Stripe.js → Stripe Vault           │
  │           ↓                                  │
  │           token: "tok_abc123"                │
  │           ↓                                  │
  │  Merchant server only sees token             │
  │  Token used for: authorize, capture, refund  │
  │                                              │
  │  PCI scope:                                  │
  │  Tokenization vault (Level 1 PCI)            │
  │  Payment service (SAQ-A: minimal scope)      │
  └──────────────────────────────────────────────┘
```

---

## 9. Fault Tolerance

- **Database:** PostgreSQL with synchronous replication; promote standby on failure
- **Payment gateway timeout:** Retry with same idempotency key; reconcile with network
- **Ledger consistency:** Transactions wrapped in DB transactions; two entries atomically
- **Split brain prevention:** Only one writer (leader) for ledger; Raft-based leader election
- **Disaster recovery:** Cross-region standby; RPO=0 (sync replication), RTO < 1 minute

---

## 10. Low-Level Design (LLD)

### API Contracts

```
# Create Payment Intent
POST /api/v1/payments
Headers: Idempotency-Key: "ik_abc123"
{
  "amount": 9999,              // cents (integer, never float)
  "currency": "USD",
  "payment_method": "pm_card_visa_4242",
  "customer_id": "cust_123",
  "metadata": {"order_id": "ord_456"},
  "capture_method": "automatic"  // automatic | manual
}
Response: {
  "payment_id": "pay_xyz789",
  "status": "requires_confirmation",
  "amount": 9999,
  "client_secret": "pay_xyz789_secret_abc"
}

# Confirm Payment (client-side 3DS)
POST /api/v1/payments/{payment_id}/confirm
{
  "return_url": "https://merchant.com/complete"
}

# Capture (for manual capture)
POST /api/v1/payments/{payment_id}/capture
{ "amount": 9999 }

# Refund
POST /api/v1/refunds
{
  "payment_id": "pay_xyz789",
  "amount": 5000,              // partial refund
  "reason": "customer_request"
}

# Webhook Delivery
POST https://merchant.com/webhooks
Headers: X-Signature: "sha256=..."
{
  "event": "payment.succeeded",
  "payment_id": "pay_xyz789",
  "amount": 9999,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### Database Schema (Ledger)

```sql
-- Payments table (source of truth)
CREATE TABLE payments (
    payment_id      UUID PRIMARY KEY,
    idempotency_key VARCHAR(255) UNIQUE,
    customer_id     UUID NOT NULL,
    amount          BIGINT NOT NULL,        -- cents
    currency        CHAR(3) NOT NULL,
    status          payment_status NOT NULL, -- ENUM
    payment_method  JSONB,
    processor_ref   VARCHAR(255),           -- PSP reference
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    captured_at     TIMESTAMPTZ,
    INDEX idx_customer (customer_id, created_at DESC),
    INDEX idx_idempotency (idempotency_key)
);

-- Double-entry ledger
CREATE TABLE ledger_entries (
    entry_id        BIGSERIAL PRIMARY KEY,
    payment_id      UUID REFERENCES payments,
    account_id      UUID NOT NULL,
    entry_type      VARCHAR(10) NOT NULL,   -- DEBIT | CREDIT
    amount          BIGINT NOT NULL,        -- always positive
    currency        CHAR(3) NOT NULL,
    balance_after   BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    -- Constraint: sum(CREDIT) = sum(DEBIT) per payment_id
    INDEX idx_account_balance (account_id, created_at DESC)
);

-- Idempotency store (prevents double-charge)
CREATE TABLE idempotency_keys (
    key             VARCHAR(255) PRIMARY KEY,
    payment_id      UUID,
    response_code   INT,
    response_body   JSONB,
    created_at      TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ     -- TTL: 24 hours
);
```

### Payment Processing Internals

```
Payment Authorization Flow:
┌──────────────────────────────────────────────────────────┐
│ 1. Idempotency Check                                    │
│    SELECT * FROM idempotency_keys WHERE key = ?          │
│    If exists → return cached response (no re-processing)│
├──────────────────────────────────────────────────────────┤
│ 2. Fraud Check (async, <50ms SLA)                       │
│    Risk score = ML model(amount, merchant, device,      │
│                          velocity, geo)                  │
│    If score > 0.8 → DECLINE                             │
│    If score > 0.5 → 3DS CHALLENGE                       │
├──────────────────────────────────────────────────────────┤
│ 3. PSP Router (select processor)                        │
│    Strategy: lowest_cost | highest_auth_rate | failover │
│    Route: Stripe (primary) → Adyen (backup)             │
│    Timeout: 5s per PSP, retry on 5xx                    │
├──────────────────────────────────────────────────────────┤
│ 4. Authorization                                        │
│    Card Network: Visa/MC → Issuing Bank                 │
│    Response: APPROVED / DECLINED / PENDING_3DS          │
├──────────────────────────────────────────────────────────┤
│ 5. Ledger Write (within DB transaction)                 │
│    BEGIN;                                                │
│      INSERT payments (status=AUTHORIZED);               │
│      INSERT ledger_entry (DEBIT customer_escrow);       │
│      INSERT ledger_entry (CREDIT merchant_pending);     │
│      INSERT idempotency_keys (response);                │
│    COMMIT;                                              │
├──────────────────────────────────────────────────────────┤
│ 6. Webhook Dispatch                                     │
│    Kafka → webhook_worker → POST merchant endpoint      │
│    Retry: exponential backoff, 3 days max               │
└──────────────────────────────────────────────────────────┘
```

## 11. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Transaction Volume** | Shard payments DB by customer_id (Vitess/CockroachDB) | 100K TPS |
| **Ledger Growth** | Time-partitioned ledger tables; archive old entries to cold storage | 10Bn entries |
| **PSP Failover** | Smart routing: primary + 2 backup processors per region | 99.99% auth availability |
| **Webhook Delivery** | Partitioned Kafka topics; independent consumer groups per merchant | 1M webhooks/min |
| **Fraud ML** | Feature store pre-computation; model inference <5ms via ONNX | Real-time at 100K TPS |

**Read Scaling:**
```
Read Patterns (handled by replicas):
  - Transaction history: read replica + Redis cache (TTL 30s)
  - Merchant dashboard: materialized views, updated every 5min
  - Reconciliation: dedicated analytics replica (never serves live traffic)
  - Reporting: OLAP warehouse (BigQuery/Redshift), ETL every 15min
```

## 12. No Data Loss

| Component | Protection Mechanism |
|-----------|---------------------|
| **Payment State** | PostgreSQL synchronous replication (primary + sync standby); ACK only after both confirm |
| **Ledger Integrity** | Double-entry invariant enforced in DB constraints; nightly reconciliation job verifies sum(DEBIT) = sum(CREDIT) |
| **Idempotency Keys** | Persisted before processing; crash-safe: replay returns same result |
| **Webhook Events** | Kafka acks=all (RF=3); consumer offset committed only after successful delivery |
| **Audit Trail** | Append-only audit log with hash chain (tamper-evident); archived to immutable storage |
| **Backups** | Continuous WAL archiving to S3; PITR granularity = 1 second; daily cross-region backup |

**Reconciliation:**
```
Daily Reconciliation Pipeline:
  1. Internal ledger balance vs PSP settlement reports
  2. Delta = internal_amount - psp_amount per transaction
  3. If |delta| > 0: flag for manual review
  4. Output: reconciliation_report_{date}.csv
  5. Unresolved after 72h → escalate to finance team
  
  Monthly: external auditor verifies ledger integrity
```

## 13. Latency

| Operation | p50 | p99 | Target |
|-----------|-----|-----|--------|
| Payment authorization | 200ms | 800ms | <1s |
| Fraud check | 10ms | 50ms | <100ms |
| Idempotency lookup | 1ms | 5ms | <10ms |
| Refund processing | 150ms | 500ms | <1s |
| Webhook delivery (first attempt) | 300ms | 2s | <5s |

**Optimization Techniques:**
- **Parallel fraud + routing**: Run fraud check and PSP selection concurrently
- **Connection pooling**: Persistent HTTPS connections to PSPs (avoid TLS handshake per request)
- **Prepared statements**: Pre-compile ledger INSERT queries
- **Async webhook**: Decouple from payment path; enqueue to Kafka immediately
- **Regional processing**: Process payments in nearest region to customer's issuing bank

## 14. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **PSP timeout** | Authorization delayed | Circuit breaker: failover to backup PSP after 5s; retry policy per PSP |
| **DB primary crash** | Writes blocked | Synchronous standby promoted in <30s; no data loss due to sync replication |
| **Double-charge** | Customer overcharged | Idempotency key prevents reprocessing; reconciliation catches mismatches |
| **Webhook endpoint down** | Merchant not notified | Exponential retry for 72h; merchant can poll GET /payments/{id} |
| **Fraud model stale** | False approvals | Fallback to rule-based engine; velocity checks always active |
| **Network partition** | Split-brain risk | Fencing tokens on DB writes; only one primary can accept writes |

## 15. Availability

**Target: 99.999% for payment processing (5.26 min downtime/year)**

```
Availability Architecture:
┌──────────────────────────────────────────────┐
│            Global Payment Router             │
│     (Route by card BIN to nearest region)    │
├──────────────────┬───────────────────────────┤
│   US Region      │     EU Region             │
│ ┌──────────────┐ │ ┌──────────────┐          │
│ │ API Gateway  │ │ │ API Gateway  │          │
│ │ (active)     │ │ │ (active)     │          │
│ ├──────────────┤ │ ├──────────────┤          │
│ │ Payment Svc  │ │ │ Payment Svc  │          │
│ │ (3 replicas) │ │ │ (3 replicas) │          │
│ ├──────────────┤ │ ├──────────────┤          │
│ │ PostgreSQL   │ │ │ PostgreSQL   │          │
│ │ Primary +    │ │ │ Primary +    │          │
│ │ Sync Standby │ │ │ Sync Standby │          │
│ └──────────────┘ │ └──────────────┘          │
│                  │                           │
│ PSPs: Stripe,    │ PSPs: Adyen, Stripe,      │
│ Braintree        │ Worldpay                  │
└──────────────────┴───────────────────────────┘

Maintenance Strategy:
  - Zero-downtime deployments (blue-green)
  - PSP maintenance windows: auto-route to backup PSP
  - DB schema migrations: backward-compatible, online DDL
```

## 16. Security

| Layer | Mechanism |
|-------|-----------|
| **PCI-DSS L1** | Card data never touches application servers; tokenized via PSP's SDK |
| **Encryption** | TLS 1.3 everywhere; AES-256 at rest for PII; column-level encryption for card tokens |
| **Tokenization** | Raw PAN → PSP token (tok_xxxx); only PSP can detokenize |
| **Authentication** | mTLS between services; API keys + HMAC signatures for merchants |
| **Webhooks** | HMAC-SHA256 signature per event; merchant verifies before processing |
| **Fraud Prevention** | ML model + rule engine; velocity limits; 3DS for high-risk transactions |
| **Access Control** | SOC2: role-based access to production; all access logged; break-glass for emergencies |
| **Data Retention** | Card data: PSP retains; transaction logs: 7 years (regulatory); PII: GDPR right-to-erasure |

## 17. Cost Constraints

**Estimated Cost (1M transactions/day, $100M GMV/month):**

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| **PSP Fees** | 2.9% + $0.30 per transaction (Stripe) | $2,900,000 (pass-through) |
| **Infrastructure** | 20× c6g.xlarge (API/workers) | $8,000 |
| **Database** | RDS PostgreSQL Multi-AZ (r6g.4xlarge) | $6,000 |
| **Kafka** | MSK 6-broker cluster | $4,000 |
| **Fraud ML** | SageMaker inference endpoints | $3,000 |
| **Compliance/Auditing** | PCI-DSS audit, SOC2, penetration testing | $15,000 (amortized) |
| **Total (infra only)** | | **~$36,000/month** |

**Cost Optimization:**
- **PSP negotiation**: Volume discounts at $10M+/month GMV → 2.5% + $0.25
- **Smart routing**: Route to cheapest PSP that has highest auth rate for card type
- **Interchange++**: Negotiate interchange-plus pricing vs blended rates — saves 0.2-0.5%
- **Batch settlements**: Reduce per-transaction fees by batching captures
- **Fraud model accuracy**: Each 0.1% false-positive reduction = $100K/year saved in declined valid transactions

## Key Interview Discussion Points

1. **How to prevent double-charging?** — Idempotency keys; server deduplicates before processing
2. **Why double-entry bookkeeping?** — Every debit has a matching credit; makes auditing trivial; balance always zero
3. **How to handle partial failures?** — Saga pattern: compensating transactions (reverse auth if capture fails)
4. **Why not FLOAT for money?** — Floating point rounding errors accumulate; use DECIMAL or integer cents
5. **How to scale ledger?** — Shard by account_id; each shard has independent double-entry integrity; cross-shard transfers use 2PC or saga
