# Part 48: Redis Caching ใน Nim

## Redis คืออะไร

```
Redis = Remote Dictionary Server
- In-memory data structure store
- ใช้เป็น: cache, session store, pub/sub, rate limiting
- ข้อมูลพักใน RAM = เร็วมาก
- รองรับ type: string, hash, list, set, sorted set, bitmap

Nim package: nimble install redis
```

## เชื่อมต่อ Redis

```nim
# redis_basics.nim
import redis, asyncdispatch, json, strutils

# Sync client
proc syncDemo() =
  let r = openRedis("localhost", 6379)
  defer: r.close()
  
  # String operations
  discard r.setk("name", "Alice")
  let name = r.get("name")
  echo "name: ", name.get("")
  
  # Set with expiration (TTL in seconds)
  discard r.setex("session:abc123", 3600, "user:42")
  let ttl = r.ttl("session:abc123")
  echo "TTL: ", ttl, " seconds"
  
  # Increment
  discard r.incr("page_views")
  discard r.incrby("page_views", 10)
  let views = r.get("page_views")
  echo "Views: ", views.get("0")
  
  # Delete
  discard r.del("name")
  
  # Check existence
  let exists = r.exists("page_views")
  echo "Exists: ", exists

syncDemo()

# Async client
proc asyncDemo() {.async.} =
  let r = await openAsyncRedis("localhost", 6379)
  defer: r.close()
  
  await r.setk("async_key", "hello")
  let val = await r.get("async_key")
  echo "Async get: ", val.get("")

waitFor asyncDemo()
```

## Caching Pattern

```nim
# cache_aside.nim
# Cache-aside pattern: ดึงจาก cache ก่อน, ถ้าไม่มี query DB

import redis, db_postgres, json, options, times

type
  UserService = ref object
    db: DbConn
    cache: Redis
    cacheTTL: int

proc newUserService(db: DbConn, cache: Redis, ttl: int = 300): UserService =
  UserService(db: db, cache: cache, cacheTTL: ttl)

proc getUserById(svc: UserService, id: int): Option[JsonNode] =
  let cacheKey = "user:" & $id
  
  # 1. Check cache first
  let cached = svc.cache.get(cacheKey)
  if cached.isSome:
    echo "[CACHE HIT] user:", id
    return some(parseJson(cached.get()))
  
  echo "[CACHE MISS] user:", id
  
  # 2. Query database
  let row = svc.db.getRow(sql"SELECT id, username, email FROM users WHERE id = ?", $id)
  if row[0].len == 0:
    return none(JsonNode)
  
  let user = %*{
    "id": parseInt(row[0]),
    "username": row[1],
    "email": row[2]
  }
  
  # 3. Store in cache
  discard svc.cache.setex(cacheKey, svc.cacheTTL, $user)
  
  some(user)

proc invalidateUser(svc: UserService, id: int) =
  discard svc.cache.del("user:" & $id)

proc updateUser(svc: UserService, id: int, email: string) =
  # Update DB
  svc.db.exec(sql"UPDATE users SET email = ? WHERE id = ?", email, $id)
  # Invalidate cache
  svc.invalidateUser(id)
```

## Redis Hash สำหรับ Session

```nim
# redis_session.nim

import redis, strutils, random, times, json

type
  RedisSessionStore = ref object
    redis: Redis
    prefix: string
    ttl: int

proc newRedisSessionStore(r: Redis, prefix = "sess:", ttl = 86400): RedisSessionStore =
  RedisSessionStore(redis: r, prefix: prefix, ttl: ttl)

proc generateId(): string =
  randomize()
  var id = ""
  for i in 0..<32:
    id &= toHex(rand(255), 2)
  id.toLowerAscii()

proc create(store: RedisSessionStore, data: JsonNode): string =
  let id = generateId()
  let key = store.prefix & id
  
  # Store each field as hash
  for k, v in data.pairs:
    discard store.redis.hset(key, k, $v)
  
  # Set expiration
  discard store.redis.expire(key, store.ttl)
  id

proc get(store: RedisSessionStore, id: string): Option[JsonNode] =
  import options
  let key = store.prefix & id
  
  # Check existence
  if store.redis.exists(key) == 0:
    return none(JsonNode)
  
  # Get all fields
  let all = store.redis.hgetall(key)
  var result = newJObject()
  var i = 0
  while i < all.len - 1:
    result[all[i]] = newJString(all[i+1])
    i += 2
  
  some(result)

proc set(store: RedisSessionStore, id, field, value: string) =
  let key = store.prefix & id
  discard store.redis.hset(key, field, value)

proc delete(store: RedisSessionStore, id: string) =
  discard store.redis.del(store.prefix & id)

proc refresh(store: RedisSessionStore, id: string) =
  discard store.redis.expire(store.prefix & id, store.ttl)

# Usage
let r = openRedis("localhost", 6379)
let store = newRedisSessionStore(r)

let sessionId = store.create(%*{"userId": "42", "role": "admin"})
echo "Session: ", sessionId

let session = store.get(sessionId)
if session.isSome:
  echo "User: ", session.get()["userId"]
```

## Rate Limiting

```nim
# rate_limiter.nim
# กำหนดจำนวน request ต่อวินาที

import redis, times, strutils, asynchttpserver, asyncdispatch

type RateLimiter = ref object
  redis: Redis
  maxRequests: int
  windowSeconds: int

proc newRateLimiter(r: Redis, maxReq = 100, windowSec = 60): RateLimiter =
  RateLimiter(redis: r, maxRequests: maxReq, windowSeconds: windowSec)

proc isAllowed(rl: RateLimiter, identifier: string): (bool, int) =
  ## Returns (allowed, remaining)
  let key = "ratelimit:" & identifier
  let now = epochTime().int
  let windowStart = now - rl.windowSeconds
  
  # Sliding window using sorted set
  # Remove old requests
  discard rl.redis.zremrangebyscore(key, "-inf", $windowStart)
  
  # Count current requests
  let count = rl.redis.zcard(key)
  
  if count >= rl.maxRequests:
    return (false, 0)
  
  # Add current request
  discard rl.redis.zadd(key, @[$(now.float + count.float * 0.001), $now])
  discard rl.redis.expire(key, rl.windowSeconds)
  
  (true, rl.maxRequests - count - 1)

template withRateLimit(req: Request, limiter: RateLimiter, body: untyped) =
  let ip = req.headers.getOrDefault("X-Real-IP", req.hostname)
  let (allowed, remaining) = limiter.isAllowed(ip)
  
  if not allowed:
    await req.respond(Http429, """{"error":"Too Many Requests"}""",
      newHttpHeaders({
        "Content-Type": "application/json",
        "Retry-After": $limiter.windowSeconds
      }))
  else:
    let extraHeaders = newHttpHeaders({"X-RateLimit-Remaining": $remaining})
    body
```

## Pub/Sub

```nim
# pubsub.nim
# Redis publish/subscribe

import redis, asyncdispatch, strutils

# Publisher
proc publishEvent(r: Redis, channel, message: string) =
  let receivers = r.publish(channel, message)
  echo "Published to ", receivers, " subscribers"

# Subscriber
proc subscribeToChannel(host: string, channel: string,
                         handler: proc(msg: string)) {.async.} =
  # Use separate connection for subscribe
  let subRedis = openRedis(host, 6379)
  subRedis.subscribe(channel)
  
  while true:
    let msg = subRedis.receiveMessage()
    if msg.kind == "message":
      handler(msg.message)

# Example: real-time notifications
proc publishNotification(r: Redis, userId: int, message: string) =
  let channel = "notifications:" & $userId
  let payload = $(% *{"type": "notification", "message": message, "time": $now()})
  publishEvent(r, channel, payload)

let r = openRedis("localhost", 6379)
publishNotification(r, 42, "Your order is ready!")
```

## สรุป Part 48

- ␅ Redis basics (get/set/expire)
- ␅ Cache-aside pattern
- ␅ Redis Hash สำหรับ session storage
- ␅ Rate limiting ด้วย sliding window
- ␅ Pub/Sub notifications

---
**Next**: [Part 49 - GraphQL](part49_graphql.md)
