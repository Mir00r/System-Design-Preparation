# 🅿️ LLD Interview: Design a Parking Lot System 🚀

**⏱️ 45 minutes** | **🎯 🟡 Intermediate** | **🔗 Prerequisites**: [OOP Design Principles](./01_OOP_Design_Principles.md)

> **"The best design problems have no right answer — but they do have many wrong ones."**

The Parking Lot is the most common LLD interview question. It tests your ability to model real-world entities, apply OOP principles, choose appropriate design patterns, and handle edge cases.

---

## 📋 Table of Contents

1. [Requirements Gathering](#1-requirements-gathering)
2. [Identify Entities & Relationships](#2-identify-entities--relationships)
3. [Class Diagram](#3-class-diagram)
4. [Core Enums & Value Objects](#4-core-enums--value-objects)
5. [Entity Classes](#5-entity-classes)
6. [Strategy: Parking Spot Finder](#6-strategy-parking-spot-finder)
7. [Facade: ParkingLot](#7-facade-parkinglot)
8. [Observer: Events](#8-observer-events)
9. [Pricing Strategy](#9-pricing-strategy)
10. [Complete Example Flow](#10-complete-example-flow)
11. [Design Decisions Explained](#11-design-decisions-explained)
12. [Interview Q&A](#12-interview-qa)

---

## 1. Requirements Gathering

**Always start with requirements** — ask the interviewer these:

**Functional Requirements:**
- Vehicles enter the parking lot (car, motorcycle, truck)
- System assigns available spots automatically
- Issue a ticket on entry, bill on exit
- Different spot types: compact, regular, large
- Display available spots count per type

**Non-Functional Requirements:**
- Single parking lot (can extend to multiple floors)
- Thread-safe (concurrent entry/exit)
- Support different pricing strategies

**Out of Scope (say this explicitly):**
- Payment processing integration
- Real-time display boards
- License plate recognition

---

## 2. Identify Entities & Relationships

```
┌─────────────────────────────────────────────────────────────┐
│                   ENTITY MAP                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Vehicle ───────> ParkingTicket ───────> ParkingSpot        │
│    (car, moto,       (entry time,         (floor, number,   │
│     truck)           exit time)            type, occupied)  │
│                           │                                 │
│                           ▼                                 │
│                      ParkingFloor ──────> ParkingLot        │
│                      (spots list)         (floors,          │
│                                           strategies)       │
│                                                             │
│  PricingStrategy ──────────────> Fee Calculation           │
│  SpotFindingStrategy ──────────> Spot Assignment           │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Class Diagram

```
┌────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Vehicle      │     │  ParkingTicket   │     │  ParkingSpot    │
│─────────────── │     │───────────────── │     │──────────────── │
│ licensePlate   │     │ ticketId         │     │ spotId          │
│ vehicleType    │     │ vehicle          │     │ type (enum)     │
│                │     │ spot             │     │ floor           │
│ +getType()     │     │ entryTime        │     │ isOccupied      │
└────────────────┘     │ exitTime         │     │                 │
                       │                 │     │ +occupy()       │
                       │ +getDuration()  │     │ +vacate()       │
                       │ +close()        │     │ +canFit(Vehicle)│
                       └─────────────────┘     └─────────────────┘
                                │
                                │ uses
                       ┌────────▼───────────┐
                       │  PricingStrategy   │
                       │─────────────────── │
                       │ +calculate(ticket) │
                       └────────────────────┘
                                ↑
           ┌────────────────────┴─────────────────┐
           │                                      │
┌──────────┴──────┐                    ┌──────────┴──────┐
│ HourlyPricing   │                    │ FlatRatePricing │
│─────────────────│                    │─────────────────│
│ ratePerHour     │                    │ flatFee         │
└─────────────────┘                    └─────────────────┘

┌────────────────────┐     ┌─────────────────────────────┐
│  ParkingFloor      │     │  ParkingLot                 │
│─────────────────── │     │──────────────────────────── │
│ floorNumber        │ ◄── │ floors                      │
│ spots: List<Spot>  │     │ spotFinder: Strategy        │
│                    │     │ activeTickets: Map           │
│ +findSpot(Vehicle) │     │                             │
│ +getAvailable()    │     │ +parkVehicle(Vehicle)       │
└────────────────────┘     │ +exitVehicle(ticketId)      │
                           │ +getAvailability()          │
                           └─────────────────────────────┘
```

---

## 4. Core Enums & Value Objects

```java
// Vehicle types supported
public enum VehicleType {
    MOTORCYCLE, CAR, TRUCK
}

// Parking spot sizes
public enum SpotType {
    COMPACT,    // motorcycles only
    REGULAR,    // cars and motorcycles
    LARGE       // trucks, cars, motorcycles
}

// Immutable value object for parking fee
public record ParkingFee(BigDecimal amount, Currency currency) {
    public ParkingFee {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Fee cannot be negative");
    }

    public static ParkingFee of(double amount) {
        return new ParkingFee(BigDecimal.valueOf(amount), Currency.getInstance("USD"));
    }
}

// Ticket identifier (type-safe)
public record TicketId(String value) {
    public TicketId {
        Objects.requireNonNull(value);
        if (value.isBlank()) throw new IllegalArgumentException("TicketId cannot be blank");
    }

    public static TicketId generate() {
        return new TicketId("TKT-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase());
    }
}
```

---

## 5. Entity Classes

```java
// Vehicle — immutable
public record Vehicle(String licensePlate, VehicleType type) {
    public Vehicle {
        Objects.requireNonNull(licensePlate, "License plate required");
        Objects.requireNonNull(type, "Vehicle type required");
        if (licensePlate.isBlank()) throw new IllegalArgumentException("License plate blank");
    }
}

// Parking Spot — mutable (state changes on park/exit)
public class ParkingSpot {
    private final String spotId;
    private final SpotType type;
    private final int floorNumber;
    private volatile boolean occupied;
    private volatile Vehicle currentVehicle;

    public ParkingSpot(String spotId, SpotType type, int floorNumber) {
        this.spotId = spotId;
        this.type = type;
        this.floorNumber = floorNumber;
        this.occupied = false;
    }

    public boolean canFit(Vehicle vehicle) {
        return switch (vehicle.type()) {
            case MOTORCYCLE -> true; // fits in any spot
            case CAR        -> type == SpotType.REGULAR || type == SpotType.LARGE;
            case TRUCK      -> type == SpotType.LARGE;
        };
    }

    public synchronized boolean occupy(Vehicle vehicle) {
        if (occupied || !canFit(vehicle)) return false;
        this.occupied = true;
        this.currentVehicle = vehicle;
        return true;
    }

    public synchronized void vacate() {
        this.occupied = false;
        this.currentVehicle = null;
    }

    public boolean isAvailable() { return !occupied; }
    public String getSpotId()    { return spotId; }
    public SpotType getType()    { return type; }
    public int getFloorNumber()  { return floorNumber; }
}

// Parking Ticket — records entry/exit
public class ParkingTicket {
    private final TicketId ticketId;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final Instant entryTime;
    private Instant exitTime;
    private ParkingFee fee;

    public ParkingTicket(Vehicle vehicle, ParkingSpot spot) {
        this.ticketId  = TicketId.generate();
        this.vehicle   = vehicle;
        this.spot      = spot;
        this.entryTime = Instant.now();
    }

    public void close(ParkingFee fee) {
        if (exitTime != null) throw new IllegalStateException("Ticket already closed");
        this.exitTime = Instant.now();
        this.fee = fee;
    }

    public Duration getDuration() {
        Instant end = exitTime != null ? exitTime : Instant.now();
        return Duration.between(entryTime, end);
    }

    public boolean isClosed()        { return exitTime != null; }
    public TicketId getTicketId()     { return ticketId; }
    public Vehicle getVehicle()       { return vehicle; }
    public ParkingSpot getSpot()      { return spot; }
    public Instant getEntryTime()     { return entryTime; }
    public Optional<ParkingFee> getFee() { return Optional.ofNullable(fee); }
}
```

---

## 6. Strategy: Parking Spot Finder

```java
// Strategy interface
@FunctionalInterface
public interface SpotFindingStrategy {
    Optional<ParkingSpot> findSpot(List<ParkingSpot> availableSpots, Vehicle vehicle);
}

// Default: find nearest available spot (first fit)
public class NearestFirstStrategy implements SpotFindingStrategy {
    @Override
    public Optional<ParkingSpot> findSpot(List<ParkingSpot> spots, Vehicle vehicle) {
        return spots.stream()
            .filter(ParkingSpot::isAvailable)
            .filter(spot -> spot.canFit(vehicle))
            .findFirst();
    }
}

// Alternative: prefer compact spots for motorcycles (resource optimization)
public class OptimalSizeStrategy implements SpotFindingStrategy {
    @Override
    public Optional<ParkingSpot> findSpot(List<ParkingSpot> spots, Vehicle vehicle) {
        SpotType preferredType = switch (vehicle.type()) {
            case MOTORCYCLE -> SpotType.COMPACT;
            case CAR        -> SpotType.REGULAR;
            case TRUCK      -> SpotType.LARGE;
        };

        // Try preferred size first, fall back to larger
        return spots.stream()
            .filter(ParkingSpot::isAvailable)
            .filter(spot -> spot.canFit(vehicle))
            .sorted(Comparator.comparing(spot ->
                spotTypePriority(spot.getType(), preferredType)))
            .findFirst();
    }

    private int spotTypePriority(SpotType actual, SpotType preferred) {
        if (actual == preferred) return 0;
        return actual.ordinal() > preferred.ordinal() ? 1 : 2;
    }
}
```

---

## 7. Facade: ParkingLot

```java
// ParkingFloor — manages spots on one floor
public class ParkingFloor {
    private final int floorNumber;
    private final List<ParkingSpot> spots;

    public ParkingFloor(int floorNumber, List<ParkingSpot> spots) {
        this.floorNumber = floorNumber;
        this.spots = new ArrayList<>(spots);
    }

    public List<ParkingSpot> getAvailableSpots(Vehicle vehicle) {
        return spots.stream()
            .filter(ParkingSpot::isAvailable)
            .filter(spot -> spot.canFit(vehicle))
            .toList();
    }

    public Map<SpotType, Long> getAvailabilityCount() {
        return spots.stream()
            .filter(ParkingSpot::isAvailable)
            .collect(Collectors.groupingBy(ParkingSpot::getType, Collectors.counting()));
    }
}

// ParkingLot — main facade
public class ParkingLot {
    private final List<ParkingFloor> floors;
    private final SpotFindingStrategy spotFinder;
    private final PricingStrategy pricingStrategy;
    private final Map<TicketId, ParkingTicket> activeTickets = new ConcurrentHashMap<>();

    public ParkingLot(List<ParkingFloor> floors,
                      SpotFindingStrategy spotFinder,
                      PricingStrategy pricingStrategy) {
        this.floors = List.copyOf(floors);
        this.spotFinder = spotFinder;
        this.pricingStrategy = pricingStrategy;
    }

    public ParkingTicket parkVehicle(Vehicle vehicle) {
        List<ParkingSpot> allAvailable = floors.stream()
            .flatMap(floor -> floor.getAvailableSpots(vehicle).stream())
            .toList();

        ParkingSpot spot = spotFinder.findSpot(allAvailable, vehicle)
            .orElseThrow(() -> new ParkingLotFullException(
                "No available spot for " + vehicle.type()));

        boolean reserved = spot.occupy(vehicle);
        if (!reserved) {
            // Race condition — retry once
            return parkVehicle(vehicle);
        }

        ParkingTicket ticket = new ParkingTicket(vehicle, spot);
        activeTickets.put(ticket.getTicketId(), ticket);
        return ticket;
    }

    public ParkingFee exitVehicle(TicketId ticketId) {
        ParkingTicket ticket = Optional.ofNullable(activeTickets.remove(ticketId))
            .orElseThrow(() -> new IllegalArgumentException("Unknown ticket: " + ticketId));

        if (ticket.isClosed()) {
            throw new IllegalStateException("Ticket already processed: " + ticketId);
        }

        ParkingFee fee = pricingStrategy.calculate(ticket);
        ticket.close(fee);
        ticket.getSpot().vacate();
        return fee;
    }

    public Map<SpotType, Long> getAvailability() {
        return floors.stream()
            .flatMap(floor -> floor.getAvailabilityCount().entrySet().stream())
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                Map.Entry::getValue,
                Long::sum));
    }

    public int getActiveTicketCount() {
        return activeTickets.size();
    }
}
```

---

## 8. Observer: Events

```java
// Event types (using sealed + records)
public sealed interface ParkingEvent
    permits ParkingEvent.VehicleParked, ParkingEvent.VehicleExited {

    record VehicleParked(Vehicle vehicle, ParkingSpot spot, ParkingTicket ticket, Instant when)
        implements ParkingEvent {}

    record VehicleExited(Vehicle vehicle, ParkingFee fee, Duration duration, Instant when)
        implements ParkingEvent {}
}

// Listener interface
@FunctionalInterface
public interface ParkingEventListener {
    void onEvent(ParkingEvent event);
}

// Example listeners
public class DisplayBoardListener implements ParkingEventListener {
    public void onEvent(ParkingEvent event) {
        switch (event) {
            case ParkingEvent.VehicleParked p ->
                System.out.println("Spot " + p.spot().getSpotId() + " occupied by " + p.vehicle().licensePlate());
            case ParkingEvent.VehicleExited e ->
                System.out.println("Vehicle exited. Fee: $" + e.fee().amount());
        }
    }
}
```

---

## 9. Pricing Strategy

```java
@FunctionalInterface
public interface PricingStrategy {
    ParkingFee calculate(ParkingTicket ticket);
}

// Hourly pricing
public class HourlyPricingStrategy implements PricingStrategy {
    private final Map<VehicleType, BigDecimal> hourlyRates;

    public HourlyPricingStrategy(Map<VehicleType, BigDecimal> hourlyRates) {
        this.hourlyRates = Map.copyOf(hourlyRates);
    }

    @Override
    public ParkingFee calculate(ParkingTicket ticket) {
        long hours = ticket.getDuration().toHours() + 1; // ceil to nearest hour
        BigDecimal rate = hourlyRates.getOrDefault(
            ticket.getVehicle().type(), BigDecimal.valueOf(5));
        return ParkingFee.of(rate.multiply(BigDecimal.valueOf(hours)).doubleValue());
    }
}

// Flat rate (e.g., events)
public class FlatRatePricingStrategy implements PricingStrategy {
    private final BigDecimal flatRate;

    public FlatRatePricingStrategy(double flatRate) {
        this.flatRate = BigDecimal.valueOf(flatRate);
    }

    @Override
    public ParkingFee calculate(ParkingTicket ticket) {
        return ParkingFee.of(flatRate.doubleValue());
    }
}
```

---

## 10. Complete Example Flow

```java
public class ParkingLotDemo {
    public static void main(String[] args) throws InterruptedException {
        // Build parking lot: 2 floors, 5 spots each
        List<ParkingSpot> floor1Spots = List.of(
            new ParkingSpot("F1-S1", SpotType.COMPACT,  1),
            new ParkingSpot("F1-S2", SpotType.REGULAR,  1),
            new ParkingSpot("F1-S3", SpotType.REGULAR,  1),
            new ParkingSpot("F1-S4", SpotType.LARGE,    1),
            new ParkingSpot("F1-S5", SpotType.LARGE,    1)
        );

        List<ParkingSpot> floor2Spots = List.of(
            new ParkingSpot("F2-S1", SpotType.COMPACT,  2),
            new ParkingSpot("F2-S2", SpotType.REGULAR,  2),
            new ParkingSpot("F2-S3", SpotType.REGULAR,  2),
            new ParkingSpot("F2-S4", SpotType.LARGE,    2),
            new ParkingSpot("F2-S5", SpotType.LARGE,    2)
        );

        ParkingLot lot = new ParkingLot(
            List.of(new ParkingFloor(1, floor1Spots),
                    new ParkingFloor(2, floor2Spots)),
            new OptimalSizeStrategy(),
            new HourlyPricingStrategy(Map.of(
                VehicleType.MOTORCYCLE, BigDecimal.valueOf(2),
                VehicleType.CAR,        BigDecimal.valueOf(5),
                VehicleType.TRUCK,      BigDecimal.valueOf(10)
            ))
        );

        // Vehicles arrive
        Vehicle moto  = new Vehicle("MOTO-001", VehicleType.MOTORCYCLE);
        Vehicle car1  = new Vehicle("CAR-001",  VehicleType.CAR);
        Vehicle truck = new Vehicle("TRUCK-001", VehicleType.TRUCK);

        ParkingTicket motoTicket  = lot.parkVehicle(moto);
        ParkingTicket car1Ticket  = lot.parkVehicle(car1);
        ParkingTicket truckTicket = lot.parkVehicle(truck);

        System.out.println("Availability: " + lot.getAvailability());
        System.out.println("Active tickets: " + lot.getActiveTicketCount());

        // Simulate 2 hours passing
        Thread.sleep(100);

        // Vehicles exit
        ParkingFee fee = lot.exitVehicle(car1Ticket.getTicketId());
        System.out.println("Car fee: $" + fee.amount());

        System.out.println("Availability after car exit: " + lot.getAvailability());
    }
}
```

---

## 11. Design Decisions Explained

| Decision | Why |
|----------|-----|
| `SpotFindingStrategy` interface | OCP — add new algorithms without changing ParkingLot |
| `PricingStrategy` interface | OCP + Strategy pattern — swap pricing models |
| `synchronized` on `occupy()` | Thread safety — prevent two threads parking in same spot |
| `ConcurrentHashMap` for tickets | Thread-safe without locking the whole lot |
| `sealed ParkingEvent` | Exhaustive pattern matching — compiler ensures all events handled |
| `records` for Vehicle, TicketId, ParkingFee | Immutable value objects — no accidental mutation |
| `ParkingFloor` class | SRP — each floor manages its own spots |
| `ParkingLot` as Facade | Simplified API hiding floor/spot complexity |

---

## 12. Interview Q&A

<details>
<summary>💡 Q: How would you make this support multiple parking lots?</summary>

Add a `ParkingLotManager` (or `ParkingNetwork`) that:
1. Holds a `Map<String, ParkingLot>` keyed by location ID
2. Routes vehicle to nearest/least-busy lot using a `LotSelectionStrategy`
3. Centralizes ticket/fee tracking

Each `ParkingLot` remains unchanged — the manager adds orchestration.

</details>

<details>
<summary>💡 Q: How would you add real-time display boards showing availability?</summary>

Use the Observer pattern. `ParkingLot` publishes `ParkingEvent` on every park/exit. `DisplayBoardListener` subscribes and updates:

```java
parkingLot.addListener(new DisplayBoardListener(displayBoard));
```

For real-time distributed updates: publish events to Kafka topic `parking-events`. Display service consumes and updates Redis cache → UI polls Redis.

</details>

<details>
<summary>💡 Q: What if the same vehicle tries to park twice?</summary>

Add validation in `parkVehicle()`:
```java
boolean alreadyParked = activeTickets.values().stream()
    .anyMatch(t -> t.getVehicle().licensePlate().equals(vehicle.licensePlate()));
if (alreadyParked) throw new VehicleAlreadyParkedException(vehicle.licensePlate());
```

For performance at scale, maintain a `Set<String> parkedLicensePlates` alongside `activeTickets`.

</details>

<details>
<summary>💡 Q: How does your design handle concurrent entries?</summary>

Two-level thread safety:
1. `ParkingSpot.occupy()` is `synchronized` — only one thread can reserve a spot
2. `ConcurrentHashMap` for `activeTickets` — thread-safe ticket storage
3. Retry in `parkVehicle()` handles rare CAS failures

For high-throughput: use `ReentrantLock` per floor (reduces contention vs one global lock).

</details>

---

*Previous: [← OOP Design Principles](./01_OOP_Design_Principles.md) | Next: [Design Patterns →](../DesignPattern/README.md)*
