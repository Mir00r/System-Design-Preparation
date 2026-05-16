# 🔌 WebSockets: Full-Duplex Real-Time Communication

> *"HTTP is request-response: client asks, server answers. WebSocket is a persistent bidirectional connection — both sides can send messages anytime. It's the foundation of real-time apps: chat, gaming, live dashboards, collaborative editing."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [RESTful APIs](./RESTful.md)

---

## 🤔 HTTP vs WebSocket

```
HTTP (half-duplex, request/response):
  Client: "Any new messages?" → Server: "No"     (1 sec later)
  Client: "Any new messages?" → Server: "No"     (1 sec later)
  Client: "Any new messages?" → Server: "Yes! Here's 1 message"
  
  Problem: Polling wastes bandwidth, adds latency (up to 1s delay)

WEBSOCKET (full-duplex, persistent):
  Client ←──────── persistent connection ──────────→ Server
  
  Server: "New message from Alice!" (instant push, no polling)
  Client: "I'm typing..." (instant send)
  Server: "Bob is typing..." (instant push)
  
  Single TCP connection stays open → both sides push anytime

HANDSHAKE (HTTP → WebSocket upgrade):
  Client: GET /chat HTTP/1.1
          Upgrade: websocket
          Connection: Upgrade
          Sec-WebSocket-Key: dGhlIHNhbXBsZQ==
          
  Server: HTTP/1.1 101 Switching Protocols
          Upgrade: websocket
          Connection: Upgrade
          
  → Connection upgraded! Now speaking WebSocket protocol (not HTTP)
```

---

## 💻 Spring Boot WebSocket Example

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new ChatWebSocketHandler(), "/ws/chat")
                .setAllowedOrigins("*");
    }
}

@Component
public class ChatWebSocketHandler extends TextWebSocketHandler {
    private final Set<WebSocketSession> sessions = ConcurrentHashMap.newKeySet();
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        sessions.add(session);
    }
    
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        // Broadcast to all connected clients
        for (WebSocketSession s : sessions) {
            if (s.isOpen()) {
                s.sendMessage(new TextMessage("User: " + message.getPayload()));
            }
        }
    }
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        sessions.remove(session);
    }
}
```

---

## ⚖️ When to Use WebSocket vs Alternatives

| Use Case | Best Option | Why |
|---|---|---|
| Chat / messaging | WebSocket | Bidirectional, low latency |
| Live notifications | SSE or WebSocket | SSE simpler if one-way |
| Real-time dashboard | WebSocket or SSE | Depends on interactivity |
| File upload progress | SSE | Server→client only |
| Online gaming | WebSocket | Bidirectional, low latency |
| API CRUD operations | REST | Request/response sufficient |
| Infrequent updates | Long polling | Simpler infrastructure |

---

## ⚠️ Common Pitfalls

1. **No reconnection logic** — WebSocket connections drop (network issues, server restart). Clients MUST implement automatic reconnection with exponential backoff. Libraries like SockJS handle this.

2. **Not handling scale** — WebSocket connections are stateful (pinned to one server). To scale: use Redis Pub/Sub or a message broker to broadcast messages across server instances.

3. **Using WebSocket for everything** — If you only need server→client push (notifications, live scores), use SSE — it's simpler, works over HTTP/2, and auto-reconnects.

---

## 📝 Interview Q&A

**Q: How would you scale a WebSocket-based chat system to millions of users?**
> A: (1) **Connection management**: each server handles ~50K-100K concurrent WebSocket connections. Use a load balancer with sticky sessions (or connection ID routing). (2) **Cross-server messaging**: when User A (on Server 1) messages User B (on Server 2), use Redis Pub/Sub or Kafka to relay the message between servers. (3) **Presence tracking**: store online status in Redis (which users are on which servers). (4) **Horizontal scaling**: add more WebSocket servers behind the LB. (5) **Fallback**: use SockJS for browsers that don't support WebSocket (falls back to long-polling).

---

## 🔗 What to Read Next

1. **[APIs/ServerSentEvents.md](./ServerSentEvents.md)** — One-way server push alternative
2. **[APIs/Webhooks.md](./Webhooks.md)** — Event-driven notifications
3. **[SystemDesignCaseStudies/DesignWhatsApp.md](../SystemDesignCaseStudies/DesignWhatsApp.md)** — WhatsApp design using WebSocket

---

*[← API Design Guidelines](./API_Design_Guidelines.md) | [Back to Index](../INDEX.md) | [Next: Server-Sent Events →](./ServerSentEvents.md)*
