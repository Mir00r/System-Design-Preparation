# 🗺️ Design Google Maps: Navigation at Planet Scale

> *"Google Maps serves 1 BILLION users monthly, processes 20 PETABYTES of map data, and provides turn-by-turn navigation to millions of drivers simultaneously. When you ask for directions, the system searches through a graph with 50+ million road segments, considers real-time traffic from millions of phones, and returns the optimal route — all in under 500ms. This is graph theory, distributed computing, and ML at planetary scale."*

**⏱️ Estimated Time**: 50 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [Caching](../../BuildingBlocks/Caching.md), [CDN](../../BuildingBlocks/CDN.md), [Load Balancing](../../BuildingBlocks/LoadBalancing.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Scale Estimation](#-scale-estimation)
3. [High-Level Architecture](#-high-level-architecture)
4. [Map Tile Rendering](#-map-tile-rendering)
5. [Routing Engine](#-routing-engine)
6. [Real-time Traffic](#-real-time-traffic)
7. [ETA Prediction](#-eta-prediction)
8. [Location Services](#-location-services)
9. [Search & Geocoding](#-search--geocoding)
10. [Data Model](#-data-model)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
✅ Display interactive maps (zoom, pan, satellite view)
✅ Search for places (restaurants, addresses, coordinates)
✅ Turn-by-turn navigation (car, walking, transit, cycling)
✅ Real-time traffic conditions
✅ Estimated Time of Arrival (ETA)
✅ Alternative routes
✅ Nearby places discovery (gas stations, restaurants)
✅ Street View
✅ Offline maps download
```

### Non-Functional Requirements
```
✅ Low latency: map tiles load in < 200ms, routes in < 500ms
✅ High availability: 99.99% (navigation is safety-critical!)
✅ Support 1B monthly active users
✅ Global coverage (every road on earth)
✅ Real-time: traffic updates every 1-2 minutes
✅ Accuracy: routes within 5% of optimal, ETA within 10%
```

---

## 📊 Scale Estimation

```
USERS:
  • 1B monthly active, 150M daily active
  • 100M active navigation sessions daily
  • Peak: 20M concurrent users viewing maps

MAP DATA:
  • Road network graph: 50M+ road segments, 30M+ intersections
  • Total map data: 20+ PB (satellite imagery, street view, terrain)
  • Map tiles (all zoom levels): ~50 TB (vector tiles)
  • Points of Interest (POI): 200M+ businesses/places

REQUESTS:
  • Tile requests: 100B/day (each pan/zoom = multiple tiles)
  • Route requests: 500M/day
  • Search requests: 1B/day
  • Location updates (traffic): 1B+ GPS pings/day from phones

BANDWIDTH:
  • Each map tile: 10-50 KB (vector) or 50-200 KB (raster)
  • Navigation session: ~1 tile/second × 30 min = 1800 tiles
  • Peak tile serving: ~5M tiles/second
```

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                   │
│    📱 Mobile App    💻 Web App    🚗 Car Display                 │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   CDN (Map Tiles)   │ ← 95% of tile requests served here!
                    └──────────┬──────────┘
                               │ (cache miss)
                    ┌──────────▼──────────┐
                    │   API Gateway       │
                    └──────────┬──────────┘
                               │
     ┌─────────────────────────┼─────────────────────────────┐
     │              │          │           │                  │
     ▼              ▼          ▼           ▼                  ▼
┌─────────┐  ┌──────────┐  ┌────────┐  ┌─────────┐  ┌────────────┐
│  Tile   │  │ Routing  │  │ Search │  │ Traffic │  │ Location   │
│ Service │  │ Service  │  │Service │  │ Service │  │ Service    │
└─────────┘  └──────────┘  └────────┘  └─────────┘  └────────────┘
     │              │          │           │
     ▼              ▼          ▼           ▼
┌──────────────────────────────────────────────────────────────────┐
│                       DATA LAYER                                  │
│ ┌────────────┐ ┌───────────┐ ┌──────────┐ ┌──────────────────┐ │
│ │ Tile Cache │ │Road Graph │ │POI Index │ │ Traffic Stream   │ │
│ │ (CDN+Redis)│ │(Sharded)  │ │(Elastic) │ │ (Kafka)         │ │
│ └────────────┘ └───────────┘ └──────────┘ └──────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🗺️ Map Tile Rendering

```
THE TILING SYSTEM:

  The world is divided into tiles at multiple zoom levels:
  
  Zoom 0: Entire world = 1 tile (256×256 pixels)
  Zoom 1: World = 4 tiles (2×2 grid)
  Zoom 2: World = 16 tiles (4×4 grid)
  ...
  Zoom 18: World = 68 BILLION tiles (streets visible!)
  Zoom 21: Individual buildings!
  
  Total tiles (all zoom levels): ~2^(2×21) per zoom = ~50 TB
  
  Each tile identified by: (zoom, x, y)
  URL: /tiles/{zoom}/{x}/{y}.png  (or .mvt for vector)

VECTOR TILES vs RASTER TILES:
  ┌───────────────┬────────────────┬──────────────────────┐
  │               │  Raster (PNG)  │  Vector (MVT/PBF)    │
  ├───────────────┼────────────────┼──────────────────────┤
  │  Size         │  50-200 KB     │  10-50 KB            │
  │  Rendering    │  Server-side   │  Client-side (GPU!)  │
  │  Rotation     │  Pixelated     │  Smooth              │
  │  Styling      │  Fixed         │  Dynamic (dark mode!)│
  │  Labels       │  Fixed angle   │  Always upright      │
  │  Bandwidth    │  Higher        │  60-80% less         │
  └───────────────┴────────────────┴──────────────────────┘
  
  Google Maps uses VECTOR tiles (since 2013):
  • Send geometry + metadata → client renders with GPU
  • Allows real-time style changes (dark mode, terrain, transit)
  • Dramatically reduces bandwidth

CACHING STRATEGY:
  CDN → Redis → Pre-rendered → On-demand render
  
  Hit rates:
    CDN: 95% (most tiles are viewed by many users)
    Redis: 3% (recently accessed, not yet in CDN)
    Pre-rendered: 1.5% (popular zoom levels pre-computed)
    On-demand: 0.5% (rare tiles rendered on request)
```

---

## 🛣️ Routing Engine

```
THE ROAD NETWORK AS A GRAPH:
  
  Nodes: Intersections (30M+ worldwide)
  Edges: Road segments between intersections (50M+)
  Edge weights: Travel time (NOT just distance!)
  
  Weight factors:
    • Road length
    • Speed limit
    • Road type (highway vs residential)
    • Number of lanes
    • Turn costs (left turns take longer)
    • Traffic lights
    • Real-time traffic conditions!

BASIC DIJKSTRA IS TOO SLOW:
  Dijkstra on 50M nodes = O(V log V) = way too slow for real-time!
  
  Google uses CONTRACTION HIERARCHIES:
  
  Preprocessing (offline, takes hours):
    1. Order nodes by "importance" (highways > local roads)
    2. Create "shortcut edges" that skip unimportant nodes
    3. Build hierarchy: highways at top, local roads at bottom
    
  Query time (online):
    1. Bidirectional search (from source AND destination)
    2. Only explore UPWARD in hierarchy
    3. Meet in the middle at highway level
    
  Result: Route found in ~1ms (instead of seconds!) 🚀
  
  ┌──────────────────────────────────────────────────────┐
  │  Level 3 (Highways):     A ═══════════════════ B     │
  │                          │ (shortcut: 45 min)  │     │
  │  Level 2 (Main roads):   │     C───D───E       │     │
  │                          │                     │     │
  │  Level 1 (Local):        │  f─g─h  i─j─k     │     │
  │                          │                     │     │
  │  Start: 'f' → go UP hierarchy → highway shortcut    │
  │  → come DOWN at destination → local routing          │
  └──────────────────────────────────────────────────────┘

ALTERNATIVE ROUTES:
  Don't just return one route — give 3 alternatives!
  
  Algorithm: Penalty method
    1. Find shortest route R1
    2. Penalize edges in R1 (increase weight by 2x)
    3. Find shortest route R2 (will use different roads)
    4. Penalize R2 edges too
    5. Find R3
  
  Filter: alternatives must differ by > 30% of edges
```

---

## 🚦 Real-time Traffic

```
DATA SOURCES:
  • 1B+ GPS pings/day from Android phones (anonymous!)
  • Partner data (taxis, delivery trucks, Waze users)
  • Traffic sensors (highways)
  • Incident reports (accidents, construction)
  • Event data (concerts, sports games)

HOW TRAFFIC SPEED IS COMPUTED:
  
  Road segment "Main St between 1st Ave and 2nd Ave":
    
  Phones traveling on this segment in last 5 minutes:
    Phone A: traveled 500m in 45 seconds → 40 km/h
    Phone B: traveled 500m in 50 seconds → 36 km/h
    Phone C: traveled 500m in 120 seconds → 15 km/h (stuck!)
    Phone D: traveled 500m in 44 seconds → 41 km/h
    
  Aggregated speed: median = 38 km/h (discard outliers)
  Normal speed for this road: 50 km/h
  Traffic ratio: 38/50 = 0.76 → YELLOW (moderate traffic)
  
  Color coding:
    Green:  > 90% of normal speed (free flow)
    Yellow: 50-90% of normal speed (moderate)
    Red:    25-50% of normal speed (heavy)
    Dark red: < 25% of normal speed (standstill!)

PROCESSING PIPELINE:
  Phones → Kafka (location events) → Flink (real-time aggregation)
  → Traffic Speed per segment (updated every 1-2 min)
  → Stored in Redis for route computation
  → Published to map tiles for visualization
```

---

## ⏱️ ETA Prediction

```
ETA ≠ simple (distance / speed limit)!

ETA MODEL FACTORS:
  • Route distance and road types
  • Current traffic speeds (per segment!)
  • Historical traffic patterns (Tuesday 8am is always slow here)
  • Traffic signals and stop signs
  • Predicted traffic (ML: "accident ahead will clear in 20 min")
  • Weather conditions
  • Time of day / day of week

ML MODEL (DeepMind partnership):
  Google uses Graph Neural Networks on the road graph!
  
  Input: Current traffic state (speed per segment) + time features
  Output: Predicted speeds for next 30/60 minutes per segment
  
  This allows: "You'll reach the highway in 10 minutes.
                By then, the current jam will have cleared."
  
  Accuracy: Within 5% of actual arrival time (most trips)

ETA UPDATES DURING NAVIGATION:
  Recalculate every 60 seconds:
    • Current position (GPS)
    • Remaining route segments
    • Updated traffic for those segments
    • Any new incidents reported ahead
```

---

## 📍 Location Services

```
GPS ACCURACY ENHANCEMENT:

  Raw GPS: ±5-15 meters accuracy
  
  Google enhances with:
    • WiFi fingerprinting (known WiFi locations)
    • Cell tower triangulation
    • Barometric pressure (altitude / floor detection)
    • Sensor fusion (accelerometer, gyroscope)
    
  Result: ±2-3 meter accuracy in cities!

MAP MATCHING:
  GPS says you're at (40.7128, -74.0060) — but that's in a building!
  Map matching: "Snap" GPS point to nearest road segment.
  
  Uses Hidden Markov Model:
    • Observation: GPS coordinates (noisy)
    • Hidden state: actual road segment
    • Transition: probability of moving between segments
    
  Result: Smooth path along actual roads, even with GPS noise

GEOFENCING:
  "Notify me when I'm near a gas station"
  → Server maintains geofence circles
  → Client reports location → server checks against geofences
  → Push notification when entering/exiting a geofence
```

---

## 🔍 Search & Geocoding

```
GEOCODING = Converting address to coordinates
  "1600 Pennsylvania Ave, Washington DC" → (38.8977, -77.0365)
  
REVERSE GEOCODING = Converting coordinates to address
  (40.7484, -73.9857) → "Empire State Building, NYC"

PLACE SEARCH:
  "Pizza near me" → 
    1. Geocode "me" → current location
    2. Search POI index for "pizza" within 5km radius
    3. Rank by: distance, rating, popularity, open now
    4. Return top 20 results with details

SEARCH ARCHITECTURE:
  ┌──────────┐  "pizza near me"  ┌──────────────┐
  │  Client  │─────────────────►│  Search API  │
  └──────────┘                  └──────┬───────┘
                                       │
                          ┌────────────┼──────────────┐
                          ▼            ▼              ▼
                   ┌──────────┐ ┌───────────┐ ┌────────────┐
                   │  Text    │ │  Geo      │ │  Ranking   │
                   │  Index   │ │  Index    │ │  Service   │
                   │  (name,  │ │  (R-tree/ │ │  (distance │
                   │  category)│ │  geohash) │ │  +rating)  │
                   └──────────┘ └───────────┘ └────────────┘

GEOHASH for spatial indexing:
  World divided into grid cells, each with a hash prefix:
  
  "9q8yy" = San Francisco area
  "9q8yyk" = more specific area in SF
  "9q8yykb" = even more specific (block-level)
  
  Nearby places share common geohash prefixes!
  Index: geohash_prefix → [poi_1, poi_2, ...]
  Query: find all POIs matching "9q8yy*" → nearby places!
```

---

## 🗄️ Data Model

```
ROAD GRAPH (Sharded by geographic region):
  Node: {
    id: int64,
    lat: float,
    lng: float,
    type: "intersection" | "turn" | "merge"
  }
  
  Edge: {
    id: int64,
    source_node: int64,
    target_node: int64,
    length_meters: int,
    road_name: string,
    road_type: "highway" | "primary" | "residential",
    speed_limit_kmh: int,
    lanes: int,
    one_way: boolean,
    current_speed_kmh: float,  // Real-time from traffic service!
    contraction_level: int     // For hierarchical routing
  }

POI (Elasticsearch):
  {
    "id": "poi_12345",
    "name": "Joe's Pizza",
    "category": ["restaurant", "pizza"],
    "location": {"lat": 40.7484, "lon": -73.9857},
    "rating": 4.5,
    "price_level": 2,
    "opening_hours": {...},
    "phone": "+1-555-0123"
  }

TILES (Object Storage + CDN):
  Path: /tiles/v3/{zoom}/{x}/{y}.mvt
  Metadata: {version, last_updated, bbox}
  
  Pre-computed for zoom levels 0-14 (global)
  On-demand for zoom 15+ (too many tiles to pre-compute!)
```

---

## 🎮 Mini Challenge

### 🧩 Design: Real-time Rerouting

A user is navigating and hits unexpected traffic 5km ahead. Design the rerouting system.

<details>
<summary>🔑 Answer</summary>

**Detection:**
1. Traffic Service detects speed drop on upcoming road segments (from phone GPS data)
2. ETA Service recalculates: if new ETA is > 10 minutes longer than alternative → trigger reroute

**Rerouting Flow:**
1. Navigation Service receives traffic update for user's remaining route
2. Compute alternative: run routing algorithm from current position to destination (with updated traffic)
3. Compare: if alternative saves > 5 minutes → notify user
4. User accepts → update navigation to new route
5. Push updated route + tiles to client

**Key Design Decisions:**
- Don't reroute for < 5 min savings (annoying!)
- Pre-compute alternatives every 60 seconds during active navigation
- Consider "future traffic": current jam might clear before user arrives
- Avoid sending ALL users on same alternative (causes NEW jam!)
  - Solution: split traffic across 2-3 alternatives proportionally

**Scale:**
- 100M active navigation sessions
- Only ~5% need rerouting at any time = 5M reroute evaluations
- Each reroute: ~10ms on contraction hierarchy
- Total: 5M × 10ms / 60s = 833K routes/sec (distributed across routing cluster)
</details>

---

## ❓ Interview Q&A

**Q1: How do you compute routes quickly on a graph with 50M nodes?**
> Contraction Hierarchies: preprocess the graph offline to create shortcuts (e.g., "highway from A to B takes 45 min" without expanding intermediate nodes). At query time, run bidirectional search that only explores upward in the hierarchy. Query time: ~1ms instead of seconds for Dijkstra. Preprocessing takes hours but only runs when map data changes.

**Q2: How does Google Maps know real-time traffic conditions?**
> Anonymous GPS data from billions of Android phones. When phones travel along road segments, their speed is calculated. For each road segment, aggregate speed from all phones in the last 5 minutes. Compare to normal speed → derive traffic level. Processing: Kafka ingests 1B+ location events/day, Flink computes real-time aggregates per segment, stores in Redis for routing.

**Q3: How do you serve map tiles to 1 billion users?**
> Tiling system: world divided into tiles at 22 zoom levels. Vector tiles (10-50KB) rendered client-side. CDN caches popular tiles globally (95% hit rate). Pre-compute tiles for zoom 0-14. Generate zoom 15+ on-demand with aggressive caching. Multiple CDN providers for redundancy. Client caches recently viewed tiles locally.

**Q4: How do you estimate arrival time accurately?**
> Not just distance/speed! Use ML model (Graph Neural Networks) that considers: current traffic per segment, historical patterns (Tuesday 8am), predicted traffic evolution (jam clearing), signals, road type, weather. Retrained daily on billions of completed trips. During navigation: recalculate every 60 seconds with updated traffic. Accuracy: within 5% for most trips.

**Q5: How do you handle offline maps?**
> User downloads a geographic region: all tiles (zoom 0-16), routing graph for that area, POI data. Stored locally (~500MB for a large city). Offline routing: run Dijkstra on local graph (no traffic data). Sync when online: upload GPS traces (contributes to traffic), download updated POI data. Challenge: keeping downloaded maps fresh (differential updates monthly).

---

## 🔗 Related Topics
- [CDN](../../BuildingBlocks/CDN.md) — Tile delivery at scale
- [Caching](../../BuildingBlocks/Caching.md) — Multi-layer tile caching
- [Search Index](../../BuildingBlocks/SearchIndex.md) — Place search
- [Design Uber](./DesignUber.md) — Similar location tracking + routing

---

*"Google Maps processes more graph queries per second than any other system on Earth. Your navigation to the grocery store involves more computation than early Moon missions." — Mapping Infrastructure* 🗺️
