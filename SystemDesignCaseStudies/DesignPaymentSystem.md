# 💳 Design a Payment System
## ACID Guarantees, Idempotency, and Moving Money Reliably at Scale

> *"Payment systems are where distributed systems theory meets real-world consequence. A bug that causes double-charging a user or losing a transaction isn't a latency spike — it's a lawsuit."*

**⏱️ Estimated Time**: 60 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [How to Approach System Design](./How_To_Approach_System_Design.md), [ACID Transactions](../Database/ACID.md)

---

## 📋 Table of Contents
1. [Requirements (RADIO: R)](#-requirements)
2. [API Design (RADIO: A)](#-api-design)
3. [Data Model (RADIO: D)](#-data-model)
4. [Infrastructure & Architecture (RADIO: I)](#-infrastructure--architecture)
5. [Optimization (RADIO: O)](#-optimization)
6. [Mini Challenge](#-mini-challenge)
7. [Interview Q&A](#-interview-qa)

---

## 📋 Requirements

### Functional Requirements (Core)

| Feature | Description |
|---|---|
| **Initiate payment** | User initiates a payment from their wallet/card to a recipient |
| **Process payment** | Charge the source (card/bank), credit the destination |
| **Payment status** | Query the status of a payment (pending, completed, failed) |
| **Transaction history** | Retrieve transaction history for a user |
| **Refund** | Reverse a completed payment fully or partially |
| **Idempotency** | Retry-safe operations — charging twice must be prevented |

### Non-Functional Requirements (Scale)

```
Scale targets:
  - 1 million transactions per day (Stripe-like scale: ~12 TPS average, 100 TPS peak)
  - 99.99% availability (< 52 minutes downtime/year) — money CANNOT disappear
  - Transaction durability: zero data loss — every committed transaction is permanent
  - Latency: P99 < 3 seconds end-to-end (card network + processing)
  - Consistency: STRONG — no money created or destroyed (double-entry bookkeeping)
  - Compliance: PCI DSS Level 1, SOC 2, regional regulations (PSD2, RBI)

Critical constraints:
  - Exactly-once payment processing (no double charges)
  - Atomic money movement (debit + credit must both succeed or both fail)
  - Complete audit trail for every state change
  - Reconciliation: our records must match bank/card network records
```

---

## 📡 API Design

### REST Endpoints

```
POST /v1/payments
  Request:
    {
      "idempotency_key": "order-7839-attempt-1",   // client-generated UUID
      "amount": 2999,                               // cents (avoid float arithmetic)
      "currency": "USD",
      "payment_method_id": "pm_card_abc123",        // tokenized card reference
      "description": "Order #7839",
      "metadata": { "order_id": "7839" }
    }
  Response 200 (success) | 202 (async, pending) | 400 (invalid) | 402 (payment failed):
    {
      "payment_id": "pay_xyz789",
      "status": "succeeded",                        // pending | processing | succeeded | failed
      "amount": 2999,
      "currency": "USD",
      "created_at": "2026-05-17T10:30:00Z",
      "receipt_url": "https://payments.co/receipts/pay_xyz789"
    }

GET /v1/payments/{payment_id}
  Response: Payment object with current status

POST /v1/payments/{payment_id}/refund
  Request: { "amount": 2999, "reason": "customer_request" }
  Response: Refund object

GET /v1/payments?user_id=u123&limit=20&cursor=<opaque>
  Response: Paginated payment history

POST /v1/payment-methods
  Request: { "card": { "number": "...", "exp_month": 12, "exp_year": 2027, "cvc": "123" } }
  Response: { "payment_method_id": "pm_card_abc123", "last4": "4242" }
  Note: Raw card data NEVER stored — tokenized immediately and discarded
```

### Idempotency Key Semantics

```
Client sends: POST /v1/payments
  Headers: Idempotency-Key: order-7839-attempt-1

First call:
  Server processes payment → SUCCESS → stores result under "order-7839-attempt-1"
  Response: { "payment_id": "pay_xyz", "status": "succeeded" }

Network timeout — client retries with SAME idempotency key:
  Server finds "order-7839-attempt-1" in idempotency store
  Returns SAME response WITHOUT processing payment again
  Response: { "payment_id": "pay_xyz", "status": "succeeded" }

Same request, different idempotency key (client bug — generates new key on retry):
  Server treats as NEW payment request
  Risk: DOUBLE CHARGE ← this is why clients MUST reuse idempotency keys on retry
```

---

## 🗄️ Data Model

### Database Selection

```
Why PostgreSQL for payments (not NoSQL):
  - ACID transactions are mandatory (atomically debit + credit)
  - Complex joins (payment → payment_method → user → merchant)
  - Financial queries require SQL aggregations and reporting
  - Row-level locking prevents race conditions
  - Foreign key constraints enforce referential integrity
  
Why NOT Cassandra/DynamoDB:
  - No multi-row ACID transactions (cannot atomically update two accounts)
  - Eventual consistency is unacceptable — money cannot be "eventually" moved
  - AP systems (Cassandra) sacrifice consistency we legally require
```

### Core Schema

```sql
-- Accounts / Wallets (double-entry bookkeeping)
CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    account_type    VARCHAR(20) NOT NULL,   -- 'wallet', 'escrow', 'merchant'
    currency        CHAR(3) NOT NULL,        -- 'USD', 'EUR'
    balance         BIGINT NOT NULL DEFAULT 0  CHECK (balance >= 0),  -- cents
    version         BIGINT NOT NULL DEFAULT 0, -- optimistic locking
    created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Payments (the intent/request)
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    idempotency_key VARCHAR(255) UNIQUE NOT NULL,  -- prevents duplicate processing
    payer_id        UUID NOT NULL REFERENCES users(id),
    payee_id        UUID NOT NULL REFERENCES users(id),
    amount          BIGINT NOT NULL CHECK (amount > 0),
    currency        CHAR(3) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- 'pending' → 'processing' → 'succeeded' | 'failed' | 'refunded'
    payment_method_id UUID REFERENCES payment_methods(id),
    external_txn_id VARCHAR(255),  -- Stripe/bank reference ID
    failure_reason  TEXT,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Ledger entries (double-entry: every payment creates 2 entries)
CREATE TABLE ledger_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id      UUID NOT NULL REFERENCES payments(id),
    account_id      UUID NOT NULL REFERENCES accounts(id),
    entry_type      VARCHAR(10) NOT NULL,   -- 'debit' or 'credit'
    amount          BIGINT NOT NULL CHECK (amount > 0),
    balance_after   BIGINT NOT NULL,        -- account balance after this entry
    created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);
-- Invariant: SUM(credits) == SUM(debits) across all ledger_entries at all times

-- Idempotency store (short-lived: 24h TTL, then purged)
CREATE TABLE idempotency_keys (
    key             VARCHAR(255) PRIMARY KEY,
    payment_id      UUID REFERENCES payments(id),
    response_body   JSONB NOT NULL,         -- cached response to return on retry
    expires_at      TIMESTAMP NOT NULL      -- purge after 24h
);

-- Payment Methods (tokenized — never store raw card data)
CREATE TABLE payment_methods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    type            VARCHAR(20) NOT NULL,   -- 'card', 'bank_account', 'wallet'
    processor_token VARCHAR(255) NOT NULL,  -- Stripe/Braintree token (not raw card)
    last4           CHAR(4),
    exp_month       SMALLINT,
    exp_year        SMALLINT,
    is_default      BOOLEAN DEFAULT false
);

-- Indexes for common queries
CREATE INDEX idx_payments_payer_id ON payments(payer_id, created_at DESC);
CREATE INDEX idx_payments_status ON payments(status) WHERE status = 'pending';
CREATE INDEX idx_idempotency_expires ON idempotency_keys(expires_at);
```

### Double-Entry Bookkeeping

```
A payment of $30 from Alice to Bob creates TWO ledger entries:

payments row:
  id=pay_xyz, payer_id=alice, payee_id=bob, amount=3000, status=succeeded

ledger_entries rows:
  { payment_id=pay_xyz, account_id=alice_wallet, entry_type=DEBIT,  amount=3000, balance_after=7000 }
  { payment_id=pay_xyz, account_id=bob_wallet,   entry_type=CREDIT, amount=3000, balance_after=8000 }

Key invariant: for every payment, the sum of debits == sum of credits
               Money is NEVER created or destroyed — only moved
               Audit: reconstruct any account balance at any point in time
               by replaying all ledger entries in order (event sourcing!)
```

---

## 🏗️ Infrastructure & Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PAYMENT SYSTEM ARCHITECTURE                      │
│                                                                     │
│  Client App                                                         │
│     │                                                               │
│     ▼                                                               │
│  [API Gateway]  ← rate limit, auth, TLS termination                 │
│     │                                                               │
│     ▼                                                               │
│  [Payment Service]                                                  │
│     │ 1. Check idempotency key (Redis cache → PostgreSQL fallback)  │
│     │ 2. Validate payment method                                    │
│     │ 3. Reserve funds (optimistic locking on account.balance)      │
│     │ 4. Call card processor (Stripe/Adyen) [async, webhook back]   │
│     │ 5. Write ledger entries (atomic DB transaction)               │
│     │ 6. Publish payment.completed event                            │
│     │                                                               │
│  ┌──▼──────────────┐    ┌─────────────────┐    ┌───────────────┐   │
│  │   PostgreSQL    │    │ Redis (idempot.) │    │     Kafka     │   │
│  │  (accounts,     │    │  key cache,      │    │ (downstream   │   │
│  │   ledger,       │    │  session,        │    │  events:      │   │
│  │   payments)     │    │  rate limit)     │    │  notif, fraud)│   │
│  └─────────────────┘    └─────────────────┘    └───────────────┘   │
│                                                                     │
│  [Stripe/Adyen]  ← external card processor                         │
│     │ webhook: payment_intent.succeeded / payment_failed           │
│     ▼                                                               │
│  [Webhook Handler Service] → updates payment status → Kafka event  │
│                                                                     │
│  [Fraud Detection Service] ← async, from Kafka                     │
│  [Notification Service]    ← async, from Kafka                     │
│  [Reconciliation Service]  ← daily batch, compares our records     │
│                              against bank statements                │
└─────────────────────────────────────────────────────────────────────┘
```

### Payment Processing Flow (Detailed)

```
Step 1: API Gateway → Payment Service
  - Validate JWT, rate limit, TLS
  - Extract Idempotency-Key header

Step 2: Idempotency Check (before ANY processing)
  - Lookup idempotency_key in Redis (< 1ms)
  - If found: return cached response (no processing — safe retry)
  - If not found: continue to step 3

Step 3: Input Validation
  - Amount > 0 and <= maximum allowed
  - Currency is supported
  - Payment method exists and belongs to the payer
  - Payer account exists and is not frozen

Step 4: Fund Reservation (Optimistic Locking)
  BEGIN TRANSACTION
    SELECT balance, version FROM accounts WHERE id = payer_id FOR UPDATE
    IF balance < amount → ROLLBACK → return insufficient_funds error
    UPDATE accounts SET balance = balance - amount,
                        version = version + 1
           WHERE id = payer_id AND version = <expected_version>
    IF rows_affected = 0 → concurrent update detected → retry
    INSERT INTO payments (id, idempotency_key, status='processing', ...)
    INSERT INTO idempotency_keys (key, payment_id, response_body, expires_at)
  COMMIT

Step 5: External Processor Call (Stripe/Adyen)
  POST https://api.stripe.com/v1/payment_intents
  → Returns: payment_intent_id, requires_action, error

  IMPORTANT: External call is OUTSIDE the DB transaction (network calls in transactions
             hold locks and cause timeouts). Payment is in 'processing' state in DB.

Step 6: Handle Processor Response
  On success: webhook from Stripe confirms charge
  → BEGIN TRANSACTION
       UPDATE payments SET status='succeeded', external_txn_id=...
       UPDATE accounts SET balance = balance + amount WHERE id = payee_id
       INSERT INTO ledger_entries (debit from payer, credit to payee)
    COMMIT
  On failure:
  → BEGIN TRANSACTION
       UPDATE payments SET status='failed', failure_reason=...
       UPDATE accounts SET balance = balance + amount WHERE id = payer_id  -- refund reservation
    COMMIT
```

---

## 📈 Optimization

### Bottleneck: High-Traffic Write Contention

```
Problem: At peak (100 TPS), all payments to popular merchants hit the same
         merchant account row → row-level lock contention → serialized writes.

Solution 1: Write-ahead accumulation
  Don't update merchant balance on every payment.
  Accumulate in a staging table, batch-apply every minute.
  Eventual balance, but individual transactions still recorded accurately.

Solution 2: Account sharding
  Merchant account split into N sub-accounts (shards)
  Payments distributed across shards (hash on payment_id % N)
  Balance = SUM of all shard balances
  N × more write throughput, N × less contention

Solution 3: CQRS for balance queries
  Write path: ledger_entries (append-only, high throughput)
  Read path: materialized account balance (pre-computed, refreshed asynchronously)
  Never compute balance at query time from full ledger scan
```

### Bottleneck: PostgreSQL Write Throughput Limits

```
Single PostgreSQL primary can handle ~10K TPS.
At 100 TPS peak, we're well within limits. For Stripe-scale (50K TPS):

1. Read replicas (most queries are reads — status checks, history)
   Route all SELECT queries to read replicas
   Primary handles only writes (INSERTs/UPDATEs)

2. Connection pooling (PgBouncer)
   App servers can hold thousands of connections.
   PostgreSQL handles ~300 effectively.
   PgBouncer multiplexes 10K app connections to 300 DB connections.

3. Partitioning
   Partition payments table by created_at (monthly)
   Hot partition: current month (all writes)
   Cold partitions: history (reads only, can be archived)

4. Separate ledger DB
   Ledger entries are append-only — never updated, only inserted.
   Can use a separate append-optimized database or table partition.
```

### Reconciliation

```
Problem: Our system says 1,247 payments succeeded today.
         Bank statement shows $47,832.50 credited.
         Do these match? How do we know if money was lost or double-credited?

Reconciliation service (runs nightly):
  1. Fetch all transactions from external processor (Stripe export)
  2. Compare against our payment records (payment.external_txn_id)
  3. Identify mismatches:
     - In Stripe, not in our DB: we missed a webhook → reconcile manually
     - In our DB as succeeded, but failed in Stripe: refund automatically
     - Amount mismatch: alert for manual review
  4. Generate reconciliation report

At Stripe/PayPal scale: reconciliation is real-time, not nightly.
```

---

## 🧩 Mini Challenge

**Scenario**: A client sends a payment request, receives a network timeout (no response), and retries. Your payment service has already charged the card (Step 5 succeeded) but the response was lost. The client retries with a NEW idempotency key (they have a bug).

**What happens, and how do you prevent the double charge at the system design level?**

<details>
<summary>💡 Click to reveal answer</summary>

**What happens with the bug**:
The client generates a new idempotency key on retry (`order-7839-attempt-2` instead of `order-7839-attempt-1`). Your server sees a new idempotency key → treats it as a new payment → charges the card a second time. **Double charge occurs.**

**Prevention strategies**:

1. **Client-side (primary prevention)**: Document clearly that clients MUST reuse the idempotency key on retries. Use exponential backoff with jitter. Most payment SDKs (Stripe, Braintree) handle this automatically. Educate integrators.

2. **Server-side deduplication (secondary prevention)**: Index on `(payer_id, payee_id, amount, created_at window)`. If a payment with matching fields exists within a 60-second window, return the existing payment with a `X-Idempotency-Replay: true` header. This catches the common case of duplicate requests without idempotency keys.

3. **Processor-level deduplication**: Stripe and Adyen have their own idempotency mechanisms. Pass a deterministic key to the card processor based on your payment ID: `stripe_idempotency_key = "internal_payment_" + payment.id`. Even if you call Stripe twice, Stripe returns the same charge if the key matches.

4. **Automated refund detection**: The reconciliation service detects a payer charged twice in a short window for the same amount/merchant. Auto-trigger refund if pattern matches defined rules, and alert the on-call engineer.

5. **Compensating transaction**: If the double charge is detected post-fact, automatically initiate a refund for the duplicate payment and notify the user with an explanation and apology credit.

</details>

---

## 📝 Interview Q&A

**Q: How do you ensure a payment is processed exactly once?**
> A: Three layers: (1) **Idempotency keys** — clients generate a unique key per payment attempt; the server stores this key with the result and returns the cached result on retry without reprocessing. (2) **Optimistic locking** on the account balance — the UPDATE fails with zero rows affected if a concurrent transaction has already modified the account, triggering a retry with fresh state. (3) **External processor idempotency** — pass a deterministic key to Stripe/Adyen based on our internal payment ID; even if we call them twice, they deduplicate.

**Q: Why is double-entry bookkeeping important in a payment system?**
> A: Double-entry bookkeeping records every transaction as both a debit from one account and a credit to another. The invariant is that total debits always equal total credits — money is never created or destroyed. This provides: (1) **auditability** — reconstruct any account's balance at any point in time by replaying ledger entries; (2) **consistency checking** — if debits ≠ credits, there's a bug; (3) **regulatory compliance** — required for financial systems; (4) **reconciliation** — compare our ledger against bank statements to detect discrepancies.

**Q: How do you handle the "money in flight" problem — when you've debited the payer but the network call to the card processor fails?**
> A: The payer's account is decremented (reserved), but the payment to the processor hasn't been confirmed. The payment is in `processing` state. Solution: (1) Use a **webhook/callback** from the processor instead of synchronous response — the processor calls back when complete, guaranteeing delivery even through network blips. (2) Run a **periodic job** (every 5 minutes) that finds payments stuck in `processing` for > 2 minutes and queries the processor's API to get the actual status. (3) If the processor has no record of the charge (request never reached them): mark payment as `failed` and credit back the reserved amount atomically. This is the "saga" pattern for distributed transactions.

---

## 🔗 What to Read Next

1. **[SystemDesignCaseStudies/DesignRateLimiter.md](./DesignRateLimiter.md)** — Rate limiting is critical for payment APIs to prevent fraud
2. **[Database/ACID.md](../Database/ACID.md)** — Deep dive into the transaction guarantees underpinning this design
3. **[Security/OWASP_Top10.md](../Security/OWASP_Top10.md)** — Payment systems are high-value targets; review A01 (Broken Access Control) and A02 (Crypto Failures)

---

*[← Design Notification System](./DesignNotificationSystem.md) | [Back to Case Studies](./README.md)*
