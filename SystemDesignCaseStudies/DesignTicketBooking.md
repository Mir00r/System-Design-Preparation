# 🎫 Design a Ticket Booking System (BookMyShow / Ticketmaster)

> *"When Taylor Swift's Eras Tour tickets went on sale, Ticketmaster received 14 MILLION requests in ONE HOUR. 3.5 billion system requests in a single day — 4x their previous record. The result? Crashes, queues, and fury. The core challenge: 50,000 seats, 14 million people wanting them simultaneously. How do you prevent overselling seat A1 to 100 people while maintaining sub-second response times?"*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [Distributed Locking](../../Microservices/DistributedLocking.md), [Rate Limiting](../../BuildingBlocks/RateLimiting.md), [Caching](../../BuildingBlocks/Caching.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [The Core Challenge: Seat Concurrency](#-the-core-challenge-seat-concurrency)
3. [High-Level Architecture](#-high-level-architecture)
4. [Seat Inventory & Locking](#-seat-inventory--locking)
5. [Booking Flow (Happy Path)](#-booking-flow-happy-path)
6. [Handling Flash Sales & Spikes](#-handling-flash-sales--spikes)
7. [Payment Integration](#-payment-integration)
8. [Fairness & Queue System](#-fairness--queue-system)
9. [Java Implementation](#-java-implementation)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
✅ Browse events (search, filter by date/location/genre)
✅ View venue seat map (real-time availability)
✅ Select seats (specific seat or "best available")
✅ Temporary hold on selected seats (5-10 min timer)
✅ Complete booking (payment + confirmation)
✅ Cancel/refund tickets
✅ QR code generation for entry
✅ Waitlist for sold-out events
✅ Dynamic pricing (demand-based)
```

### Non-Functional Requirements
```
✅ NO double-booking (absolutely CRITICAL!)
✅ Handle extreme spikes (10M+ concurrent for popular events)
✅ Low latency (seat selection < 2 seconds)
✅ High availability (99.99% during on-sale)
✅ Fair ordering (first-come-first-served)
✅ Scalable (different events = different load)
✅ Payment reliability (exactly-once charge!)
```

### Scale Estimation
```
Normal day: 100K bookings/day
Popular event on-sale: 5M requests in 5 minutes = 16,600 RPS!
Taylor Swift level: 14M requests/hour = 3,900 RPS sustained

Seats per event: 50,000 (stadium)
Concurrent seat selectors: 500,000+ (10x seats available!)
→ 90% will get "sold out" → must handle gracefully!
```

---

## 🔒 The Core Challenge: Seat Concurrency

```
THE DOUBLE-BOOKING PROBLEM:

  Time T1: User A selects Seat 5A     → "Available!" ✅
  Time T1: User B selects Seat 5A     → "Available!" ✅ (RACE!)
  Time T2: User A clicks "Book"        → Writes to DB ✅
  Time T2: User B clicks "Book"        → Also writes to DB! 💀
  
  Result: Seat 5A sold to BOTH users! Two people show up!
  
  THIS IS NOT ACCEPTABLE. Not for concerts. Not for flights. Not ever.

SOLUTIONS SPECTRUM:
  ┌────────────────────────────────────────────────────────────────┐
  │  Approach          │  Pros            │  Cons                  │
  ├────────────────────────────────────────────────────────────────┤
  │  Pessimistic Lock  │  Guarantees no   │  Blocks others,        │
  │  (DB row lock)     │  conflicts       │  poor concurrency      │
  ├────────────────────────────────────────────────────────────────┤
  │  Optimistic Lock   │  High concurrency│  Retry storms on       │
  │  (version check)   │  no blocking     │  hot seats             │
  ├────────────────────────────────────────────────────────────────┤
  │  Temporary Hold    │  User-friendly   │  Complex timer mgmt    │
  │  (Redis TTL)       │  time to pay     │  need cleanup          │
  ├────────────────────────────────────────────────────────────────┤
  │  Queue-based       │  Fair ordering   │  Higher latency        │
  │  (serialize all)   │  no conflicts    │  single bottleneck     │
  └────────────────────────────────────────────────────────────────┘
  
  BEST: Temporary Hold + Optimistic Locking (combination!)
```

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      USERS (5M+ concurrent!)                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │   CDN + Load Balancer    │  ← Static assets cached!
              └────────────┬────────────┘
                           │
         ┌─────────────────┼─────────────────────────┐
         ▼                 ▼                         ▼
┌──────────────┐  ┌─────────────────┐      ┌──────────────────┐
│  Event       │  │  Booking        │      │  Search          │
│  Service     │  │  Service        │      │  Service         │
│  (CRUD)      │  │  (CRITICAL!)    │      │  (Elasticsearch) │
└──────────────┘  └────────┬────────┘      └──────────────────┘
                           │
              ┌────────────┼────────────────────┐
              ▼            ▼                    ▼
     ┌──────────────┐ ┌──────────┐    ┌──────────────────┐
     │  Seat        │ │  Redis   │    │  Payment         │
     │  Inventory   │ │  (Holds) │    │  Service         │
     │  (PostgreSQL)│ │  TTL=10m │    │  (Stripe/PayPal) │
     └──────────────┘ └──────────┘    └──────────────────┘
              │                                │
              ▼                                ▼
     ┌──────────────┐                 ┌──────────────────┐
     │  Queue        │                 │  Notification    │
     │  Service     │                 │  Service         │
     │  (Fairness)  │                 │  (Email/SMS)     │
     └──────────────┘                 └──────────────────┘
```

---

## 🪑 Seat Inventory & Locking

```
DATABASE SCHEMA:

  events               seats                    bookings
  ┌──────────┐         ┌───────────────────┐    ┌──────────────────┐
  │ id       │         │ id                │    │ id               │
  │ name     │────────►│ event_id          │    │ seat_id          │
  │ venue    │         │ section           │    │ user_id          │
  │ datetime │         │ row               │    │ status           │
  │ capacity │         │ number            │    │ payment_id       │
  └──────────┘         │ status            │◄───│ amount           │
                       │ price_tier        │    │ booked_at        │
                       │ held_by           │    │ qr_code          │
                       │ held_until        │    └──────────────────┘
                       │ version (optimistic)│
                       └───────────────────┘

  Seat statuses: AVAILABLE → HELD → BOOKED (or HELD → RELEASED)

HOLD MECHANISM (Redis + DB):

  Step 1: User selects seat 5A
  Step 2: Atomic operation in Redis:
    SETNX "hold:event123:seat5A" "user456" EX 600  (10-min TTL)
    
    If SETNX returns 1 → Hold acquired! User has 10 minutes to pay.
    If SETNX returns 0 → Already held by someone else! Show "unavailable"
    
  Step 3: Also update DB (for persistence):
    UPDATE seats SET status='HELD', held_by='user456', 
    held_until=NOW()+INTERVAL '10 minutes', version=version+1
    WHERE id='seat5A' AND status='AVAILABLE' AND version=current_version;
    
  Step 4: If user doesn't pay within 10 minutes:
    Redis key expires automatically → seat available again
    Background job resets DB: SET status='AVAILABLE', held_by=NULL
    
  This is FAST (Redis) + SAFE (DB backup) + USER-FRIENDLY (10 min to pay!)
```

---

## ✅ Booking Flow (Happy Path)

```
COMPLETE BOOKING FLOW:

  USER                  SYSTEM                         PAYMENT
   │                      │                              │
   │ 1. "Show me Sec A"   │                              │
   │─────────────────────►│                              │
   │ Available seats      │                              │
   │◄─────────────────────│                              │
   │                      │                              │
   │ 2. "Hold seat A5"    │                              │
   │─────────────────────►│                              │
   │                      │─── Redis SETNX ──►           │
   │                      │◄── OK (got hold!) ───        │
   │ "Held! 10 min timer" │                              │
   │◄─────────────────────│                              │
   │                      │                              │
   │ 3. "Pay $150"        │                              │
   │─────────────────────►│                              │
   │                      │ 3a. Verify hold still valid  │
   │                      │ 3b. Create payment intent    │
   │                      │─────────────────────────────►│
   │                      │     Payment confirmed!       │
   │                      │◄─────────────────────────────│
   │                      │ 3c. Update seat → BOOKED     │
   │                      │ 3d. Generate QR code         │
   │                      │ 3e. Send confirmation        │
   │ "Booking confirmed!" │                              │
   │◄─────────────────────│                              │
   │ 📧 Email + QR code   │                              │
   │◄─────────────────────│                              │

IMPORTANT: Payment and booking must be ATOMIC!
  If payment succeeds but DB update fails → REFUND!
  If DB update succeeds but payment fails → RELEASE SEAT!
  
  Use: Saga pattern or 2-phase approach
    1. Create payment in PENDING state
    2. Update seat to BOOKED
    3. Confirm/capture payment
    4. If any step fails → compensate previous steps!
```

---

## 🚀 Handling Flash Sales & Spikes

```
THE THUNDERING HERD PROBLEM:
  Event: Taylor Swift concert goes on sale at 10:00 AM
  10:00:00 → 5 million users hit "Buy" simultaneously!
  
  Without protection:
  • Database: 5M SELECT queries in 1 second → 💀 DEAD
  • Application servers: overwhelmed → timeout errors
  • Users: rage-clicking makes it 10x worse!

DEFENSE IN DEPTH:

  Layer 1: CDN + STATIC CACHING
    Seat map image → CDN (don't hit DB for each view!)
    Only "check availability" goes to backend
    Reduces 80% of read traffic!
    
  Layer 2: VIRTUAL WAITING ROOM
    At 9:55 AM: users arrive, placed in random queue
    At 10:00 AM: release first 10,000 users to booking page
    Every 30 seconds: release next batch
    
    Why random? Prevents advantage from "fast clicking"
    Why batched? Controls load on booking service
    
    ┌────────────────────────────────────────────┐
    │  WAITING ROOM                               │
    │  5,000,000 users queued                    │
    │  Your position: 142,503                    │
    │  Estimated wait: 12 minutes                │
    │  ████████░░░░░░░░░░░░ 35% through          │
    └────────────────────────────────────────────┘
    
  Layer 3: RATE LIMITING
    Per-user: max 10 requests/second
    Per-IP: max 50 requests/second (shared office)
    Global: max 50,000 booking attempts/second
    
  Layer 4: INVENTORY IN REDIS (not DB!)
    Pre-load all seat availability into Redis at sale start
    Check availability from Redis (microseconds!)
    Only hit PostgreSQL for the actual HOLD/BOOK operation
    
  Layer 5: OVERFLOW QUEUE
    If booking service overloaded → don't reject!
    Place request in queue → process in order
    User sees: "Processing your request..."
```

---

## 💳 Payment Integration

```
PAYMENT FLOW (Exactly-Once Booking):

  Challenge: Network can fail between payment charge and booking!
  
  SAGA PATTERN:
  
  Step 1: Reserve Seat (reversible!)
    UPDATE seats SET status='CONFIRMING' WHERE id=? AND status='HELD';
    
  Step 2: Charge Payment  
    Stripe: create PaymentIntent with idempotency_key = booking_id
    If charge fails → Release seat back to AVAILABLE
    
  Step 3: Confirm Booking
    UPDATE seats SET status='BOOKED';
    INSERT INTO bookings (...);
    
  Step 4: Send Confirmation
    Email + SMS + Push notification with QR code
    
  FAILURE SCENARIOS:
  ┌────────────────────────────────────────────────────────────┐
  │  Failure Point     │  Action                               │
  ├────────────────────────────────────────────────────────────┤
  │  After Step 1      │  Release seat (was just CONFIRMING)   │
  │  After Step 2      │  Check payment status, if charged     │
  │  (network timeout) │  → retry Step 3. If not → release    │
  │  After Step 3      │  Payment + booking done, retry        │
  │  (notification     │  notification only (idempotent)       │
  │   fails)           │                                       │
  └────────────────────────────────────────────────────────────┘
  
  KEY INSIGHT: Use idempotency keys everywhere!
  Same booking_id → same payment intent → no double-charge!
```

---

## 🎯 Fairness & Queue System

```
VIRTUAL WAITING ROOM DESIGN:

  When: Event on-sale is announced
  How: Users who arrive before on-sale time get randomized position
       Users who arrive after get FIFO position

  ┌──────────────────────────────────────────────────────────┐
  │  QUEUE MECHANISM                                         │
  │                                                          │
  │  1. User arrives at 9:55 AM (before 10 AM on-sale)     │
  │     → Assigned random token: position = random(1, N)    │
  │     → Stored in Redis sorted set: ZADD queue score=rand │
  │                                                          │
  │  2. User arrives at 10:02 AM (after on-sale)           │
  │     → Assigned incremental token: position = MAX + 1    │
  │     → FIFO from this point on                          │
  │                                                          │
  │  3. Every 30 seconds: pop first 5000 from queue        │
  │     → Give them access token (JWT, 10-min validity)     │
  │     → They can now access booking page!                │
  │                                                          │
  │  4. User completes booking OR token expires            │
  │     → Slot freed → next batch released!                │
  └──────────────────────────────────────────────────────────┘

  WHY RANDOMIZE pre-arrival users?
  • Prevents advantage from "I arrived 2 hours early!"
  • Prevents bot farms camping the page
  • Makes it fair: everyone who showed up before sale has equal chance!

BOT PROTECTION:
  • CAPTCHA before entering queue (stops 80% of bots)
  • Device fingerprinting (same browser = same queue slot)
  • Rate limiting on queue joins (1 per device per event)
  • Behavioral analysis (no human clicks 100x/second!)
  • Payment verification (real credit card = real human, usually)
```

---

## 💻 Java Implementation

### Seat Hold Service

```java
@Service
public class SeatHoldService {
    
    @Autowired private StringRedisTemplate redis;
    @Autowired private SeatRepository seatRepository;
    
    private static final Duration HOLD_DURATION = Duration.ofMinutes(10);
    
    /**
     * Attempt to hold a seat for a user. 
     * Uses Redis for speed + DB for persistence.
     */
    @Transactional
    public HoldResult holdSeat(String eventId, String seatId, String userId) {
        String holdKey = "hold:" + eventId + ":" + seatId;
        
        // Step 1: Atomic Redis SETNX (prevents double-hold!)
        Boolean acquired = redis.opsForValue()
            .setIfAbsent(holdKey, userId, HOLD_DURATION);
        
        if (!Boolean.TRUE.equals(acquired)) {
            // Someone else holds this seat!
            String holder = redis.opsForValue().get(holdKey);
            if (userId.equals(holder)) {
                return HoldResult.alreadyHeld(); // Same user re-clicking
            }
            return HoldResult.unavailable();
        }
        
        // Step 2: Update database (with optimistic locking!)
        int updated = seatRepository.holdSeat(
            seatId, userId, 
            Instant.now().plus(HOLD_DURATION),
            SeatStatus.AVAILABLE); // Only if currently AVAILABLE!
        
        if (updated == 0) {
            // DB says not available (maybe stale Redis?)
            redis.delete(holdKey); // Cleanup Redis
            return HoldResult.unavailable();
        }
        
        // Step 3: Track user's held seats (max 6 per event)
        String userHoldsKey = "user-holds:" + eventId + ":" + userId;
        Long holdCount = redis.opsForSet().add(userHoldsKey, seatId);
        redis.expire(userHoldsKey, HOLD_DURATION);
        
        if (redis.opsForSet().size(userHoldsKey) > 6) {
            // Too many seats! Release this one.
            releaseHold(eventId, seatId, userId);
            return HoldResult.tooManyHolds();
        }
        
        return HoldResult.success(HOLD_DURATION);
    }
    
    public void releaseHold(String eventId, String seatId, String userId) {
        String holdKey = "hold:" + eventId + ":" + seatId;
        
        // Only release if this user holds it!
        String currentHolder = redis.opsForValue().get(holdKey);
        if (userId.equals(currentHolder)) {
            redis.delete(holdKey);
            seatRepository.releaseSeat(seatId, SeatStatus.AVAILABLE);
        }
    }
    
    /**
     * Background job: cleanup expired holds in DB
     * (Redis expires automatically, but DB needs explicit cleanup)
     */
    @Scheduled(fixedRate = 30_000) // Every 30 seconds
    public void cleanupExpiredHolds() {
        int released = seatRepository.releaseExpiredHolds(Instant.now());
        if (released > 0) {
            log.info("Released {} expired seat holds", released);
        }
    }
}
```

### Booking Service with Payment Saga

```java
@Service
public class BookingService {
    
    @Autowired private SeatHoldService holdService;
    @Autowired private PaymentGateway paymentGateway;
    @Autowired private BookingRepository bookingRepo;
    @Autowired private NotificationService notificationService;
    
    public BookingResult completeBooking(BookingRequest request) {
        String bookingId = UUID.randomUUID().toString();
        
        try {
            // Step 1: Verify hold is still valid
            if (!holdService.isHeldByUser(
                    request.getEventId(), 
                    request.getSeatId(), 
                    request.getUserId())) {
                return BookingResult.holdExpired();
            }
            
            // Step 2: Transition seat to CONFIRMING (point of no return)
            seatRepository.updateStatus(
                request.getSeatId(), 
                SeatStatus.HELD, 
                SeatStatus.CONFIRMING);
            
            // Step 3: Charge payment (idempotent!)
            PaymentResult payment = paymentGateway.charge(
                ChargeRequest.builder()
                    .idempotencyKey(bookingId) // Prevents double-charge!
                    .amount(request.getAmount())
                    .currency("USD")
                    .customerId(request.getUserId())
                    .description("Ticket: " + request.getEventName())
                    .build());
            
            if (!payment.isSuccess()) {
                // Payment failed → release seat!
                seatRepository.updateStatus(
                    request.getSeatId(), 
                    SeatStatus.CONFIRMING, 
                    SeatStatus.AVAILABLE);
                holdService.releaseHold(
                    request.getEventId(), 
                    request.getSeatId(), 
                    request.getUserId());
                return BookingResult.paymentFailed(payment.getError());
            }
            
            // Step 4: Create booking record
            Booking booking = Booking.builder()
                .id(bookingId)
                .seatId(request.getSeatId())
                .userId(request.getUserId())
                .eventId(request.getEventId())
                .paymentId(payment.getId())
                .amount(request.getAmount())
                .status(BookingStatus.CONFIRMED)
                .qrCode(generateQRCode(bookingId))
                .bookedAt(Instant.now())
                .build();
            
            bookingRepo.save(booking);
            
            // Step 5: Finalize seat status
            seatRepository.updateStatus(
                request.getSeatId(), 
                SeatStatus.CONFIRMING, 
                SeatStatus.BOOKED);
            
            // Step 6: Send confirmation (async, don't block!)
            notificationService.sendBookingConfirmation(booking);
            
            return BookingResult.success(booking);
            
        } catch (Exception e) {
            // Compensate! Check what was done and undo.
            compensateFailure(bookingId, request, e);
            throw new BookingException("Booking failed", e);
        }
    }
}
```

### Virtual Waiting Room

```java
@Service
public class WaitingRoomService {
    
    @Autowired private StringRedisTemplate redis;
    
    private static final int BATCH_SIZE = 5000;
    private static final Duration ACCESS_TOKEN_VALIDITY = Duration.ofMinutes(10);
    
    /**
     * User enters waiting room before on-sale.
     * Position is RANDOM (fair for all early arrivals!)
     */
    public QueueTicket enterQueue(String eventId, String userId) {
        String queueKey = "queue:" + eventId;
        Instant onSaleTime = getOnSaleTime(eventId);
        
        double score;
        if (Instant.now().isBefore(onSaleTime)) {
            // Before on-sale: random position (FAIR!)
            score = ThreadLocalRandom.current().nextDouble();
        } else {
            // After on-sale: FIFO (incremental)
            score = System.currentTimeMillis(); // Timestamp as score
        }
        
        redis.opsForZSet().add(queueKey, userId, score);
        
        // Get position
        Long position = redis.opsForZSet().rank(queueKey, userId);
        long totalInQueue = redis.opsForZSet().size(queueKey);
        
        return QueueTicket.builder()
            .position(position + 1)
            .totalInQueue(totalInQueue)
            .estimatedWaitMinutes(
                (position / BATCH_SIZE) * 0.5) // 30s per batch
            .build();
    }
    
    /**
     * Release next batch of users (called every 30 seconds).
     * Gives them access tokens to proceed to booking page.
     */
    @Scheduled(fixedRate = 30_000)
    public void releaseNextBatch() {
        for (String eventId : getActiveOnSaleEvents()) {
            String queueKey = "queue:" + eventId;
            
            // Pop first BATCH_SIZE users from sorted set
            Set<String> batch = redis.opsForZSet()
                .range(queueKey, 0, BATCH_SIZE - 1);
            
            if (batch == null || batch.isEmpty()) continue;
            
            for (String userId : batch) {
                // Generate access token (JWT with 10-min expiry)
                String accessToken = generateAccessToken(userId, eventId);
                
                // Store token for validation
                String tokenKey = "access:" + eventId + ":" + userId;
                redis.opsForValue().set(tokenKey, accessToken, 
                    ACCESS_TOKEN_VALIDITY);
                
                // Notify user via WebSocket: "It's your turn!"
                webSocketService.send(userId, 
                    new YourTurnEvent(eventId, accessToken));
            }
            
            // Remove released users from queue
            redis.opsForZSet().removeRange(queueKey, 0, BATCH_SIZE - 1);
        }
    }
}
```

---

## 🎮 Mini Challenge

### 🧩 Design: "Best Available" Seat Algorithm

User doesn't pick a specific seat — they just say "Give me 4 best available seats together!" How do you find 4 adjacent seats in the best section quickly?

<details>
<summary>🔑 Answer</summary>

```java
public List<Seat> findBestAvailable(String eventId, int count) {
    // Strategy: Find consecutive available seats in best sections first
    
    List<Section> sections = getSectionsByPriority(eventId); // VIP first!
    
    for (Section section : sections) {
        List<Row> rows = section.getRows();
        
        for (Row row : rows) {
            List<Seat> available = row.getAvailableSeats(); // Sorted by number!
            
            // Find longest consecutive sequence >= count
            List<Seat> consecutive = findConsecutive(available, count);
            
            if (consecutive != null) {
                // Prefer CENTER seats (better view!)
                // Score: distance from center of row
                return selectMostCentered(consecutive, count, row.getSize());
            }
        }
    }
    
    return Collections.emptyList(); // No consecutive seats available!
}

private List<Seat> findConsecutive(List<Seat> seats, int needed) {
    int streak = 1;
    int streakStart = 0;
    
    for (int i = 1; i < seats.size(); i++) {
        if (seats.get(i).getNumber() == seats.get(i-1).getNumber() + 1) {
            streak++;
            if (streak >= needed) {
                return seats.subList(streakStart, streakStart + needed);
            }
        } else {
            streak = 1;
            streakStart = i;
        }
    }
    return null;
}
```

**Optimization:** Pre-compute a bitmap per row (1=available, 0=taken). Finding consecutive 1s in a bitmap is O(row_size). Cache bitmaps in Redis for real-time updates!
</details>

---

## ❓ Interview Q&A

**Q1: How do you prevent overselling (double-booking) a seat?**
> Three-layer defense: (1) Redis SETNX for fast atomic hold (first check), (2) Database optimistic locking with version column (second check), (3) Unique constraint on (event_id, seat_id) in bookings table (final safety net). Even if Redis and app logic somehow fail, the database constraint prevents duplicate bookings. This is defense-in-depth.

**Q2: Why use a virtual waiting room instead of just handling all requests?**
> Without queue: 5M users hit the system at once → all get errors → all retry → thundering herd → system dies. With queue: controlled flow of ~5000 users every 30 seconds → system handles gracefully → users get clear expectation ("your position is X, wait ~Y minutes"). It's also FAIRER — random queue position prevents bots and fast-clickers from having advantage.

**Q3: How do you handle payment failures after seat is held?**
> Saga pattern with compensation: (1) Seat moves to CONFIRMING state during payment, (2) If payment fails → seat goes back to AVAILABLE (compensation), (3) If payment succeeds but DB update fails → check payment status on retry, if charged → retry booking, if not → release seat. Idempotency keys prevent double-charges on retries. Background reconciliation job catches any inconsistencies.

**Q4: How would you implement dynamic pricing?**
> Price = f(demand, supply, time_to_event). Signals: (1) Queue depth (many people waiting → increase price), (2) Sell velocity (selling fast → increase), (3) Time to event (closer = more expensive for popular, less for unpopular), (4) Section/row quality. Algorithm: start with base price, apply multipliers. Cap maximum increase (ethical/legal limits). Show original price + demand premium for transparency. Update prices every 5 minutes, not per-request (stability).

**Q5: How do you handle the case where 50K seats sell out in 2 minutes?**
> Pre-compute everything: (1) Load all seat availability into Redis at on-sale start (O(1) lookups!), (2) Seat map rendered as static image with clickable overlay (CDN-served), (3) Only "hold" and "book" operations hit the backend, (4) Once inventory depleted in Redis → immediately show "SOLD OUT" (don't even try DB), (5) Graceful degradation: show waitlist option with one click, (6) Track real-time inventory count → display to users ("23 seats remaining!").

---

## 🔗 Related Topics
- [Distributed Locking](../../Microservices/DistributedLocking.md) — Seat hold mechanism
- [Rate Limiting](../../BuildingBlocks/RateLimiting.md) — Protecting against flash sale spikes
- [Idempotency](../../APIs/Idempotency.md) — Preventing double-charges
- [Caching](../../BuildingBlocks/Caching.md) — Redis for inventory & holds
- [Message Queues](../../BuildingBlocks/MessageQueues.md) — Async processing & fairness queue

---

*"The difference between a good ticket system and a bad one: a good system tells you 'Sorry, sold out' in 2 seconds. A bad system tells you 'Something went wrong' after you waited 45 minutes, entered your credit card, and your soul left your body." — Every Ticketmaster Victim* 🎫
