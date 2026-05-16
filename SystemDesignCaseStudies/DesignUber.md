# 🚗 Design Uber
## Geospatial Indexing, Real-Time Matching, and Location at Scale

> *"At any given second, Uber knows the precise GPS coordinates of 5 million drivers worldwide, updates them every 4 seconds, and matches a new rider in under 5 seconds — all while calculating dynamic surge pricing based on real-time supply and demand across 10,000+ cities."*

**⏱️ Estimated Time**: 75 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [Design Twitter](./DesignTwitter.md), [BuildingBlocks/](../BuildingBlocks/)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [Requirements (R)](#-requirements-r)
3. [API Design (A)](#-api-design-a)
4. [Data Model (D)](#-data-model-d)
5. [Infrastructure (I)](#-infrastructure-i)
6. [The Hard Part: Geospatial Indexing](#-the-hard-part-geospatial-indexing)
7. [Driver-Rider Matching Algorithm](#-driver-rider-matching-algorithm)
8. [Real-Time Location Tracking](#-real-time-location-tracking)
9. [Surge Pricing](#-surge-pricing)
10. [Optimization (O)](#-optimization-o)
11. [Code Examples](#-code-examples)
12. [Industry Examples](#-industry-examples)
13. [Common Pitfalls](#-common-pitfalls)
14. [Mini Challenge](#-mini-challenge)
15. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

You open Uber at midnight in Times Square, New York.

```
Your app must:
1. Show all available drivers within 5km — in under 200ms
2. Match you with the optimal driver — in under 5 seconds
3. Track your driver's location in real-time as they approach
4. Calculate surge pricing based on current supply/demand in your area
5. Do this simultaneously for 5 million rides happening globally right now
```

The core challenges:

**Challenge 1 — Location at scale**: 5M drivers × 1 location update every 4 seconds = **1.25M GPS writes/second**. A traditional database cannot handle this.

**Challenge 2 — Proximity search**: "Find all drivers within 5km of lat 40.7580, lon -73.9855" is a 2D range query. Standard B-tree indexes don't work in 2 dimensions.

**Challenge 3 — Matching**: Once you have 20 nearby drivers, which one do you pick? Nearest isn't always best — a driver 600m away heading toward you is better than one 400m away heading away.

---

## 📐 Requirements (R)

### Functional Requirements
```
Core features:
  ✅ Rider requests a ride (current location, destination)
  ✅ App shows nearby available drivers on map
  ✅ System matches rider with optimal available driver
  ✅ Real-time driver location tracking during ride
  ✅ Ride fare calculation (with surge pricing)
  ✅ Ride history for both rider and driver

Out of scope:
  ❌ Driver onboarding / background checks
  ❌ Payments (separate service)
  ❌ Reviews / ratings algorithm
  ❌ Scheduled rides
```

### Non-Functional Requirements
```
Scale:
  - 100M rides/day globally
  - 5M active drivers (at peak: ~500K concurrent)
  - 75M active riders
  - Driver location updates: every 4 seconds per driver
  → Location write QPS: 500K drivers ÷ 4 seconds = 125K writes/sec

Performance:
  - Show nearby drivers on open app: < 200ms
  - Driver-rider match time: < 5 seconds
  - Location update propagation: < 5 seconds (rider sees driver move)

Reliability:
  - 99.99% availability for matching (core business)
  - Location updates can be eventually consistent (< 5s lag acceptable)
```

---

## 🌐 API Design (A)

```
# Rider: Request a ride
POST   /v1/rides
  Request:  { "pickupLat": 40.758, "pickupLon": -73.985, "destLat": 40.712, "destLon": -74.006 }
  Response: { "rideId": "...", "estimatedFare": 12.50, "surgeMultiplier": 1.2, "eta": 4 }

GET    /v1/rides/{rideId}
  Response: { "status": "en_route|arrived|in_progress|completed", "driver": {...}, "location": {...} }

# Driver: Update location (called every 4 seconds)
POST   /v1/drivers/location
  Request:  { "lat": 40.756, "lon": -73.989, "heading": 270, "speed": 35 }
  Response: 204 No Content

# Driver: Go online/offline
PUT    /v1/drivers/availability
  Request:  { "status": "available|unavailable" }

# Rider: Get nearby drivers (for map display)
GET    /v1/drivers/nearby?lat=40.758&lon=-73.985&radius=5000
  Response: { "drivers": [{ "id": "...", "lat": 40.756, "lon": -73.989, "eta": 3 }] }

# Matching webhook (internal)
POST   /v1/rides/{rideId}/match    → { "driverId": "...", "eta": 4 }
```

---

## 🗄️ Data Model (D)

### Storage Choices

```
Driver current location    → Redis (in-memory, geospatial ops, < 1ms reads)
Driver location history    → Cassandra (time-series, append-heavy, audit trail)
Rides                      → PostgreSQL (ACID, complex status lifecycle)
Users/Drivers              → PostgreSQL (relational, auth data)
Geospatial index           → Redis GEOADD (built-in spherical distance)
Trip analytics             → BigQuery / ClickHouse (OLAP, batch analytics)
```

### Schemas

```sql
-- PostgreSQL: Drivers
CREATE TABLE drivers (
    driver_id     BIGINT PRIMARY KEY,
    user_id       BIGINT UNIQUE REFERENCES users(user_id),
    vehicle_type  VARCHAR(20),     -- 'uberX', 'uberXL', 'black'
    license_plate VARCHAR(20),
    status        VARCHAR(20) DEFAULT 'offline',  -- offline|available|on_trip
    rating        DECIMAL(3,2),
    created_at    TIMESTAMP
);

-- PostgreSQL: Rides (full lifecycle)
CREATE TABLE rides (
    ride_id        BIGINT PRIMARY KEY,
    rider_id       BIGINT REFERENCES users(user_id),
    driver_id      BIGINT REFERENCES drivers(driver_id),
    status         VARCHAR(20),    -- requested|matched|en_route|arrived|in_progress|completed|cancelled
    pickup_lat     DECIMAL(9,6),
    pickup_lon     DECIMAL(9,6),
    dest_lat       DECIMAL(9,6),
    dest_lon       DECIMAL(9,6),
    base_fare      DECIMAL(8,2),
    surge_mult     DECIMAL(3,2) DEFAULT 1.0,
    final_fare     DECIMAL(8,2),
    requested_at   TIMESTAMP,
    completed_at   TIMESTAMP
);
CREATE INDEX idx_rides_rider  ON rides(rider_id, requested_at DESC);
CREATE INDEX idx_rides_driver ON rides(driver_id, requested_at DESC);
```

```
-- Redis: Driver current location (geospatial sorted set)
GEOADD available_drivers <lon> <lat> <driver_id>
GEODIST available_drivers driver_1 driver_2 km
GEORADIUS available_drivers -73.985 40.758 5 km ASC COUNT 20

-- Cassandra: Driver location history (for playback + analytics)
CREATE TABLE driver_locations (
    driver_id    BIGINT,
    recorded_at  TIMESTAMP,
    lat          DECIMAL,
    lon          DECIMAL,
    heading      INT,
    speed        FLOAT,
    PRIMARY KEY  (driver_id, recorded_at)
) WITH CLUSTERING ORDER BY (recorded_at DESC);
```

---

## 🏗️ Infrastructure (I)

### High-Level Architecture

```
  ┌───────────┐        ┌───────────┐
  │ Rider App │        │Driver App │
  └─────┬─────┘        └─────┬─────┘
        │ HTTPS               │ HTTPS (every 4s)
  ┌─────▼─────────────────────▼─────┐
  │         API Gateway              │
  │   (Auth + Rate Limit + Routing)  │
  └──┬──────────────────────────┬───┘
     │                          │
┌────▼──────────┐      ┌────────▼──────────┐
│  Ride Service │      │ Location Service  │
│  (lifecycle)  │      │ (125K writes/sec) │
└────┬──────────┘      └────────┬──────────┘
     │                          │
     │             ┌────────────▼────────────┐
     │             │   Redis Geo Cluster      │
     │             │  (driver positions,      │
     │             │   GEORADIUS queries)     │
     │             └────────────┬────────────┘
     │                          │ async
     │             ┌────────────▼────────────┐
     │             │   Kafka                  │
     │             │  (location_updated)      │
     │             └────────────┬────────────┘
     │                          │
┌────▼──────────┐  ┌────────────▼────────────┐
│  Matching     │  │  Cassandra               │
│  Service      │  │  (location history,      │
│  (algorithm)  │  │   ride archive)          │
└────┬──────────┘  └─────────────────────────┘
     │
┌────▼──────────────────────────────────────┐
│         Notification Service              │
│  (push driver request to driver's app)    │
└───────────────────────────────────────────┘
```

### Ride Request Flow (Critical Path)

```
1. Rider opens app → GET /drivers/nearby?lat=...
   → Location Service queries Redis: GEORADIUS → returns 20 nearest driver IDs
   → App displays drivers on map

2. Rider taps "Request Ride"
   → POST /rides { pickup, destination }
   → Ride Service: calculate estimated fare (base + surge multiplier)
   → Ride Service: create ride record in PostgreSQL (status = 'requested')
   → Matching Service triggered (async via Kafka event)

3. Matching Service:
   → GEORADIUS: find 10 nearest available drivers
   → Score each driver (proximity + heading + rating + recent acceptance rate)
   → Select top driver → send ride request via push notification
   → Wait for acceptance (15 second timeout)
   → If rejected/timeout → try next driver

4. Driver accepts:
   → Ride status updated: 'matched' → 'en_route'
   → Driver status: 'available' → 'on_trip' (removed from GEORADIUS index)
   → Rider receives push: "Your driver is X minutes away"

5. During ride:
   → Driver sends location every 4 seconds
   → Rider app receives location via WebSocket (real-time tracking)
   → On arrival: status → 'in_progress'

6. Ride complete:
   → Status → 'completed'
   → Payment triggered
   → Driver added back to available pool in Redis
```

---

## 🗺️ The Hard Part: Geospatial Indexing

This is the core algorithmic challenge of the Uber interview.

### Why Standard DB Indexes Don't Work

```
SQL query: "Find all drivers within 5km of (40.758, -73.985)"

Naive approach: SELECT * FROM drivers WHERE lat BETWEEN 40.71 AND 40.80 AND lon BETWEEN -74.03 AND -73.94
  - Works! But is a full table scan unless you have 2D index
  - PostgreSQL's PostGIS extension adds 2D R-tree indexes
  - But: Redis GEORADIUS is 10-100x faster for this specific access pattern

Why Redis GEORADIUS is used:
  - All driver locations fit in memory (500K active drivers × 32 bytes = 16 MB)
  - In-memory ops: < 1ms vs PostgreSQL: 5-50ms
  - GEORADIUS is O(N+log(M)) where M = members in sorted set — very fast
```

### Approach 1: Redis GEORADIUS (Uber's actual approach for real-time)

```
How Redis stores geo coordinates:
  - Encodes (lat, lon) as a 52-bit Geohash integer
  - Stores it as a Sorted Set (ZSET) score
  - GEORADIUS uses spherical distance calculation (Haversine formula)

Operations:
  GEOADD  available_drivers -73.989 40.756 "driver_42"   # add/update driver
  GEORADIUS available_drivers -73.985 40.758 5 km ASC COUNT 20 WITHDIST
  # → returns up to 20 nearest drivers within 5km, sorted by distance

  When driver goes on_trip: ZREM available_drivers "driver_42"
  When driver completes trip: GEOADD available_drivers <new_lon> <new_lat> "driver_42"
```

### Approach 2: Geohash (Alternative — understand this for interviews)

```
Geohash divides the world into a grid of rectangles.
Each cell has a string code: "dr5ru" = a 4.9km × 4.9km box around Times Square

Properties:
  - Longer code = smaller area (more precision)
  - Nearby locations share a common prefix: "dr5ru" and "dr5rv" are adjacent
  - Prefix search finds nearby cells: all drivers in "dr5r*" are near each other

How to use for proximity search:
  1. Compute geohash of rider's location: "dr5ru7" (5 chars = ~4.9km cell)
  2. Find all 9 neighboring cells (center + 8 surrounding)
  3. Query: SELECT * FROM driver_locations WHERE geohash LIKE "dr5ru7%" OR geohash LIKE "dr5ru5%" ...
  4. Within those candidates, compute exact distance and filter

  Stored in: Redis Hash (geohash_cell → list of driver_ids) OR in a DB with index on geohash column
```

### Approach 3: QuadTree (for interviews — explains the concept)

```
QuadTree recursively divides a 2D area into 4 quadrants until each cell
contains a manageable number of objects (e.g., ≤ 50 drivers per cell).

Urban area (Times Square): high driver density → small quadrant cells (200m × 200m)
Rural area: low density → large cells (10km × 10km)

Pros: Adapts to density, handles arbitrary areas
Cons: Harder to implement, harder to distribute across machines
Used by: Google Maps, OpenStreetMap for map tile rendering

For Uber matching: Redis GEORADIUS is simpler and sufficient at their scale.
Mention QuadTree to show breadth of knowledge.
```

---

## 🎯 Driver-Rider Matching Algorithm

Finding the nearest driver is only step one. Optimal matching is more nuanced:

```
Scoring formula (simplified):
  score = (1/distance_km) × heading_bonus × availability_penalty × driver_rating

Where:
  distance_km:     Physical distance (from GEORADIUS)
  heading_bonus:   1.5 if driver is heading toward rider, 1.0 otherwise
                   (a driver heading toward you will arrive faster than distance alone suggests)
  availability_penalty: Reduce score if driver recently rejected requests
  driver_rating:   Multiply by (rating / 5.0) to prefer highly-rated drivers

Example:
  Driver A: 400m away, heading away from rider   → score = (1/0.4) × 1.0 = 2.5
  Driver B: 600m away, heading toward rider       → score = (1/0.6) × 1.5 = 2.5  (tie!)
  Driver C: 500m away, heading perpendicular      → score = (1/0.5) × 1.0 = 2.0
  
  → Offer to Driver A or B first (tied), C second

Reality: Uber uses a much more complex ML model that also considers:
  - Traffic patterns (ETA from routing engine, not just distance)
  - Driver's destination preference (end-of-shift proximity)
  - Historical acceptance rates
  - Ride pool eligibility
```

### Sequential vs Broadcast Matching

```
Sequential matching (Uber's original approach):
  - Offer ride to best-scored driver
  - Wait 15 seconds for acceptance
  - If no response → move to next driver
  - Slower (each refusal adds 15 second delay) but cleaner

Broadcast matching (used for high-demand situations):
  - Offer ride to top 3-5 drivers simultaneously
  - First to accept wins; cancel offers to others
  - Faster for rider, but rejected drivers may feel spammed
  - Risk: multiple drivers start heading to pickup → one is wasted
```

---

## 📈 Surge Pricing

```
Surge = demand exceeds supply in a geographic cell

Algorithm:
  1. Divide each city into 1km × 1km geohash cells
  2. Every 60 seconds: for each cell, compute:
     supply = count of available drivers in cell
     demand = count of ride requests in cell in last 60 seconds
     surge_ratio = demand / supply
     
  3. Apply surge multiplier:
     surge_ratio < 1.0 → multiplier = 1.0 (no surge)
     1.0 - 1.5        → multiplier = 1.1
     1.5 - 2.0        → multiplier = 1.3
     2.0 - 3.0        → multiplier = 1.8
     > 3.0            → multiplier = 2.5 (cap)

  4. Store in Redis: SET surge:{geohash} 1.8 EX 120

  5. At ride request: 
     cell = geohash(rider_lat, rider_lon, precision=5)
     multiplier = redis.GET("surge:" + cell) || 1.0
     estimated_fare = base_fare × multiplier
```

---

## 💻 Code Examples

### Location Service — Driver Position Update

```java
@RestController
@RequestMapping("/v1/drivers")
public class LocationController {

    @Autowired private RedisTemplate<String, String> redis;
    @Autowired private KafkaTemplate<String, LocationEvent> kafka;

    private static final String GEO_KEY = "available_drivers";

    @PostMapping("/location")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void updateLocation(@RequestBody DriverLocationUpdate update,
                               @RequestHeader("X-Driver-Id") String driverId) {
        // 1. Update Redis geospatial index (available drivers pool)
        redis.opsForGeo().add(GEO_KEY,
                new Point(update.getLon(), update.getLat()),  // Redis uses lon,lat order
                driverId);

        // 2. Publish to Kafka for: Cassandra persistence, real-time tracking, analytics
        kafka.send("location.updated", driverId,
                new LocationEvent(driverId, update.getLat(), update.getLon(),
                        update.getHeading(), update.getSpeed(), Instant.now()));
    }
}
```

### Matching Service — Find & Score Drivers

```java
@Service
public class MatchingService {

    @Autowired private RedisTemplate<String, String> redis;
    @Autowired private DriverService driverService;
    @Autowired private NotificationService notificationService;

    private static final String GEO_KEY = "available_drivers";

    public Optional<String> findBestDriver(RideRequest request) {
        // 1. Find candidates within 5km radius
        Distance radius = new Distance(5, Metrics.KILOMETERS);
        Point riderLocation = new Point(request.getPickupLon(), request.getPickupLat());

        GeoResults<RedisGeoCommands.GeoLocation<String>> results =
                redis.opsForGeo().radius(GEO_KEY, riderLocation, radius,
                        RedisGeoCommands.GeoRadiusCommandArgs.newGeoRadiusArgs()
                                .includeDistance()
                                .sortAscending()
                                .limit(20));

        if (results == null || results.getContent().isEmpty()) {
            return Optional.empty();
        }

        // 2. Score each candidate
        return results.getContent().stream()
                .map(result -> {
                    String driverId = result.getContent().getName();
                    double distKm = result.getDistance().getValue();
                    DriverInfo driver = driverService.getDriverInfo(driverId);

                    double score = calculateScore(distKm, driver, request);
                    return new ScoredDriver(driverId, score);
                })
                .max(Comparator.comparingDouble(ScoredDriver::getScore))
                .map(ScoredDriver::getDriverId);
    }

    private double calculateScore(double distKm, DriverInfo driver, RideRequest request) {
        double proximityScore = 1.0 / Math.max(distKm, 0.1);  // avoid division by zero

        // Heading bonus: is driver moving toward rider?
        double headingBonus = 1.0;
        if (driver.getHeading() != null) {
            double bearingToRider = calculateBearing(driver.getLat(), driver.getLon(),
                    request.getPickupLat(), request.getPickupLon());
            double headingDiff = Math.abs(driver.getHeading() - bearingToRider);
            if (headingDiff < 45 || headingDiff > 315) {
                headingBonus = 1.5;  // driver is heading toward rider
            }
        }

        double ratingBonus = driver.getRating() / 5.0;

        return proximityScore * headingBonus * ratingBonus;
    }
}
```

### Surge Pricing — Real-Time Calculation

```java
@Scheduled(fixedRate = 60_000)  // every 60 seconds
public void recalculateSurgePricing() {
    // Get all active geohash cells (1km precision = geohash length 5)
    Set<String> activeCells = rideRequestRepository.getActiveCellsLastMinute();

    for (String geohashCell : activeCells) {
        // Count available drivers in cell
        long supply = countDriversInCell(geohashCell);
        // Count ride requests in last 60 seconds
        long demand = countRequestsInCell(geohashCell, Duration.ofMinutes(1));

        double surgeMultiplier = calculateMultiplier(supply, demand);

        // Store in Redis with 2-minute TTL (auto-expires if no activity)
        redis.opsForValue().set("surge:" + geohashCell,
                String.valueOf(surgeMultiplier), Duration.ofMinutes(2));
    }
}

private double calculateMultiplier(long supply, long demand) {
    if (supply == 0) return 2.5;  // no drivers → max surge
    double ratio = (double) demand / supply;
    if (ratio < 1.0) return 1.0;
    if (ratio < 1.5) return 1.1;
    if (ratio < 2.0) return 1.3;
    if (ratio < 3.0) return 1.8;
    return 2.5;
}
```

---

## 🏢 Industry Examples

| Company | Geospatial Approach | Notes |
|---|---|---|
| **Uber** | Redis GEORADIUS + H3 (Hexagonal Hierarchical Geospatial) | H3 used for surge pricing grids — hexagons tessellate perfectly with no gaps |
| **Lyft** | Similar Redis-based approach | Also uses geohash for zone-based pricing |
| **DoorDash** | PostGIS + Redis | Delivery zone boundaries use PostGIS polygon operations |
| **Google Maps** | QuadTree + R-tree | Complex hierarchy for multi-scale rendering |
| **Foursquare** | S2 geometry library | Google's S2 cells used for geo queries |

**Uber's actual tech** (public engineering blog):
- Switched from geohash to **H3** (Uber's own hexagonal grid) in 2018
- H3's advantage: hexagonal cells have uniform distance to all 6 neighbors (square cells have different distances to edge vs corner neighbors)
- Geofencing using S2 geometry for driver zone boundaries
- Ringpop (consistent hashing) for distributing driver ownership across services

---

## ⚠️ Common Pitfalls

1. **Storing driver locations in a relational database** — 125K location writes/second will overwhelm any SQL database. Redis geospatial is purpose-built for this access pattern and handles it trivially.

2. **Not considering the matching timeout problem** — What if the best driver doesn't respond in 15 seconds? You must cascade to the next driver. Design this explicitly with a timeout + retry mechanism.

3. **Forgetting to remove drivers from the available pool** — When a driver accepts a ride, they must be immediately removed from Redis `available_drivers` (ZREM). Otherwise, multiple riders could be matched to the same driver.

4. **Not distinguishing between driver states** — Drivers have at least 4 states: offline, available, on_trip, in_offer (waiting for response). The matching service must only query `available` drivers.

5. **Ignoring the "heading" dimension** — Pure distance matching is suboptimal. A driver 400m away but stuck in a one-way street heading away from you is worse than one 700m away heading toward you. Always discuss the heading bonus.

---

## 🧩 Mini Challenge

**Design question**: Uber is launching "Uber Pool" (shared rides where multiple riders share one car). What new matching challenges does this introduce? How would you change the matching service?

<details>
<summary>💡 Click to reveal answer</summary>

**New Challenges**:

1. **Multi-rider optimization**: Instead of 1 rider → nearest driver, you need to find a driver whose current route can accommodate a second pickup with minimal detour. This is the **Vehicle Routing Problem (VRP)** — NP-hard in its general form.

2. **Sequential matching**: For a driver already carrying Rider A, the system must check if Rider B's pickup and destination are "on the way" — within X% of detour from the current route.

**Simplified Matching for Pool**:

```
When Pool request comes in from Rider B:
1. Check active Pool rides (driver has 1 passenger, not yet picked up all riders)
2. For each candidate: calculate detour = route(A_pickup → B_pickup → A_dest → B_dest)
                                          vs original: route(A_pickup → A_dest)
3. If detour < 5 minutes AND both riders save money → match
4. Otherwise → treat as standard ride

Key data structure: 
  - Store current pool ride state: { driver_id, current_stops: [A_dest, B_pickup, B_dest] }
  - Route re-optimization after each pickup: reorder stops to minimize total distance
```

**Why this is complex at Uber's scale**: With 500K concurrent rides, checking every active Pool ride for every new Pool request would be O(active_rides) per request — too slow. Solution: Geospatial pruning first (only consider Pool rides within 3km of new rider), then route optimization on that small set.

</details>

---

## 📝 Interview Q&A

**Q: How does Uber track a driver's location with < 5 second latency on the rider's screen?**
> A: The driver app sends GPS coordinates to the Location Service every 4 seconds. The Location Service updates Redis (for matching) and publishes to Kafka. A streaming processor consumes location events and pushes them to the rider's app via WebSocket or Server-Sent Events. The rider's map smoothly interpolates between received positions using dead reckoning (predicting position based on speed/heading between updates).

**Q: What happens if a driver loses internet connection mid-ride?**
> A: The driver app buffers location updates locally and batches them when reconnected. The rider sees the last known location on their map. The ride is not cancelled — the driver_id is still associated with the ride in PostgreSQL. After 5 minutes without a location update, the backend flags the ride as "location unknown" and support is notified. The ride can still be completed manually.

**Q: How does Uber handle the case where there are no available drivers in an area?**
> A: If GEORADIUS returns 0 results within 5km, the Ride Service: (1) expands search radius to 10km, (2) if still none, notifies the rider with ETA from nearest available driver, (3) optionally increases surge multiplier to attract more drivers to that area. The rider can choose to wait or cancel.

**Q: How does Uber prevent a driver from being matched with multiple riders simultaneously?**
> A: The matching service uses Redis transactions (MULTI/EXEC) to atomically: (1) verify driver is still in `available_drivers`, (2) remove driver from `available_drivers`, (3) update driver status to `in_offer`. This is an atomic operation — if the driver was already taken by another match, the EXEC fails and the matcher tries the next candidate.

**Q: How would you design the ETA calculation?**
> A: ETA = (distance) / (average speed), but this is naive. Real ETAs use: (1) routing engine (Google Maps API or internal Hopper router) for road network distance, (2) real-time traffic data (Uber collects from all driver GPS histories), (3) historical pickup times for specific pickup locations (some streets are consistently slower). Uber's internal routing engine processes 1M+ route calculations per second.

---

## 🔗 What to Read Next

1. **[KeyConcepts/Consistent_Hashing.md](../KeyConcepts/Consistent_Hashing.md)** — How Uber distributes driver location state across Redis nodes
2. **[Design Notification System](./DesignNotificationSystem.md)** — How Uber pushes ride requests to driver apps in real-time
3. **[BuildingBlocks/RateLimiting.md](../BuildingBlocks/RateLimiting.md)** — Preventing location update abuse from malicious driver clients

---

*[← Design Netflix](./DesignNetflix.md) | [Back to Case Studies](./README.md) | [Next: Design Notification System →](./DesignNotificationSystem.md)*
