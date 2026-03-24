# System Design — URL Shortener

A URL shortener is one of the most common system design interview questions.
It's deceptively simple on the surface but opens up into a rich discussion
of scale, hashing, caching, and database design.

---

## The Ask

> "Design a URL shortening service like bit.ly"

Clarifying questions to ask first:
- How many URLs per day? (assume 100M writes/day, 10B reads/day)
- How long should shortened URLs live? (assume 5 years)
- Should custom aliases be supported? (yes)
- Should analytics be tracked? (yes, click counts at minimum)

---

## High Level Design
```
Client → CDN → Load Balancer → App Servers → Cache (Redis) → DB (PostgreSQL)
                                           ↓
                                     Analytics Service
```

---

## Core Components

### 1. URL Shortening (Write Path)

**Generate a short code:**
- Take the long URL
- Generate a unique 7-character code
- Two approaches:

**Option A — Hash based:**
```typescript
import crypto from 'crypto'

function generateShortCode(longUrl: string): string {
  const hash = crypto
    .createHash('md5')
    .update(longUrl + Date.now())
    .digest('base64url')
  return hash.slice(0, 7)
}
```

**Option B — Base62 counter (preferred at scale):**
```typescript
const BASE62 = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'

function toBase62(num: number): string {
  let result = ''
  while (num > 0) {
    result = BASE62[num % 62] + result
    num = Math.floor(num / 62)
  }
  return result.padStart(7, 'a')
}
```

Base62 counter is better because:
- No collision risk
- Predictable length
- Can use a distributed ID generator (Snowflake)

---

### 2. Redirection (Read Path)
```typescript
// Express handler — the hot path, must be fast
app.get('/:code', async (req, res) => {
  const { code } = req.params

  // 1. Check Redis cache first
  const cached = await redis.get(`url:${code}`)
  if (cached) {
    await incrementClickCount(code) // async, non-blocking
    return res.redirect(301, cached)
  }

  // 2. Fall back to database
  const url = await db.query(
    'SELECT long_url FROM urls WHERE short_code = $1',
    [code]
  )

  if (!url.rows[0]) return res.status(404).send('Not found')

  // 3. Populate cache
  await redis.setex(`url:${code}`, 86400, url.rows[0].long_url)

  await incrementClickCount(code)
  return res.redirect(301, url.rows[0].long_url)
})
```

**301 vs 302:**
- `301` Permanent — browser caches it, reduces server load, but kills analytics
- `302` Temporary — hits server every time, accurate analytics
- Use `302` if you need click tracking

---

### 3. Database Schema
```sql
CREATE TABLE urls (
  id          BIGSERIAL PRIMARY KEY,
  short_code  VARCHAR(10)  NOT NULL UNIQUE,
  long_url    TEXT         NOT NULL,
  user_id     BIGINT,
  created_at  TIMESTAMPTZ  DEFAULT NOW(),
  expires_at  TIMESTAMPTZ,
  click_count BIGINT       DEFAULT 0
);

CREATE INDEX idx_short_code ON urls(short_code);

-- For analytics
CREATE TABLE clicks (
  id         BIGSERIAL PRIMARY KEY,
  url_id     BIGINT REFERENCES urls(id),
  clicked_at TIMESTAMPTZ DEFAULT NOW(),
  country    VARCHAR(2),
  referrer   TEXT
);
```

---

## Scale Considerations

### Caching
- Cache hot URLs in Redis with a TTL of 24 hours
- 80/20 rule — 20% of URLs get 80% of traffic
- Cache hit ratio should be ~99% at scale

### Database
- Read replicas for the read-heavy workload (100:1 read/write ratio)
- Shard by short_code hash if single DB becomes a bottleneck
- Archive expired URLs to cold storage

### Rate Limiting
- Limit URL creation to 100/hour per IP to prevent abuse
- Use Redis sliding window counter

### CDN
- Put a CDN in front for the most popular URLs
- Cache redirects at the edge for sub-10ms response times

---

## Tradeoffs to Discuss

| Decision | Option A | Option B | Winner |
|---|---|---|---|
| ID generation | MD5 hash | Base62 counter | Base62 — no collisions |
| Redirect type | 301 | 302 | 302 if analytics matter |
| Storage | SQL | NoSQL | SQL — relational data, ACID |
| Cache | Redis | Memcached | Redis — richer data structures |

---

## Follow-up Questions Interviewers Ask

- "How would you handle custom aliases?" — check availability, store alongside auto-generated codes
- "How would you expire URLs?" — TTL column + background job with `DELETE WHERE expires_at < NOW()`
- "How would you scale to 1B URLs?" — horizontal sharding, consistent hashing
- "How would you prevent abuse?" — rate limiting, URL scanning, blacklist

---

## My Talking Points

- I'd start with a simple monolith and extract the analytics service first since it has different scaling characteristics than the core redirect path
- The read path is the critical path — optimize ruthlessly with caching
- Base62 counter with a distributed ID generator (like Twitter's Snowflake) is production-grade from day one
- PostgreSQL handles this comfortably up to ~500M URLs before you need to think about sharding
