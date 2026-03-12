---
name: performance-optimization
description: Full-stack performance optimization patterns for profiling, caching, database tuning, memory management, and frontend performance across Python, Node.js, and web applications.
---

# Performance Optimization Patterns

Systematic approaches to identifying and resolving performance bottlenecks across the full stack.

## When to Activate

- Profiling slow endpoints or functions
- Designing caching strategies (in-memory, Redis, CDN, HTTP)
- Optimizing database queries (N+1, indexes, connection pooling)
- Reducing frontend bundle size and improving rendering
- Memory leak detection and resolution
- Capacity planning and load testing

## Core Principles

### 1. Measure Before Optimizing

Never optimize based on intuition. Profile first.

```python
# Python: cProfile for function-level profiling
import cProfile
import pstats

def profile(func):
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()
        result = func(*args, **kwargs)
        profiler.disable()
        stats = pstats.Stats(profiler).sort_stats("cumulative")
        stats.print_stats(20)  # Top 20 hotspots
        return result
    return wrapper
```

### 2. Cache at the Right Layer

```
Request → CDN Cache → Reverse Proxy Cache → App Cache → DB Cache → Database

Layer        | TTL        | Best For
CDN          | hours-days | Static assets, public API responses
HTTP Cache   | minutes    | GET requests with ETag/Last-Modified
Application  | seconds    | Computed results, serialized responses
Query Cache  | seconds    | Frequent identical queries
```

### 3. Optimize the Critical Path

Focus on the hot 20% that consumes 80% of resources.

## Database Optimization

### N+1 Query Detection & Fix

```python
# BAD: N+1 — 1 query for plates + N queries for vehicles
plates = db.query(Plate).all()
for plate in plates:
    vehicle = db.query(Vehicle).get(plate.vehicle_id)  # N queries!

# GOOD: Eager loading — 1 query with JOIN
plates = db.query(Plate).options(joinedload(Plate.vehicle)).all()

# GOOD: Batch loading — 2 queries total
plates = db.query(Plate).all()
vehicle_ids = [p.vehicle_id for p in plates]
vehicles = db.query(Vehicle).filter(Vehicle.id.in_(vehicle_ids)).all()
vehicle_map = {v.id: v for v in vehicles}
```

### Index Strategy

```sql
-- Composite index for common query patterns
CREATE INDEX idx_plates_region_status ON license_plates (region, status)
  WHERE status = 'active';  -- Partial index: smaller, faster

-- Covering index (includes all needed columns)
CREATE INDEX idx_plates_search ON license_plates (plate_number)
  INCLUDE (region, created_at);

-- GIN index for full-text search
CREATE INDEX idx_plates_text ON license_plates
  USING GIN (to_tsvector('english', plate_number));
```

### Connection Pooling

```python
# SQLAlchemy connection pool tuning
engine = create_engine(
    DATABASE_URL,
    pool_size=20,           # Steady-state connections
    max_overflow=10,        # Burst connections
    pool_timeout=30,        # Wait for connection (seconds)
    pool_recycle=3600,      # Recycle connections after 1 hour
    pool_pre_ping=True,     # Test connections before use
)
```

## Caching Patterns

### In-Memory Cache with TTL

```python
from functools import lru_cache
from cachetools import TTLCache
import threading

class ThreadSafeCache:
    def __init__(self, maxsize: int = 1024, ttl: int = 300):
        self._cache = TTLCache(maxsize=maxsize, ttl=ttl)
        self._lock = threading.Lock()

    def get(self, key: str):
        with self._lock:
            return self._cache.get(key)

    def set(self, key: str, value):
        with self._lock:
            self._cache[key] = value

# Usage
plate_cache = ThreadSafeCache(maxsize=10000, ttl=60)

def get_plate(plate_id: str) -> dict:
    cached = plate_cache.get(plate_id)
    if cached is not None:
        return cached
    plate = db.query(Plate).get(plate_id)
    plate_cache.set(plate_id, plate)
    return plate
```

### HTTP Cache Headers

```python
from datetime import timedelta

# Immutable static assets (hashed filenames)
@app.get("/static/{filename}")
def static_file(filename: str):
    return FileResponse(
        f"static/{filename}",
        headers={"Cache-Control": "public, max-age=31536000, immutable"},
    )

# API responses with ETag
@app.get("/api/plates/{plate_id}")
def get_plate(plate_id: str, request: Request):
    plate = plate_service.get(plate_id)
    etag = hashlib.md5(json.dumps(plate).encode()).hexdigest()

    if request.headers.get("If-None-Match") == etag:
        return Response(status_code=304)

    return JSONResponse(plate, headers={"ETag": etag, "Cache-Control": "private, max-age=60"})
```

## Python-Specific Optimization

### Async I/O for Concurrent Operations

```python
import asyncio

# BAD: Sequential I/O
async def get_dashboard():
    plates = await get_plates()       # 100ms
    stats = await get_stats()         # 100ms
    alerts = await get_alerts()       # 100ms
    return {**plates, **stats, **alerts}  # Total: 300ms

# GOOD: Concurrent I/O
async def get_dashboard():
    plates, stats, alerts = await asyncio.gather(
        get_plates(),
        get_stats(),
        get_alerts(),
    )
    return {**plates, **stats, **alerts}  # Total: ~100ms
```

### Memory Optimization

```python
# Use generators for large datasets
def process_large_dataset(file_path: str):
    # BAD: Load all into memory
    # data = json.load(open(file_path))

    # GOOD: Stream processing
    with open(file_path) as f:
        for line in f:
            yield json.loads(line)

# Use __slots__ for memory-efficient objects
class Detection:
    __slots__ = ("bbox", "confidence", "class_id", "timestamp")

    def __init__(self, bbox, confidence, class_id, timestamp):
        self.bbox = bbox
        self.confidence = confidence
        self.class_id = class_id
        self.timestamp = timestamp
```

## Frontend Performance

### Bundle Optimization

```typescript
// Dynamic imports for code splitting
const HeavyChart = lazy(() => import('./HeavyChart'))

// Tree-shakeable imports
import { debounce } from 'lodash-es'  // NOT: import _ from 'lodash'
```

### Image Optimization

```html
<!-- Responsive images with modern formats -->
<picture>
  <source srcset="plate.avif" type="image/avif" />
  <source srcset="plate.webp" type="image/webp" />
  <img src="plate.jpg" alt="License plate" loading="lazy"
       width="640" height="480" decoding="async" />
</picture>
```

## Profiling Commands

```bash
# Python profiling
python -m cProfile -o profile.out app.py
python -m snakeviz profile.out              # Visual profiler

# PostgreSQL query analysis
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT ...;

# Memory profiling
python -m memory_profiler script.py
objgraph.show_most_common_types()

# Load testing
wrk -t12 -c400 -d30s http://localhost:8000/api/plates
ab -n 1000 -c 50 http://localhost:8000/api/plates
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Premature optimization | Profile first, optimize hotspots |
| Cache everything | Cache expensive + frequently accessed only |
| No cache invalidation strategy | Use TTL + event-based invalidation |
| SELECT * in queries | Select only needed columns |
| No connection pooling | Configure pool size for expected load |
| Sync I/O in async context | Use async libraries or run_in_executor |
| Loading entire dataset into memory | Stream/paginate large datasets |
| No compression | Enable gzip/brotli for API responses |
