---
name: realtime-patterns
description: Real-time communication patterns for WebSocket, Server-Sent Events (SSE), pub/sub, message queues, and event-driven architectures across Python, Node.js, and Go.
---

# Real-Time Communication Patterns

Patterns for building real-time features — live updates, notifications, streaming, and event-driven systems.

## When to Activate

- Building WebSocket or SSE connections for live data
- Implementing pub/sub messaging (Redis, RabbitMQ, Kafka)
- Designing event-driven architectures
- Adding real-time notifications or live feeds
- Streaming data processing or log tailing
- Building collaborative or multiplayer features

## Core Principles

### 1. Choose the Right Transport

```
| Feature | WebSocket | SSE | Polling |
|---------|-----------|-----|---------|
| Direction | Bidirectional | Server→Client | Client→Server |
| Complexity | High | Low | Lowest |
| Reconnection | Manual | Automatic | N/A |
| Binary data | Yes | No (text only) | Yes |
| Firewall friendly | Sometimes issues | Yes (HTTP) | Yes |
| Best for | Chat, gaming | Notifications, feeds | Simple updates |
```

### 2. Always Plan for Disconnection

Clients disconnect. Design for it.

```python
# Pattern: Reconnection with last-event-id
class SSEClient:
    def __init__(self, url: str):
        self.url = url
        self.last_event_id: str | None = None

    async def connect(self):
        headers = {}
        if self.last_event_id:
            headers["Last-Event-ID"] = self.last_event_id

        async with aiohttp.ClientSession() as session:
            async with session.get(self.url, headers=headers) as resp:
                async for line in resp.content:
                    event = parse_sse(line)
                    if event.id:
                        self.last_event_id = event.id
                    yield event
```

### 3. Fan-Out Through Pub/Sub

Don't broadcast from app servers. Use a message bus.

```
┌──────────┐     ┌──────────┐     ┌──────────────┐
│ App Srv 1 │────▶│  Redis   │────▶│ App Srv 1    │──▶ Client A
│ (publish) │     │  Pub/Sub │     │ (subscribers) │──▶ Client B
└──────────┘     │          │     └──────────────┘
                  │          │     ┌──────────────┐
                  │          │────▶│ App Srv 2    │──▶ Client C
                  └──────────┘     │ (subscribers) │──▶ Client D
                                   └──────────────┘
```

## WebSocket Patterns

### Connection Manager

```python
import asyncio
from dataclasses import dataclass, field

@dataclass
class ConnectionManager:
    """Manage WebSocket connections with room support."""
    _rooms: dict[str, set] = field(default_factory=dict)

    async def connect(self, websocket, room: str = "default"):
        await websocket.accept()
        if room not in self._rooms:
            self._rooms[room] = set()
        self._rooms[room].add(websocket)

    def disconnect(self, websocket, room: str = "default"):
        self._rooms.get(room, set()).discard(websocket)

    async def broadcast(self, message: dict, room: str = "default"):
        """Send message to all connections in a room."""
        connections = self._rooms.get(room, set()).copy()
        disconnected = set()
        for ws in connections:
            try:
                await ws.send_json(message)
            except Exception:
                disconnected.add(ws)
        # Clean up dead connections
        for ws in disconnected:
            self.disconnect(ws, room)

manager = ConnectionManager()
```

### FastAPI WebSocket Endpoint

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

@app.websocket("/ws/detections/{camera_id}")
async def detection_feed(websocket: WebSocket, camera_id: str):
    await manager.connect(websocket, room=f"camera:{camera_id}")
    try:
        while True:
            data = await websocket.receive_json()
            # Handle client messages (e.g., filter changes)
            if data.get("type") == "set_filter":
                await handle_filter(websocket, data)
    except WebSocketDisconnect:
        manager.disconnect(websocket, room=f"camera:{camera_id}")
```

## Server-Sent Events (SSE)

### Python SSE Endpoint

```python
from fastapi.responses import StreamingResponse

@app.get("/api/stream/detections")
async def stream_detections(request: Request):
    async def event_generator():
        last_id = request.headers.get("Last-Event-ID", "0")

        # Send any missed events
        missed = await get_events_since(last_id)
        for event in missed:
            yield format_sse(event)

        # Stream new events
        async for event in subscribe("detections"):
            if await request.is_disconnected():
                break
            yield format_sse(event)

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Disable Nginx buffering
        },
    )

def format_sse(event: dict) -> str:
    """Format event as SSE message."""
    lines = []
    if event.get("id"):
        lines.append(f"id: {event['id']}")
    if event.get("event"):
        lines.append(f"event: {event['event']}")
    lines.append(f"data: {json.dumps(event['data'])}")
    lines.append("")  # Empty line terminates event
    return "\n".join(lines) + "\n"
```

### Client-Side SSE

```typescript
function connectSSE(url: string, onEvent: (data: any) => void): EventSource {
  const source = new EventSource(url);

  source.addEventListener("detection", (event) => {
    const data = JSON.parse(event.data);
    onEvent(data);
  });

  source.onerror = () => {
    // EventSource auto-reconnects with Last-Event-ID
    console.log("SSE reconnecting...");
  };

  return source;
}
```

## Event-Driven Architecture

### Event Bus Pattern

```python
from collections import defaultdict
from typing import Callable, Awaitable

class EventBus:
    def __init__(self):
        self._handlers: dict[str, list[Callable]] = defaultdict(list)

    def on(self, event_type: str, handler: Callable[..., Awaitable]):
        self._handlers[event_type].append(handler)

    async def emit(self, event_type: str, data: dict):
        for handler in self._handlers[event_type]:
            try:
                await handler(data)
            except Exception as e:
                logger.error(f"Handler error for {event_type}: {e}")

# Usage
bus = EventBus()
bus.on("plate_detected", notify_subscribers)
bus.on("plate_detected", update_statistics)
bus.on("plate_detected", log_detection)

await bus.emit("plate_detected", {"plate": "AB1234", "camera": "cam-01"})
```

## Message Queue Integration

```python
# Redis pub/sub for cross-process communication
import redis.asyncio as redis

async def publish_event(channel: str, data: dict):
    r = redis.from_url("redis://localhost:6379")
    await r.publish(channel, json.dumps(data))

async def subscribe_events(channel: str):
    r = redis.from_url("redis://localhost:6379")
    pubsub = r.pubsub()
    await pubsub.subscribe(channel)

    async for message in pubsub.listen():
        if message["type"] == "message":
            yield json.loads(message["data"])
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Polling for real-time data | Use WebSocket or SSE |
| No reconnection logic | Auto-reconnect with backoff |
| Broadcasting from app server | Use pub/sub (Redis, RabbitMQ) |
| No message ordering | Include sequence numbers/timestamps |
| Unbounded connection count | Limit connections per user, use rooms |
| No heartbeat/ping | Send periodic pings to detect dead connections |
| Storing events only in memory | Persist events for replay after reconnect |
| No backpressure | Buffer or drop messages when client is slow |
