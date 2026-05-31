# ✈️ Design a Flight Booking System

> *"Flight booking is the ultimate test of distributed systems under pressure: thousands of users competing for the same seat, prices changing every millisecond, inventory spread across hundreds of airlines, and a transaction that must coordinate flights + hotels + payments atomically. This system combines the reservation pattern (hold before confirming), distributed transactions (Saga), dynamic pricing, and the critical concept of 'eventual consistency with compensation' — all under the constraint that double-booking a seat is UNACCEPTABLE."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [ACID](../Database/ACID.md), [Distributed Locking](../Microservices/DistributedLocking.md), [Message Queues](../BuildingBlocks/MessageQueues.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [High-Level Architecture](#-high-level-architecture)
3. [Search & Availability](#-search--availability)
4. [Seat Reservation Pattern](#-seat-reservation-pattern)
5. [Booking & Payment (Saga)](#-booking--payment-saga)
6. [Dynamic Pricing](#-dynamic-pricing)
7. [Java Implementation](#-java-implementation)
8. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Search flights by origin, destination, dates, passengers
  • Filter by airline, stops, time, price
  • View seat map and select seats
  • HOLD seats during booking (temporary reservation!)
  • Complete booking with payment
  • Support multi-leg journeys (connecting flights!)
  • Cancellation and refund
  • E-ticket generation and email
  • Price alerts (notify when price drops!)

NON-FUNCTIONAL:
  • 10M searches per day, 100K bookings per day
  • Search latency: < 2 seconds (aggregating 200+ airlines!)
  • Seat hold: 15 minutes (prevent phantom inventory!)
  • Zero double-bookings! (CRITICAL!)
  • Payment processing: < 5 seconds
  • Availability: 99.99% (travel is time-sensitive!)
  • Handle 10× spikes during sales/holidays!

SCALE:
  • 500+ airlines, 100K+ routes
  • 5M+ flights per month
  • Concurrent searches: 50K/second during peak!
  • Inventory: 50M+ seats managed at any time
```

---

## 🏗️ High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                      FLIGHT BOOKING SYSTEM                                   │
│                                                                              │
│  ┌────────────┐     ┌─────────────┐     ┌────────────────────────────┐    │
│  │  Web/Mobile│────►│ API Gateway │────►│  Load Balancer (L7)        │    │
│  │   Client   │     └─────────────┘     └────────────┬───────────────┘    │
│  └────────────┘                                       │                     │
│                                    ┌──────────────────┼────────────────┐   │
│                                    ▼                  ▼                ▼    │
│                          ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │
│                          │Search Service│  │Booking Service│  │ User Svc │ │
│                          │(read-heavy!) │  │(write-heavy!) │  │          │ │
│                          └──────┬───────┘  └──────┬───────┘  └──────────┘ │
│                                 │                  │                        │
│               ┌─────────────────┤                  │                        │
│               ▼                 ▼                  ▼                        │
│  ┌───────────────────┐  ┌────────────┐  ┌──────────────────┐             │
│  │  Flight Inventory │  │   Cache    │  │  Reservation     │             │
│  │  (aggregator!)    │  │  (Redis)   │  │  Engine          │             │
│  │  • GDS systems    │  │  search    │  │  • Seat locking  │             │
│  │  • Airline APIs   │  │  results   │  │  • TTL holds     │             │
│  │  • Charter feeds  │  │            │  │  • Payment coord │             │
│  └───────────────────┘  └────────────┘  └──────────────────┘             │
│                                                    │                       │
│                                    ┌───────────────┼───────────────┐      │
│                                    ▼               ▼               ▼       │
│                          ┌──────────────┐  ┌────────────┐  ┌──────────┐  │
│                          │   Payment    │  │  Pricing   │  │  Noti-   │  │
│                          │   Service    │  │  Engine    │  │  fication │  │
│                          │   (Stripe)   │  │  (dynamic!)│  │  Service │  │
│                          └──────────────┘  └────────────┘  └──────────┘  │
│                                                                            │
│  DATA STORES:                                                              │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐           │
│  │Postgres│  │  Redis │  │ Elastic│  │ Kafka  │  │  S3    │           │
│  │(booking│  │(seats, │  │(flight │  │(events)│  │(tickets│           │
│  │ data)  │  │ cache) │  │ search)│  │        │  │  PDF)  │           │
│  └────────┘  └────────┘  └────────┘  └────────┘  └────────┘           │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔍 Search & Availability

```
SEARCH CHALLENGE:
  "Show me flights from NYC to London on Dec 15"
  → Must query 50+ airlines in parallel!
  → Must consider connecting flights (combinatorial explosion!)
  → Must return in < 2 seconds!

SEARCH ARCHITECTURE:
  ┌────────────────────────────────────────────────────────────────┐
  │  Request: NYC → LON, Dec 15, 2 passengers                      │
  │                                                                 │
  │  1. CHECK CACHE (Redis): same search in last 5 min?            │
  │     HIT: return cached results (80% hit rate!)                  │
  │     Prices may be slightly stale (acceptable for search!)       │
  │                                                                 │
  │  2. DIRECT FLIGHTS:                                             │
  │     Query route index: NYC → LON                                │
  │     Airlines on this route: BA, AA, VS, UA, DL                  │
  │     Parallel API calls to each airline's inventory!             │
  │     Timeout: 3 seconds per airline (don't wait for slow ones!) │
  │                                                                 │
  │  3. CONNECTING FLIGHTS (if requested):                          │
  │     Find valid connections: NYC → ??? → LON                     │
  │     Constraints:                                                │
  │     • Connection time: 1h - 8h                                  │
  │     • Same terminal preferred                                   │
  │     • Both legs available!                                      │
  │     This is a GRAPH SEARCH! (BFS with constraints!)            │
  │                                                                 │
  │  4. AGGREGATE & RANK:                                           │
  │     Merge results from all airlines                             │
  │     Sort by: price (default) / duration / departure time        │
  │     Apply filters: stops, time range, airline                   │
  │                                                                 │
  │  5. CACHE & RETURN:                                             │
  │     Cache results for 5 min (TTL!)                              │
  │     Return with "prices may change" disclaimer!                 │
  └────────────────────────────────────────────────────────────────┘

GDS (Global Distribution System):
  Amadeus, Sabre, Travelport — aggregators of airline inventory!
  One API call to GDS = availability across 400+ airlines!
  BUT: expensive per query! Cache aggressively!
  
  Alternative: direct airline connections (NDC standard!)
  Lower cost, more control, but more complex integration!
```

---

## 🔒 Seat Reservation Pattern

```
THE RESERVATION PROBLEM:
  100 users looking at the same seat → only 1 can book it!
  
  WRONG approach: "first to click 'book' wins"
  → User fills form for 10 minutes → seat taken! TERRIBLE UX! 😡
  
  RIGHT approach: TEMPORARY HOLD (TTL-based reservation!)

HOLD MECHANISM:
  ┌────────────────────────────────────────────────────────────────┐
  │  STEP 1: User selects seat 14A                                  │
  │  → Create HOLD record:                                          │
  │    {seat: "14A", flight: "BA123", user: "U42",                 │
  │     hold_until: now() + 15 min, status: "HELD"}                │
  │  → Redis: SETEX "hold:BA123:14A" 900 "user-42"                │
  │    (900 seconds = 15 minutes! Auto-expires!)                   │
  │                                                                 │
  │  STEP 2: Other users see seat 14A as "unavailable"             │
  │  → Before showing seat map: check holds in Redis!               │
  │  → Seat 14A grayed out: "temporarily held"                     │
  │                                                                 │
  │  STEP 3a: User completes booking within 15 min                 │
  │  → Hold converted to CONFIRMED booking!                         │
  │  → Permanent record in PostgreSQL!                              │
  │                                                                 │
  │  STEP 3b: User abandons (timeout!)                             │
  │  → Redis key expires automatically!                             │
  │  → Seat becomes available again!                                │
  │  → No cleanup needed! (TTL does the work!)                     │
  └────────────────────────────────────────────────────────────────┘

PREVENTING DOUBLE-BOOKING (CRITICAL!):
  Two users try to hold same seat simultaneously!
  
  Redis solution: SETNX (Set if Not eXists!) → atomic!
  
  SET "hold:BA123:14A" "user-42" NX EX 900
  ← Returns OK if set (got the hold!)
  ← Returns nil if already exists (someone else has it!)
  
  ONLY ONE user gets the hold! Atomic, no race condition! ✅
  
  After payment confirmed: write to PostgreSQL with UNIQUE constraint!
  (seat_number, flight_id) = UNIQUE → database-level safety net!
```

---

## 💰 Booking & Payment (Saga)

```
MULTI-STEP BOOKING (Saga Pattern!):

  Book round-trip NYC → LON → NYC with hotel:
  
  ┌─────────────────────────────────────────────────────────────────┐
  │  SAGA STEPS (choreography!):                                     │
  │                                                                  │
  │  Step 1: HOLD outbound flight seat                               │
  │    Success → Step 2                                              │
  │    Fail → ABORT (no compensation needed!)                       │
  │                                                                  │
  │  Step 2: HOLD return flight seat                                 │
  │    Success → Step 3                                              │
  │    Fail → COMPENSATE: release outbound hold!                    │
  │                                                                  │
  │  Step 3: RESERVE hotel room                                      │
  │    Success → Step 4                                              │
  │    Fail → COMPENSATE: release both flight holds!                │
  │                                                                  │
  │  Step 4: CHARGE payment                                          │
  │    Success → Step 5                                              │
  │    Fail → COMPENSATE: release flights + hotel!                  │
  │                                                                  │
  │  Step 5: CONFIRM all reservations                                │
  │    Convert holds → confirmed bookings!                           │
  │    Generate e-tickets!                                           │
  │    Send confirmation email!                                      │
  │                                                                  │
  │  ANY step fails → compensating transactions undo previous steps!│
  └─────────────────────────────────────────────────────────────────┘

WHY NOT 2PC (Two-Phase Commit)?
  Airlines have INDEPENDENT databases! Can't do distributed 2PC!
  Saga = eventual consistency with compensations!
  
  Compensation examples:
  • Hold expired? → No compensation needed (TTL handles it!)
  • Payment charged but flight cancelled? → Refund payment!
  • Flight confirmed but hotel fails? → Cancel flight (may have fee!)
```

---

## 💻 Java Implementation

### Seat Hold Service

```java
@Service
public class SeatHoldService {
    
    @Autowired private StringRedisTemplate redis;
    @Autowired private SeatRepository seatRepo;
    
    private static final int HOLD_DURATION_SECONDS = 900; // 15 minutes!
    
    /**
     * Attempt to hold a seat for a user.
     * Uses Redis SETNX for atomic, race-condition-free locking!
     */
    public HoldResult holdSeat(String flightId, String seatNumber, 
                                String userId) {
        String holdKey = buildHoldKey(flightId, seatNumber);
        
        // Atomic: set only if not exists! (prevents double-hold!)
        Boolean acquired = redis.opsForValue().setIfAbsent(
            holdKey, userId, Duration.ofSeconds(HOLD_DURATION_SECONDS));
        
        if (Boolean.TRUE.equals(acquired)) {
            // Got the hold! 🎉
            return HoldResult.success(
                flightId, seatNumber, userId,
                Instant.now().plusSeconds(HOLD_DURATION_SECONDS));
        }
        
        // Someone else holds this seat!
        String currentHolder = redis.opsForValue().get(holdKey);
        if (userId.equals(currentHolder)) {
            // User already holds it (idempotent!)
            Long ttl = redis.getExpire(holdKey, TimeUnit.SECONDS);
            return HoldResult.alreadyHeld(
                Instant.now().plusSeconds(ttl != null ? ttl : 0));
        }
        
        return HoldResult.unavailable("Seat held by another user");
    }
    
    /**
     * Release a hold (user cancelled or changing seat).
     */
    public void releaseHold(String flightId, String seatNumber, String userId) {
        String holdKey = buildHoldKey(flightId, seatNumber);
        
        // Only release if THIS user holds it! (Lua script for atomicity!)
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        redis.execute(new DefaultRedisScript<>(script, Long.class),
            List.of(holdKey), userId);
    }
    
    /**
     * Confirm hold → permanent booking (after payment success!).
     */
    @Transactional
    public Booking confirmHold(String flightId, String seatNumber, 
                               String userId, String paymentId) {
        String holdKey = buildHoldKey(flightId, seatNumber);
        
        // Verify hold still exists and belongs to user!
        String holder = redis.opsForValue().get(holdKey);
        if (!userId.equals(holder)) {
            throw new HoldExpiredException("Hold expired! Seat may be taken.");
        }
        
        // Write confirmed booking to PostgreSQL (UNIQUE constraint!)
        Booking booking = Booking.builder()
            .id(UUID.randomUUID().toString())
            .flightId(flightId)
            .seatNumber(seatNumber)
            .userId(userId)
            .paymentId(paymentId)
            .status(BookingStatus.CONFIRMED)
            .bookedAt(Instant.now())
            .build();
        
        try {
            seatRepo.confirmSeat(booking); // UNIQUE(flight_id, seat_number)!
        } catch (DuplicateKeyException e) {
            throw new SeatAlreadyBookedException("Seat already booked!");
        }
        
        // Remove hold (no longer needed — DB has truth!)
        redis.delete(holdKey);
        
        return booking;
    }
    
    private String buildHoldKey(String flightId, String seatNumber) {
        return "hold:" + flightId + ":" + seatNumber;
    }
}
```

### Booking Saga Orchestrator

```java
@Service
public class BookingSagaOrchestrator {
    
    @Autowired private SeatHoldService seatHoldService;
    @Autowired private PaymentService paymentService;
    @Autowired private NotificationService notificationService;
    
    /**
     * Orchestrate multi-step booking with compensation on failure!
     */
    public BookingResult executeBookingSaga(BookingRequest request) {
        List<CompensationAction> compensations = new ArrayList<>();
        
        try {
            // Step 1: Hold outbound seat
            HoldResult outbound = seatHoldService.holdSeat(
                request.getOutboundFlightId(),
                request.getOutboundSeat(),
                request.getUserId());
            
            if (!outbound.isSuccess()) {
                return BookingResult.failed("Outbound seat unavailable");
            }
            compensations.add(() -> seatHoldService.releaseHold(
                request.getOutboundFlightId(),
                request.getOutboundSeat(),
                request.getUserId()));
            
            // Step 2: Hold return seat (if round-trip!)
            if (request.isRoundTrip()) {
                HoldResult returnFlight = seatHoldService.holdSeat(
                    request.getReturnFlightId(),
                    request.getReturnSeat(),
                    request.getUserId());
                
                if (!returnFlight.isSuccess()) {
                    compensate(compensations);
                    return BookingResult.failed("Return seat unavailable");
                }
                compensations.add(() -> seatHoldService.releaseHold(
                    request.getReturnFlightId(),
                    request.getReturnSeat(),
                    request.getUserId()));
            }
            
            // Step 3: Charge payment
            PaymentResult payment = paymentService.charge(
                request.getUserId(),
                request.getTotalPrice(),
                request.getPaymentMethod());
            
            if (!payment.isSuccess()) {
                compensate(compensations);
                return BookingResult.failed("Payment failed: " + payment.getError());
            }
            compensations.add(() -> paymentService.refund(payment.getTransactionId()));
            
            // Step 4: Confirm all holds → permanent bookings!
            Booking outboundBooking = seatHoldService.confirmHold(
                request.getOutboundFlightId(),
                request.getOutboundSeat(),
                request.getUserId(),
                payment.getTransactionId());
            
            // Step 5: Send confirmation!
            notificationService.sendBookingConfirmation(outboundBooking);
            
            return BookingResult.success(outboundBooking);
            
        } catch (Exception e) {
            // Any unexpected failure → compensate everything!
            compensate(compensations);
            throw new BookingException("Booking failed, all reservations released", e);
        }
    }
    
    private void compensate(List<CompensationAction> compensations) {
        // Execute compensations in REVERSE order!
        Collections.reverse(compensations);
        for (CompensationAction action : compensations) {
            try {
                action.execute();
            } catch (Exception e) {
                // Log and continue — best effort!
                log.error("Compensation failed!", e);
                // Alert for manual intervention!
            }
        }
    }
}
```

---

## ❓ Interview Q&A

**Q1: How do you prevent double-booking without distributed locks on every seat?**
> Two layers of protection: (1) Redis SETNX for the "hold" phase — atomic, single-threaded, impossible to double-hold. This handles 99.9% of cases. (2) PostgreSQL UNIQUE constraint on (flight_id, seat_number) as the final safety net — even if Redis fails, DB rejects duplicates! The hold is a "soft lock" (TTL-based, auto-releases), the DB constraint is the "hard lock" (permanent). This avoids heavyweight distributed locks while guaranteeing no double-bookings.

**Q2: Why use a Saga instead of a distributed transaction (2PC)?**
> Airlines, hotels, and payment processors are INDEPENDENT systems you don't control! You can't wrap Stripe + American Airlines + Marriott in a 2PC — they don't support it! Saga works with autonomous services: each step is a local transaction with a compensating action. If Step 3 fails: undo Step 2, undo Step 1 (compensation). Trade-off: temporary inconsistency (you might hold a flight seat for 2 seconds after payment fails before compensation runs). But for booking: this is acceptable! The alternative (2PC) would: hold locks across services (terrible latency!), not work with external APIs, and create availability coupling.

**Q3: How do you handle price changes between search and booking?**
> The search result is a SNAPSHOT (cached price at time T). By the time user clicks "book" (time T+10min), price may differ. Solutions: (1) Display "price may change" disclaimer during search, (2) At booking time: re-query real-time price from airline, (3) If price INCREASED: show user the new price, ask to confirm! (4) If price DECREASED: pleasant surprise — book at lower price! (5) Use a "price guarantee" window: cache result + guarantee that price for 15 minutes (absorb the difference as business cost for better UX). Airlines implement this via fare locks (hold a price for $5-20 fee for 24-72 hours!).

**Q4: How would you design the search to return in < 2 seconds across 200+ airlines?**
> (1) Don't query all 200 airlines in real-time! Use pre-cached availability from GDS (updated every 5-30 minutes). (2) Only query airlines that fly the requested route (route index: NYC→LON = 8 airlines, not 200!). (3) Parallel fan-out: query all relevant airlines simultaneously, use fastest responses. (4) Timeout: if an airline doesn't respond in 2s, exclude them from results (show "more results loading" async!). (5) Aggressive caching: same route+date search = serve from Redis cache (5-minute TTL). 80% of searches are repeats! (6) Pre-compute popular routes during off-peak (NYC→LON, LAX→NYC are always searched!).

---

## 🔗 Related Topics
- [Distributed Locking](../Microservices/DistributedLocking.md) — Seat hold mechanism
- [Design Ticket Booking](./DesignTicketBooking.md) — Similar reservation pattern
- [ACID Transactions](../Database/ACID.md) — Booking consistency
- [Circuit Breaker](../BuildingBlocks/CircuitBreaker.md) — Airline API resilience

---

*"A flight booking system is the perfect interview question because it touches EVERY distributed systems concept: consistency (no double-booking!), availability (99.99%!), partition tolerance (airline APIs go down!), saga pattern (multi-step booking!), caching (search performance!), and TTL-based resource management (seat holds!). If you can design this well, you can design anything."* ✈️
