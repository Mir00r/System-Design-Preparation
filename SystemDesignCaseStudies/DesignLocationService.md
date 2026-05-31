# 🌐 Design a Location-Based Service (Uber Nearby Drivers / Yelp)

> *"When you open Uber, it shows the 5 nearest drivers within seconds. Behind that 'simple' feature: a spatial index storing 5 MILLION active drivers updating their location every 4 seconds (1.25 million location updates per second!). The system must answer 'find everything within 2km of me' across millions of moving objects in under 100ms. This is the foundation of ride-sharing, food delivery, dating apps, and local search."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Database Indexing](../../Database/Indexing.md), [Caching](../../BuildingBlocks/Caching.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Geospatial Indexing Approaches](#-geospatial-indexing-approaches)
3. [Geohash Deep Dive](#-geohash-deep-dive)
4. [QuadTree Approach](#-quadtree-approach)
5. [Architecture for Moving Objects](#-architecture-for-moving-objects)
6. [Proximity Search Algorithm](#-proximity-search-algorithm)
7. [Scaling & Optimization](#-scaling--optimization)
8. [Java Implementation](#-java-implementation)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
✅ Update location (driver/rider sends GPS every 4 seconds)
✅ Find nearby objects (drivers within 5km, restaurants within 2km)
✅ Return results sorted by distance
✅ Support geofencing (alert when user enters/exits area)
✅ Handle moving objects (driver locations change constantly!)
✅ Search with filters (available drivers, open restaurants, etc.)
```

### Non-Functional Requirements
```
✅ Low latency: proximity query < 100ms
✅ High write throughput: 1M+ location updates/second
✅ Scale: 10M+ active objects being tracked
✅ Accuracy: results within 100m of actual distance
✅ Real-time: location updates reflected within 5 seconds
✅ Availability: 99.99% (ride-sharing can't go down!)
```

---

## 🗺️ Geospatial Indexing Approaches

```
THE CORE PROBLEM:
  Given point (lat=40.7128, lon=-74.0060) — find all objects within 3km.
  
  NAIVE APPROACH: Check every object!
  For 10M objects: calculate distance to each → O(N)! WAY too slow!
  
  We need a SPATIAL INDEX that narrows the search space!

FOUR APPROACHES:
  ┌────────────────────────────────────────────────────────────────┐
  │  Approach     │  How It Works          │  Best For             │
  ├────────────────────────────────────────────────────────────────┤
  │  GEOHASH      │  Divide world into     │  Static/slow-moving   │
  │               │  grid cells, encode    │  (restaurants, ATMs)  │
  │               │  as string prefix      │  Easy to implement!   │
  ├────────────────────────────────────────────────────────────────┤
  │  QUADTREE     │  Recursively divide    │  Non-uniform density  │
  │               │  into 4 quadrants      │  (cities vs rural)    │
  │               │  until max per cell    │  In-memory, fast!     │
  ├────────────────────────────────────────────────────────────────┤
  │  R-TREE       │  Bounding rectangles   │  Complex shapes       │
  │               │  balanced tree          │  (PostGIS uses this!) │
  ├────────────────────────────────────────────────────────────────┤
  │  H3 (Uber!)   │  Hexagonal grid        │  Moving objects       │
  │               │  Uber's own system!    │  Better than geohash  │
  │               │  Uniform distances!    │  for distances!       │
  └────────────────────────────────────────────────────────────────┘
```

---

## #️⃣ Geohash Deep Dive

```
HOW GEOHASH WORKS:
  Encode latitude/longitude into a string!
  
  Precision levels:
  "9"     → 5,000km × 5,000km cell (continent level!)
  "9q"    → 1,250km × 625km  
  "9q8"   → 156km × 156km  (region)
  "9q8y"  → 39km × 19.5km  (city area)
  "9q8yy" → 4.9km × 4.9km  (neighborhood!) ← USEFUL!
  "9q8yyk" → 1.2km × 0.6km (few blocks)
  "9q8yyk8" → 150m × 150m  (street level)
  
  KEY INSIGHT: Same prefix = nearby!
  "9q8yyk2" and "9q8yyk7" are in same ~1km area!
  
  PROXIMITY SEARCH:
  1. Compute geohash of query point: "9q8yy" (5-char precision)
  2. Find all objects with same prefix!
     SELECT * FROM objects WHERE geohash LIKE '9q8yy%';
  3. Also search 8 NEIGHBORING cells! (edge case!)
     "9q8yy" + its 8 neighbors = 9 cells to search
  4. Filter by actual distance (some results might be just outside radius)
  
  ┌───────┬───────┬───────┐
  │9q8yx  │9q8yz  │9q8yb  │ ← neighbor cells
  ├───────┼───────┼───────┤
  │9q8yv  │9q8yy★│9q8y8  │ ← ★ = my cell + neighbors!
  ├───────┼───────┼───────┤
  │9q8yt  │9q8yw  │9q8y2  │
  └───────┴───────┴───────┘

GEOHASH LIMITATIONS:
  ⚠️ Edge problem: Two points VERY close together but in different cells!
     Point A at cell boundary → geohash "9q8yy"
     Point B 10 meters away → geohash "9q8yz" (different prefix!)
     
  Solution: Always search neighboring cells too!
  
  ⚠️ Non-uniform cell sizes at different latitudes
     Near equator: cells are square-ish
     Near poles: cells are very elongated!
```

---

## 🌳 QuadTree Approach

```
QUADTREE: Recursive spatial subdivision

  Start with entire world as ONE rectangle.
  If a cell has > K objects (e.g., K=100): split into 4 quadrants!
  Repeat until each cell has ≤ K objects.
  
  Result: Dense areas (cities) have SMALL cells → fine granularity!
         Sparse areas (ocean) have HUGE cells → no wasted space!
  
  VISUALIZATION:
  ┌─────────────────────────────────────────┐
  │                                         │
  │        Rural area                       │
  │        (one big cell,                   │
  │         few objects)                    │
  │                                         │
  ├──────────┬──────────┬───────────────────┤
  │ ┌──┬──┐  │          │                   │
  │ ├──┼──┤  │ Suburban  │                   │
  │ └──┴──┘  │          │                   │
  │ Downtown │          │                   │
  │ (many    ├──────────┤                   │
  │  small   │          │                   │
  │  cells!) │          │                   │
  └──────────┴──────────┴───────────────────┘
  
  Manhattan: quadtree cells might be 100m × 100m (dense!)
  Wyoming: quadtree cells might be 50km × 50km (sparse!)

PROXIMITY SEARCH IN QUADTREE:
  1. Find the cell containing query point → O(log N) (tree traversal)
  2. Collect all objects in that cell
  3. Check neighboring cells (overlap with search radius!)
  4. Keep expanding until search radius fully covered
  5. Filter by actual distance
  
  GREAT FOR: Non-uniform distributions (cities vs rural)
  NOT GREAT FOR: In-memory requirement (full tree in RAM!)
```

---

## 🚗 Architecture for Moving Objects

```
UBER'S CHALLENGE: 5 million drivers updating location every 4 seconds!
  = 1.25 MILLION writes per second!
  
  CAN'T use database for this! Too slow for writes!
  
  ARCHITECTURE:
  
  ┌─────────────────────────────────────────────────────────────┐
  │  DRIVERS (sending GPS every 4 seconds)                      │
  │  driver_123: {lat: 40.7128, lon: -74.0060, ts: now}        │
  └─────────────────────────┬───────────────────────────────────┘
                            │ (1.25M updates/second!)
                            ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  LOCATION UPDATE SERVICE                                     │
  │  • Receives GPS updates via WebSocket/UDP                   │
  │  • Updates in-memory spatial index (geohash grid)           │
  │  • Publishes to Kafka for downstream consumers             │
  └─────────────────────────┬───────────────────────────────────┘
                            │
          ┌─────────────────┼──────────────────────┐
          ▼                 ▼                      ▼
  ┌──────────────┐  ┌──────────────┐     ┌──────────────────┐
  │  IN-MEMORY   │  │  Kafka       │     │  Analytics       │
  │  SPATIAL     │  │  (location   │     │  (trip tracking, │
  │  INDEX       │  │   events)    │     │   heatmaps)      │
  │  (Redis/     │  └──────────────┘     └──────────────────┘
  │   custom)    │
  └──────┬───────┘
         │
         ▼ (proximity queries: "find drivers near me")
  ┌──────────────────────────────────────────────┐
  │  MATCHING SERVICE                             │
  │  "User at (40.71, -74.00) needs a ride"      │
  │  → Query spatial index for drivers within 3km │
  │  → Filter: available, correct car type        │
  │  → Rank: by distance, rating, ETA             │
  │  → Return top 5 nearest available drivers!    │
  └──────────────────────────────────────────────┘

DATA STRUCTURE FOR MOVING OBJECTS:
  Redis GEO (built-in!):
    GEOADD drivers 40.7128 -74.0060 "driver_123"
    GEOADD drivers 40.7150 -74.0025 "driver_456"
    
    GEORADIUS drivers 40.7130 -74.0055 3 km ASC COUNT 10
    → Returns nearest 10 drivers within 3km! Done!
    
  Update: just GEOADD again (overwrites old position!)
  
  Performance: O(N+log(M)) where N=results, M=total items
  For 5M drivers, 3km radius in Manhattan: ~50 results in <1ms!
```

---

## 🔍 Proximity Search Algorithm

```
FULL PROXIMITY SEARCH FLOW:

  User opens Uber at (40.7128, -74.0060):
  
  Step 1: DETERMINE SEARCH AREA
    radius = 3km
    geohash precision = 6 (cells ~1.2km × 0.6km)
    My cell: "dr5ru7"
    Need to also search 8 neighbors:
    "dr5ru5", "dr5ru6", "dr5ruh", "dr5rud", "dr5rue", 
    "dr5rus", "dr5ruk", "dr5rum"
    
  Step 2: FETCH CANDIDATES
    For each of 9 cells:
      Get all drivers in that cell from Redis
    Collect ~50-200 candidates
    
  Step 3: FILTER
    Remove: unavailable, wrong car type, too far (actual distance check)
    Actual distance: haversine formula
    
    d = 2r × arcsin(√(sin²((φ₂-φ₁)/2) + cos(φ₁)cos(φ₂)sin²((λ₂-λ₁)/2)))
    
  Step 4: RANK
    Sort by: distance (primary), then ETA (considering traffic), 
             then driver rating
    
  Step 5: RETURN TOP-K
    Return nearest 5-10 with ETA estimates
    
  TOTAL TIME: < 50ms (mostly network, computation is trivial!)
```

---

## 📈 Scaling & Optimization

```
SCALING LOCATION UPDATES:

  Problem: 1.25M writes/second → single server can't handle!
  
  Solution: GEOGRAPHIC SHARDING
  
  Shard by geohash prefix:
    Server 1: handles all drivers in geohash "9q*" (West US)
    Server 2: handles all drivers in geohash "dr*" (East US)  
    Server 3: handles all drivers in geohash "gc*" (London)
    ...
    
  Each shard handles ~100K-200K updates/second → manageable!
  
  Rebalancing: If one area gets too hot (Manhattan rush hour!)
    → Split that shard further (finer geohash prefix)

HANDLING CROSS-BOUNDARY QUERIES:
  What if search radius crosses shard boundaries?
  
  Query: "find drivers within 5km of me"
  My shard: handles "dr5r*"
  But 5km radius overlaps into "dr5q*" (different shard!)
  
  Solution: SCATTER-GATHER
  1. Determine which shards overlap with search radius
  2. Query ALL relevant shards in parallel
  3. Merge results, re-sort by distance
  4. Return top-K
  
  Most queries stay within 1 shard (small radius, dense areas)
  Only large-radius or boundary queries need multi-shard!

CACHING:
  Popular queries (airports, stadiums) → cache results!
  But locations change every 4 seconds...
  Cache TTL = 4 seconds (aligned with update interval!)
  Cache key: geohash(query_point, precision=5) + radius + filters
```

---

## 💻 Java Implementation

### Location Update Service

```java
@Service
public class LocationUpdateService {
    
    @Autowired private StringRedisTemplate redis;
    @Autowired private KafkaTemplate<String, LocationEvent> kafka;
    
    /**
     * Driver sends location update every 4 seconds.
     * ~1.25 million calls/second at Uber scale!
     */
    public void updateLocation(String driverId, double lat, double lon) {
        // 1. Update Redis GEO index (primary spatial index!)
        redis.opsForGeo().add("active-drivers", 
            new Point(lon, lat), driverId);
        
        // 2. Store metadata (availability, car type)
        // Already stored separately — just update position!
        
        // 3. Update geohash membership (for shard routing)
        String geohash = Geohash.encode(lat, lon, 6); // precision 6
        redis.opsForValue().set(
            "driver-geohash:" + driverId, geohash,
            Duration.ofSeconds(10)); // Expires if driver goes offline
        
        // 4. Publish event for downstream consumers (analytics, trip tracking)
        kafka.send("location-updates", driverId, 
            new LocationEvent(driverId, lat, lon, Instant.now()));
    }
    
    /**
     * Driver goes offline (app closed, trip ended)
     */
    public void removeDriver(String driverId) {
        redis.opsForGeo().remove("active-drivers", driverId);
        redis.delete("driver-geohash:" + driverId);
    }
}
```

### Proximity Search Service

```java
@Service
public class ProximitySearchService {
    
    @Autowired private StringRedisTemplate redis;
    @Autowired private DriverMetadataService driverMetadata;
    
    /**
     * Find nearest available drivers to a rider.
     * Returns sorted by distance, filtered by car type.
     */
    public List<NearbyDriver> findNearbyDrivers(
            double lat, double lon, 
            double radiusKm,
            CarType carType,
            int limit) {
        
        // 1. Redis GEORADIUS: find all drivers within radius!
        GeoResults<RedisGeoCommands.GeoLocation<String>> results = 
            redis.opsForGeo().radius(
                "active-drivers",
                new Circle(new Point(lon, lat), 
                    new Distance(radiusKm, Metrics.KILOMETERS)),
                RedisGeoCommands.GeoRadiusCommandArgs.newGeoRadiusArgs()
                    .includeDistance()
                    .includeCoordinates()
                    .sortAscending() // Nearest first!
                    .limit(limit * 3) // Fetch extra (some will be filtered)
            );
        
        if (results == null) return Collections.emptyList();
        
        // 2. Filter by availability and car type
        return results.getContent().stream()
            .map(result -> {
                String driverId = result.getContent().getName();
                double distance = result.getDistance().getValue();
                Point driverLocation = result.getContent().getPoint();
                
                return new NearbyDriver(driverId, distance, driverLocation);
            })
            .filter(driver -> {
                DriverInfo info = driverMetadata.get(driver.getId());
                return info != null 
                    && info.isAvailable() 
                    && info.getCarType() == carType;
            })
            .limit(limit)
            .collect(Collectors.toList());
    }
    
    /**
     * Geofencing: check if a driver entered/exited a zone.
     * Used for: airport queues, surge zones, geo-restrictions.
     */
    public boolean isInGeofence(String driverId, Geofence fence) {
        List<Point> positions = redis.opsForGeo()
            .position("active-drivers", driverId);
        
        if (positions == null || positions.isEmpty()) return false;
        
        Point driverPos = positions.get(0);
        return fence.contains(driverPos.getX(), driverPos.getY());
    }
}
```

### Custom Geohash Implementation

```java
public class Geohash {
    
    private static final String BASE32 = "0123456789bcdefghjkmnpqrstuvwxyz";
    
    /**
     * Encode lat/lon to geohash string of given precision.
     */
    public static String encode(double lat, double lon, int precision) {
        double[] latRange = {-90.0, 90.0};
        double[] lonRange = {-180.0, 180.0};
        
        StringBuilder geohash = new StringBuilder();
        boolean isLon = true; // Alternate lon and lat bits
        int bit = 0;
        int ch = 0;
        
        while (geohash.length() < precision) {
            double mid;
            if (isLon) {
                mid = (lonRange[0] + lonRange[1]) / 2;
                if (lon >= mid) {
                    ch |= (1 << (4 - bit));
                    lonRange[0] = mid;
                } else {
                    lonRange[1] = mid;
                }
            } else {
                mid = (latRange[0] + latRange[1]) / 2;
                if (lat >= mid) {
                    ch |= (1 << (4 - bit));
                    latRange[0] = mid;
                } else {
                    latRange[1] = mid;
                }
            }
            
            isLon = !isLon;
            bit++;
            
            if (bit == 5) {
                geohash.append(BASE32.charAt(ch));
                bit = 0;
                ch = 0;
            }
        }
        
        return geohash.toString();
    }
    
    /**
     * Get 8 neighboring geohash cells (for boundary search!)
     */
    public static List<String> getNeighbors(String geohash) {
        // Decode center, offset in 8 directions, re-encode
        double[] center = decode(geohash);
        double latStep = getLatStep(geohash.length());
        double lonStep = getLonStep(geohash.length());
        
        List<String> neighbors = new ArrayList<>(8);
        int[][] directions = {{-1,-1},{-1,0},{-1,1},{0,-1},{0,1},{1,-1},{1,0},{1,1}};
        
        for (int[] dir : directions) {
            double newLat = center[0] + dir[0] * latStep;
            double newLon = center[1] + dir[1] * lonStep;
            neighbors.add(encode(newLat, newLon, geohash.length()));
        }
        
        return neighbors;
    }
}
```

---

## 🎮 Mini Challenge

### 🧩 Design: Surge Pricing Zone Detection

Uber needs to detect when a geographic area has too many riders and not enough drivers (high demand), and dynamically apply surge pricing. Design the zone detection system.

<details>
<summary>🔑 Answer</summary>

```java
public class SurgeDetector {
    
    // Divide city into hexagonal zones (H3 resolution 8 ≈ 0.7 km²)
    
    @Scheduled(fixedRate = 60_000) // Check every minute
    public void detectSurgeZones() {
        Map<String, ZoneMetrics> zones = new HashMap<>();
        
        // Count riders requesting rides per zone
        List<RideRequest> recentRequests = 
            getRequestsLastMinutes(5);
        
        for (RideRequest req : recentRequests) {
            String zone = H3.geoToH3(req.getLat(), req.getLon(), 8);
            zones.computeIfAbsent(zone, k -> new ZoneMetrics())
                .incrementDemand();
        }
        
        // Count available drivers per zone  
        List<DriverLocation> drivers = getActiveDrivers();
        for (DriverLocation driver : drivers) {
            String zone = H3.geoToH3(driver.getLat(), driver.getLon(), 8);
            zones.computeIfAbsent(zone, k -> new ZoneMetrics())
                .incrementSupply();
        }
        
        // Calculate surge multiplier
        for (var entry : zones.entrySet()) {
            ZoneMetrics metrics = entry.getValue();
            double ratio = metrics.getDemand() / 
                Math.max(metrics.getSupply(), 1.0);
            
            double surge = 1.0;
            if (ratio > 2.0) surge = 1.5;  // 2x demand vs supply
            if (ratio > 3.0) surge = 2.0;  // 3x
            if (ratio > 5.0) surge = 2.5;  // 5x (cap!)
            
            surgeCache.put(entry.getKey(), surge);
        }
    }
}
```

**Smooth surge:** Don't jump from 1x to 3x! Ramp up gradually over 2-3 minutes. Ramp down slowly too (prevent oscillation).
</details>

---

## ❓ Interview Q&A

**Q1: Why use Redis GEO over PostGIS for real-time location tracking?**
> Redis GEO: in-memory, sub-millisecond reads/writes, handles 500K+ ops/second per node. PostGIS: disk-based, better for complex spatial queries (polygons, intersections) but slower for high-throughput point updates. For moving objects (drivers, riders) updating every 4 seconds: Redis is 100x faster. For static data (restaurants, ATMs): PostGIS is fine and offers richer query capabilities.

**Q2: How do you handle the geohash boundary problem?**
> Two points 10 meters apart can have completely different geohash prefixes if they're on a cell boundary. Solution: always search the target cell AND all 8 neighboring cells. This guarantees coverage. Then apply exact distance calculation (Haversine formula) to filter results within the actual radius. The 9-cell search over-fetches slightly but guarantees no misses.

**Q3: How would you scale to handle 10M+ moving objects?**
> Geographic sharding: partition by geohash prefix. Each shard handles a geographic region (e.g., one city). Shard key = first 2-3 chars of geohash. Benefits: location updates only hit one shard (local writes), most proximity queries stay within one shard (local reads). For cross-boundary queries: scatter-gather across relevant shards in parallel. Auto-split hot shards (Manhattan at rush hour) into finer granularity.

**Q4: Geohash vs QuadTree vs H3 — when to use which?**
> Geohash: simplest, works with standard databases (LIKE prefix queries!), good for static objects. QuadTree: adaptive granularity (dense areas get small cells), great for non-uniform distributions, but requires in-memory tree structure. H3 (Uber's hexagonal system): uniform distance property (all neighbors equidistant, unlike geohash squares), best for distance-based queries, used by Uber/Lyft. For interviews: geohash is usually sufficient and easiest to explain.

**Q5: How does Uber's system handle a driver crossing shard boundaries?**
> When a driver's location update puts them in a different geohash region (shard): (1) Remove from old shard, (2) Add to new shard. Both operations are atomic per shard. Brief inconsistency window (~4 seconds until next update) is acceptable. For queries near shard boundaries: scatter-gather ensures we check both sides. The system is eventually consistent with a maximum staleness of one update interval (4 seconds).

---

## 🔗 Related Topics
- [Database Indexing](../../Database/Indexing.md) — Spatial index fundamentals
- [Consistent Hashing](../../KeyConcepts/ConsistentHashing.md) — Geographic sharding
- [Redis Deep Dive](../../Database/Redis_Deep_Dive.md) — GEO commands internals
- [Design Google Maps](DesignGoogleMaps.md) — Map tiles and routing

---

*"The hardest part of a location service isn't the math (Haversine is just a formula). It's handling millions of objects that MOVE, while answering 'who's near me?' in milliseconds, and making sure no driver appears as 'available' on two riders' screens simultaneously." — Uber Engineering* 🌐
