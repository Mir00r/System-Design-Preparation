# 🏠 Design Airbnb / Hotel Booking System

> *"Airbnb manages 7.7 MILLION listings across 100,000 cities. On New Year's Eve alone, 5.5 million guests check in worldwide. The engineering challenge: when 1000 people search for 'Paris apartment Dec 31', show accurate availability in real-time — while handling double-booking prevention, dynamic pricing, trust & safety (fraudulent listings!), and search ranking that balances guest preferences with host revenue."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Search Index](../../BuildingBlocks/SearchIndex.md), [Caching](../../BuildingBlocks/Caching.md), [Database Sharding](../../Database/Sharding.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [High-Level Architecture](#-high-level-architecture)
3. [Search & Discovery](#-search--discovery)
4. [Availability & Booking](#-availability--booking)
5. [Pricing Engine](#-pricing-engine)
6. [Trust & Safety](#-trust--safety)
7. [Payments & Payouts](#-payments--payouts)
8. [Java Implementation](#-java-implementation)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
✅ Host: List property (photos, description, amenities, pricing)
✅ Guest: Search listings (location, dates, guests, filters)
✅ View listing details & reviews
✅ Check real-time availability
✅ Book a listing (dates, guest count)
✅ Payment processing (hold → charge after check-in)
✅ Reviews (both guest reviews host AND host reviews guest!)
✅ Messaging between host and guest
✅ Host calendar management (block dates, set pricing)
✅ Cancellation policies
```

### Non-Functional Requirements
```
✅ Search latency < 500ms (complex geo + date queries!)
✅ Booking consistency (NO double-booking!)
✅ High availability (people book at all hours, all zones)
✅ Scale: 7M+ listings, 150M+ users, 2M bookings/day
✅ Global: multi-region for low latency worldwide
✅ Accurate availability (real-time, consistent)
```

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     CLIENTS (Web / Mobile)                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  API Gateway    │ (Auth, Rate Limiting, Routing)
                    └────────┬────────┘
                             │
       ┌─────────────────────┼──────────────────────────────┐
       ▼                     ▼                              ▼
┌────────────┐      ┌──────────────┐              ┌──────────────┐
│  Search    │      │  Booking     │              │  User        │
│  Service   │      │  Service     │              │  Service     │
│(Elasticsearch)│   │  (CRITICAL!) │              │  (Profiles)  │
└─────┬──────┘      └──────┬───────┘              └──────────────┘
      │                     │
      │               ┌─────┼─────────────────┐
      ▼               ▼     ▼                 ▼
┌──────────┐   ┌──────────┐ ┌──────────┐ ┌──────────────┐
│ Listing  │   │Availability│ │ Payment │ │ Notification │
│ Service  │   │  Service  │ │ Service │ │ Service      │
└──────────┘   └──────────┘ └──────────┘ └──────────────┘
      │              │           │
      ▼              ▼           ▼
┌──────────┐   ┌──────────┐ ┌──────────┐
│PostgreSQL│   │ Redis +   │ │ Stripe   │
│(listings)│   │ PostgreSQL│ │ (payment)│
└──────────┘   │(calendar) │ └──────────┘
               └──────────┘

ADDITIONAL SERVICES:
  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
  │  Pricing     │ │  Review      │ │  Messaging   │
  │  Engine      │ │  Service     │ │  Service     │
  └──────────────┘ └──────────────┘ └──────────────┘
  ┌──────────────┐ ┌──────────────┐
  │  Trust &     │ │  Analytics   │
  │  Safety      │ │  Service     │
  └──────────────┘ └──────────────┘
```

---

## 🔍 Search & Discovery

```
SEARCH QUERY EXAMPLE:
  "2 guests, Paris, Dec 28-31, under $200/night, WiFi, 2+ bedrooms"

THIS REQUIRES:
  1. Geo search (properties in Paris bounds)
  2. Availability check (free Dec 28-31)
  3. Capacity filter (2+ guests)
  4. Price filter (< $200/night for those dates!)
  5. Amenity filter (has WiFi, 2+ bedrooms)
  6. RANKING (not just matching — best results first!)

SEARCH ARCHITECTURE:
  ┌─────────────────────────────────────────────────┐
  │  Elasticsearch Cluster                           │
  │                                                  │
  │  Index: listings                                │
  │  Fields:                                        │
  │   • location (geo_point) → geo distance queries │
  │   • bedrooms (integer) → range filter           │
  │   • max_guests (integer) → range filter         │
  │   • amenities (keyword[]) → terms filter        │
  │   • base_price (float) → range filter           │
  │   • rating (float) → scoring factor             │
  │   • instant_book (boolean)                      │
  │   • listing_type (keyword)                      │
  │   • availability_bitmap → 365 bits (1=available)│
  │                                                  │
  └─────────────────────────────────────────────────┘

AVAILABILITY BITMAP TRICK:
  Instead of checking calendar DB for each listing:
  Pre-compute 365-day bitmap in Elasticsearch!
  
  Day 0 = today. Bit 1 = available, Bit 0 = booked.
  
  listing_123: 1111111111100000111111111111... (365 bits)
                          ↑ booked 5 nights!
  
  Search "available Dec 28-31" → check bits 100-103 are all 1!
  Done in Elasticsearch without hitting calendar DB! FAST!
  
  Update bitmap: when booking confirmed → flip bits to 0
  Refresh: nightly rebuild from source-of-truth (calendar DB)

RANKING SIGNALS:
  relevance_score = 
    0.3 × quality_score (photos, description, amenities)
    + 0.25 × review_score (rating × review_count)
    + 0.2 × price_competitiveness (vs similar listings)
    + 0.15 × conversion_rate (how often viewed → booked)
    + 0.1 × host_response_rate (superhost bonus!)
    
  Personal boost:
    + Previously booked similar style? Boost!
    + Wishlist overlap with search? Boost!
```

---

## 📅 Availability & Booking

```
CALENDAR MODEL:

  ┌────────────────────────────────────────────────────────┐
  │  Table: listing_calendar                                │
  │                                                         │
  │  listing_id │ date       │ status    │ price │ note    │
  │  ──────────────────────────────────────────────────── │
  │  L123       │ 2024-12-28 │ AVAILABLE │ $180  │         │
  │  L123       │ 2024-12-29 │ AVAILABLE │ $200  │ NYE!    │
  │  L123       │ 2024-12-30 │ BLOCKED   │ -     │ Personal│
  │  L123       │ 2024-12-31 │ AVAILABLE │ $250  │ NYE     │
  │  L123       │ 2025-01-01 │ BOOKED    │ $220  │ book#99 │
  └────────────────────────────────────────────────────────┘
  
  Status: AVAILABLE | BLOCKED (by host) | BOOKED | PENDING

BOOKING FLOW (Preventing Double-Booking):

  Guest wants Dec 28-31 (4 nights):
  
  1. CHECK: Are all 4 nights AVAILABLE?
     SELECT COUNT(*) FROM listing_calendar 
     WHERE listing_id='L123' 
     AND date IN ('2024-12-28','2024-12-29','2024-12-30','2024-12-31')
     AND status = 'AVAILABLE';
     → Must equal 4!
     
  2. LOCK: Atomic status update (optimistic locking!)
     UPDATE listing_calendar 
     SET status = 'PENDING', booking_id = 'B789'
     WHERE listing_id = 'L123'
     AND date IN ('2024-12-28', '2024-12-29', '2024-12-30', '2024-12-31')
     AND status = 'AVAILABLE';
     → If affected_rows = 4: SUCCESS! All dates locked!
     → If affected_rows < 4: CONFLICT! Someone booked a night already!
     
  3. PAYMENT: Process payment (while dates are PENDING)
  
  4. CONFIRM: 
     If payment succeeds: UPDATE status = 'BOOKED'
     If payment fails: UPDATE status = 'AVAILABLE' (release!)
     
  5. TIMEOUT: If PENDING > 15 minutes (payment abandoned):
     Background job resets to AVAILABLE

WHY THIS PREVENTS DOUBLE-BOOKING:
  The UPDATE with WHERE status='AVAILABLE' is ATOMIC!
  If two guests try simultaneously:
  Guest A: UPDATE... WHERE status='AVAILABLE' → 4 rows! ✅
  Guest B: UPDATE... WHERE status='AVAILABLE' → 0 rows! ❌ (already PENDING!)
  
  Database guarantee: one of them will succeed, other will fail!
```

---

## 💰 Pricing Engine

```
DYNAMIC PRICING (Smart Pricing):

  base_price = host's set price ($150/night)
  
  Adjustments:
  ┌──────────────────────────────────────────────────────────┐
  │  Factor              │  Multiplier                        │
  ├──────────────────────────────────────────────────────────┤
  │  Weekend             │  × 1.2 (20% more on weekends)    │
  │  Holiday/NYE         │  × 2.0 - 3.0 (peak demand!)      │
  │  Last minute (< 3d)  │  × 0.8 (discount to fill!)       │
  │  High demand area    │  × 1.3 (events, conferences)     │
  │  Low occupancy       │  × 0.9 (fill the gaps!)          │
  │  Long stay (7+ days) │  × 0.85 (weekly discount)        │
  │  Competitor pricing  │  Adjust to market rate            │
  └──────────────────────────────────────────────────────────┘
  
  final_price = base_price × Π(multipliers) 
  
  AIRBNB'S APPROACH:
  ML model trained on:
  • Historical booking data (what price → booked quickly?)
  • Comparable listings (similar property, location, amenities)
  • Demand signals (search volume for this area + dates)
  • Supply (how many available listings in area?)
  
  Output: "Recommended price: $175 (80% chance of booking)"
  Host can override! But smart pricing fills more nights.

SERVICE FEE STRUCTURE:
  Guest pays: listing_price + cleaning_fee + service_fee (14-16%)
  Host receives: listing_price + cleaning_fee - host_fee (3%)
  Airbnb keeps: service_fee + host_fee (~17-19% total take rate!)
```

---

## 🛡️ Trust & Safety

```
FRAUD DETECTION:
  ┌────────────────────────────────────────────────────────────┐
  │  Risk Signal          │  Action                            │
  ├────────────────────────────────────────────────────────────┤
  │  New host, no reviews │  Manual review of listing photos   │
  │  Suspiciously low $$  │  Flag for review (bait & switch?)  │
  │  Multiple listings    │  Verify not a scam operation       │
  │  same photos          │  (reverse image search!)           │
  │  Guest: new account,  │  Require ID verification before    │
  │  booking expensive    │  allowing booking                  │
  │  property             │                                    │
  │  Off-platform comms   │  Block external links in messages  │
  │  (payment bypass!)    │                                    │
  └────────────────────────────────────────────────────────────┘

IDENTITY VERIFICATION:
  • Government ID scan (passport, driver's license)
  • Selfie matching (biometric comparison to ID photo)
  • Phone verification (SMS OTP)
  • Email verification
  • Social connections (Facebook, LinkedIn — trust signals)
  
REVIEW SYSTEM:
  • Dual-blind: neither sees other's review until both submit!
  • 14-day window to leave review after checkout
  • Can't edit after publication
  • Response option for hosts (address concerns publicly)
  • Minimum 3 reviews for "Superhost" status

CONTENT MODERATION:
  Photos: ML model detects inappropriate content
  Text: NLP filters for harassment, discrimination
  Listing descriptions: verified against reality (check-in reports)
```

---

## 💳 Payments & Payouts

```
PAYMENT FLOW:
  ┌──────────────────────────────────────────────────────────┐
  │  BOOKING CONFIRMED                                        │
  │    │                                                      │
  │    ▼                                                      │
  │  Charge guest's card (HOLD, not capture!)                │
  │  $800 for 4 nights → authorized, not charged yet!        │
  │    │                                                      │
  │    ▼  (24 hours before check-in)                         │
  │  CAPTURE payment ($800 actually charged!)                │
  │    │                                                      │
  │    ▼  (24 hours after check-in)                          │
  │  PAYOUT to host ($800 - 3% fee = $776)                  │
  │  via bank transfer / PayPal                              │
  └──────────────────────────────────────────────────────────┘
  
  WHY HOLD FIRST?
  • If guest cancels → release hold (no charge!)
  • If host cancels → no charge + Airbnb pays guest rebooking credit
  • Protects both parties during uncertainty period
  
  MULTI-CURRENCY:
  Guest in US pays in USD → Airbnb holds in USD
  Host in France receives in EUR → Airbnb converts at payout
  Exchange rate locked at booking time (protects both parties!)

CANCELLATION HANDLING:
  Flexible: Full refund if cancelled 24h+ before check-in
  Moderate: 50% refund if cancelled 5+ days before
  Strict: No refund within 7 days of check-in
  
  Implementation:
    cancellation_deadline = check_in_date - policy.days
    if NOW() < cancellation_deadline → full/partial refund
    else → no refund (host keeps money)
```

---

## 💻 Java Implementation

### Search Service

```java
@Service
public class ListingSearchService {
    
    @Autowired private ElasticsearchClient esClient;
    
    public SearchResult searchListings(SearchRequest request) {
        // Build Elasticsearch query
        BoolQuery.Builder query = new BoolQuery.Builder();
        
        // 1. Geo filter: listings within bounding box / radius
        query.filter(QueryBuilders.geoBoundingBox()
            .field("location")
            .boundingBox(b -> b.tlbr(tl -> tl
                .lat(request.getNorthEastLat())
                .lon(request.getSouthWestLon()),
                br -> br.lat(request.getSouthWestLat())
                    .lon(request.getNorthEastLon())))
            .build()._toQuery());
        
        // 2. Availability filter: check bitmap for requested dates
        int startDay = daysSinceToday(request.getCheckIn());
        int endDay = daysSinceToday(request.getCheckOut());
        for (int day = startDay; day < endDay; day++) {
            query.filter(QueryBuilders.term()
                .field("availability_bitmap." + day)
                .value(1) // Must be available!
                .build()._toQuery());
        }
        
        // 3. Guest count filter
        query.filter(QueryBuilders.range()
            .field("max_guests")
            .gte(JsonData.of(request.getGuests()))
            .build()._toQuery());
        
        // 4. Price range filter
        if (request.getMaxPrice() != null) {
            query.filter(QueryBuilders.range()
                .field("base_price")
                .lte(JsonData.of(request.getMaxPrice()))
                .build()._toQuery());
        }
        
        // 5. Amenity filters
        if (request.getAmenities() != null) {
            for (String amenity : request.getAmenities()) {
                query.filter(QueryBuilders.term()
                    .field("amenities")
                    .value(amenity)
                    .build()._toQuery());
            }
        }
        
        // 6. Scoring (ranking!)
        FunctionScoreQuery functionScore = FunctionScoreQuery.of(fs -> fs
            .query(query.build()._toQuery())
            .functions(
                // Boost by rating
                FunctionScore.of(f -> f.fieldValueFactor(
                    fvf -> fvf.field("rating").modifier(Modifier.Log1p))),
                // Boost by review count
                FunctionScore.of(f -> f.fieldValueFactor(
                    fvf -> fvf.field("review_count").modifier(Modifier.Sqrt))),
                // Boost Superhosts
                FunctionScore.of(f -> f.filter(
                    QueryBuilders.term().field("superhost").value(true).build()._toQuery())
                    .weight(1.3))
            )
            .scoreMode(FunctionScoreMode.Multiply)
        );
        
        // Execute search
        var response = esClient.search(s -> s
            .index("listings")
            .query(functionScore._toQuery())
            .from(request.getOffset())
            .size(request.getLimit()),
            ListingDocument.class);
        
        return mapToSearchResult(response);
    }
}
```

### Booking Service (Double-Booking Prevention)

```java
@Service
public class BookingService {
    
    @Autowired private CalendarRepository calendarRepo;
    @Autowired private BookingRepository bookingRepo;
    @Autowired private PaymentService paymentService;
    @Autowired private SearchIndexUpdater indexUpdater;
    
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public BookingResult createBooking(BookingRequest request) {
        String bookingId = UUID.randomUUID().toString();
        
        LocalDate checkIn = request.getCheckIn();
        LocalDate checkOut = request.getCheckOut();
        List<LocalDate> dates = checkIn.datesUntil(checkOut)
            .collect(Collectors.toList());
        
        // Step 1: Atomic availability check + lock
        int lockedDates = calendarRepo.lockDatesForBooking(
            request.getListingId(),
            dates,
            bookingId,
            CalendarStatus.AVAILABLE,  // Only if currently available!
            CalendarStatus.PENDING     // Transition to PENDING
        );
        
        if (lockedDates != dates.size()) {
            // Some dates were NOT available → conflict!
            calendarRepo.releasePendingDates(bookingId); // Cleanup
            return BookingResult.datesUnavailable();
        }
        
        // Step 2: Calculate total price
        BigDecimal totalPrice = calendarRepo
            .getPricesForDates(request.getListingId(), dates)
            .stream()
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        BigDecimal serviceFee = totalPrice.multiply(new BigDecimal("0.14"));
        BigDecimal totalCharge = totalPrice.add(serviceFee);
        
        // Step 3: Authorize payment (hold, don't capture yet!)
        PaymentAuth auth = paymentService.authorize(
            request.getUserId(), 
            totalCharge,
            bookingId); // Idempotency key!
        
        if (!auth.isSuccess()) {
            calendarRepo.releasePendingDates(bookingId);
            return BookingResult.paymentFailed(auth.getError());
        }
        
        // Step 4: Confirm booking
        Booking booking = Booking.builder()
            .id(bookingId)
            .listingId(request.getListingId())
            .guestId(request.getUserId())
            .checkIn(checkIn)
            .checkOut(checkOut)
            .guests(request.getGuests())
            .totalPrice(totalPrice)
            .serviceFee(serviceFee)
            .paymentAuthId(auth.getId())
            .status(BookingStatus.CONFIRMED)
            .build();
        
        bookingRepo.save(booking);
        
        // Step 5: Update calendar to BOOKED
        calendarRepo.confirmBooking(bookingId, CalendarStatus.BOOKED);
        
        // Step 6: Update search index (flip availability bits!)
        indexUpdater.markDatesUnavailable(request.getListingId(), dates);
        
        // Step 7: Notify host + guest (async)
        notificationService.sendBookingConfirmation(booking);
        
        return BookingResult.success(booking);
    }
}
```

---

## 🎮 Mini Challenge

### 🧩 Design: Instant Book vs. Request to Book

Some listings allow instant booking (confirmed immediately), others require host approval (request-to-book). Design the workflow differences.

<details>
<summary>🔑 Answer</summary>

```java
public BookingResult processBooking(BookingRequest request) {
    Listing listing = listingService.get(request.getListingId());
    
    if (listing.isInstantBook()) {
        // INSTANT BOOK: Lock dates + charge immediately!
        return createConfirmedBooking(request);
        
    } else {
        // REQUEST TO BOOK: Hold dates + notify host + wait for approval
        return createBookingRequest(request);
    }
}

private BookingResult createBookingRequest(BookingRequest request) {
    // 1. Soft-hold dates (PENDING status, 24h timeout)
    lockDates(request, CalendarStatus.REQUESTED, Duration.ofHours(24));
    
    // 2. Pre-authorize payment (hold on card, don't charge!)
    paymentService.authorize(request); // Card validated, funds held
    
    // 3. Notify host: "You have a booking request!"
    notificationService.notifyHost(request, NotificationType.BOOKING_REQUEST);
    
    // 4. Host has 24 hours to accept/decline
    // If accept → confirm booking, capture payment
    // If decline → release dates, release payment hold
    // If timeout → auto-decline, release everything
    
    return BookingResult.requestPending();
}
```

**Key difference:** Instant book locks dates permanently; request-to-book creates a *soft hold* that expires if host doesn't respond. The 24h timer prevents dates from being blocked forever.
</details>

---

## ❓ Interview Q&A

**Q1: How do you handle search with real-time availability?**
> Pre-compute availability bitmaps in Elasticsearch (365 bits per listing). On booking: flip bits to 0 via async event. Trade-off: bitmap might be slightly stale (seconds), but actual booking still validates against calendar DB (source of truth). This gives fast search (milliseconds for millions of listings) with eventual consistency, and strong consistency at booking time.

**Q2: How do you prevent double-booking when two guests try to book the same dates?**
> Database-level atomic update: `UPDATE calendar SET status='PENDING' WHERE listing_id=X AND date IN (...) AND status='AVAILABLE'`. Check affected_rows = requested dates. If less: another booking got there first → reject and release. PostgreSQL's row-level locking guarantees only one transaction succeeds. This is simpler and more reliable than application-level locking.

**Q3: How would you design the search ranking algorithm?**
> Multi-factor scoring: quality (photo count, description length, amenities), popularity (booking rate, click-through rate), host reliability (response rate, superhost status), freshness (new listings get temporary boost), price competitiveness (vs similar listings). Use function_score in Elasticsearch to combine these factors. A/B test weights continuously. Also personalize: if user previously booked apartments, boost apartments over hotels.

**Q4: How do you handle peak demand (New Year's Eve, Olympics)?**
> Multi-layer: (1) CDN cache search results (30s TTL for popular locations), (2) Pre-compute availability for peak dates, (3) Rate limit searches per user (not more than 10/minute), (4) Auto-scale search cluster 2 weeks before known peaks, (5) Degrade gracefully: if overloaded, show cached results (slightly stale) rather than errors, (6) Queue booking requests (controlled admission to prevent DB overload).

**Q5: How does the payment flow protect both host and guest?**
> Authorization hold at booking → actual charge 24h before check-in → payout to host 24h after check-in. This protects: Guest (can cancel during free period, get full refund from released hold), Host (payment is guaranteed once captured, even if guest's card later fails), Airbnb (holds money during stay, can mediate disputes before releasing to host). Dispute resolution: if guest complains during stay, Airbnb can withhold payout pending investigation.

---

## 🔗 Related Topics
- [Search Index](../../BuildingBlocks/SearchIndex.md) — Elasticsearch for listing search
- [Caching](../../BuildingBlocks/Caching.md) — Caching search results & listings
- [Database Sharding](../../Database/Sharding.md) — Scaling calendar/booking data
- [Rate Limiting](../../BuildingBlocks/RateLimiting.md) — Protecting search under peak load
- [Idempotency](../../APIs/Idempotency.md) — Payment safety

---

*"Airbnb's real product isn't the platform — it's trust. Trust between strangers to sleep in each other's homes. The engineering challenge is building systems that make this trust possible at scale: identity verification, reviews, insurance, and instant conflict resolution." — Airbnb Engineering* 🏠
