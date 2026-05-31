# 🔌 Long Polling vs WebSockets vs Server-Sent Events

> *"Slack delivers messages to 20 million daily active users in real-time. They tried long polling first — it worked but crushed their servers with millions of hanging HTTP connections. They switched to WebSockets and reduced their connection overhead by 90%. But for their notification feed? They use Server-Sent Events — because not everything needs bidirectional communication."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [HTTP](../APIs/HTTP.md), [WebSockets](../APIs/WebSockets.md)

---

## 📋 Table of Contents
1. [The Real-Time Problem](#-the-real-time-problem)
2. [Regular Polling](#-regular-polling)
3. [Long Polling](#-long-polling)
4. [WebSockets](#-websockets)
5. [Server-Sent Events (SSE)](#-server-sent-events-sse)
6. [Comparison Table](#-comparison-table)
7. [Java/Spring Boot Examples](#-javaspring-boot-examples)
8. [Decision Framework](#-decision-framework)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 The Real-Time Problem

```
╔══════════════════════════════════════════════════════════════════╗
║  HTTP is request-response: Client asks → Server answers.       ║
║  But what if the SERVER needs to send data WITHOUT being asked? ║
║  (New message, stock price change, live score update...)       ║
║                                                                ║
║  Solutions: Polling, Long Polling, WebSockets, SSE             ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔄 Regular Polling

```
  Client asks "any new messages?" every N seconds:
  
  Client: "New messages?" ──► Server: "Nope"  (wasted!)
  Client: "New messages?" ──► Server: "Nope"  (wasted!)
  Client: "New messages?" ──► Server: "Nope"  (wasted!)
  Client: "New messages?" ──► Server: "Yes! Here: [msg]" ✅
  Client: "New messages?" ──► Server: "Nope"  (wasted!)
  
  Timeline:
  ├──1s──├──1s──├──1s──├──1s──├──1s──├──1s──├
  ↑req   ↑req   ↑req   ↑req   ↑req   ↑req
  ↓empty ↓empty ↓empty ↓DATA! ↓empty ↓empty
  
  PROBLEM: 90% of requests return EMPTY! Wastes bandwidth and CPU.
  With 1M users polling every 1s = 1M requests/second (mostly empty!)
```

### When Polling is Actually Fine

```
  ✅ Low-frequency data (weather every 30 min)
  ✅ Simple implementation needed
  ✅ Client can tolerate delay (email checking every 5 min)
  ✅ Very few concurrent users (admin dashboards)
```

---

## ⏳ Long Polling

```
  Client asks "any new messages?" — Server HOLDS the connection
  until there IS a message (or timeout)!
  
  Client: "New messages?" ──► Server: "Let me hold this..."
                                     ... waiting ...
                                     ... waiting ... (30 seconds)
                                     ... new data arrives! ...
  Client: ◄─── "Yes! Here: [msg]"
  
  Client immediately reconnects: "New messages?" ──► Server holds...
  
  Timeline:
  ├────────────────────────30s────────────────────────├
  ↑req            DATA! ↓immediately                  ↑req again
  [=======WAITING=======]                             [===WAITING===]
  
  MUCH BETTER than polling:
  • No wasted empty responses!
  • Near-real-time delivery (as soon as data is available)
  • Still uses regular HTTP (proxy-friendly!)
```

### How Long Polling Works

```
  ┌────────────┐                          ┌────────────┐
  │   Client   │                          │   Server   │
  └─────┬──────┘                          └─────┬──────┘
        │  GET /messages?since=lastId            │
        │───────────────────────────────────────►│
        │                                        │ Hold connection
        │                                        │ ... waiting ...
        │                                        │ New message arrives!
        │◄───────────────────────────────────────│
        │  200 OK {messages: [...]}              │
        │                                        │
        │  GET /messages?since=newLastId         │
        │───────────────────────────────────────►│  (immediately reconnect!)
        │                                        │ Hold connection
        │                                        │ ... waiting ...
```

---

## 🔌 WebSockets

```
  Full-duplex, persistent connection. Both sides send anytime!
  
  1. Client initiates HTTP upgrade:
     GET /chat HTTP/1.1
     Upgrade: websocket
     Connection: Upgrade
     
  2. Server accepts:
     HTTP/1.1 101 Switching Protocols
     Upgrade: websocket
     
  3. Now: PERSISTENT TCP connection, both send freely!
  
  Client: "Hello!" ───────────► Server
  Client: ◄─────────────────── Server: "Hi back!"
  Client: ◄─────────────────── Server: "New notification!"
  Client: "Send to Bob: hi" ──► Server
  Client: ◄─────────────────── Server: "Bob says: hey!"
  
  Timeline:
  ├═══════════════════PERSISTENT CONNECTION═══════════════════════├
  ↑upgrade  ↕ messages flow freely in BOTH directions  
  
  TRUE bidirectional real-time communication!
  No overhead of HTTP headers on each message (after handshake)
```

---

## 📡 Server-Sent Events (SSE)

```
  One-way: Server pushes events to client over HTTP.
  Client CANNOT send data back on same connection!
  
  Client: GET /events (Accept: text/event-stream) ──► Server
  Client: ◄─── data: {"price": 150.23}\n\n
  Client: ◄─── data: {"price": 151.05}\n\n
  Client: ◄─── data: {"price": 149.88}\n\n
  ...continues indefinitely...
  
  Timeline:
  ├══════════PERSISTENT HTTP CONNECTION══════════════════════════├
  ↑connect   ↓event  ↓event  ↓event  ↓event (server → client only)
  
  LIKE WebSocket but:
  • ONE-WAY (server to client only)
  • Regular HTTP (works through all proxies!)
  • Auto-reconnection built into browsers!
  • Simpler than WebSocket (just HTTP with streaming response)
```

---

## 📊 Comparison Table

```
┌─────────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│  Feature        │  Polling     │  Long Poll   │  WebSocket   │  SSE         │
├─────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│  Direction      │  Client→Svr  │  Client→Svr  │  Bidirectional│ Server→Client│
│  Real-time      │  ❌ Delayed  │  ✅ Near     │  ✅ True     │  ✅ True     │
│  Connection     │  Short-lived │  Held (30s)  │  Persistent  │  Persistent  │
│  Protocol       │  HTTP        │  HTTP        │  WS (TCP)    │  HTTP        │
│  Overhead       │  High        │  Medium      │  Low         │  Low         │
│  Proxy support  │  ✅ Great    │  ✅ Good     │  ⚠️ Some issues│  ✅ Great   │
│  Auto-reconnect │  N/A         │  Manual      │  Manual      │  ✅ Built-in │
│  Binary data    │  ❌          │  ❌          │  ✅          │  ❌ (text)   │
│  Scalability    │  Poor        │  Medium      │  Complex     │  Good        │
│  Max connections│  N/A         │  Limited*    │  Limited*    │  6 per domain│
│  Complexity     │  Simple      │  Medium      │  Complex     │  Simple      │
└─────────────────┴──────────────┴──────────────┴──────────────┴──────────────┘

* Most servers: 10K-100K concurrent connections (C10K problem)
```

---

## 💻 Java/Spring Boot Examples

### Long Polling

```java
@RestController
public class LongPollingController {
    
    private final Map<String, DeferredResult<List<Message>>> waitingClients = 
        new ConcurrentHashMap<>();
    
    @GetMapping("/api/messages/poll")
    public DeferredResult<List<Message>> pollMessages(
            @RequestParam String userId,
            @RequestParam Long sinceId) {
        
        // DeferredResult allows async response (hold connection!)
        DeferredResult<List<Message>> result = new DeferredResult<>(30000L);
        
        // Check if messages already available
        List<Message> pending = messageService.getNewMessages(userId, sinceId);
        if (!pending.isEmpty()) {
            result.setResult(pending); // Return immediately!
            return result;
        }
        
        // No messages yet — hold the connection
        waitingClients.put(userId, result);
        
        // Timeout handler (return empty after 30s, client will reconnect)
        result.onTimeout(() -> {
            waitingClients.remove(userId);
            result.setResult(Collections.emptyList());
        });
        
        return result;
    }
    
    // Called when new message arrives for a user
    public void notifyNewMessage(String userId, Message message) {
        DeferredResult<List<Message>> waiting = waitingClients.remove(userId);
        if (waiting != null) {
            waiting.setResult(List.of(message)); // Release held connection!
        }
    }
}
```

### WebSocket

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").withSockJS(); // Fallback for old browsers
    }
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
    }
}

@Controller
public class ChatWebSocketController {
    
    @MessageMapping("/chat.send")  // Client sends to /app/chat.send
    @SendTo("/topic/public")       // Broadcast to all subscribers
    public ChatMessage sendMessage(@Payload ChatMessage message) {
        return message;
    }
    
    // Send to specific user
    @Autowired private SimpMessagingTemplate messagingTemplate;
    
    public void sendPrivateMessage(String userId, Object message) {
        messagingTemplate.convertAndSendToUser(userId, "/queue/messages", message);
    }
}
```

### Server-Sent Events (SSE)

```java
@RestController
public class SSEController {
    
    @GetMapping(value = "/api/events/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter streamEvents(@RequestParam String userId) {
        SseEmitter emitter = new SseEmitter(Long.MAX_VALUE); // No timeout
        
        // Register this emitter for the user
        sseRegistry.register(userId, emitter);
        
        // Cleanup on disconnect
        emitter.onCompletion(() -> sseRegistry.remove(userId));
        emitter.onTimeout(() -> sseRegistry.remove(userId));
        
        // Send initial connection event
        try {
            emitter.send(SseEmitter.event()
                .name("connected")
                .data("Stream established"));
        } catch (IOException e) {
            emitter.completeWithError(e);
        }
        
        return emitter;
    }
    
    // Push event to specific user
    public void pushEvent(String userId, String eventType, Object data) {
        SseEmitter emitter = sseRegistry.get(userId);
        if (emitter != null) {
            try {
                emitter.send(SseEmitter.event()
                    .name(eventType)
                    .data(data));
            } catch (IOException e) {
                sseRegistry.remove(userId);
            }
        }
    }
}
```

---

## 📊 Decision Framework

```
START HERE:
  ┌─────────────────────────────────────┐
  │ Does client need to send data       │
  │ back to server on same connection?  │
  └───────────────┬─────────────────────┘
                  │
        ┌─────────┴─────────┐
        │YES                │NO
        ▼                   ▼
    WebSocket         Need real-time?
                          │
                ┌─────────┴─────────┐
                │YES                │NO
                ▼                   ▼
              SSE              Regular Polling
              (or Long Poll     (simple, low frequency)
               if SSE not 
               available)

SPECIFIC USE CASES:
  • Chat application → WebSocket (bidirectional messaging)
  • Live sports scores → SSE (server pushes updates)
  • Stock ticker → SSE (one-way, high frequency)
  • Collaborative editing → WebSocket (bidirectional, low latency)
  • Notification feed → SSE (server pushes)
  • Email checking → Polling (low frequency, simple)
  • IoT data → WebSocket or MQTT (bidirectional, high throughput)
  • File upload progress → SSE (server reports progress)
```

---

## 🎮 Mini Challenge

### 🧩 Choose the Right Technology

For each scenario, pick the best real-time approach:
1. Live cricket score updates to 10M users
2. Multiplayer online game (100ms latency requirement)
3. Social media "someone liked your post" notifications
4. Collaborative whiteboard (multiple users drawing)
5. Uber driver location tracking (driver → server → rider)

<details>
<summary>🔑 Answers</summary>

1. **SSE** — One-way (server → clients), simple, auto-reconnect. 10M users can use CDN with SSE edge servers.
2. **WebSocket** — Bidirectional, ultra-low latency, binary data support for game state.
3. **SSE** (or Long Polling as fallback) — Server pushes notifications. Client doesn't send on notification channel.
4. **WebSocket** — Bidirectional (users send drawing commands AND receive others' drawings). Low latency critical.
5. **WebSocket** — Driver sends location (client→server), rider receives location (server→client). Bidirectional needed but could split: driver uses HTTP POST, rider uses SSE.
</details>

---

## ❓ Interview Q&A

**Q1: Explain long polling and when you'd use it over WebSockets.**
> Long polling: client sends request, server holds it open until data is available (or timeout), then responds. Client immediately reconnects. Use over WebSockets when: (1) you need proxy/firewall compatibility (all support HTTP), (2) infrequent updates (holding 1 connection vs persistent WS), (3) simple implementation needed, (4) server infrastructure doesn't support WebSocket.

**Q2: What's the main advantage of SSE over WebSockets?**
> SSE uses plain HTTP, so it works through ALL proxies, load balancers, and firewalls without special configuration. It has built-in browser auto-reconnection and event ID tracking (last-event-id for replay after disconnect). WebSocket requires special proxy config, manual reconnection logic, and doesn't work through some corporate proxies.

**Q3: How do you scale WebSocket connections to millions of users?**
> (1) Use sticky sessions (load balancer routes same user to same server), (2) Pub/Sub backbone (Redis Pub/Sub or Kafka) to broadcast across servers, (3) Connection management servers (separate from business logic), (4) Horizontal scaling with shared state (which user is on which server), (5) Consider using managed services (AWS API Gateway WebSocket).

**Q4: When would you NOT use WebSockets?**
> When: (1) Communication is one-way server→client (use SSE, simpler), (2) Updates are infrequent (polling every 30s is fine), (3) Need HTTP caching, (4) Behind restrictive corporate proxies, (5) Need request-response semantics (REST is clearer), (6) Stateless architecture required (WS is inherently stateful).

**Q5: What happens when a long-polling or WebSocket connection drops?**
> Long polling: Client detects timeout/error, reconnects with last-seen message ID, server sends missed messages. WebSocket: Client detects close event, attempts reconnection with exponential backoff, server sends missed messages from event log. Both need: message ID tracking, server-side message buffer, and reconnection logic.

---

## 🔗 Related Topics
- [WebSockets](../APIs/WebSockets.md) — WebSocket protocol deep dive
- [Server-Sent Events](../APIs/ServerSentEvents.md) — SSE details
- [Load Balancing](../BuildingBlocks/LoadBalancing.md) — Sticky sessions for WS
- [Push vs Pull](./Push_vs_Pull_Architecture.md) — Broader architectural context

---

*"Long polling is the mullet of real-time: HTTP in the front, persistent connection in the back." — Unknown Web Developer* 🔌
