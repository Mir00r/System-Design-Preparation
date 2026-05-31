# 🍕 Design a Food Delivery System (Uber Eats/DoorDash/Zomato)

> *"DoorDash processes 25 million orders per month across 7,000 cities. The system must match orders to the OPTIMAL delivery driver within 30 seconds, predict delivery time within 5-minute accuracy, handle dinner rush spikes of 10x normal traffic, and ensure your food arrives hot — all while tracking real-time location of 6 million drivers. This is logistics orchestration at its finest."*

**⏱️ Estimated Time**: 50 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [Load Balancing](../../BuildingBlocks/LoadBalancing.md), [Message Queues](../../BuildingBlocks/MessageQueues.md), [Caching](../../BuildingBlocks/Caching.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Scale Estimation](#-scale-estimation)
3. [High-Level Architecture](#-high-level-architecture)
4. [Core Services](#-core-services)
5. [Order Flow](#-order-flow)
6. [Driver Matching Algorithm](#-driver-matching-algorithm)
7. [Location Tracking](#-location-tracking)
8. [ETA Prediction](#-eta-prediction)
9. [Handling Dinner Rush](#-handling-dinner-rush)
10. [Data Model](#-data-model)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
CUSTOMER:
  ✅ Browse restaurants near their location
  ✅ Search menu items, filter by cuisine/rating/price
  ✅ Place orders (cart → checkout → payment)
  ✅ Track order + delivery in real-time on map
  ✅ Rate restaurants and drivers
  
RESTAURANT:
  ✅ Manage menu (items, prices, availability)
  ✅ Accept/reject orders
  ✅ Update preparation time and order status
  
DRIVER:
  ✅ Go online/offline
  ✅ Accept/reject delivery offers
  ✅ Navigate to restaurant → pick up → deliver
  ✅ Mark order as picked up / delivered
```

### Non-Functional Requirements
```
  ✅ Low latency: < 3s to display restaurants, < 30s to match driver
  ✅ High availability: 99.99% (especially during peak hours!)
  ✅ Real-time tracking: Location updates every 3-5 seconds
  ✅ Handle 10x traffic spikes (dinner rush, rainy days)
  ✅ Accurate ETA: within 5 minutes
  ✅ Payment consistency: exactly-once payment processing
```

---

## 📊 Scale Estimation

```
USERS:
  • 50M registered customers, 10M DAU
  • 500K restaurants
  • 2M registered drivers, 200K online at peak
  • 2M orders/day, peak 100K orders/hour (dinner rush)

LOCATION DATA:
  • 200K drivers × update every 4 seconds = 50K location updates/sec
  • Each update: {driver_id, lat, lng, timestamp, heading, speed} = ~100 bytes
  • Daily location data: 50K × 86,400 sec × 100 bytes = ~430 GB/day!

API CALLS:
  • Restaurant search: 1M searches/hour at peak
  • Order tracking: 100K concurrent tracking sessions
  • Menu loads: 5M/hour
  
STORAGE:
  • Orders: 2M/day × 365 days × 2KB = ~1.5 TB/year
  • Location history: 430 GB/day × 30 days = 13 TB (hot storage)
```

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                   │
│  📱 Customer App    📱 Driver App    💻 Restaurant Dashboard    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   API Gateway   │  Auth, Rate Limit, Routing
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────────────┐
        │                    │                            │
        ▼                    ▼                            ▼
┌──────────────┐   ┌─────────────────┐         ┌──────────────────┐
│  Restaurant  │   │  Order Service  │         │  User/Auth       │
│  Service     │   │                 │         │  Service         │
└──────────────┘   └────────┬────────┘         └──────────────────┘
                            │
              ┌─────────────┼──────────────────┐
              │             │                  │
              ▼             ▼                  ▼
┌───────────────┐  ┌──────────────┐   ┌──────────────────┐
│  Payment      │  │  Matching    │   │  Notification    │
│  Service      │  │  Service     │   │  Service         │
└───────────────┘  └──────┬───────┘   └──────────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
┌────────────────┐ ┌────────────┐ ┌──────────────────┐
│  Location      │ │  ETA       │ │  Pricing/Surge   │
│  Service       │ │  Service   │ │  Service         │
└────────────────┘ └────────────┘ └──────────────────┘

DATA STORES:
┌──────────────────────────────────────────────────────────────────┐
│  PostgreSQL (orders, users, restaurants) — strong consistency    │
│  Redis (driver locations, sessions, cache) — speed              │
│  Elasticsearch (restaurant/menu search) — full-text             │
│  Cassandra (location history, analytics) — write throughput     │
│  Kafka (events: orders, locations, notifications) — async       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🧩 Core Services

### Restaurant Service
```
Responsibilities:
  • Restaurant catalog (CRUD)
  • Menu management
  • Operating hours & availability
  • Search by location, cuisine, rating
  
Search flow:
  1. User sends location (lat, lng) + filters
  2. Geospatial query: restaurants within 5km radius
  3. Filter by: open now, minimum rating, cuisine type
  4. Rank by: distance, rating, delivery time, promotion
  5. Return paginated results with estimated delivery time
  
Technology:
  • Elasticsearch with geo_distance query
  • Redis geo for nearby driver count (affects ETA)
  • Pre-computed restaurant scores updated hourly
```

### Order Service (State Machine)
```
ORDER STATES:
  ┌──────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐
  │  PLACED  │───►│ ACCEPTED │───►│ PREPARING │───►│  READY   │
  └──────────┘    └──────────┘    └───────────┘    └──────────┘
       │                                                │
       │ (rejected)                                     │
       ▼                                                ▼
  ┌──────────┐                                   ┌──────────┐
  │ CANCELLED│                                   │ PICKED_UP│
  └──────────┘                                   └──────────┘
                                                       │
                                                       ▼
                                                 ┌───────────┐
                                                 │ DELIVERED │
                                                 └───────────┘
  
  Each state transition → Kafka event → notifications pushed to all parties
```

---

## 🚗 Order Flow

```
COMPLETE ORDER LIFECYCLE:

1. CUSTOMER PLACES ORDER:
   Client → Order Service: Create order
   Order Service:
     • Validate items + prices
     • Calculate total (subtotal + delivery fee + tax + tip)
     • Create order record (status: PLACED)
     • Publish OrderPlaced event to Kafka

2. RESTAURANT ACCEPTS:
   Restaurant Dashboard → Accept order
   Order Service: status → ACCEPTED
   Timer: If no response in 5 min → auto-accept or cancel

3. DRIVER MATCHING (parallel with restaurant prep):
   Matching Service receives OrderPlaced event
   • Find available drivers near restaurant
   • Score candidates (distance, rating, capacity, direction)
   • Send offer to best candidate
   • Driver has 30 seconds to accept
   • If declined → offer to next candidate
   • If no one accepts in 3 attempts → expand radius

4. FOOD PREPARATION:
   Restaurant marks stages: Preparing → Ready for Pickup
   ETA Service updates customer's expected delivery time

5. PICKUP:
   Driver navigates to restaurant
   Driver marks "Picked Up" → status: PICKED_UP
   Customer gets notification + live map tracking

6. DELIVERY:
   Driver navigates to customer
   Driver marks "Delivered" → status: DELIVERED
   Payment finalized, ratings requested

TIMING (average):
  Order placed → Accepted: 2 min
  Accepted → Ready: 15-20 min (restaurant prep)
  Driver matched: 30 sec - 3 min (parallel)
  Pickup → Delivery: 10-15 min
  TOTAL: ~35-45 minutes
```

---

## 🎯 Driver Matching Algorithm

```
MATCHING CRITERIA (weighted score):

  Score = w1 × distance_score 
        + w2 × direction_score
        + w3 × rating_score
        + w4 × acceptance_rate
        + w5 × delivery_capacity

WHERE:
  distance_score = 1 - (driver_to_restaurant / max_radius)
  direction_score = heading alignment (is driver already going that way?)
  rating_score = driver_rating / 5.0
  acceptance_rate = recent acceptance % (penalize frequent decliners)
  delivery_capacity = can handle stacked orders?

MATCHING ALGORITHM:
  ┌─────────────────────────────────────────────────────────────┐
  │  1. New order arrives                                       │
  │  2. Query: drivers within 3km of restaurant, status=IDLE    │
  │  3. Score each candidate                                    │
  │  4. Sort by score descending                                │
  │  5. Offer to top candidate (30s timeout)                    │
  │  6. If declined/timeout → offer to #2                       │
  │  7. If 3 rejections → expand radius to 5km                  │
  │  8. If still no match → surge pricing to attract drivers    │
  └─────────────────────────────────────────────────────────────┘

BATCHED MATCHING (DoorDash approach):
  Instead of matching one-by-one:
  • Collect orders for 30-second windows
  • Run global optimization: assign ALL orders to ALL drivers
  • Minimize total delivery time across the batch!
  • This is an assignment problem (Hungarian algorithm / linear programming)
```

---

## 📍 Location Tracking

```
DRIVER LOCATION UPDATES:
  Driver app sends location every 3-5 seconds when online.
  
  ┌──────────┐  GPS update   ┌──────────────┐
  │  Driver  │──────────────►│  Location    │
  │  App     │  every 4s     │  Service     │
  └──────────┘               └──────┬───────┘
                                    │
                      ┌─────────────┼─────────────┐
                      ▼             ▼             ▼
              ┌──────────┐  ┌────────────┐  ┌──────────────┐
              │  Redis   │  │  Kafka     │  │  Customer    │
              │  (GeoSet)│  │  (history) │  │  WebSocket   │
              └──────────┘  └────────────┘  └──────────────┘
              Current pos    Persist for     Push to tracking
              for matching   analytics       customers

REDIS GEO for nearby drivers:
  GEOADD drivers 77.1025 28.7041 "driver_123"
  GEORADIUS drivers 77.1025 28.7041 3 km COUNT 20
  → Returns 20 nearest drivers within 3km!

CUSTOMER LIVE TRACKING:
  Customer opens tracking → WebSocket connection established
  Location Service pushes driver position updates every 4 seconds
  Client renders on map with smooth interpolation between points
```

---

## ⏱️ ETA Prediction

```
ETA = Prep Time + Pickup Time + Delivery Time

FACTORS:
  • Restaurant historical prep time (by time of day, order size)
  • Driver distance to restaurant (routing, not straight line!)
  • Restaurant to customer distance
  • Traffic conditions (time of day, real-time data)
  • Weather (rain → slower driving, more orders!)
  • Restaurant backlog (20 pending orders = longer wait)

ML MODEL:
  Features: [time_of_day, day_of_week, restaurant_id, order_items_count,
             driver_distance, traffic_index, weather_code, restaurant_queue_size]
  
  Output: predicted_minutes_to_delivery
  
  Model: Gradient Boosted Trees (XGBoost)
  Training data: Millions of historical deliveries with actual times
  Updated: Daily retraining
  
  Accuracy target: 80% of predictions within 5 minutes of actual
```

---

## 🌊 Handling Dinner Rush

```
TRAFFIC PATTERN:
  
  Orders/hour:
  100K ┤                    ╭──╮
       │                   ╱    ╲
  80K  ┤                  ╱      ╲
       │                 ╱        ╲
  60K  ┤       ╭──╮    ╱          ╲
       │      ╱    ╲  ╱            ╲
  40K  ┤     ╱      ╲╱              ╲
       │    ╱                        ╲
  20K  ┤───╱                          ╲───
       │
  10K  ┤
       └────┬────┬────┬────┬────┬────┬────
            8am  11am 2pm  5pm  8pm  11pm

  Lunch: 3x normal | Dinner: 10x normal!
  Rain: +40% orders | Game night: +60% in delivery areas near stadium

STRATEGIES:
  1. Auto-scaling: Pre-warm instances before 5pm
  2. Queue-based: Orders buffered in Kafka (absorb spikes)
  3. Surge pricing: Higher fees → attract more drivers online
  4. Rate limiting: Cap orders per restaurant (kitchen capacity)
  5. Degradation: Disable non-essential features (recommendations)
  6. Caching: Aggressively cache restaurant/menu data
```

---

## 🗄️ Data Model

```sql
-- Core entities (PostgreSQL)
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    restaurant_id UUID NOT NULL,
    driver_id UUID,
    status VARCHAR(20) NOT NULL DEFAULT 'PLACED',
    items JSONB NOT NULL,
    subtotal DECIMAL(10,2),
    delivery_fee DECIMAL(10,2),
    tax DECIMAL(10,2),
    tip DECIMAL(10,2),
    total DECIMAL(10,2),
    delivery_address JSONB,
    placed_at TIMESTAMP DEFAULT NOW(),
    accepted_at TIMESTAMP,
    ready_at TIMESTAMP,
    picked_up_at TIMESTAMP,
    delivered_at TIMESTAMP,
    estimated_delivery_time TIMESTAMP
);

CREATE TABLE restaurants (
    id UUID PRIMARY KEY,
    name VARCHAR(200),
    cuisine_type VARCHAR(50)[],
    location GEOGRAPHY(POINT, 4326), -- PostGIS
    rating DECIMAL(2,1),
    avg_prep_time_minutes INT,
    is_open BOOLEAN,
    operating_hours JSONB
);

-- Driver location (Redis)
-- GEOADD active_drivers <lng> <lat> <driver_id>
-- Refreshed every 4 seconds per driver

-- Location history (Cassandra - high write throughput)
CREATE TABLE driver_locations (
    driver_id UUID,
    timestamp TIMESTAMP,
    latitude DOUBLE,
    longitude DOUBLE,
    heading FLOAT,
    speed FLOAT,
    PRIMARY KEY (driver_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC)
  AND default_time_to_live = 2592000; -- 30 day TTL
```

---

## 🎮 Mini Challenge

### 🧩 Design: Stacked Orders (Batching)

DoorDash allows drivers to pick up from 2 restaurants on the same trip. Design this feature considering:
- Customer A orders from Restaurant X (1km from driver)
- Customer B orders from Restaurant Y (1.5km from driver, 0.5km from X)
- Both deliveries are in the same direction

<details>
<summary>🔑 Answer</summary>

**Matching Algorithm Enhancement:**
1. When driver picks up Order A, check for "stackable" orders:
   - Restaurant B is within 1km of Restaurant A or en route
   - Delivery B is within 2km of Delivery A or en route
   - Adding order B increases A's delivery time by < 10 minutes

2. **Routing optimization:**
   - Calculate: A_restaurant → B_restaurant → A_delivery → B_delivery
   - Compare with: A_restaurant → A_delivery (solo)
   - Accept if A's delay < 10 min AND total distance saved > 2km

3. **Customer communication:**
   - Customer A: "Driver making one more pickup nearby. +5 min"
   - Customer B: Gets lower delivery fee (batched = cheaper!)

4. **Compensation:**
   - If Customer A's delivery is delayed > 10 min due to batching → credit
   - Driver gets bonus for completing stacked delivery efficiently
</details>

---

## ❓ Interview Q&A

**Q1: How would you handle the driver matching problem?**
> Geospatial index (Redis GEO) to find nearby available drivers. Score candidates by: distance to restaurant, heading direction, rating, acceptance rate. Offer to highest-scoring driver with 30-second timeout. On decline: cascade to next. Advanced: batch orders in 30-second windows and solve global assignment optimization (minimize total delivery time across all orders and drivers).

**Q2: How do you handle real-time location tracking for millions of drivers?**
> Drivers send GPS updates every 4 seconds. Location Service writes to: (1) Redis GEO for real-time nearest-driver queries, (2) Kafka for event streaming to tracking consumers, (3) Cassandra for historical analysis. Customer tracking via WebSocket: Location Service pushes driver position to customer's active WebSocket connection. Scale: 200K drivers × 0.25 updates/sec = 50K writes/sec to Redis (easily handled).

**Q3: How do you estimate delivery time accurately?**
> ETA = restaurant prep time + driver-to-restaurant time + restaurant-to-customer time. Use ML model (XGBoost) trained on historical deliveries with features: time of day, restaurant historical prep time, current queue length, traffic conditions, weather, distance. Update prediction in real-time as order progresses (prep done earlier/later than expected). Target: 80% within 5 minutes of actual.

**Q4: How do you handle the dinner rush (10x traffic spike)?**
> (1) Pre-scale: auto-scaling rules triggered at 4:30pm based on historical patterns, (2) Queue buffering: Kafka absorbs order burst (restaurant confirms at its pace), (3) Surge pricing: higher delivery fees attract more drivers online, (4) Rate limiting: cap orders per restaurant based on kitchen capacity, (5) Graceful degradation: disable non-essential features (recommendations, reviews) during peak.

**Q5: How do you ensure payment consistency?**
> Two-step payment: (1) Authorization hold when order placed (reserve funds, don't charge), (2) Capture on delivery confirmation. Use idempotency keys for payment API calls (prevent double-charge on retry). If delivery fails: release hold, refund. Saga pattern for the full flow: if any step fails, compensating transactions undo previous steps.

---

## 🔗 Related Topics
- [Design Uber/Lyft](./DesignUber.md) — Similar driver matching + location tracking
- [Rate Limiting](../../BuildingBlocks/RateLimiting.md) — Handling traffic spikes
- [WebSockets](../../APIs/WebSockets.md) — Real-time tracking
- [Consistent Hashing](../../KeyConcepts/ConsistentHashing.md) — Partitioning location data

---

*"Food delivery is where logistics meets real-time systems meets machine learning. Get the matching wrong by 2 minutes, and someone's pizza is cold." — DoorDash Engineering* 🍕
