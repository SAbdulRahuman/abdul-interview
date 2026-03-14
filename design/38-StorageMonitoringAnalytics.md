# Design a Storage Monitoring and Analytics Platform

## Overview
Design a storage infrastructure monitoring platform like NetApp Active IQ / Cloud Insights — collecting telemetry from thousands of storage systems, predicting failures, providing capacity forecasting, and generating actionable recommendations through ML-powered analytics.

## 1. Requirements

**Functional:**
- Collect performance metrics (IOPS, latency, throughput) every 5-60 seconds
- Capacity trending, forecasting (days-to-full prediction)
- Health scoring per system/volume/disk
- Anomaly detection (performance regression, unusual patterns)
- Risk assessment: firmware vulnerabilities, EOL components, known issues
- Recommendation engine: optimization, tiering, right-sizing

**Non-Functional:**
- Metric ingestion: 1M metrics/second
- Dashboard latency: <2s for 30-day range queries
- Alerting latency: <60s from event to notification
- Retention: raw 90 days, downsampled 3 years
- Availability: 99.9% (monitoring SaaS platform)

## 2. Scale Estimation

| Parameter | Value |
|-----------|-------|
| **Monitored systems** | 10,000 storage controllers |
| **Metrics per system** | 5,000 time series (volumes, disks, ports, aggregates) |
| **Total time series** | 50 million |
| **Ingestion rate** | 1M data points/second |
| **Daily data volume** | ~200GB (compressed TSDB) |
| **Hot data (90 days)** | 18TB |
| **Historical (3 years)** | 50TB (downsampled) |

## 3. High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│       Storage Monitoring & Analytics Platform                     │
│                                                                   │
│  Storage Systems (on-prem):                                      │
│  ┌────────┐ ┌────────┐ ┌────────┐                               │
│  │ONTAP-1 │ │ONTAP-2 │ │E-Series│                               │
│  │  Agent  │ │  Agent  │ │  Agent  │                               │
│  └───┬────┘ └───┬────┘ └───┬────┘                               │
│      │          │          │                                      │
│      └──────────┴──────────┘                                     │
│               │ HTTPS (AutoSupport/ASUP)                         │
│               ▼                                                   │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  Ingestion Layer                                         │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │     │
│  │  │ API Gateway  │  │ Kafka Cluster │  │ Stream Proc  │  │     │
│  │  │ (Rate Limit) │─►│ (Buffer)     │─►│ (Flink)      │  │     │
│  │  └──────────────┘  └──────────────┘  └──────┬───────┘  │     │
│  └─────────────────────────────────────────────┼───────────┘     │
│                                                 │                 │
│  ┌─────────────────────────────────────────────┼───────────┐     │
│  │  Storage & Query Layer                      ▼           │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │     │
│  │  │ TimescaleDB/ │  │ Config/Asset │  │ Alert Engine │  │     │
│  │  │ VictoriaM.   │  │ PostgreSQL   │  │              │  │     │
│  │  │ (metrics)    │  │ (inventory)  │  │              │  │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  Analytics Layer                                         │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │     │
│  │  │ ML Pipeline  │  │ Forecasting  │  │ Recommend.   │  │     │
│  │  │ (Anomaly)    │  │ (Prophet)    │  │ Engine       │  │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  Presentation Layer                                      │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │     │
│  │  │ Dashboard UI │  │ REST API     │  │ Mobile App   │  │     │
│  │  │ (React+D3)   │  │ (GraphQL)    │  │              │  │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │     │
│  └─────────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────┘
```

## 4. Metric Collection & Pipeline

```
AutoSupport (ASUP) Message Structure:
┌──────────────────────────────────────────────────┐
│ ASUP from cluster: prod-ontap-01                  │
│ Timestamp: 2024-01-15T12:00:00Z                  │
│ Type: performance (every 5 min)                  │
│                                                  │
│ System Metrics:                                  │
│   cpu_busy: 45%                                  │
│   memory_used: 72%                               │
│   disk_busy_max: 38%                             │
│                                                  │
│ Volume Metrics (per volume, 200 volumes):        │
│   vol_prod_data1:                                │
│     read_iops: 15000                             │
│     write_iops: 5000                             │
│     read_latency_us: 250                         │
│     write_latency_us: 400                        │
│     space_used_gb: 800                           │
│     space_available_gb: 200                      │
│                                                  │
│ Disk Health (per disk, 96 disks):                │
│   disk_0a.00.0:                                  │
│     media_errors: 0                              │
│     predicted_life_remaining: 92%                │
│     temperature_c: 38                            │
│     power_on_hours: 12450                        │
│                                                  │
│ Events:                                          │
│   [WARN] aggr1 85% full                          │
│   [INFO] firmware update available: 9.12.1P3     │
└──────────────────────────────────────────────────┘

Stream Processing (Flink):
  1. Parse ASUP → extract individual metrics
  2. Enrich with asset metadata (cluster name, tier, customer)
  3. Compute aggregations: 5min → 1hr → 1day rollups
  4. Run anomaly detection (online)
  5. Route to: TSDB (metrics), PostgreSQL (inventory), Alert Engine
```

## 5. Anomaly Detection & Forecasting

```
Anomaly Detection Pipeline:

  Model: Isolation Forest + seasonal decomposition

  class AnomalyDetector:
    def detect(self, metric_series, window="7d"):
      # 1. Remove seasonality (daily, weekly patterns)
      trend, seasonal, residual = seasonal_decompose(
        metric_series, period=288  # 5-min intervals per day
      )
      
      # 2. Isolation Forest on residuals
      model = IsolationForest(contamination=0.01)
      scores = model.fit_predict(residual)
      
      # 3. Contextual check: is this anomaly actionable?
      for anomaly in scores.anomalies:
        severity = self.assess_severity(anomaly, metric_series)
        if severity > threshold:
          yield Alert(metric=metric_series.name,
                     value=anomaly.value,
                     expected_range=anomaly.expected,
                     severity=severity)

Capacity Forecasting:

  Model: Facebook Prophet (handles seasonality + trends)
  
  def forecast_days_to_full(volume):
    history = get_space_used_history(volume, days=90)
    
    model = Prophet(
      yearly_seasonality=False,
      weekly_seasonality=True,    # batch jobs on weekends
      changepoint_prior_scale=0.1 # trend sensitivity
    )
    model.fit(history)
    
    future = model.predict(periods=365)  # forecast 1 year
    
    capacity = volume.total_capacity
    days_to_full = find_first_crossing(future, capacity)
    
    return {
      "days_to_full": days_to_full,
      "confidence_interval": (days_low, days_high),
      "growth_rate_gb_per_day": trend_slope,
      "recommendation": generate_recommendation(days_to_full)
    }
```

## 6. Low-Level Design (LLD)

### API Contracts

```
# Query Metrics
POST /api/v1/metrics/query
{
  "query": "avg(volume_read_latency_us{cluster='prod-01'})",
  "start": "2024-01-01T00:00:00Z",
  "end": "2024-01-15T00:00:00Z",
  "step": "1h"                   // rollup interval
}

# Get Health Score
GET /api/v1/systems/{cluster_id}/health
→ {
  "overall_score": 87,
  "components": {
    "performance": 92,
    "capacity": 75,              // nearing thresholds
    "protection": 95,
    "configuration": 88
  },
  "risks": [
    {"severity": "high", "msg": "aggr1 projected full in 45 days"},
    {"severity": "medium", "msg": "firmware CVE-2024-1234 unpatched"}
  ]
}

# Get Recommendations
GET /api/v1/systems/{cluster_id}/recommendations
→ {
  "recommendations": [
    {
      "type": "capacity",
      "priority": "high",
      "title": "Expand aggr1 or tier cold data",
      "detail": "Add 10TB or enable FabricPool to tier 3TB cold data",
      "estimated_savings": "$5,000/month with tiering"
    },
    {
      "type": "performance",
      "priority": "medium",
      "title": "Move vol_db1 to SSD aggregate",
      "detail": "Current latency 8ms avg; SSD would reduce to <500μs"
    }
  ]
}

# Configure Alert Rule
POST /api/v1/alerts/rules
{
  "name": "High Latency",
  "condition": "volume_write_latency_us > 5000 for 10m",
  "severity": "critical",
  "notification": ["pagerduty", "slack:#storage-alerts"],
  "suppression": {
    "during_maintenance": true,
    "cooldown_minutes": 30
  }
}
```

### Time-Series Storage Schema

```
TSDB Schema (VictoriaMetrics/TimescaleDB):

Metric Format (OpenMetrics):
  storage_volume_iops{
    cluster="prod-01",
    svm="svm1",
    volume="db_data",
    type="read",
    tier="ssd"
  } 15000 1705312800

Rollup Strategy:
  Raw (5-min):    keep 90 days     → 200GB/day
  1-hour avg:     keep 1 year      → ~8GB/day
  1-day avg:      keep 3 years     → ~0.3GB/day
  
Cardinality Management:
  - 50M active time series
  - Label combinations: cluster × svm × volume × metric × type
  - Inverted index for fast label lookups
  - Compression: Gorilla (timestamp) + XOR (value) → ~1.5 bytes/point
```

## 7. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Ingestion** | Kafka partitioned by cluster_id; Flink parallelism | 1M metrics/sec |
| **Storage** | TSDB sharded by metric group; tiered to object storage | 100TB total |
| **Query** | Pre-computed rollups; materialized views for dashboards | <2s for 30-day range |
| **Monitored systems** | Horizontal scaling of collectors and stream processors | 100K+ systems |
| **Alerting** | Distributed alert evaluation per shard; dedup via grouping | 10K active rules |

## 8. No Data Loss

| Scenario | Protection |
|----------|-----------|
| **ASUP delivery failure** | Agent retries with exponential backoff; local buffer (72h) |
| **Kafka broker failure** | Replication factor 3; ISR-based ack |
| **TSDB crash** | WAL replay; TSDB replication across zones |
| **Metric gap** | Gap detection in stream processor; interpolation for dashboards; raw gap preserved |

## 9. Latency

| Operation | Target |
|-----------|--------|
| **Metric ingestion to queryable** | <30 seconds |
| **Alert detection** | <60 seconds from event |
| **Dashboard rendering (30-day)** | <2 seconds |
| **Forecast computation** | <10 seconds per volume |
| **Full system health score** | <5 seconds |

## 10. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Collector agent failure** | Metrics gap for one system | Heartbeat monitoring; auto-restart; alert on missed ASUP |
| **Stream processor failure** | Processing delay | Flink checkpointing; Kafka replay from last offset |
| **ML model drift** | False positive anomalies | Regular retraining; feedback loop from user acknowledgments |
| **Alert storm** | Operator fatigue | Alert grouping, dedup, and suppression; rate limiting notifications |

## 11. Availability

**Target: 99.9% for SaaS monitoring platform**

```
Architecture:
  - Multi-AZ deployment (3 availability zones)
  - Ingestion: Kafka across AZs → survives AZ failure
  - TSDB: replicated across AZs
  - Dashboard: CDN + multi-region API servers
  
  Degraded mode:
    TSDB partial outage → serve from cache + stale data indicator
    ML service down → disable recommendations; monitoring continues
```

## 12. Security

| Layer | Mechanism |
|-------|-----------|
| **Data in transit** | TLS 1.3 for ASUP upload; mutual TLS for agent-to-platform |
| **Data at rest** | AES-256 encryption for TSDB and PostgreSQL |
| **Authentication** | SAML/OIDC for web UI; API keys for programmatic access |
| **Authorization** | RBAC: admin, operator, viewer roles per customer/cluster |
| **Data privacy** | PII stripped from ASUPs; no customer data content — only metadata/metrics |
| **Multi-tenancy** | Hard isolation: separate TSDB namespaces per customer |

## 13. Cost Constraints

**Estimated Cost (10K monitored systems, SaaS):**

| Component | Cost/Month |
|-----------|------------|
| **Kafka cluster** | 9 brokers: $5,400 |
| **Flink cluster** | 20 workers: $6,000 |
| **TSDB (VictoriaMetrics)** | 10 nodes + 100TB storage: $8,000 |
| **PostgreSQL (inventory)** | 2 nodes HA: $1,200 |
| **ML compute** | GPU for training + CPU for inference: $3,000 |
| **CDN + API servers** | $2,000 |
| **Total** | **~$25,600/month** |

Revenue model: $5-15/system/month → $50K-150K/month (profitable at 5K+ systems).

## Key Interview Discussion Points

1. **Why Kafka before TSDB?** — Decouple ingestion from storage; handle burst traffic; replay capability for reprocessing; fan-out to multiple consumers (TSDB + alerts + ML)
2. **How to handle 50M time series?** — Inverted index for label lookups; Gorilla compression (1.5 bytes/point); partitioned by time + metric group; rollup reduces cardinality for old data
3. **Anomaly detection: supervised vs unsupervised?** — Unsupervised (Isolation Forest) because labeled anomaly data is scarce. Supplement with rule-based alerts for known patterns
4. **Alert fatigue mitigation?** — Grouping (same root cause → 1 alert), suppression windows, cooldown periods, severity escalation, and feedback loops from operators
5. **Capacity forecasting accuracy?** — Prophet handles seasonality well; accuracy degrades beyond 90 days. Confidence intervals communicate uncertainty. Re-forecast weekly
