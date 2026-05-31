# 💰 Design a Digital Wallet (Payment System)

> *"Every time you tap your phone to pay, transfer money to a friend, or pay a bill online — a digital wallet system ensures the right amount moves from point A to point B, exactly ONCE, with full auditability. Financial systems have ZERO tolerance for errors: a lost transaction means lost money, a duplicate means you charged someone twice. This is where distributed systems meets accounting meets regulatory compliance."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [ACID](../Database/ACID.md), [Idempotency](../APIs/Idempotency.md), [Distributed Locking](../Microservices/DistributedLocking.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Core Concepts](#-core-concepts)
3. [High-Level Architecture](#-high-level-architecture)
4. [Double-Entry Bookkeeping](#-double-entry-bookkeeping)
5. [Transaction Processing](#-transaction-processing)
6. [Idempotency & Exactly-Once](#-idempotency--exactly-once)
7. [Scalability & Partitioning](#-scalability--partitioning)
8. [Java Implementation](#-java-implementation)
9. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Create wallet (user registration → wallet with $0 balance)
  • Add money (bank transfer, card, external deposit)
  • Send money (peer-to-peer transfer!)
  • Pay merchant (purchase goods/services)
  • Transaction history (full audit trail!)
  • Balance inquiry (real-time, ACCURATE!)
  • Refunds & reversals
  • Multi-currency support
  
NON-FUNCTIONAL:
  • Consistency: NEVER lose money! Balance always correct!
  • Idempotency: network retry must NOT double-charge!
  • Availability: 99.99% (payment failures = lost revenue!)
  • Latency: < 500ms for transfers
  • Throughput: 10K transactions per second
  • Auditability: every cent traceable (regulatory!)
  • Security: PCI DSS compliance, fraud detection

INVARIANTS (must NEVER be violated!):
  • Sum of all wallet balances = sum of all deposits - sum of all withdrawals
  • No wallet balance goes negative (unless credit line!)
  • Every transaction has exactly two sides (double-entry!)
  • Completed transactions are IMMUTABLE (append-only ledger!)
```

---

## 🧱 Core Concepts

```
DOUBLE-ENTRY BOOKKEEPING:
  Every transaction has TWO entries:
  • DEBIT: money leaves one account
  • CREDIT: money enters another account
  
  Transfer $50 from Alice to Bob:
    Debit:  Alice's wallet  -$50
    Credit: Bob's wallet    +$50
    Net change to system: $0 ✅ (money neither created nor destroyed!)
  
  This is the FUNDAMENTAL principle of all financial systems!
  If debits ≠ credits → something is WRONG! (instant detection!)

LEDGER (Immutable Transaction Log):
  NEVER update balances directly! Instead:
  • Record every transaction in an immutable ledger
  • Balance = SUM of all credits - SUM of all debits for that account!
  
  Ledger:
  ┌──────────────────────────────────────────────────────────────┐
  │  TxID  │  From     │  To      │  Amount  │  Time            │
  ├──────────────────────────────────────────────────────────────┤
  │  TX001 │  SYSTEM   │  Alice   │  $100    │  2024-01-01 10:00│
  │  TX002 │  Alice    │  Bob     │  $50     │  2024-01-01 11:00│
  │  TX003 │  SYSTEM   │  Bob     │  $30     │  2024-01-01 12:00│
  └──────────────────────────────────────────────────────────────┘
  
  Alice's balance: +$100 - $50 = $50 ✅
  Bob's balance: +$50 + $30 = $80 ✅
  
  WHY IMMUTABLE?
  • Auditability (regulators can trace every cent!)
  • No "mysterious" balance changes
  • Easy reconciliation (check: total in = total out!)
  • Temporal queries ("what was balance at 3pm yesterday?")

ACCOUNT TYPES:
  • User Wallet: end-user's balance
  • System Account: for deposits/withdrawals (money in/out of system)
  • Merchant Account: business receiving payments
  • Fee Account: where transaction fees accumulate
  • Escrow Account: holds money during disputes/pending transactions
```

---

## 🏗️ High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        DIGITAL WALLET SYSTEM                              │
│                                                                           │
│  ┌─────────┐    ┌──────────────────────────────────────────────────┐    │
│  │  Client  │───►│  API Gateway (auth, rate limit, idempotency!)   │    │
│  └─────────┘    └──────┬──────────────────┬────────────────────────┘    │
│                         │                  │                              │
│           ┌─────────────▼────┐   ┌────────▼──────────┐                  │
│           │  Wallet Service   │   │  Transaction Svc  │                  │
│           │  • Balance query  │   │  • Transfer        │                  │
│           │  • Create wallet  │   │  • Deposit         │                  │
│           │                   │   │  • Withdraw        │                  │
│           └────────┬──────────┘   └────────┬──────────┘                  │
│                    │                        │                              │
│                    │              ┌─────────▼──────────────┐              │
│                    │              │  Ledger Service         │              │
│                    │              │  • Double-entry         │              │
│                    │              │  • Immutable log        │              │
│                    │              │  • Balance computation  │              │
│                    │              └─────────┬──────────────┘              │
│                    │                        │                              │
│           ┌────────▼────────────────────────▼──────────────┐             │
│           │  PostgreSQL (SERIALIZABLE transactions!)        │             │
│           │  Tables: accounts, ledger_entries, transactions │             │
│           │  Sharded by account_id!                         │             │
│           └────────────────────────────────────────────────┘             │
│                                                                           │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────┐          │
│  │  Idempotency   │  │  Fraud         │  │  Notification    │          │
│  │  Store (Redis)  │  │  Detection     │  │  Service         │          │
│  │  (dedup keys!)  │  │  (ML models!)  │  │  (email, push)   │          │
│  └────────────────┘  └────────────────┘  └──────────────────┘          │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Event Bus (Kafka) — transaction events for analytics, compliance  │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 📒 Double-Entry Bookkeeping

```
EVERY TRANSACTION CREATES TWO LEDGER ENTRIES:

  Transfer $50: Alice → Bob
  
  ┌──────────────────────────────────────────────────────────────────┐
  │  Entry 1 (DEBIT):                                                │
  │    account_id: alice_wallet                                      │
  │    amount: -50.00                                                │
  │    tx_id: TX-123                                                 │
  │    type: TRANSFER_OUT                                            │
  │    counterparty: bob_wallet                                      │
  │    timestamp: 2024-01-15T10:30:00Z                               │
  │                                                                  │
  │  Entry 2 (CREDIT):                                               │
  │    account_id: bob_wallet                                        │
  │    amount: +50.00                                                │
  │    tx_id: TX-123                                                 │
  │    type: TRANSFER_IN                                             │
  │    counterparty: alice_wallet                                    │
  │    timestamp: 2024-01-15T10:30:00Z                               │
  └──────────────────────────────────────────────────────────────────┘
  
  VERIFICATION: For EVERY transaction:
  SUM(all entries for TX-123) = (-50) + (+50) = 0 ✅
  
  BALANCE CALCULATION:
  Alice's balance = SUM(amount) WHERE account_id = 'alice_wallet'
  = +100 (deposit) + (-50) (transfer) = $50 ✅

WITH FEES:
  Transfer $50 Alice → Bob, fee = $1
  
  Entry 1: alice_wallet    -$51  (amount + fee!)
  Entry 2: bob_wallet      +$50
  Entry 3: fee_account     +$1
  
  Sum: -51 + 50 + 1 = 0 ✅ (system balanced!)

BALANCE OPTIMIZATION:
  Computing SUM(all entries) for every balance check is SLOW!
  
  Solution: Materialized balance (cached, updated on each tx!)
  
  accounts table:
  | account_id   | balance | last_updated     |
  | alice_wallet | 50.00   | 2024-01-15 10:30 |
  | bob_wallet   | 80.00   | 2024-01-15 10:30 |
  
  Update balance atomically within the SAME transaction as ledger entry!
  Periodically: verify balance = SUM(ledger entries) (reconciliation!)
```

---

## ⚡ Transaction Processing

```
TRANSFER FLOW (the critical path!):

  Alice sends $50 to Bob:
  
  1. REQUEST VALIDATION:
     • Alice exists? ✅
     • Bob exists? ✅
     • Amount > 0? ✅
     • Alice's balance >= 50? ✅ (read current balance!)
     
  2. FRAUD CHECK:
     • Unusual amount? Transaction velocity? New device?
     • If suspicious: HOLD for review! (don't process!)
  
  3. EXECUTE TRANSACTION (SINGLE DB TRANSACTION!):
     BEGIN;
     
     -- Check balance again (inside transaction! No race conditions!)
     SELECT balance FROM accounts WHERE id = 'alice' FOR UPDATE;
     -- balance = 100, OK to proceed!
     
     -- Debit Alice
     UPDATE accounts SET balance = balance - 50 WHERE id = 'alice';
     
     -- Credit Bob  
     UPDATE accounts SET balance = balance + 50 WHERE id = 'bob';
     
     -- Record in ledger (immutable!)
     INSERT INTO ledger_entries (tx_id, account_id, amount, ...) VALUES
       ('TX-123', 'alice', -50, ...),
       ('TX-123', 'bob', +50, ...);
     
     -- Record transaction
     INSERT INTO transactions (id, from, to, amount, status) VALUES
       ('TX-123', 'alice', 'bob', 50, 'COMPLETED');
     
     COMMIT;
  
  4. POST-PROCESSING (async!):
     • Send notifications (push, email)
     • Publish event to Kafka (analytics!)
     • Update spending stats
  
WHAT IF ALICE AND BOB ARE ON DIFFERENT SHARDS?!
  Can't do single DB transaction across shards!
  
  Solution: TWO-PHASE approach:
  1. Debit Alice (Shard A): alice.balance -= 50, status = PENDING
  2. Credit Bob (Shard B): bob.balance += 50, status = PENDING
  3. Confirm both: status = COMPLETED
  
  If step 2 fails: rollback step 1 (alice.balance += 50!)
  
  Or: Saga pattern with compensation!
  Or: Route Alice & Bob to same shard! (hot accounts → same partition!)
```

---

## 🔒 Idempotency & Exactly-Once

```
THE MOST CRITICAL REQUIREMENT FOR PAYMENT SYSTEMS!

  Scenario: Client sends "Transfer $50" → network timeout!
  Did it go through? Client retries...
  WITHOUT IDEMPOTENCY: $50 transferred TWICE! 💀💀💀

IDEMPOTENCY KEY:
  Client generates unique key per intended transaction:
  POST /transfer
  Headers: Idempotency-Key: "client-abc-transfer-20240115-001"
  Body: {from: "alice", to: "bob", amount: 50}

SERVER LOGIC:
  1. Check: has this idempotency key been processed before?
     → Redis: GET "idempotent:client-abc-transfer-20240115-001"
     
  2. YES (already processed): Return SAME previous result! (no re-execution!)
  
  3. NO (first time):
     → Process transaction!
     → Store result in Redis: 
       SET "idempotent:client-abc-transfer-20240115-001" "{result}" EX 86400
     → Return result to client!

  Retry with SAME key → same result! No duplicate charges! ✅

IMPLEMENTATION DETAIL:
  Race condition: two retries arrive simultaneously!
  
  Solution: Redis SETNX (Set If Not Exists) as a lock!
  
  if (redis.setIfAbsent(idempotencyKey, "PROCESSING", 30s)) {
      // First one wins! Process transaction.
      result = processTransfer(...);
      redis.set(idempotencyKey, serialize(result), 24h);
  } else {
      // Already being processed or was processed!
      while (redis.get(idempotencyKey).equals("PROCESSING")) {
          Thread.sleep(50); // Wait for first one to finish!
      }
      return deserialize(redis.get(idempotencyKey)); // Return cached result!
  }
```

---

## 📊 Scalability & Partitioning

```
SCALING A PAYMENT SYSTEM:

CHALLENGE: Can't easily shard because transfers span two accounts!
  Alice (Shard 1) → Bob (Shard 2) = distributed transaction! 😰

SHARDING STRATEGIES:
  
  1. SHARD BY ACCOUNT ID (most common!):
     Shard 1: accounts A-M
     Shard 2: accounts N-Z
     
     Same-shard transfers: single transaction! ✅
     Cross-shard transfers: Saga pattern! 😐
     
  2. HOT ACCOUNT HANDLING:
     Merchant "Amazon" receives 10K payments/second!
     All hit same shard → bottleneck!
     
     Solution: Split hot accounts into sub-accounts!
     amazon_001, amazon_002, ... amazon_100 (100 shards!)
     Route: hash(transaction_id) % 100 → sub-account!
     Periodically: sweep sub-accounts → main account!
     
  3. EVENT SOURCING:
     Don't update balances in real-time for hot accounts!
     Append events → compute balance from event stream!
     Can process events in parallel (per account partition!)

READ SCALING:
  Balance queries: 99% of requests!
  Solution: Read replicas + caching!
  
  But: balance MUST be accurate after a write!
  After transfer: invalidate cache, read from primary!
  "Read-your-own-writes" consistency!
```

---

## 💻 Java Implementation

### Transfer Service

```java
@Service
public class TransferService {
    
    @Autowired private AccountRepository accountRepo;
    @Autowired private LedgerRepository ledgerRepo;
    @Autowired private TransactionRepository txRepo;
    @Autowired private IdempotencyService idempotencyService;
    @Autowired private FraudDetectionService fraudService;
    @Autowired private KafkaTemplate<String, TransactionEvent> kafka;
    
    /**
     * Execute money transfer with full safety guarantees:
     * - Idempotency (no double-processing!)
     * - ACID (all-or-nothing!)
     * - Double-entry (balanced ledger!)
     * - Fraud check
     */
    public TransferResult transfer(TransferRequest request) {
        // 1. Idempotency check!
        String idempotencyKey = request.getIdempotencyKey();
        Optional<TransferResult> cached = idempotencyService.get(idempotencyKey);
        if (cached.isPresent()) {
            return cached.get(); // Already processed! Return same result.
        }
        
        // 2. Acquire idempotency lock (prevent concurrent duplicates!)
        if (!idempotencyService.tryLock(idempotencyKey)) {
            throw new ConcurrentRequestException("Request already in progress");
        }
        
        try {
            // 3. Fraud check
            FraudResult fraud = fraudService.check(request);
            if (fraud.isSuspicious()) {
                TransferResult held = TransferResult.held("Under review");
                idempotencyService.store(idempotencyKey, held);
                return held;
            }
            
            // 4. Execute the transfer (ACID transaction!)
            TransferResult result = executeTransfer(request);
            
            // 5. Store idempotency result
            idempotencyService.store(idempotencyKey, result);
            
            // 6. Publish event (async — after commit!)
            if (result.isSuccess()) {
                kafka.send("transactions", request.getFromAccount(),
                    new TransactionEvent("COMPLETED", result.getTransactionId()));
            }
            
            return result;
            
        } finally {
            idempotencyService.releaseLock(idempotencyKey);
        }
    }
    
    @Transactional(isolation = Isolation.SERIALIZABLE)
    private TransferResult executeTransfer(TransferRequest request) {
        String fromId = request.getFromAccount();
        String toId = request.getToAccount();
        BigDecimal amount = request.getAmount();
        
        // Lock accounts in consistent order (prevent deadlock!)
        String first = fromId.compareTo(toId) < 0 ? fromId : toId;
        String second = fromId.compareTo(toId) < 0 ? toId : fromId;
        
        Account firstAccount = accountRepo.findByIdForUpdate(first);
        Account secondAccount = accountRepo.findByIdForUpdate(second);
        
        Account from = fromId.equals(first) ? firstAccount : secondAccount;
        Account to = fromId.equals(first) ? secondAccount : firstAccount;
        
        // Sufficient balance check!
        if (from.getBalance().compareTo(amount) < 0) {
            return TransferResult.failed("Insufficient balance");
        }
        
        // Update balances!
        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
        accountRepo.save(from);
        accountRepo.save(to);
        
        // Record in immutable ledger (double-entry!)
        String txId = UUID.randomUUID().toString();
        ledgerRepo.save(new LedgerEntry(txId, fromId, amount.negate(), "TRANSFER_OUT"));
        ledgerRepo.save(new LedgerEntry(txId, toId, amount, "TRANSFER_IN"));
        
        // Record transaction
        Transaction tx = new Transaction(txId, fromId, toId, amount, 
            TransactionStatus.COMPLETED);
        txRepo.save(tx);
        
        return TransferResult.success(txId);
    }
}
```

### Idempotency Service

```java
@Service
public class IdempotencyService {
    
    @Autowired private RedisTemplate<String, String> redis;
    private final ObjectMapper mapper = new ObjectMapper();
    
    private static final Duration LOCK_TTL = Duration.ofSeconds(30);
    private static final Duration RESULT_TTL = Duration.ofHours(24);
    
    public boolean tryLock(String key) {
        return Boolean.TRUE.equals(
            redis.opsForValue().setIfAbsent(
                "lock:" + key, "PROCESSING", LOCK_TTL));
    }
    
    public void releaseLock(String key) {
        redis.delete("lock:" + key);
    }
    
    public Optional<TransferResult> get(String key) {
        String stored = redis.opsForValue().get("result:" + key);
        if (stored == null) return Optional.empty();
        return Optional.of(mapper.readValue(stored, TransferResult.class));
    }
    
    public void store(String key, TransferResult result) {
        redis.opsForValue().set("result:" + key, 
            mapper.writeValueAsString(result), RESULT_TTL);
    }
}
```

---

## ❓ Interview Q&A

**Q1: Why use double-entry bookkeeping in a digital wallet?**
> Three critical reasons: (1) Error detection — if sum of all entries for a transaction ≠ 0, something is WRONG (instant detection, not discovered weeks later!), (2) Auditability — every cent is traceable, regulators can verify the complete money flow at any point in time, (3) Immutability — the ledger is append-only, no one can "silently" change a balance (corrections are recorded as new entries: reversals!). This is why every bank, payment processor, and fintech uses double-entry — it's been proven for 700+ years of accounting!

**Q2: How do you handle a transfer between accounts on different database shards?**
> Saga pattern: (1) Debit source account on Shard A (status: PENDING), (2) Credit destination on Shard B (status: PENDING), (3) Confirm both (status: COMPLETED). If step 2 fails: compensate step 1 (re-credit source). Alternative: use a single-shard "holding" account — transfer to holding (same shard as source!), then transfer from holding to destination (same shard as dest!). Two single-shard transactions are simpler than one cross-shard! Key: the holding account is on a "hot shard" optimized for high throughput.

**Q3: How do you prevent double-spending in a concurrent system?**
> SELECT ... FOR UPDATE (pessimistic lock) on the account row within a SERIALIZABLE transaction. This ensures: (1) Balance check and deduction are atomic, (2) Two concurrent transfers from same account serialize properly. If Alice has $100 and two $80 transfers hit simultaneously: first one succeeds (balance → $20), second sees $20 < $80 → rejected! The database lock prevents both from reading $100 and both deducting. For higher throughput: optimistic locking with version field + retry on conflict.

**Q4: How do you ensure exactly-once processing for payment transactions?**
> Three layers: (1) Client-side: idempotency key (unique per intended action, sent with every retry), (2) Server-side: check Redis for existing result before processing, use SETNX as lock against concurrent duplicates, (3) Database: unique constraint on (idempotency_key) in transactions table (final safety net!). Even if Redis fails, the DB constraint catches duplicates. The combination makes double-processing virtually impossible. Additionally: use database transactions (ACID) so partial failures don't corrupt state.

---

## 🔗 Related Topics
- [ACID Transactions](../Database/ACID.md) — Why financial systems need ACID
- [Idempotency](../APIs/Idempotency.md) — Preventing double-processing
- [Distributed Locking](../Microservices/DistributedLocking.md) — Cross-shard safety
- [Event-Driven Architecture](../Architectures/Event_Driven.md) — Async processing
- [Design Ticket Booking](DesignTicketBooking.md) — Similar reservation patterns

---

*"In a payment system, a bug that loses $0.01 per transaction costs you millions at scale. A bug that creates $0.01 per transaction makes your books wrong and gets you fined by regulators. The only acceptable outcome is ZERO error — which is why payment systems are the most carefully designed, most thoroughly tested systems in all of software engineering." — Fintech Architect* 💰
