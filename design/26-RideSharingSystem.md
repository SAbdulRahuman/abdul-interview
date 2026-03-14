# Design a Ride-Sharing System (Uber/Lyft)

Examples: Uber, Lyft, Grab, DiDi

---

## 1. Requirements

### Functional
- Rider requests a ride with pickup and dropoff locations
- Match rider to nearest available driver
- Real-time driver location tracking
- Dynamic pricing (surge pricing)
- ETA estimation with live traffic
- Trip lifecycle management
- Payment integration

### Non-Functional
- Match rider to driver in < 10 seconds
- Location updates every 3-5 seconds from drivers
- Handle 1M+ concurrent rides
- 99.99% dispatch availability
- Support peak events (New Year's Eve: 10× normal)

---

## 2. Scale Estimation

```
Active drivers: 5M worldwide
Concurrent rides: 1M
Location updates: 5M drivers × 1 update/4s = 1.25M updates/sec
Ride requests: ~10K requests/sec (peak)
ETA queries: ~50K/sec
Trip data/day: 20M trips × 5 KB = 100 GB/day
```

---

## 3. High-Level Architecture

```
  ┌──────────┐              ┌──────────┐
  │  Rider   │              │  Driver  │
  │  App     │              │  App     │
  └────┬─────┘              └────┬─────┘
       │                         │
       │  ride request           │  location updates (every 4s)
       │                         │
  ┌────▼─────────────────────────▼────────────────────┐
  │                   API Gateway                      │
  │  (auth, rate limit, request routing)               │
  └──────────┬──────────────────┬─────────────────────┘
             │                  │
  ┌──────────▼──────┐   ┌──────▼──────────────────┐
  │  Ride Service   │   │  Location Service       │
  │  (trip state    │   │  (ingest, store,         │
  │   machine)      │   │   query nearby drivers)  │
  └────────┬────────┘   └──────────┬──────────────┘
           │                       │
  ┌────────▼────────┐   ┌──────────▼──────────────┐
  │  Dispatch       │   │  Geospatial Index       │
  │  Service        │◀──│  (drivers by location)  │
  │  (matching)     │   │  Geohash / S2 Cells     │
  └────────┬────────┘   └─────────────────────────┘
           │
  ┌────────▼────────┐   ┌─────────────────────────┐
  │  Pricing        │   │  ETA Service            │
  │  Service        │   │  (routing engine)       │
  │  (surge)        │   │                         │
  └─────────────────┘   └─────────────────────────┘
```

---

## 4. Geospatial Indexing

```
  How to find nearest drivers efficiently?

  GEOHASH:
  ┌──────────────────────────────────────────────┐
  │  Divide the world into grid cells            │
  │  Each cell has a string prefix               │
  │                                              │
  │  Precision 6: ~1.2km × 0.6km cells          │
  │                                              │
  │  ┌────┬────┬────┬────┐                       │
  │  │9xj6│9xj7│9xj8│9xj9│                      │
  │  ├────┼────┼────┼────┤                       │
  │  │9xj2│9xj3│9xj4│9xj5│                      │
  │  ├────┼────┼────┼────┤                       │
  │  │9xjA│9xjB│9xjC│9xjD│  ← driver is here   │
  │  └────┴────┴────┴────┘                       │
  │                                              │
  │  Query: "drivers in cell 9xjC and neighbors" │
  │  → 9 cells to search                         │
  │                                              │
  │  Redis: GEOADD drivers:city_id lng lat id    │
  │         GEORADIUS drivers:city_id lng lat 5km│
  └──────────────────────────────────────────────┘

  S2 CELLS (Google's approach):
  ┌──────────────────────────────────────────────┐
  │  Projects Earth onto faces of a cube         │
  │  Hierarchical cell IDs (level 0-30)          │
  │  Level 12: ~3.3km² cells                     │
  │  Level 16: ~153m² cells                      │
  │                                              │
  │  Pro: Uniform cell sizes (unlike geohash)    │
  │  Pro: Efficient containment/intersection     │
  │  Used by: Uber H3 (hexagonal variant)        │
  └──────────────────────────────────────────────┘

  Location update flow:
  Driver → Location Service → Update geospatial index
  1.25M updates/sec → sharded by city/region
```

---

## 5. Driver-Rider Matching (Dispatch)

```
  Rider requests ride:
  ┌──────────────────────────────────────────────────┐
  │  1. Get rider location and destination           │
  │  2. Query geospatial index:                      │
  │     Find available drivers within 5km radius     │
  │     → returns [D1: 0.3km, D2: 0.8km, D3: 1.2km]│
  │                                                  │
  │  3. Score each candidate:                        │
  │     ┌─────────┬──────┬──────┬───────┐            │
  │     │ Driver  │ Dist │ ETA  │ Score │            │
  │     ├─────────┼──────┼──────┼───────┤            │
  │     │ D1      │ 0.3km│ 2min │ 0.95  │            │
  │     │ D2      │ 0.8km│ 4min │ 0.82  │            │
  │     │ D3      │ 1.2km│ 3min │ 0.85  │            │
  │     └─────────┴──────┴──────┴───────┘            │
  │     Score = f(distance, ETA, driver_rating,       │
  │              vehicle_type, acceptance_rate)       │
  │                                                  │
  │  4. Send ride offer to best driver (D1)          │
  │     Timeout: 15 seconds to accept                │
  │     If declined/timeout → offer to D3 → D2       │
  │                                                  │
  │  5. Driver accepts → trip created                │
  │     Mark driver as unavailable in index          │
  └──────────────────────────────────────────────────┘

  Preventing double-dispatch:
  ┌──────────────────────────────────────────────┐
  │  Optimistic locking:                         │
  │  SET driver:{id}:status "dispatched"         │
  │    NX (only if not exists / not dispatched)  │
  │  If SET fails → driver already taken         │
  │  → offer to next candidate                  │
  └──────────────────────────────────────────────┘
```

---

## 6. Trip State Machine

```
  ┌──────────┐  request  ┌──────────┐  accept   ┌──────────┐
  │ REQUESTED│─────────▶│ MATCHING │─────────▶│ ACCEPTED │
  └──────────┘          └──────────┘          └──────────┘
                             │                     │
                        no drivers             driver arrives
                             │                     │
                             ▼                     ▼
                        ┌──────────┐          ┌──────────┐
                        │ CANCELLED│          │ ARRIVED  │
                        └──────────┘          └──────────┘
                                                   │
                                              rider boards
                                                   │
                                                   ▼
                                              ┌──────────┐
                                              │  TRIP    │
                                              │  STARTED │
                                              └──────────┘
                                                   │
                                              reach dest.
                                                   │
                                                   ▼
                                              ┌──────────┐
                                              │ COMPLETED│
                                              └──────────┘
                                                   │
                                              payment
                                                   │
                                                   ▼
                                              ┌──────────┐
                                              │  PAID    │
                                              └──────────┘
```

---

## 7. Dynamic Pricing (Surge)

```
  Surge pricing algorithm:
  ┌──────────────────────────────────────────────┐
  │  For each hex cell (H3):                     │
  │  demand = ride_requests in last 5 min        │
  │  supply = available_drivers in cell          │
  │                                              │
  │  ratio = demand / supply                     │
  │                                              │
  │  if ratio > 2.0: surge = 1.5×               │
  │  if ratio > 3.0: surge = 2.0×               │
  │  if ratio > 5.0: surge = 3.0×               │
  │                                              │
  │  Smooth transitions:                         │
  │  Don't jump from 1× to 3× instantly         │
  │  Apply exponential smoothing:                │
  │  surge_new = α × raw_surge + (1-α) × surge_old│
  │                                              │
  │  Show surge multiplier to rider BEFORE       │
  │  requesting ride (transparent pricing)       │
  └──────────────────────────────────────────────┘
```

---

## 8. ETA Estimation

```
  Road network as weighted graph:
  ┌──────────────────────────────────────────────┐
  │  Nodes: intersections                        │
  │  Edges: road segments                        │
  │  Weights: travel time (NOT distance)         │
  │                                              │
  │  Static weights: speed limit × road length   │
  │  Dynamic weights: live traffic data          │
  │                                              │
  │  Algorithm: A* or Contraction Hierarchies    │
  │  (not Dijkstra — too slow for global graph)  │
  │                                              │
  │  Hierarchical routing:                       │
  │  Local roads → arterials → highways          │
  │  Pre-compute highway-to-highway shortest     │
  │  paths; compute local connections at query   │
  │                                              │
  │  Live traffic adjustment:                    │
  │  Aggregate driver GPS traces → segment speed │
  │  Update edge weights every 2 minutes         │
  │  Confidence: more traces = higher confidence │
  └──────────────────────────────────────────────┘
```

---

## 9. Fault Tolerance

- **Dispatch service failure:** Active-passive HA; in-flight dispatches recovered from state store
- **Location service failure:** Sharded by city; failure affects only one region
- **Driver app disconnects:** Timeout → mark driver unavailable; reconnect → resume
- **Payment failure:** Trip marked COMPLETED; async payment retry; don't block rider
- **Peak load:** Pre-scale for known events; degrade: disable surge display, simplify matching

---

## 10. Low-Level Design (LLD)

### API Contracts

```
# Request Ride
POST /api/v1/rides/request
{
  "rider_id": "r_abc123",
  "pickup": {"lat": 37.7749, "lng": -122.4194},
  "dropoff": {"lat": 37.3382, "lng": -121.8863},
  "ride_type": "standard",     // standard | premium | pool
  "payment_method": "pm_card_1234"
}
Response: {
  "ride_id": "ride_xyz789",
  "status": "matching",
  "estimated_fare": {"min": 2500, "max": 3200, "currency": "USD"},
  "surge_multiplier": 1.5,
  "estimated_pickup_eta_sec": 240
}

# Driver Location Update (every 3-5 sec)
PUT /api/v1/drivers/{driver_id}/location
{
  "lat": 37.7750,
  "lng": -122.4180,
  "heading": 45,               // degrees
  "speed_kmh": 35,
  "timestamp": 1700000000,
  "ride_id": "ride_xyz789"     // null if idle
}

# Accept Ride (driver)
POST /api/v1/rides/{ride_id}/accept
Headers: X-Driver-Id: d_def456

# Complete Ride
POST /api/v1/rides/{ride_id}/complete
{
  "actual_route": [{"lat":..,"lng":..,"t":..}, ...],
  "distance_meters": 42500,
  "duration_sec": 2400
}

# ETA Estimation
GET /api/v1/eta?origin_lat=37.77&origin_lng=-122.42&dest_lat=37.34&dest_lng=-121.89
Response: {
  "eta_seconds": 2520,
  "distance_meters": 42500,
  "route_polyline": "encoded_polyline_string"
}

# Surge Pricing
GET /api/v1/surge?lat=37.77&lng=-122.42&radius_km=5
Response: {
  "surge_multiplier": 1.8,
  "demand_level": "high",
  "supply_drivers": 45,
  "demand_requests": 120
}
```

### Geospatial Index Internals

```
Geospatial Index (Redis + H3):

H3 Resolution 9 (~174m edge):
  Cell "892a1008003ffff" → {
    drivers: [
      {id: "d_001", lat: 37.7750, lng: -122.4180, status: "available", 
       heading: 45, updated_at: 1700000000},
      {id: "d_002", lat: 37.7752, lng: -122.4175, status: "available",
       heading: 180, updated_at: 1700000002}
    ]
  }

Redis Data Structures:
  # Driver locations (sorted set per H3 cell)
  GEOADD driver_locations:{h3_cell} lng lat driver_id
  
  # Driver status
  HSET driver:{driver_id} status available lat 37.775 lng -122.418
  
  # Find nearby drivers
  GEORADIUS driver_locations:{cell} lng lat 2000 m ASC COUNT 20
  + also query k-ring neighbor cells

Matching Algorithm:
┌──────────────────────────────────────────────────┐
│ 1. Compute pickup H3 cell (resolution 9)        │
│ 2. Get k-ring(cell, k=2) → 19 cells             │
│ 3. GEORADIUS in each cell → candidate drivers    │
│ 4. Filter: status=available, ride_type capable   │
│ 5. For each candidate:                           │
│    score = w1 × (1/ETA) +                        │
│            w2 × driver_rating +                  │
│            w3 × acceptance_rate +                │
│            w4 × heading_alignment                │
│ 6. Sort by score DESC                            │
│ 7. Dispatch to top driver (15s accept window)    │
│ 8. If declined → next driver (cascade)           │
│ 9. After 3 declines → expand radius (k=3)       │
└──────────────────────────────────────────────────┘
```

### Trip State Machine (Internal)

```
Trip Lifecycle (with DB transitions):

  REQUESTED ──dispatch──→ MATCHING ──accepted──→ DRIVER_ASSIGNED
       │                      │                        │
       │ (timeout 30s)        │ (no driver 60s)        │ (driver arrives)
       ▼                      ▼                        ▼
    CANCELLED             NO_DRIVERS              ARRIVED
                          (notify rider)              │
                                                      │ (rider boards)
                                                      ▼
                                                  IN_PROGRESS
                                                      │
                                         ┌────────────┤
                                         │            │
                                    (rider cancels)   │ (reaches dest)
                                         ▼            ▼
                                    CANCELLED     COMPLETED
                                    (cancel fee)      │
                                                      ▼
                                                  PAYMENT_PENDING
                                                      │
                                                      ▼
                                                  PAYMENT_COMPLETE

DB: rides table
  ride_id | rider_id | driver_id | status | pickup | dropoff |
  surge | fare_estimate | actual_fare | created_at | completed_at
```

## 11. Scalability

| Dimension | Strategy | Capacity |
|-----------|----------|----------|
| **Location Updates** | Redis cluster with H3-partitioned keys; 3-5 sec interval | 2M drivers × 0.25 RPS = 500K updates/sec |
| **Ride Matching** | Regional matching services; each handles one metro area | 10K matches/sec per region |
| **ETA Computation** | Pre-computed contraction hierarchies + live traffic overlay; cached routes | 100K ETA requests/sec |
| **Surge Calculation** | Aggregate demand/supply per H3 resolution 7 cell; recompute every 30s | Real-time for 1000 cities |
| **Trip History** | Cassandra partitioned by rider_id; time-bucketed | 1Bn trips/year |

**Geographic Scaling:**
```
Regional Architecture:
  Each city/metro = independent matching service
  
  City Launch Checklist:
    1. Deploy matching service in nearest region
    2. Load map data + traffic model for city
    3. Pre-compute contraction hierarchies
    4. Configure surge zones (airport, downtown, events)
    5. Set base fare + per-mile/min rates
  
  Cross-City:
    - Global account service (shared across cities)
    - Global payment service
    - Per-city: matching, surge, routing, dispatch
```

## 12. No Data Loss

| Component | Protection Mechanism |
|-----------|---------------------|
| **Ride State** | PostgreSQL with synchronous replication; state machine transitions are ACID |
| **Driver Location** | Redis with AOF persistence; locations are ephemeral but ride GPS trail persisted to Kafka |
| **Payment Records** | Double-entry ledger (same as Payment System); idempotency keys prevent double-charge |
| **Trip GPS Trail** | Driver app buffers GPS points; uploads in batches; server stores in Kafka → S3 |
| **Matching Decisions** | Logged to Kafka audit topic; replay for dispute resolution |

**Recovery:**
```
Matching Service Crash:
  1. In-flight ride requests in MATCHING state
  2. New service instance queries rides WHERE status='MATCHING' AND updated_at < now()-30s
  3. Re-runs matching algorithm for each
  4. Riders see brief delay (30s max)

Driver App GPS Gap:
  1. App buffers GPS points locally when offline
  2. On reconnect: uploads buffered points with timestamps
  3. Server fills in GPS trail gaps
  4. Fare calculated from actual GPS trail (not straight-line)
```

## 13. Latency

| Operation | p50 | p99 | Target |
|-----------|-----|-----|--------|
| Location update ingestion | 5ms | 20ms | <50ms |
| Find nearest drivers | 10ms | 50ms | <100ms |
| Ride matching (end-to-end) | 3s | 15s | <30s |
| ETA calculation | 20ms | 100ms | <200ms |
| Surge price lookup | 5ms | 20ms | <50ms |
| Map tile loading | 50ms | 200ms | <500ms |

**Optimization Techniques:**
- **H3 indexing**: O(1) cell lookup + k-ring neighbor scan (vs R-tree range query)
- **ETA caching**: Cache popular routes (airport→downtown); invalidate on traffic change
- **Contraction hierarchies**: Pre-compute shortcuts in road graph → 1000× faster routing
- **Optimistic dispatch**: Start routing driver while rider confirms; cancel if rider declines
- **Batch location updates**: Driver app batches 3-5 GPS points per request (reduce API calls)
- **Edge computing**: Matching service deployed in local PoP per metro area

## 14. Reliability

| Failure Mode | Impact | Mitigation |
|--------------|--------|------------|
| **Matching service crash** | Rides stuck in MATCHING | Timeout (30s) → re-match; standby instance takes over |
| **Redis cluster failure** | Can't find nearby drivers | Fallback: query PostgreSQL driver table (slower, 500ms vs 10ms) |
| **Driver app disconnect** | Location stale | Mark driver unavailable after 30s no heartbeat; don't dispatch to stale drivers |
| **Payment failure** | Can't charge rider | Complete ride anyway; retry payment async; block future rides if unpaid |
| **GPS drift** | Wrong fare calculation | Cross-validate GPS with map-matched route; use road-snapped distance |
| **Surge spike** | Riders see 10× price | Cap surge multiplier (max 5×); show fare estimate before request |

## 15. Availability

**Target: 99.99% for ride requests, 99.9% for matching**

```
Availability Architecture:
┌──────────────────────────────────────────────┐
│          Global API Gateway                  │
│   (Route by lat/lng to nearest region)       │
├────────────┬────────────┬────────────────────┤
│ US-West    │ US-East    │ EU-West            │
│ ┌────────┐ │ ┌────────┐ │ ┌────────┐        │
│ │Matching│ │ │Matching│ │ │Matching│        │
│ │Service │ │ │Service │ │ │Service │        │
│ │(3 inst)│ │ │(3 inst)│ │ │(3 inst)│        │
│ ├────────┤ │ ├────────┤ │ ├────────┤        │
│ │ Redis  │ │ │ Redis  │ │ │ Redis  │        │
│ │Cluster │ │ │Cluster │ │ │Cluster │        │
│ ├────────┤ │ ├────────┤ │ ├────────┤        │
│ │Postgres│ │ │Postgres│ │ │Postgres│        │
│ │Primary │ │ │Primary │ │ │Primary │        │
│ └────────┘ │ └────────┘ │ └────────┘        │
└────────────┴────────────┴────────────────────┘

Graceful Degradation:
  1. Matching slow → Expand search radius, accept suboptimal match
  2. Surge service down → Default surge=1.0 (no surge pricing)
  3. ETA service down → Show "ETA unavailable"; still allow ride request
  4. Region down → Riders see "service unavailable in area" (no cross-region fallback for real-time matching)
```

## 16. Security

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | Phone OTP + device fingerprint; OAuth for social login |
| **Driver Verification** | Background check; license validation; photo matching on every trip |
| **Location Privacy** | Rider's exact home address masked from driver after dropoff; fuzzy pickup pins |
| **Payment Security** | PCI-DSS compliant; tokenized cards; no raw PAN in system |
| **Trip Safety** | Real-time trip monitoring; anomaly detection (unexpected stops, route deviation) |
| **Emergency** | In-app SOS button → shares live location with local emergency services |
| **Data Retention** | Trip data retained 7 years (regulatory); GPS trails anonymized after 90 days |
| **Access Control** | Role-based: rider, driver, support, ops, admin; audit log for all data access |

## 17. Cost Constraints

**Estimated Cost (500K rides/day, 200K active drivers, 5 metro areas):**

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| **Matching Services** | 15× c6g.2xlarge (3 per metro) | $6,000 |
| **Redis Clusters** | 5× r6g.xlarge clusters (1 per metro) | $5,000 |
| **PostgreSQL** | RDS r6g.2xlarge Multi-AZ × 5 metros | $15,000 |
| **Kafka** | MSK 9-broker cluster | $6,000 |
| **Map/Routing** | Google Maps API (500K rides × 4 calls) | $12,000 |
| **SMS/Push** | Twilio + FCM | $5,000 |
| **S3 (GPS trails)** | 50TB/month | $1,200 |
| **Total** | | **~$50,200/month** |

**Cost Optimization:**
- **Own routing engine**: Build with OpenStreetMap + OSRM → eliminate $12K/month Maps API
- **Dynamic driver pings**: Reduce location update frequency to 10s when no nearby riders (50% fewer writes)
- **Matching radius tuning**: Smaller radius in dense areas → fewer candidates to score → less compute
- **Shared rides (UberPool)**: 30% more revenue per vehicle-mile; better driver utilization
- **Spot instances**: Surge compute capacity for event spikes (NYE, concerts) on spot — 70% savings

## Key Interview Discussion Points

1. **How to find nearest drivers efficiently?** — Geospatial index (geohash/S2/H3); query cell + neighbors; filter by availability
2. **How to prevent double-dispatch?** — Atomic lock on driver status (Redis SETNX); check-and-claim
3. **How to handle New Year's Eve load?** — Pre-scale; queue ride requests; extend matching radius; increase surge to balance demand
4. **Why not just use distance for matching?** — ETA considers traffic, one-way streets, road type; distance is misleading
5. **How to compute ETAs at scale?** — Pre-computed contraction hierarchies; live traffic overlay; cache popular routes
