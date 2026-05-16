# 📡 Server-Sent Events (SSE): One-Way Real-Time Streaming

> *"Not every real-time feature needs bidirectional communication. If you only need the server to push updates to the client — live scores, stock tickers, notifications — SSE is simpler, lighter, and works over standard HTTP."*

**⏱️ Estimated Time**: 15 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [RESTful APIs](./RESTful.md), [WebSockets](./WebSockets.md)

---

## 🤔 Why SSE?

```
POLLING:            Client asks repeatedly → wasteful
LONG POLLING:       Client waits → complex, still half-duplex
WEBSOCKET:          Full-duplex → overkill if only server pushes
SERVER-SENT EVENTS: Server pushes over HTTP → simple, efficient

SSE FLOW:
  ┌────────┐     GET /events (text/event-stream)     ┌────────┐
  │ Client │ ────────────────────────────────────────→│ Server │
  │        │←────── data: {"score": "2-1"}\n\n ──────│        │
  │        │←────── data: {"score": "2-2"}\n\n ──────│        │
  │        │←────── data: {"score": "3-2"}\n\n ──────│        │
  └────────┘    (connection stays open, server pushes) └────────┘
  
  - Standard HTTP connection (no upgrade needed)
  - Built-in browser support (EventSource API)
  - Automatic reconnection with Last-Event-ID
  - Works with HTTP/2 multiplexing
```

---

## 💻 Spring Boot SSE Example

```java
@RestController
@RequestMapping("/api/events")
public class SSEController {
    
    private final List<SseEmitter> emitters = new CopyOnWriteArrayList<>();
    
    @GetMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter subscribe() {
        SseEmitter emitter = new SseEmitter(Long.MAX_VALUE);
        emitters.add(emitter);
        
        emitter.onCompletion(() -> emitters.remove(emitter));
        emitter.onTimeout(() -> emitters.remove(emitter));
        emitter.onError(e -> emitters.remove(emitter));
        
        return emitter;
    }
    
    // Called when you want to push an event to all subscribers
    public void pushEvent(String eventName, Object data) {
        List<SseEmitter> deadEmitters = new ArrayList<>();
        
        for (SseEmitter emitter : emitters) {
            try {
                emitter.send(SseEmitter.event()
                    .name(eventName)
                    .data(data, MediaType.APPLICATION_JSON));
            } catch (IOException e) {
                deadEmitters.add(emitter);
            }
        }
        emitters.removeAll(deadEmitters);
    }
}
```

**Client-side (JavaScript):**
```javascript
const source = new EventSource('/api/events');
source.addEventListener('score-update', (event) => {
    const data = JSON.parse(event.data);
    updateScoreboard(data);
});
source.onerror = () => console.log('Reconnecting...'); // auto-reconnects
```

---

## 📊 SSE vs WebSocket vs Long Polling

| Feature | SSE | WebSocket | Long Polling |
|---|---|---|---|
| Direction | Server → Client | Bidirectional | Server → Client |
| Protocol | HTTP | WS (upgraded HTTP) | HTTP |
| Auto-reconnect | Built-in | Manual | Manual |
| Binary data | No (text only) | Yes | Yes |
| HTTP/2 multiplexing | Yes | No | Yes |
| Browser support | All modern | All modern | All |
| Max connections/browser | 6 (HTTP/1.1) | Unlimited | 6 |
| Proxy-friendly | Yes (standard HTTP) | Sometimes blocked | Yes |
| Complexity | Low | Medium | Medium |

---

## ⚠️ Common Pitfalls

1. **HTTP/1.1 connection limit** — Browsers limit 6 SSE connections per domain on HTTP/1.1. Solution: use HTTP/2 (multiplexes streams) or consolidate into a single SSE endpoint with event types.

2. **Not setting Last-Event-ID** — SSE supports resume after reconnection. Server should include `id:` field in events; on reconnect, the browser sends `Last-Event-ID` header so the server can replay missed events.

3. **Memory leaks from dead emitters** — Always remove emitters on completion/timeout/error. Without cleanup, the emitter list grows unbounded.

---

## 📝 Interview Q&A

**Q: When would you choose SSE over WebSocket?**
> A: Use SSE when: (1) communication is unidirectional (server→client only), (2) you want simpler infrastructure (standard HTTP, no upgrade), (3) you need automatic reconnection with event replay, (4) your proxy/CDN doesn't support WebSocket, (5) you're already on HTTP/2. Use WebSocket when you need bidirectional (chat, gaming) or binary data (file transfer).

---

## 🔗 What to Read Next

1. **[APIs/WebSockets.md](./WebSockets.md)** — Full-duplex alternative
2. **[APIs/Webhooks.md](./Webhooks.md)** — Server-to-server event notifications
3. **[BuildingBlocks/MessageQueues.md](../BuildingBlocks/MessageQueues.md)** — Async messaging patterns

---

*[← WebSockets](./WebSockets.md) | [Back to Index](../INDEX.md) | [Next: Webhooks →](./Webhooks.md)*
