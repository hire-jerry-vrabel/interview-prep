# System Design — Real-Time Chat System

A real-time chat system is a classic senior interview question that
covers WebSockets, pub/sub messaging, message persistence, presence,
and scale. It maps directly to production experience with Node.js.

---

## The Ask

> "Design a real-time chat system like Slack or WhatsApp"

Clarifying questions to ask first:
- 1:1 chat only or group channels too? (both)
- How many users? (assume 50M DAU)
- How many messages per day? (assume 1B messages/day)
- Do messages need to persist? (yes, indefinitely)
- Do we need read receipts and presence? (yes)
- Mobile and web clients? (yes)

---

## High Level Design
```
Client (browser/mobile)
    ↕ WebSocket
WebSocket Servers (stateful, horizontally scaled)
    ↕ Pub/Sub
Redis (message broker + presence)
    ↕
Message Service → PostgreSQL (persistence)
    ↕
Notification Service → Push notifications (offline users)
```

---

## Core Components

### 1. WebSocket Connection Management

Each client opens a persistent WebSocket connection to a server.
The challenge at scale — users in the same channel may be connected
to different servers. Redis pub/sub bridges them.
```typescript
import { WebSocketServer, WebSocket } from 'ws'
import { createClient } from 'redis'

const wss = new WebSocketServer({ port: 8080 })
const pub = createClient()
const sub = createClient()

await pub.connect()
await sub.connect()

// Map of userId -> WebSocket connection (local to this server)
const connections = new Map<string, WebSocket>()

wss.on('connection', (ws, req) => {
  const userId = getUserIdFromRequest(req)
  connections.set(userId, ws)

  // Subscribe to this user's personal channel
  await sub.subscribe(`user:${userId}`, (message) => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(message)
    }
  })

  ws.on('message', async (data) => {
    const event = JSON.parse(data.toString())
    await handleEvent(event, userId)
  })

  ws.on('close', async () => {
    connections.delete(userId)
    await setUserOffline(userId)
  })

  setUserOnline(userId)
})
```

---

### 2. Sending a Message
```typescript
async function handleEvent(event: ChatEvent, senderId: string) {
  if (event.type === 'message') {
    const { channelId, content } = event

    // 1. Persist to database
    const message = await db.query(`
      INSERT INTO messages (channel_id, sender_id, content, created_at)
      VALUES ($1, $2, $3, NOW())
      RETURNING *
    `, [channelId, senderId, content])

    // 2. Get all members of this channel
    const members = await getChannelMembers(channelId)

    // 3. Publish to each member's Redis channel
    // All WebSocket servers are subscribed and will deliver
    // to whichever server holds each member's connection
    const payload = JSON.stringify({
      type: 'message',
      data: message.rows[0]
    })

    await Promise.all(
      members.map(memberId =>
        pub.publish(`user:${memberId}`, payload)
      )
    )

    // 4. Push notify offline members
    const offlineMembers = members.filter(id => !isOnline(id))
    await notifyOfflineUsers(offlineMembers, message.rows[0])
  }
}
```

---

### 3. Database Schema
```sql
CREATE TABLE users (
  id         BIGSERIAL PRIMARY KEY,
  username   VARCHAR(50) UNIQUE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE channels (
  id         BIGSERIAL PRIMARY KEY,
  name       VARCHAR(100),
  is_direct  BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE channel_members (
  channel_id BIGINT REFERENCES channels(id),
  user_id    BIGINT REFERENCES users(id),
  joined_at  TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (channel_id, user_id)
);

CREATE TABLE messages (
  id         BIGSERIAL PRIMARY KEY,
  channel_id BIGINT REFERENCES channels(id),
  sender_id  BIGINT REFERENCES users(id),
  content    TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Critical index for message history queries
CREATE INDEX idx_messages_channel_created
  ON messages(channel_id, created_at DESC);
```

---

### 4. Presence System

Track who is online using Redis with a TTL heartbeat:
```typescript
const PRESENCE_TTL = 30 // seconds

async function setUserOnline(userId: string) {
  await redis.setex(`presence:${userId}`, PRESENCE_TTL, '1')
  await pub.publish('presence', JSON.stringify({
    userId,
    status: 'online'
  }))
}

async function isOnline(userId: string): Promise<boolean> {
  return (await redis.exists(`presence:${userId}`)) === 1
}

// Client sends heartbeat every 20 seconds
// If TTL expires, user is considered offline
ws.on('message', async (data) => {
  const event = JSON.parse(data.toString())
  if (event.type === 'heartbeat') {
    await redis.expire(`presence:${userId}`, PRESENCE_TTL)
  }
})
```

---

### 5. Message History (Pagination)
```typescript
app.get('/channels/:channelId/messages', async (req, res) => {
  const { channelId } = req.params
  const { before, limit = 50 } = req.query

  const messages = await db.query(`
    SELECT m.*, u.username
    FROM messages m
    JOIN users u ON m.sender_id = u.id
    WHERE m.channel_id = $1
      AND ($2::timestamptz IS NULL OR m.created_at < $2)
    ORDER BY m.created_at DESC
    LIMIT $3
  `, [channelId, before || null, limit])

  res.json({
    messages: messages.rows.reverse(),
    hasMore: messages.rows.length === Number(limit)
  })
})
```

---

## Scale Considerations

### WebSocket Servers
- Stateless at the app level — all state lives in Redis
- Scale horizontally behind a load balancer with sticky sessions
  OR use Redis to route messages regardless of which server
  holds the connection
- Each server can handle ~10k concurrent WebSocket connections

### Message Fan-out
- Small channels (< 1000 members) — fan out at send time
- Large channels (> 1000 members) — use a dedicated fan-out
  service with a worker queue to avoid blocking the send path

### Database
- Partition messages table by channel_id or created_at
- Archive messages older than 1 year to cold storage (S3)
- Read replicas for message history queries

### Redis
- Use Redis Cluster for horizontal scaling of pub/sub
- Separate Redis instances for pub/sub vs presence vs caching

---

## Tradeoffs to Discuss

| Decision | Option A | Option B | Winner |
|---|---|---|---|
| Transport | WebSockets | Long polling | WebSockets — lower latency |
| Broker | Redis pub/sub | Kafka | Redis for chat, Kafka for analytics |
| Storage | PostgreSQL | Cassandra | PostgreSQL to start, Cassandra at scale |
| Fan-out | Push | Pull | Push for small channels, pull for large |

---

## Follow-up Questions Interviewers Ask

- "How would you handle message ordering?" — sequence numbers per channel,
  monotonically increasing via DB serial or Redis INCR
- "How would you support read receipts?" — store last_read_at per user per
  channel, publish receipt events via WebSocket
- "What happens if a message is lost?" — client-side sequence numbers,
  gap detection, re-fetch on reconnect
- "How would you handle 1M member channels?" — pull model, paginated
  member lists, fan-out workers

---

## My Talking Points

- Redis pub/sub is the key insight — it decouples WebSocket servers
  so any server can deliver to any connected client
- Start with PostgreSQL — it handles billions of messages with the right
  indexes and partitioning before you need Cassandra
- The presence system is underrated — heartbeat TTL in Redis is elegant
  and battle-tested
- I'd ship 1:1 chat first, validate, then add channels — same
  architecture, incremental complexity
