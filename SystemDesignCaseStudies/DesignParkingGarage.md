# 🚗 Design a Parking Garage System

> *"This is the classic OOP-meets-system-design interview question! It starts simple — 'model a parking lot' — but quickly evolves into real engineering challenges: concurrency (two cars see the same 'empty' spot), distributed sensors, payment processing, and scaling to multi-level garages in a city. Master this and you prove you can take a simple problem and think through it systematically."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [ACID](../Database/ACID.md), [Caching](../BuildingBlocks/Caching.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Object-Oriented Design](#-object-oriented-design)
3. [High-Level Architecture](#-high-level-architecture)
4. [Concurrency Handling](#-concurrency-handling)
5. [Scaling to Multiple Garages](#-scaling-to-multiple-garages)
6. [Java Implementation](#-java-implementation)
7. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Multiple floors, multiple spot types (compact, regular, large, handicap, EV)
  • Vehicle entry: assign nearest available spot, issue ticket
  • Vehicle exit: calculate fee based on duration, process payment
  • Display available spots per floor (LED boards!)
  • Support multiple entry/exit points
  • Reservation (pre-book a spot for specific time!)
  
NON-FUNCTIONAL:
  • Concurrency: multiple cars entering simultaneously!
  • Real-time: availability display updates instantly!
  • Reliability: payment processing must not lose transactions
  • Throughput: handle peak rush (100 cars/min entering!)

CAPACITY EXAMPLE:
  5 floors × 200 spots/floor = 1000 total spots
  Types: 20% compact, 50% regular, 20% large, 5% handicap, 5% EV
```

---

## 🏗️ Object-Oriented Design

```
CORE ENTITIES:

  ┌─────────────────┐     ┌─────────────────┐
  │   ParkingGarage  │────►│   Floor          │
  │   - name         │     │   - floorNumber   │
  │   - address      │     │   - spots[]       │
  │   - floors[]     │     │   - displayBoard  │
  └─────────────────┘     └────────┬─────────┘
                                    │
                           ┌────────▼─────────┐
                           │   ParkingSpot     │
                           │   - spotId        │
                           │   - type (enum)   │
                           │   - status (FREE/ │
                           │     OCCUPIED/     │
                           │     RESERVED)     │
                           │   - vehicle       │
                           └──────────────────┘
  
  ┌─────────────────┐     ┌─────────────────┐
  │   Vehicle        │     │   Ticket         │
  │   - licensePlate │     │   - ticketId     │
  │   - type         │     │   - entryTime    │
  │   - color        │     │   - exitTime     │
  └─────────────────┘     │   - spot         │
                           │   - vehicle      │
                           │   - amount       │
                           │   - isPaid       │
                           └─────────────────┘
  
  ┌─────────────────┐     ┌─────────────────┐
  │   EntryGate      │     │   ExitGate       │
  │   - scan ticket  │     │   - scan ticket  │
  │   - assign spot  │     │   - calculate fee│
  │   - raise barrier│     │   - process pay  │
  └─────────────────┘     └─────────────────┘

SPOT ASSIGNMENT STRATEGY:
  • Nearest to entry → minimize walking distance!
  • Fill floor by floor (floor 1 first, then 2, etc.)
  • Match vehicle type to spot type:
    Motorcycle → compact only
    Car → compact or regular
    Truck/Van → large only
    Handicap → handicap spots (with permit!)
```

---

## 🖥️ High-Level Architecture

```
SYSTEM ARCHITECTURE:

  ┌──────────────────────────────────────────────────────────────────┐
  │                     PARKING GARAGE SYSTEM                          │
  │                                                                    │
  │  Entry Gates (IoT)         Core Services         Exit Gates (IoT) │
  │  ┌────────────┐            ┌──────────────┐      ┌────────────┐  │
  │  │ Camera     │───────────►│  Parking     │◄─────│ Camera     │  │
  │  │ (plate OCR)│            │  Service     │      │ (plate OCR)│  │
  │  │ Sensor     │            │  • assign()  │      │ Payment    │  │
  │  │ Barrier    │            │  • release() │      │ Terminal   │  │
  │  └────────────┘            │  • reserve() │      └────────────┘  │
  │                            └──────┬───────┘                       │
  │                                   │                                │
  │  Display Boards                   │                                │
  │  ┌────────────┐            ┌──────▼───────┐                       │
  │  │ "Floor 1:  │◄───────────│  Spot        │                       │
  │  │  23 avail" │            │  Manager     │                       │
  │  │ "Floor 2:  │            │  (Redis +    │                       │
  │  │  45 avail" │            │   Postgres)  │                       │
  │  └────────────┘            └──────┬───────┘                       │
  │                                   │                                │
  │                            ┌──────▼───────┐                       │
  │                            │  Payment     │                       │
  │                            │  Service     │                       │
  │                            │  (Stripe/    │                       │
  │                            │   Square)    │                       │
  │                            └──────────────┘                       │
  └──────────────────────────────────────────────────────────────────┘

DATA STORAGE:
  PostgreSQL: tickets, payments, reservations (transactional!)
  Redis: real-time availability counts, active sessions
  
  Redis keys:
  • garage:1:floor:1:available:regular → 23 (counter!)
  • garage:1:floor:1:available:compact → 10
  • spot:1:F1:S023:status → "OCCUPIED" (per-spot status)
  • active_ticket:ABC123 → {spotId, entryTime, vehiclePlate}
```

---

## 🔒 Concurrency Handling

```
PROBLEM: Two cars arrive at same time, both see "1 spot available"!
  Car A: check → 1 available! → assign!
  Car B: check → 1 available! → assign! ← DOUBLE BOOKING! 💀

SOLUTION 1: PESSIMISTIC LOCKING (Database)
  BEGIN;
  SELECT * FROM spots WHERE floor=1 AND type='regular' AND status='FREE'
  FOR UPDATE SKIP LOCKED  -- Lock the row! Skip already-locked ones!
  LIMIT 1;
  
  UPDATE spots SET status='OCCUPIED', vehicle_id=? WHERE id=?;
  COMMIT;
  
  FOR UPDATE SKIP LOCKED: if someone else locked that row → skip to next!
  No waiting, no deadlock! ✅

SOLUTION 2: ATOMIC OPERATIONS (Redis)
  -- Lua script (atomic in Redis!)
  local spot = redis.call('LPOP', 'available:floor1:regular')
  if spot then
    redis.call('HSET', 'occupied', spot, vehicleId)
    return spot
  end
  return nil
  
  LPOP is atomic → only one car gets each spot! ✅

SOLUTION 3: OPTIMISTIC LOCKING (CAS)
  1. Read spot: {id: S023, status: FREE, version: 5}
  2. UPDATE spots SET status='OCCUPIED', version=6 
     WHERE id='S023' AND version=5;
  3. If affected_rows == 0 → someone else took it! Retry!

WHICH TO USE:
  Low contention (parking garage): Pessimistic lock ← simple, correct!
  High contention (concert tickets): Redis atomic ← fast, no DB pressure!
  Very high scale: Queue-based (first come, first served from a queue)
```

---

## 📈 Scaling to Multiple Garages

```
SINGLE GARAGE: Simple monolith is fine!
  1000 spots, 100 entries/hour → single server handles easily!

MULTI-GARAGE (city-wide parking platform):
  ┌─────────────────────────────────────────────────────────────┐
  │                    CITY PARKING PLATFORM                      │
  │                                                              │
  │  Mobile App                                                  │
  │  "Find parking near me"                                     │
  │       │                                                      │
  │       ▼                                                      │
  │  ┌───────────────┐                                          │
  │  │  API Gateway   │                                          │
  │  └───────┬───────┘                                          │
  │          │                                                   │
  │  ┌───────▼───────┐     ┌────────────────┐                  │
  │  │  Search Svc    │────►│  GeoIndex      │                  │
  │  │  "spots near   │     │  (PostGIS /    │                  │
  │  │   lat/lng"     │     │   Redis GEO)   │                  │
  │  └───────────────┘     └────────────────┘                  │
  │                                                              │
  │  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
  │  │ Garage A   │  │ Garage B   │  │ Garage C   │           │
  │  │ Service    │  │ Service    │  │ Service    │           │
  │  │ (downtown) │  │ (airport)  │  │ (mall)     │           │
  │  └────────────┘  └────────────┘  └────────────┘           │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘

FEATURES AT SCALE:
  • Dynamic pricing (rush hour = higher rates!)
  • Reservation system (pre-book for airport parking!)
  • Navigation to spot (floor + section + spot number!)
  • EV charging integration (assign to charging-capable spot!)
  • License plate recognition (ticketless entry/exit!)
  • Monthly/seasonal passes (subscription model!)
```

---

## 💻 Java Implementation

### Core Domain

```java
public enum VehicleType { MOTORCYCLE, CAR, TRUCK, VAN }
public enum SpotType { COMPACT, REGULAR, LARGE, HANDICAP, EV_CHARGING }
public enum SpotStatus { FREE, OCCUPIED, RESERVED, OUT_OF_SERVICE }

@Entity
public class ParkingSpot {
    @Id private String id; // "F1-S023"
    private int floor;
    private int spotNumber;
    @Enumerated private SpotType type;
    @Enumerated private SpotStatus status;
    private String vehicleLicensePlate;
    @Version private int version; // Optimistic locking!
}

@Entity
public class ParkingTicket {
    @Id private String ticketId;
    private String licensePlate;
    private String spotId;
    private LocalDateTime entryTime;
    private LocalDateTime exitTime;
    private BigDecimal amount;
    private boolean paid;
}
```

### Parking Service (with Concurrency!)

```java
@Service
@Transactional
public class ParkingService {
    
    @Autowired private ParkingSpotRepository spotRepo;
    @Autowired private TicketRepository ticketRepo;
    @Autowired private RedisTemplate<String, String> redis;
    @Autowired private PaymentService paymentService;
    
    /**
     * Assign a spot to incoming vehicle.
     * Uses pessimistic locking to prevent double-assignment!
     */
    public ParkingTicket assignSpot(String licensePlate, VehicleType vehicleType) {
        SpotType requiredType = mapVehicleToSpot(vehicleType);
        
        // Pessimistic lock: find and lock first available spot!
        ParkingSpot spot = spotRepo.findFirstAvailableWithLock(requiredType)
            .orElseThrow(() -> new NoSpotAvailableException(
                "No " + requiredType + " spots available!"));
        
        // Assign the spot
        spot.setStatus(SpotStatus.OCCUPIED);
        spot.setVehicleLicensePlate(licensePlate);
        spotRepo.save(spot);
        
        // Create ticket
        ParkingTicket ticket = new ParkingTicket();
        ticket.setTicketId(generateTicketId());
        ticket.setLicensePlate(licensePlate);
        ticket.setSpotId(spot.getId());
        ticket.setEntryTime(LocalDateTime.now());
        ticketRepo.save(ticket);
        
        // Update availability counter (Redis for real-time display!)
        redis.opsForValue().decrement(
            "available:" + spot.getFloor() + ":" + spot.getType());
        
        return ticket;
    }
    
    /**
     * Process vehicle exit: calculate fee and release spot.
     */
    public PaymentReceipt processExit(String ticketId) {
        ParkingTicket ticket = ticketRepo.findById(ticketId)
            .orElseThrow(() -> new TicketNotFoundException(ticketId));
        
        // Calculate fee
        ticket.setExitTime(LocalDateTime.now());
        BigDecimal fee = calculateFee(ticket);
        ticket.setAmount(fee);
        
        // Process payment
        PaymentReceipt receipt = paymentService.charge(fee, ticket);
        ticket.setPaid(true);
        ticketRepo.save(ticket);
        
        // Release the spot
        ParkingSpot spot = spotRepo.findById(ticket.getSpotId()).orElseThrow();
        spot.setStatus(SpotStatus.FREE);
        spot.setVehicleLicensePlate(null);
        spotRepo.save(spot);
        
        // Update availability counter
        redis.opsForValue().increment(
            "available:" + spot.getFloor() + ":" + spot.getType());
        
        return receipt;
    }
    
    /**
     * Fee calculation: tiered pricing!
     */
    private BigDecimal calculateFee(ParkingTicket ticket) {
        long minutes = Duration.between(
            ticket.getEntryTime(), ticket.getExitTime()).toMinutes();
        
        if (minutes <= 15) return BigDecimal.ZERO;          // Free! (grace)
        if (minutes <= 60) return new BigDecimal("3.00");   // First hour
        
        long hours = (minutes + 59) / 60; // Round up!
        BigDecimal hourlyRate = new BigDecimal("2.50");
        BigDecimal maxDaily = new BigDecimal("25.00");
        
        BigDecimal total = new BigDecimal("3.00")
            .add(hourlyRate.multiply(BigDecimal.valueOf(hours - 1)));
        
        return total.min(maxDaily); // Cap at daily maximum!
    }
    
    private SpotType mapVehicleToSpot(VehicleType v) {
        return switch (v) {
            case MOTORCYCLE -> SpotType.COMPACT;
            case CAR -> SpotType.REGULAR;
            case TRUCK, VAN -> SpotType.LARGE;
        };
    }
}
```

### Repository with Pessimistic Lock

```java
public interface ParkingSpotRepository extends JpaRepository<ParkingSpot, String> {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("""
        SELECT s FROM ParkingSpot s 
        WHERE s.type = :type AND s.status = 'FREE'
        ORDER BY s.floor ASC, s.spotNumber ASC
        """)
    Optional<ParkingSpot> findFirstAvailableWithLock(@Param("type") SpotType type);
    
    @Query("SELECT COUNT(s) FROM ParkingSpot s WHERE s.floor = :floor AND s.status = 'FREE'")
    int countAvailableByFloor(@Param("floor") int floor);
}
```

---

## ❓ Interview Q&A

**Q1: How do you handle the race condition where two cars try to get the same spot?**
> Pessimistic locking with `SELECT ... FOR UPDATE SKIP LOCKED`. The first transaction locks the row, second transaction skips that row and gets the next available spot. No waiting, no deadlock, no double-booking. For higher throughput: use Redis with atomic LPOP from an "available spots" queue — only one consumer can pop each item. For reservations: use optimistic locking with version field (retry on conflict).

**Q2: How would you scale this to a city-wide platform with 1000 garages?**
> Each garage runs independently (own database, own Redis) — no cross-garage transactions needed. A central platform provides: (1) Search service with geo-index (PostGIS/Redis GEO) to find nearby garages, (2) Real-time availability aggregation (garages push counts every 10s), (3) Unified reservation system, (4) Central billing/subscriptions. Communication: async events (garage publishes availability changes to Kafka) → search service consumes and updates index.

**Q3: How would you implement dynamic pricing?**
> Factors: (1) Current occupancy (>80% → surge pricing), (2) Time of day (rush hour multiplier), (3) Day of week (weekend vs weekday), (4) Events nearby (concert at venue = 3x), (5) Historical demand patterns. Implementation: pricing engine evaluates rules, updates rate card every 15 minutes. Display at entry gate shows current rate. For reservations: lock price at booking time (honesty!). This is similar to Uber surge pricing but simpler (fewer variables).

**Q4: What happens if the system crashes mid-transaction (car entered but ticket not issued)?**
> Database ACID guarantees: if crash happens during transaction, it's rolled back — spot freed, no ticket. But what about hardware (barrier raised, car entered, system crashed before writing to DB)? Solutions: (1) Write-ahead: create ticket FIRST (status: PENDING), then raise barrier, then confirm ticket. If crash: pending tickets are cleaned up, spot re-freed, (2) License plate camera: on restart, scan parked cars, reconcile with DB (find cars without tickets → create retroactive tickets).

---

## 🎮 Mini Challenge

Extend this system to support:
1. Monthly subscription passes (unlimited parking!)
2. EV charging spots (charge fee = parking fee + kWh used)
3. Valet parking (staff moves car to remote lot, retrieves on exit!)

*How does each feature change the data model and flow?*

---

## 🔗 Related Topics
- [ACID Transactions](../Database/ACID.md) — Why we need locking
- [Rate Limiting](../BuildingBlocks/RateLimiting.md) — Controlling entry rate
- [Design Ticket Booking](DesignTicketBooking.md) — Similar concurrency challenges
- [Distributed Locking](../Microservices/DistributedLocking.md) — For multi-server setup

---

*"The parking garage problem seems trivial until you add concurrency. Then it becomes a masterclass in: atomic operations, race conditions, optimistic vs pessimistic locking, and eventually consistent counters. The best system design answers take simple problems and reveal their hidden complexity." — Interview Coach* 🚗
