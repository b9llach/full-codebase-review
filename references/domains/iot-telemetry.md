# Domain Pack: IoT / Telemetry / Time-Series / Location Data

Extends the `logic-math`, `data-layer`, and `performance` agents. Load when the stack processes device-emitted data, location coordinates, sensor readings, or any high-volume time-series ingestion.

Three core challenges:

1. **Scale.** A single deployment can emit millions of points per day. Algorithms that work at 10k rows die at 10M.
2. **Clock trust.** Device clocks drift. Networks delay. Events arrive out of order.
3. **Location semantics.** Coordinates aren't just numbers. Distance, intersection, containment — each has edge cases (antimeridian, poles, projection choice).

## Time handling

### Clock sources and skew

Devices, servers, and clients all have clocks. They disagree.

Flag:
- Code that trusts device-reported timestamps for security / ordering decisions
- Use of device timestamp without a "received at" server timestamp
- Comparisons between timestamps without specifying which clock
- Storage of only one timestamp per event (can't reconstruct delivery lag)

Canonical pattern: store both `event_ts` (device-reported) and `ingested_ts` (server). Query on `ingested_ts` for ordering; display `event_ts` for human interpretation.

### Out-of-order ingestion

Events arriving out of order is the norm, not the exception.

Flag:
- Stream processing that assumes monotonic timestamps
- Aggregations that fail or corrupt when events arrive late (e.g., "events per hour" where an event for hour N arrives in hour N+2)
- Watermark / lateness handling missing in stream systems
- Dedupe logic that compares against only "recent" events (late duplicates slip through)

### Time zones and DST

Devices in different zones; servers in UTC; users in local. Flag:
- Timestamps stored without TZ info
- "Daily" aggregations in local time without documented TZ (which day is 00:05 UTC?)
- DST transitions not handled in historical aggregations (23-hour and 25-hour days)
- Time zone offsets hard-coded (transitions happen; 2024 Morocco dropped DST mid-year; etc.)

### Timestamp precision

Flag:
- Microsecond / nanosecond precision stored but compared at second granularity (subtle equality bugs)
- Second precision where millisecond is needed (two events at the "same second" coalesce)
- Client-reported fractional seconds trusted (many devices round)

## Location / geospatial

### Coordinate representation

Flag:
- Storing lat/lng as separate `float` columns when `geography`/`geometry` types exist (Postgres + PostGIS, MySQL spatial, etc.)
- Precision loss: float32 vs float64 (lat/lng needs at least float64)
- Swapped lat/lng (especially easy with GeoJSON which is `[lng, lat]`)
- Coordinate strings parsed with locale-sensitive parsers (`"-73,5"` could be a number with comma decimal separator)

### Distance calculations

Flag:
- Euclidean distance on lat/lng (wrong — earth is spherical; 1° longitude ≠ 1° latitude)
- Haversine used when more precision is needed (Vincenty or geodesic is more accurate over long distances)
- Distance compared using squared distance to avoid `sqrt`, but then compared against an unsquared threshold
- Unit confusion (meters vs feet, km vs miles)

### Containment / intersection

For point-in-polygon (is the device inside the geofence?):
- Use a spatial library, not hand-rolled
- Watch for self-intersecting polygons (behavior undefined in some libraries)
- Watch for polygons crossing the antimeridian (180°/-180°) — many algorithms break

Flag:
- Polygon data without validation (invalid polygons silently give wrong results)
- Containment checks using bounding box only (fast but wrong for complex shapes)
- Spatial index missing on location columns (containment queries will be slow)

### Geofencing specifics

For arrival / departure / stay-duration events:

Flag:
- Single-point "is inside" check used to trigger arrival event — GPS noise causes repeated entries
- No debouncing: a device sitting on the boundary of a geofence causes flapping events
- No minimum stay duration before "arrival" is confirmed
- Departure threshold same as arrival threshold — flapping behavior
- Events fired during GPS dropout (last known location assumed valid for too long)

Canonical pattern:
- Arrival: must be inside the fence for ≥ N minutes
- Departure: must be outside the fence for ≥ M minutes (often larger than N)
- GPS dropout handling: if no fix for > X minutes, pause event evaluation

### GPS quality

Flag:
- Trust of all GPS points equally (should use HDOP / accuracy / num-satellites as quality filter)
- No filtering of impossible movements (device reporting an 800 mph jump between two fixes)
- No smoothing for known noise patterns (multipath in urban canyons)
- Spoofing not considered (GPS can be spoofed with cheap equipment)

## High-volume ingestion

### Batching

Flag:
- Per-event database insert (doesn't scale past a few hundred events/sec)
- Batching without a max time window (low traffic → events sit in buffer too long)
- Batching without a max buffer size (memory bloat under spike)
- Retry on batch failure that retries the whole batch (one bad event poisons the rest)

### Backpressure

Flag:
- Unbounded queues between producer and consumer (runaway memory)
- No way to shed load when consumer is overwhelmed
- Synchronous acknowledgment patterns in high-throughput ingestion (ack should be async)

### Partitioning

Flag:
- Single-table design for multi-tenant high-volume data (partitioning by tenant or time should be considered)
- Time-series tables without time partitioning (Postgres native partitioning, Timescale hypertables, etc.)
- Partition key that concentrates load (all traffic hits the newest partition)

### Downsampling / rollups

Flag:
- Queries against raw points over long time ranges (should query pre-aggregated rollups)
- Rollup tables that don't have a backfill / rebuild path (schema changes are painful)
- Rollup granularity that doesn't match query needs (store 1-minute rollups, users ask for 5-minute — OK, 1-minute → 1-hour → 1-day pyramid usually works)

## Data quality

### Schema drift

IoT devices often evolve — firmware updates change field names, add new sensors, deprecate old ones.

Flag:
- No versioning on incoming event shapes
- Strict schemas that reject new fields (breaks on firmware update)
- Permissive schemas that accept anything (data quality degrades silently)
- No data quality monitoring (who knows the device stopped sending field X three weeks ago?)

### Validation at ingest

Flag:
- Trust of all incoming fields (devices can be compromised; firmware can have bugs)
- Physical impossibility not checked (negative speed, monotonic counter going backward, temperature sensor reading -400°C)
- No rejection or quarantine queue for bad data

### Deduplication

Flag:
- Natural key for dedup unclear (device_id + event_ts is usually right, but devices can emit duplicate timestamps)
- Dedup window too short (late duplicates slip through)
- Dedup based only on hash of payload (subtle timestamp or order differences leak duplicates through)

## Aggregation correctness

### Scoring / derived metrics

For derived scoring or rating systems built on top of telemetry:

Flag:
- Score formula not documented in code
- Score that changes retroactively (user sees score drop without anything changing — bad UX)
- Edge cases not handled: entity with < N events, entity with no events, entity with only low-confidence events
- Scoring weighted by recency without a clear policy for how much "recent" matters

### Summary statistics

Flag:
- Mean used where median would be more robust (outliers dominate)
- Standard deviation calculated wrong (sample vs population; one-pass Welford is numerically stable, but many implementations are naive two-pass)
- Percentiles approximated with streaming algorithms (t-digest, HDR histogram) without understanding the error bounds
- Counts vs rates confused — "1000 events today" vs "1000 events per hour" are very different

### Windowing

Flag:
- Sliding window implementations with off-by-one on window boundaries
- Tumbling window that doesn't handle events at exact window boundary consistently
- Sessionization (events grouped by gaps) with hardcoded gap duration that doesn't match domain reality

## Storage choices

Flag:
- Time-series data in OLTP database without partitioning at volume
- Historical data retained in hot storage indefinitely (cost)
- No tier-down to cold storage (S3, Glacier)
- Aggregated rollups stored alongside raw points without a clear retention policy for each

## Geospatial database specifics

### PostGIS (if used)

Flag:
- Spatial reference system (SRID) mismatched between columns (4326 = WGS84 lat/lng; 3857 = web mercator — conversions are silent)
- Spatial index missing (`CREATE INDEX USING GIST`)
- Spatial queries using latitude/longitude comparisons instead of spatial operators (`ST_DWithin`)
- Geometry stored when geography would be more appropriate (geography uses earth's curvature automatically; geometry is flat)

### Timescale (if used)

Flag:
- Hypertable chunk time interval inappropriate for volume (default 7 days often too large for high-volume, too small for low)
- Continuous aggregates without refresh policy
- Compression not enabled on old chunks (storage cost)
- Queries not taking advantage of chunk exclusion (time predicate missing)

## Regulated telemetry (if applicable)

Some telemetry domains have industry-specific compliance regimes (transportation, healthcare devices, energy metering, etc.).

Flag:
- Regulated telemetry stored without tamper-evidence
- Compliance-mandated calculations hand-rolled instead of referencing the regulator's published rules
- No separation of business telemetry from personal/operator data subject to different legal treatment
- Personal data mixed with derived analytics without consent clarity

## Process

For an IoT / telemetry codebase:

1. Identify the ingestion path(s) — direct push from device, pull from a third-party data provider, webhook from a vendor, etc.
2. Walk the ingestion code looking for the checks above (batching, dedup, backpressure, validation)
3. Walk the storage schema for partitioning, retention, indexes
4. Walk aggregation / scoring code for edge cases
5. Walk any user-facing query paths for efficiency (raw vs rollup)
6. Write findings.

## Severity calibration

- **Critical**: data loss on ingest (events dropped silently); scoring/aggregation formula bug that affects all entities
- **High**: GPS-derived events that flap; timezone bug in daily aggregates; missing backpressure under load
- **Medium**: inefficient queries on raw points; validation gaps
- **Low**: minor observability gaps in ingestion; retention policy not documented
