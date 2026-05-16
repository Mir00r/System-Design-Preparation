# 🪝 Webhooks: Event-Driven Server-to-Server Notifications

> *"Instead of polling an API every 5 seconds asking 'anything new?', webhooks flip the model: 'Hey, I'll call YOU when something happens.' It's the difference between checking your mailbox every hour vs getting a doorbell ring when mail arrives."*

**⏱️ Estimated Time**: 18 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [RESTful APIs](./RESTful.md)

---

## 🤔 Polling vs Webhooks

```
POLLING (wasteful):
  Client: "Any new payment?" → Server: "No"       (every 5 sec)
  Client: "Any new payment?" → Server: "No"       
  Client: "Any new payment?" → Server: "No"       
  Client: "Any new payment?" → Server: "Yes! Payment #123 succeeded"
  
  95% of requests return "no" → wasted bandwidth and compute

WEBHOOK (efficient):
  Client registers: "Call https://myapp.com/hooks/payment when payment succeeds"
  
  ... time passes ... (no requests made)
  
  Server: POST https://myapp.com/hooks/payment
          { "event": "payment.succeeded", "data": { "id": 123, "amount": 99.99 } }
  
  Zero wasted requests. Instant notification when event occurs.

ARCHITECTURE:
  ┌──────────┐  1. Register URL   ┌──────────────┐
  │ Your App │ ──────────────────→ │ Provider     │
  │          │                     │ (Stripe,     │
  │          │ ←─── 2. POST event  │  GitHub,     │
  │          │      when triggered  │  Slack)      │
  └──────────┘                     └──────────────┘
```

---

## 💻 Spring Boot Webhook Receiver

```java
@RestController
@RequestMapping("/api/webhooks")
public class WebhookController {
    
    private static final String WEBHOOK_SECRET = "whsec_..."; // from env
    
    @PostMapping("/stripe")
    public ResponseEntity<String> handleStripeWebhook(
            @RequestBody String payload,
            @RequestHeader("Stripe-Signature") String signature) {
        
        // 1. VERIFY SIGNATURE (critical for security!)
        if (!verifySignature(payload, signature, WEBHOOK_SECRET)) {
            return ResponseEntity.status(401).body("Invalid signature");
        }
        
        // 2. Parse event
        WebhookEvent event = parseEvent(payload);
        
        // 3. Process idempotently (webhooks may be delivered multiple times)
        if (alreadyProcessed(event.getId())) {
            return ResponseEntity.ok("Already processed");
        }
        
        // 4. Handle event type
        switch (event.getType()) {
            case "payment_intent.succeeded":
                handlePaymentSuccess(event.getData());
                break;
            case "payment_intent.failed":
                handlePaymentFailure(event.getData());
                break;
            default:
                log.info("Unhandled event type: {}", event.getType());
        }
        
        // 5. Respond 200 quickly (provider will retry if no 2xx within ~5s)
        return ResponseEntity.ok("Received");
    }
    
    private boolean verifySignature(String payload, String signature, String secret) {
        // HMAC-SHA256 verification
        String expectedSig = HmacUtils.hmacSha256Hex(secret, payload);
        return MessageDigest.isEqual(
            expectedSig.getBytes(), signature.getBytes());
    }
}
```

---

## 🔐 Webhook Security Best Practices

```
1. SIGNATURE VERIFICATION:
   Provider signs payload with shared secret (HMAC-SHA256)
   Your server verifies signature before processing
   → Prevents forged webhook attacks

2. IDEMPOTENCY:
   Store processed event IDs in database
   Skip duplicates (providers retry on timeout/failure)
   → Prevents double-processing

3. RESPOND QUICKLY:
   Return 200/202 immediately, process async
   Provider timeout: 5-30 seconds (retries on timeout)
   → Use a queue for heavy processing

4. HTTPS ONLY:
   Never expose webhook endpoint on HTTP
   Payload may contain sensitive data (payments, user info)
   → TLS required

5. IP ALLOWLISTING:
   Some providers publish webhook source IPs
   Restrict your endpoint to those IPs only
   → Defense in depth
```

---

## 📊 Webhook Retry Strategies (by Provider)

| Provider | Timeout | Retry Attempts | Backoff Strategy |
|---|---|---|---|
| Stripe | 5 sec | 3 over 72 hrs | Exponential |
| GitHub | 10 sec | 3 retries | Fixed interval |
| Slack | 3 sec | 3 over 30 min | Exponential |
| Twilio | 15 sec | Configurable | Exponential |

---

## ⚠️ Common Pitfalls

1. **Not verifying signatures** — Anyone who discovers your webhook URL can send fake events. ALWAYS verify the HMAC signature using your shared secret.

2. **Synchronous processing** — If your webhook handler takes >5 seconds (heavy DB writes, external API calls), the provider times out and retries. Accept the webhook, queue it, process async.

3. **No idempotency** — Providers retry on failure/timeout. If your handler isn't idempotent, you'll process the same payment twice, send duplicate emails, etc.

4. **Ignoring failed deliveries** — If your server is down, you miss webhooks. Implement: (a) a "replay missed events" polling fallback, (b) dead-letter queue monitoring, or (c) provider's event log API.

---

## 📝 Interview Q&A

**Q: How would you design a reliable webhook delivery system?**
> A: (1) **Event store**: persist all events to a durable log (Kafka/DB) before attempting delivery. (2) **Delivery worker**: reads from event store, POSTs to subscriber URLs with configurable timeout. (3) **Retry with exponential backoff**: on failure, retry at 1min, 5min, 30min, 1hr, 6hr, 24hr. (4) **Signature**: sign payloads with per-subscriber HMAC secret. (5) **Dead letter**: after max retries, move to dead-letter queue, alert subscriber. (6) **Replay API**: let subscribers request missed events by time range. (7) **Circuit breaker**: if a subscriber fails consistently, pause deliveries and notify them.

---

## 🔗 What to Read Next

1. **[APIs/WebSockets.md](./WebSockets.md)** — Real-time bidirectional
2. **[APIs/ServerSentEvents.md](./ServerSentEvents.md)** — Real-time server push
3. **[APIs/API_Versioning.md](./API_Versioning.md)** — Managing API changes

---

*[← Server-Sent Events](./ServerSentEvents.md) | [Back to Index](../INDEX.md) | [Next: API Versioning →](./API_Versioning.md)*
