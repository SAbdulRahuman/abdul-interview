# Design a Video Streaming Platform

Examples: YouTube, Netflix, Twitch

---

## 1. Requirements

### Functional
- Upload videos (various formats and sizes)
- Stream videos with adaptive bitrate
- Search and browse video catalog
- Recommendation engine
- Live streaming support

### Non-Functional
- 1B+ DAU, 500 hours video uploaded/minute
- Start playback in < 2 seconds
- Smooth playback with adaptive quality
- Global low-latency delivery via CDN
- 99.9% availability

---

## 2. Scale Estimation

```
Upload:  500 hours/min × 60 min × 24h = 720K hours/day
         Average raw video: 1 GB/hour → 720 TB/day raw
         Multiple encodings (5 resolutions × 3 codecs) = ~10 PB/day total
Streams: 1B views/day → ~12K concurrent streams
Storage: 720 TB/day × 365 days × 3 replicas = ~790 PB/year
CDN:     Serve 80%+ of traffic from edge caches
```

---

## 3. High-Level Architecture

```
  UPLOAD PATH:
  ┌──────────┐     ┌──────────────┐     ┌──────────────┐
  │  Creator  │────▶│  Upload      │────▶│  Object      │
  │  Client   │     │  Service     │     │  Store (S3)  │
  └──────────┘     └──────┬───────┘     └──────────────┘
                          │
                   ┌──────▼───────┐
                   │  Transcoding  │
                   │  Pipeline     │
                   │  (DAG of     │
                   │   tasks)      │
                   └──────┬───────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  1080p   │     │  720p    │     │  480p    │
  │  H.264   │     │  H.264   │     │  H.264   │
  │  H.265   │     │  H.265   │     │  VP9     │
  │  AV1     │     │  AV1     │     │          │
  └────┬─────┘     └────┬─────┘     └────┬─────┘
       └────────────────┼────────────────┘
                        ▼
              ┌──────────────────┐
              │  CDN Origin      │
              │  (distribute to  │
              │   edge caches)   │
              └──────────────────┘

  STREAMING PATH:
  ┌──────────┐  manifest   ┌──────────┐  segments   ┌──────────┐
  │  Viewer  │────────────▶│  CDN     │◀───────────│  Origin  │
  │  Client  │◀────────────│  Edge    │            │  Server  │
  │  (ABR)   │  video      │  Cache   │            │          │
  └──────────┘  chunks     └──────────┘            └──────────┘
```

---

## 4. Video Processing Pipeline

```
  Upload → Transcode → Package → Distribute

  Step 1: Upload
  ┌──────────────────────────────────────────────┐
  │  Multipart upload to S3                      │
  │  • Resumable (handle network failures)       │
  │  • Pre-signed URL for direct S3 upload       │
  │  • Webhook callback when upload complete     │
  └──────────────────────────────────────────────┘

  Step 2: Transcode (computationally expensive)
  ┌──────────────────────────────────────────────┐
  │  Input: 4K raw video (H.264)                 │
  │  Output: multiple renditions                 │
  │                                              │
  │  ┌─────────┬──────────┬──────────┐           │
  │  │ Quality │ Bitrate  │ Codec    │           │
  │  ├─────────┼──────────┼──────────┤           │
  │  │ 2160p   │ 15 Mbps  │ H.265   │           │
  │  │ 1080p   │ 5 Mbps   │ H.264   │           │
  │  │ 720p    │ 2.5 Mbps │ H.264   │           │
  │  │ 480p    │ 1 Mbps   │ H.264   │           │
  │  │ 360p    │ 0.5 Mbps │ VP9     │           │
  │  │ 240p    │ 0.3 Mbps │ H.264   │           │
  │  └─────────┴──────────┴──────────┘           │
  │                                              │
  │  Parallelism: split video into segments      │
  │  (e.g., 4-second chunks), transcode each     │
  │  segment independently on different workers  │
  └──────────────────────────────────────────────┘

  Step 3: Package for streaming
  ┌──────────────────────────────────────────────┐
  │  HLS (Apple):                                │
  │  • .m3u8 manifest (playlist)                 │
  │  • .ts segment files (2-10 sec each)         │
  │                                              │
  │  DASH (open standard):                       │
  │  • .mpd manifest                             │
  │  • .m4s segment files                        │
  │                                              │
  │  Manifest example:                           │
  │  #EXTM3U                                     │
  │  #EXT-X-STREAM-INF:BANDWIDTH=5000000        │
  │  1080p/playlist.m3u8                         │
  │  #EXT-X-STREAM-INF:BANDWIDTH=2500000        │
  │  720p/playlist.m3u8                          │
  │  #EXT-X-STREAM-INF:BANDWIDTH=1000000        │
  │  480p/playlist.m3u8                          │
  └──────────────────────────────────────────────┘
```

---

## 5. Adaptive Bitrate Streaming (ABR)

```
  Client measures bandwidth continuously:

  Time ──────────────────────────────────────▶

  Bandwidth: ████████████▒▒▒▒████████████████
             High (WiFi)  Low   High again

  Quality:   ┌──────────┐     ┌──────────────┐
             │  1080p   │     │    1080p     │
             └─────┐    │     │    ┌─────────┘
                   │    │     │    │
                   ┌────▼─────▼────┐
                   │    480p       │
                   └──────────────┘

  ABR algorithm (simplified):
  ┌──────────────────────────────────────────────┐
  │  1. Download segment N, measure throughput T │
  │  2. Buffer level B = seconds of video ahead  │
  │  3. If B > 30s AND T > bitrate(next_quality):│
  │       → switch UP to higher quality          │
  │  4. If B < 10s OR T < bitrate(current):      │
  │       → switch DOWN to lower quality         │
  │  5. Otherwise: stay at current quality       │
  └──────────────────────────────────────────────┘
```

---

## 6. CDN Architecture

```
  Multi-tier CDN:
  ┌────────────────────────────────────────────────────┐
  │                    Viewer                          │
  │                      │                             │
  │         DNS resolves to nearest edge               │
  │                      │                             │
  │              ┌───────▼───────┐                     │
  │  Tier 1:     │  Edge POP    │  (100s of locations) │
  │              │  (L1 cache)  │  Cache hit: ~95%     │
  │              └───────┬──────┘                      │
  │                      │ miss                        │
  │              ┌───────▼───────┐                     │
  │  Tier 2:     │  Regional    │  (10s of locations)  │
  │              │  (L2 cache)  │  Cache hit: ~4%      │
  │              └───────┬──────┘                      │
  │                      │ miss                        │
  │              ┌───────▼───────┐                     │
  │  Tier 3:     │  Origin      │  (2-3 locations)     │
  │              │  (S3)        │  Hit: ~1%            │
  │              └──────────────┘                      │
  └────────────────────────────────────────────────────┘

  Cache key: {video_id}/{quality}/{segment_number}
  Popular videos: cached at L1 (edge) → lowest latency
  Long-tail videos: fetched from origin on demand
```

---

## 7. Video Metadata & Search

```
  Video metadata DB (MySQL/PostgreSQL):
  ┌──────────────────────────────────────┐
  │  video_id  (PK)    │ BIGINT         │
  │  title              │ VARCHAR        │
  │  description        │ TEXT           │
  │  uploader_id        │ BIGINT         │
  │  duration_sec       │ INT            │
  │  upload_status      │ ENUM           │
  │  view_count         │ BIGINT         │
  │  created_at         │ TIMESTAMP      │
  │  thumbnail_url      │ VARCHAR        │
  │  tags               │ JSON           │
  └──────────────────────────────────────┘

  Search: Elasticsearch index over title + description + tags
  Recommendations: collaborative filtering + content-based signals
```

---

## 8. Fault Tolerance

- **Transcode failure:** Retry individual segments; dead-letter queue for persistent failures
- **CDN edge failure:** DNS health checks; automatic failover to next-nearest POP
- **Origin failure:** Multi-region S3 with cross-region replication
- **Metadata DB failure:** Read replicas serve reads; promote on primary failure
- **Upload failure:** Resumable multipart upload; client retries from last successful part

---

---

## 9. Low-Level Design (LLD)

### API Contract
```
  Upload:
  POST   /api/v1/videos/upload
         Content-Type: multipart/form-data
         Body: { file: <binary>, title: "...", description: "...",
                 tags: ["tech","tutorial"], visibility: "public" }
         Response: { "video_id": "v123", "upload_url": "s3://...",
                     "status": "processing" }

  POST   /api/v1/videos/{id}/upload-complete   (after S3 upload)
         → triggers transcoding pipeline

  Playback:
  GET    /api/v1/videos/{id}/manifest.m3u8
         Response: HLS master playlist with bitrate variants

  GET    /api/v1/videos/{id}/stream/{quality}/{segment}.ts
         → 302 redirect to CDN URL with signed token

  Metadata:
  GET    /api/v1/videos/{id}
         Response: { title, description, views, likes, duration,
                     thumbnails: [url1, url2], captions: [...] }

  GET    /api/v1/search?q=kubernetes+tutorial&page=1
  GET    /api/v1/recommendations?user_id=456&limit=20
```

### Transcoding Pipeline Internals
```
  ┌────────────────────────────────────────────────────────┐
  │  Input: original video (MP4/MOV/AVI, up to 256GB)      │
  │                                                        │
  │  1. Probe: FFprobe extracts metadata (codec, bitrate,  │
  │     resolution, duration, audio tracks)                 │
  │                                                        │
  │  2. Split: divide into 10-second segments              │
  │     1h video → 360 segments → parallel processing      │
  │                                                        │
  │  3. Transcode each segment (FFmpeg):                   │
  │     Input segment → [encode_worker] →                   │
  │     ├── 2160p (4K): H.265, 15 Mbps                    │
  │     ├── 1080p: H.264, 5 Mbps                           │
  │     ├── 720p: H.264, 2.5 Mbps                          │
  │     ├── 480p: H.264, 1 Mbps                            │
  │     └── 360p: H.264, 0.5 Mbps                          │
  │                                                        │
  │  4. Audio: separate encode (AAC 128kbps, multiple lang)│
  │                                                        │
  │  5. Package: generate HLS/DASH manifests               │
  │     Master playlist → variant playlists → segments     │
  │                                                        │
  │  6. Quality check: VMAF score > 80 per variant         │
  │                                                        │
  │  7. Upload segments to origin S3 → CDN pre-warm        │
  │                                                        │
  │  State machine:                                         │
  │  uploaded → probing → splitting → encoding →           │
  │  packaging → quality_check → available | failed        │
  └────────────────────────────────────────────────────────┘
```

### HLS Manifest Structure
```
  Master Playlist (manifest.m3u8):
  #EXTM3U
  #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
  1080p/index.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
  720p/index.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=854x480
  480p/index.m3u8

  Variant Playlist (1080p/index.m3u8):
  #EXTM3U
  #EXT-X-TARGETDURATION:10
  #EXTINF:10.0,
  segment_001.ts
  #EXTINF:10.0,
  segment_002.ts
  ...
```

---

## 10. Scalability

```
  ┌─────────────────────────────────────────────────────────┐
  │  Ingest Scaling:                                        │
  │  • Upload: direct-to-S3 (presigned URL, no proxy)       │
  │  • Transcoding: Kubernetes Jobs / AWS Batch              │
  │  • Auto-scale: 0-1000 workers based on queue depth      │
  │  • GPU instances for H.265/AV1 encoding                 │
  │  • Spot instances for non-urgent transcoding             │
  │                                                         │
  │  Delivery Scaling:                                      │
  │  • CDN: 200+ edge PoPs globally                         │
  │  • Origin shield: reduced origin hits by 80%            │
  │  • Multi-CDN: route to cheapest/fastest CDN per region  │
  │  • P2P assist: WebRTC-based peer delivery for live      │
  │                                                         │
  │  Metadata:                                              │
  │  • Video metadata: PostgreSQL (sharded by video_id)     │
  │  • Search: Elasticsearch cluster                         │
  │  • Recommendations: ML model serving (TensorFlow)       │
  │  • View counts: Redis HyperLogLog (unique) + counter   │
  │                                                         │
  │  Live Streaming:                                        │
  │  • RTMP ingest → real-time transcoding → HLS/DASH      │
  │  • 2-6 second latency (standard HLS)                    │
  │  • LL-HLS/CMAF: sub-2 second latency                    │
  └─────────────────────────────────────────────────────────┘
```

---

## 11. No Data Loss

```
  ┌─────────────────────────────────────────────────────────┐
  │  Video Content (irreplaceable):                         │
  │  • Original uploaded to S3 (11 nines durability)        │
  │  • Transcoded variants in separate S3 bucket            │
  │  • Cross-region replication for originals               │
  │  • S3 versioning enabled                                │
  │                                                         │
  │  Transcoding State:                                     │
  │  • Pipeline state in PostgreSQL (transactional)          │
  │  • If worker crashes: segment retried automatically     │
  │  • Idempotent: re-encoding same segment is safe         │
  │  • Progress checkpointed per segment                    │
  │                                                         │
  │  Metadata:                                              │
  │  • PostgreSQL: sync replication to standby              │
  │  • Daily snapshots + WAL archiving to S3                │
  │  • Video deletion: soft delete (retain 30 days)         │
  │                                                         │
  │  User-Generated Content:                                │
  │  • Comments, likes in Cassandra (RF=3)                   │
  │  • Titles, descriptions: versioned history               │
  │  • Caption files: stored alongside video in S3          │
  └─────────────────────────────────────────────────────────┘
```

---

## 12. Latency

```
  Latency Targets:
  ┌───────────────────────────────┬──────────┬──────────┐
  │ Operation                    │ p50      │ p99      │
  ├───────────────────────────────┼──────────┼──────────┤
  │ Video start (time to first   │ 500 ms   │ 2 s     │
  │ frame after play click)      │          │          │
  │ Segment fetch (CDN)          │ 50 ms    │ 200 ms  │
  │ ABR quality switch           │ 1 seg    │ 3 seg   │
  │ Search query                 │ 100 ms   │ 500 ms  │
  │ Upload processing (1h video) │ 10 min   │ 30 min  │
  │ Thumbnail generation         │ 30 s     │ 2 min   │
  └───────────────────────────────┴──────────┴──────────┘

  Optimizations:
  ┌─────────────────────────────────────────────────────────┐
  │  • CDN: cache segments at edge (cache hit ratio > 95%) │
  │  • Predictive prefetch: buffer next 2-3 segments       │
  │  • QUIC/HTTP3: faster connection setup, less buffering │
  │  • Origin shield: reduces origin round-trips            │
  │  • Codec choice: AV1 for better quality at lower bitrate│
  │  • Pre-warm CDN: push popular videos to edge proactively│
  │  • Byte-range requests: fast seek without full download │
  └─────────────────────────────────────────────────────────┘
```

---

## 13. Reliability

```
  Failure Modes & Mitigation:
  ┌────────────────────────┬─────────────────────────────────┐
  │ Failure                │ Mitigation                      │
  ├────────────────────────┼─────────────────────────────────┤
  │ CDN PoP failure        │ Multi-CDN: DNS-based failover   │
  │                        │ to alternate CDN provider       │
  ├────────────────────────┼─────────────────────────────────┤
  │ Transcoding worker     │ SQS/Kafka: unacked segment re-  │
  │ crash                  │ queued; retry on different worker│
  ├────────────────────────┼─────────────────────────────────┤
  │ Origin S3 outage       │ Cross-region replica; CDN cache │
  │                        │ serves existing content          │
  ├────────────────────────┼─────────────────────────────────┤
  │ Bitrate detection      │ Client-side ABR adapts to       │
  │ failure                │ network; fallback to lowest     │
  │                        │ quality guarantees playback      │
  ├────────────────────────┼─────────────────────────────────┤
  │ Upload corruption      │ Client-side checksum; resume    │
  │                        │ upload (multipart S3)            │
  └────────────────────────┴─────────────────────────────────┘
```

---

## 14. Availability

```
  Target: 99.99% for playback; 99.9% for uploads

  ┌─────────────────────────────────────────────────────────┐
  │  Playback (highest priority):                           │
  │  • CDN: 99.99% SLA, serves from edge cache              │
  │  • Multi-CDN: if primary CDN fails, fallback            │
  │  • Even if origin is down: popular content cached        │
  │  • API: multiple AZs, auto-healing instances             │
  │                                                         │
  │  Upload & Processing:                                   │
  │  • Direct-to-S3: highly available upload path           │
  │  • Processing queue: SQS/Kafka survives worker outage   │
  │  • Delayed processing acceptable (not real-time)         │
  │                                                         │
  │  Graceful Degradation:                                  │
  │  • Search down → browse/recommendations still work      │
  │  • Recommendation model down → show trending/popular    │
  │  • Comments down → video plays (core functionality)     │
  └─────────────────────────────────────────────────────────┘
```

---

## 15. Security

```
  ┌─────────────────────────────────────────────────────────┐
  │  Content Protection:                                    │
  │  • DRM: Widevine (Chrome/Android), FairPlay (Apple),   │
  │    PlayReady (Microsoft)                                │
  │  • Signed URLs: CDN URLs expire after 4 hours           │
  │  • Token authentication: embed user + IP in token       │
  │  • Watermarking: invisible per-user watermark for       │
  │    piracy tracking                                      │
  │                                                         │
  │  Copyright:                                             │
  │  • Content fingerprinting on upload (Content ID)        │
  │  • Hash frames against known copyrighted material       │
  │  • DMCA takedown: automated + manual review             │
  │                                                         │
  │  User Safety:                                           │
  │  • NSFW detection: ML model on video frames             │
  │  • Violent content flagging                              │
  │  • Age-gating: require verification for mature content  │
  │  • Comment moderation: ML + human review                │
  │                                                         │
  │  Infrastructure:                                        │
  │  • Encryption in transit: TLS for all streams           │
  │  • Encryption at rest: SSE-S3 for stored videos         │
  │  • Access control: per-video permissions (private/      │
  │    unlisted/public)                                     │
  └─────────────────────────────────────────────────────────┘
```

---

## 16. Cost Constraints

```
  ┌─────────────────────────────────────────────────────────┐
  │  Main Cost Drivers (at YouTube scale):                   │
  │                                                         │
  │  CDN Bandwidth (60% of cost):                           │
  │  • 500K concurrent users × 5 Mbps avg = 2.5 Tbps       │
  │  • Multi-CDN: $0.01-0.05/GB depending on region         │
  │  • ~$2M/month for bandwidth alone                       │
  │                                                         │
  │  Transcoding (20% of cost):                             │
  │  • GPU instances: $3/hr per p3.2xlarge                   │
  │  • 500K videos/day × 5 variants × 10 min each           │
  │  • ~$500K/month                                         │
  │                                                         │
  │  Storage (15%):                                         │
  │  • 1 PB originals + 5 PB transcoded = 6 PB              │
  │  • S3: $0.023/GB → ~$140K/month                         │
  │  • S3 Glacier for rarely watched: $0.004/GB             │
  │                                                         │
  │  Cost Optimization:                                     │
  │  • AV1 codec: 30% smaller files → 30% CDN savings      │
  │  • Lazy transcoding: only 720p initially; 4K on demand  │
  │  • Storage tiering: move unwatched videos to Glacier    │
  │  • CDN caching: serve from cache > 95% of requests      │
  │  • Spot instances for transcoding: 70% compute savings  │
  │  • P2P delivery: offload 10-20% of CDN traffic          │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Interview Discussion Points

1. **Why segment-based transcoding?** — Parallelism: 100 workers can transcode a 1-hour video in minutes instead of hours
2. **HLS vs DASH?** — HLS: Apple ecosystem, wider compatibility. DASH: open standard, more flexible. Most platforms support both.
3. **How to handle live streaming?** — Direct ingest via RTMP; real-time transcoding; ultra-low latency segments (1-2s); WebRTC for sub-second
4. **CDN cost optimization?** — Cache popular content aggressively; use cheaper regions for long-tail; pre-warm caches for predicted viral content
5. **How to handle copyright?** — Content fingerprinting (Content ID); hash uploaded frames against database; block or monetize matched content
