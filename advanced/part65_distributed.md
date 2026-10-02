# Part 65 - Distributed Systems Concepts in Nim

## บทนำ

Distributed systems คือระบบที่ประกอบด้วยหลาย nodes ทำงานร่วมกันผ่านเครือข่าย
เรียนรู้การสร้าง distributed components ด้วย Nim: circuit breaker, distributed lock,
service discovery, vector clocks, และ leader election

---

## 1. CAP Theorem (ทฤษฎีพื้นฐาน)

```nim
# cap_theorem.nim
# CAP Theorem: Consistency, Availability, Partition Tolerance
# เลือกได้แค่ 2 ใน 3 เสมอ

# CP systems (Consistency + Partition Tolerance): HBase, ZooKeeper, etcd
# - ข้อมูลถูกต้องเสมอ แต่อาจ unavailable ช่วง partition
# AP systems (Availability + Partition Tolerance): Cassandra, CouchDB, DynamoDB  
# - ข้อมูลอาจ stale แต่ always available
# CA systems (Consistency + Availability): ใช้ได้แค่ single-node (ไม่มี partition)

type
  ConsistencyLevel* = enum
    clStrong    # อ่านข้อมูลล่าสุดเสมอ (CP)
    clEventual  # อาจอ่านข้อมูลเก่า แต่ eventually consistent (AP)
    clCausal    # อ่านข้อมูลที่ causally consistent

  AvailabilityMode* = enum
    amAlwaysAvailable  # ตอบสนองทุก request แม้ partition (AP)
    amMayBlock         # อาจ block ระหว่าง partition (CP)
```

---

## 2. Circuit Breaker Pattern

```nim
# circuit_breaker.nim
# ป้องกัน cascading failures ใน microservices

import std/[times, asyncdispatch, httpclient, json, strformat]

type
  CircuitState* = enum
    csClosed    # ปกติ — request ผ่านได้
    csOpen      # trip — block ทุก request
    csHalfOpen  # ทดสอบ — ให้ผ่านแค่บาง request

  CircuitBreaker* = ref object
    name*: string
    state*: CircuitState
    failureCount*: int
    successCount*: int
    failureThreshold*: int   # จำนวนครั้งที่ fail ก่อน open
    successThreshold*: int   # จำนวนครั้งที่ success ก่อน close จาก half-open
    timeout*: Duration        # ระยะเวลาที่ open ก่อนเปลี่ยนเป็น half-open
    lastFailureTime*: Time
    halfOpenMaxCalls*: int
    halfOpenCalls*: int

proc newCircuitBreaker*(name: string, failureThreshold = 5,
    successThreshold = 2, timeoutSecs = 30): CircuitBreaker =
  CircuitBreaker(
    name: name,
    state: csClosed,
    failureThreshold: failureThreshold,
    successThreshold: successThreshold,
    timeout: initDuration(seconds = timeoutSecs),
    halfOpenMaxCalls: 3
  )

proc isOpen*(cb: CircuitBreaker): bool =
  if cb.state == csOpen:
    if getTime() - cb.lastFailureTime > cb.timeout:
      cb.state = csHalfOpen
      cb.halfOpenCalls = 0
      cb.successCount = 0
      echo &"[{cb.name}] Circuit -> HalfOpen"
      return false
    return true
  false

proc canRequest*(cb: CircuitBreaker): bool =
  case cb.state
  of csClosed: true
  of csOpen: not cb.isOpen()
  of csHalfOpen:
    cb.halfOpenCalls < cb.halfOpenMaxCalls

proc recordSuccess*(cb: CircuitBreaker) =
  case cb.state
  of csClosed:
    cb.failureCount = 0
  of csHalfOpen:
    inc cb.successCount
    if cb.successCount >= cb.successThreshold:
      cb.state = csClosed
      cb.failureCount = 0
      echo &"[{cb.name}] Circuit -> Closed (recovered)"
  of csOpen:
    discard

proc recordFailure*(cb: CircuitBreaker) =
  cb.lastFailureTime = getTime()
  case cb.state
  of csClosed:
    inc cb.failureCount
    if cb.failureCount >= cb.failureThreshold:
      cb.state = csOpen
      echo &"[{cb.name}] Circuit -> Open (failures: {cb.failureCount})"
  of csHalfOpen:
    cb.state = csOpen
    echo &"[{cb.name}] Circuit -> Open (half-open test failed)"
  of csOpen:
    discard

type CircuitOpenError* = object of CatchableError

proc callWithBreaker*[T](cb: CircuitBreaker, fn: proc(): Future[T]): Future[T] {.async.} =
  if not cb.canRequest():
    raise newException(CircuitOpenError, &"Circuit '{cb.name}' is open")
  if cb.state == csHalfOpen:
    inc cb.halfOpenCalls
  try:
    let result = await fn()
    cb.recordSuccess()
    return result
  except CatchableError as e:
    cb.recordFailure()
    raise

# ตัวอย่างการใช้งาน
proc makeHttpCall(url: string): Future[string] {.async.} =
  let client = newAsyncHttpClient()
  try:
    let resp = await client.getContent(url)
    return resp
  finally:
    client.close()

proc exampleUsage() {.async.} =
  let cb = newCircuitBreaker("payment-service", failureThreshold = 3)
  
  for i in 1..10:
    try:
      let result = await cb.callWithBreaker(proc(): Future[string] {.async.} =
        return await makeHttpCall("http://payment-service/charge")
      )
      echo &"Request {i}: OK"
    except CircuitOpenError:
      echo &"Request {i}: Circuit open, using fallback"
      # Fallback logic here
    except CatchableError as e:
      echo &"Request {i}: Error: {e.msg}"
    await sleepAsync(500)
```

---

## 3. Distributed Lock (Redis-based)

```nim
# distributed_lock.nim
# Distributed locking ผ่าน Redis ด้วย SET NX PX pattern
# ป้องกัน race condition ใน distributed environment

import std/[asyncdispatch, asyncnet, strformat, strutils, random, base64]
import std/[times, options]

type
  RedisClient* = ref object
    sock*: AsyncSocket
    host*: string
    port*: int

  DistributedLock* = ref object
    client*: RedisClient
    key*: string
    token*: string      # unique token ของเจ้าของ lock
    ttlMs*: int
    acquired*: bool

proc newRedisClient*(host = "127.0.0.1", port = 6379): RedisClient =
  RedisClient(host: host, port: port)

proc connect*(rc: RedisClient) {.async.} =
  rc.sock = newAsyncSocket()
  await rc.sock.connect(rc.host, Port(rc.port))

proc sendCommand*(rc: RedisClient, args: varargs[string]): Future[string] {.async.} =
  var cmd = &"*{args.len}\r\n"
  for arg in args:
    cmd &= &"${arg.len}\r\n{arg}\r\n"
  await rc.sock.send(cmd)
  
  var response = ""
  var line = await rc.sock.recvLine()
  response = line
  
  if line.startsWith("$"):
    let length = parseInt(line[1..^1])
    if length >= 0:
      let data = await rc.sock.recv(length + 2)  # +2 for \r\n
      response = data[0..^3]  # remove \r\n
  elif line.startsWith("*"):
    let count = parseInt(line[1..^1])
    for _ in 0..<count:
      let sizeLine = await rc.sock.recvLine()
      let size = parseInt(sizeLine[1..^1])
      let item = await rc.sock.recv(size + 2)
      response &= "\n" & item[0..^3]
  
  return response

proc generateToken(): string =
  randomize()
  var bytes: array[16, byte]
  for i in 0..<16:
    bytes[i] = byte(rand(255))
  encode(bytes)

proc acquire*(lock: DistributedLock, retries = 3, retryDelayMs = 100): Future[bool] {.async.} =
  ## ขอ lock — คืน true ถ้าได้ lock
  ## ใช้ SET key token NX PX ttl (atomic compare-and-set)
  for attempt in 0..<retries:
    let result = await lock.client.sendCommand(
      "SET", lock.key, lock.token,
      "NX",           # Only set if Not eXists
      "PX", $lock.ttlMs  # expire ใน milliseconds
    )
    
    if result == "OK":
      lock.acquired = true
      echo &"[Lock] Acquired '{lock.key}' (token: {lock.token[0..7]}...)"
      return true
    
    if attempt < retries - 1:
      await sleepAsync(retryDelayMs * (attempt + 1))  # exponential backoff
  
  echo &"[Lock] Failed to acquire '{lock.key}'"
  return false

proc release*(lock: DistributedLock): Future[bool] {.async.} =
  ## ปล่อย lock — ต้องตรวจว่า token ตรงก่อน (ป้องกัน release ของคนอื่น)
  ## ใช้ Lua script เพื่อ atomic check-and-delete
  let luaScript = """
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
"""
  let result = await lock.client.sendCommand("EVAL", luaScript, "1", lock.key, lock.token)
  let released = result == "1" or result == ":1"
  if released:
    lock.acquired = false
    echo &"[Lock] Released '{lock.key}'"
  else:
    echo &"[Lock] Release failed (token mismatch or already expired)"
  return released

proc withLock*(lock: DistributedLock, fn: proc(): Future[void]): Future[void] {.async.} =
  ## Helper: acquire lock, run fn, release lock
  let acquired = await lock.acquire()
  if not acquired:
    raise newException(IOError, &"Could not acquire lock: {lock.key}")
  try:
    await fn()
  finally:
    discard await lock.release()

proc newDistributedLock*(client: RedisClient, key: string, ttlMs = 30000): DistributedLock =
  DistributedLock(
    client: client,
    key: &"lock:{key}",
    token: generateToken(),
    ttlMs: ttlMs
  )

# ตัวอย่าง
proc exampleDistributedLock() {.async.} =
  let rc = newRedisClient()
  await rc.connect()
  
  let lock = newDistributedLock(rc, "critical-section", ttlMs = 5000)
  
  await lock.withLock(proc(): Future[void] {.async.} =
    echo "Executing critical section..."
    await sleepAsync(1000)
    echo "Critical section done"
  )
```

---

## 4. Service Registry & Discovery

```nim
# service_registry.nim
# Simple in-memory service registry with health checking

import std/[asyncdispatch, asynchttpserver, httpclient, json, tables]
import std/[times, strformat, sequtils, options]

type
  ServiceStatus* = enum
    ssHealthy
    ssUnhealthy
    ssUnknown

  ServiceInstance* = object
    id*: string
    name*: string
    host*: string
    port*: int
    tags*: seq[string]
    meta*: Table[string, string]
    status*: ServiceStatus
    lastCheck*: Time
    registeredAt*: Time

  ServiceRegistry* = ref object
    services*: Table[string, seq[ServiceInstance]]  # name -> instances
    checkInterval*: int  # seconds

proc newRegistry*(checkInterval = 10): ServiceRegistry =
  ServiceRegistry(
    services: initTable[string, seq[ServiceInstance]](),
    checkInterval: checkInterval
  )

proc register*(r: ServiceRegistry, inst: ServiceInstance) =
  if r.services.hasKey(inst.name):
    # ลบ instance เก่าถ้ามี id ซ้ำ
    r.services[inst.name] = r.services[inst.name].filterIt(it.id != inst.id)
    r.services[inst.name].add(inst)
  else:
    r.services[inst.name] = @[inst]
  echo &"[Registry] Registered {inst.name}/{inst.id} at {inst.host}:{inst.port}"

proc deregister*(r: ServiceRegistry, name, id: string) =
  if r.services.hasKey(name):
    r.services[name] = r.services[name].filterIt(it.id != id)
    echo &"[Registry] Deregistered {name}/{id}"

proc discover*(r: ServiceRegistry, name: string, onlyHealthy = true): seq[ServiceInstance] =
  if not r.services.hasKey(name):
    return @[]
  let instances = r.services[name]
  if onlyHealthy:
    return instances.filterIt(it.status == ssHealthy)
  return instances

proc pickRoundRobin*(r: ServiceRegistry, name: string): Option[ServiceInstance] =
  ## Simple round-robin load balancing
  let healthy = r.discover(name)
  if healthy.len == 0:
    return none(ServiceInstance)
  # Use time-based rotation (simple, stateless)
  let idx = int(epochTime().int) mod healthy.len
  return some(healthy[idx])

proc healthCheck*(r: ServiceRegistry) {.async.} =
  ## Background goroutine ตรวจสอบ health ทุก checkInterval วินาที
  let client = newAsyncHttpClient()
  client.timeout = 5000  # 5 second timeout
  
  while true:
    for name, instances in r.services.mpairs:
      for inst in instances.mitems:
        let url = &"http://{inst.host}:{inst.port}/health"
        try:
          let resp = await client.get(url)
          inst.status = if resp.code == Http200: ssHealthy else: ssUnhealthy
          inst.lastCheck = getTime()
        except:
          inst.status = ssUnhealthy
          inst.lastCheck = getTime()
    
    await sleepAsync(r.checkInterval * 1000)

proc toJson*(inst: ServiceInstance): JsonNode =
  %*{
    "id": inst.id,
    "name": inst.name,
    "host": inst.host,
    "port": inst.port,
    "status": $inst.status,
    "tags": inst.tags
  }

# Registry HTTP Server
proc startRegistryServer*(r: ServiceRegistry, port = 8500) {.async.} =
  let server = newAsyncHttpServer()
  
  proc handleRequest(req: Request) {.async.} =
    let path = req.url.path
    
    if path.startsWith("/v1/agent/service/register") and req.reqMethod == HttpPost:
      let body = parseJson(req.body)
      let inst = ServiceInstance(
        id: body["ID"].getStr(),
        name: body["Name"].getStr(),
        host: body["Address"].getStr("127.0.0.1"),
        port: body["Port"].getInt(),
        tags: body.getOrDefault("Tags").getElems().mapIt(it.getStr()),
        status: ssUnknown,
        registeredAt: getTime()
      )
      r.register(inst)
      await req.respond(Http200, "{}")
    
    elif path.startsWith("/v1/health/service/"):
      let name = path["/v1/health/service/".len..^1]
      let instances = r.discover(name)
      let response = %instances.mapIt(it.toJson())
      await req.respond(Http200, $response)
    
    else:
      await req.respond(Http404, "Not Found")
  
  asyncCheck r.healthCheck()
  echo &"[Registry] Server listening on :{port}"
  await server.serve(Port(port), handleRequest)
```

---

## 5. Vector Clocks (Causal Ordering)

```nim
# vector_clock.nim
# Vector clocks สำหรับ causal ordering ใน distributed systems

import std/[tables, strformat, algorithm]

type
  VectorClock* = Table[string, int]
  
  CausalRelation* = enum
    crBefore      # A happened before B
    crAfter       # A happened after B  
    crConcurrent  # Concurrent — no causal relationship
    crEqual       # Same clock values

proc newVectorClock*(): VectorClock = initTable[string, int]()

proc tick*(vc: var VectorClock, nodeId: string) =
  ## Increment clock for this node (called on local event)
  if vc.hasKey(nodeId):
    inc vc[nodeId]
  else:
    vc[nodeId] = 1

proc merge*(vc: var VectorClock, other: VectorClock) =
  ## Merge with received clock (take max of each component)
  for nodeId, val in other:
    if vc.hasKey(nodeId):
      vc[nodeId] = max(vc[nodeId], val)
    else:
      vc[nodeId] = val

proc receive*(vc: var VectorClock, other: VectorClock, nodeId: string) =
  ## Process received message: merge + tick own component
  vc.merge(other)
  vc.tick(nodeId)

proc compare*(a, b: VectorClock): CausalRelation =
  ## Compare two vector clocks
  var aLessOrEqual = true
  var bLessOrEqual = true
  
  # Check all keys in a
  for nodeId, aVal in a:
    let bVal = if b.hasKey(nodeId): b[nodeId] else: 0
    if aVal > bVal: bLessOrEqual = false
    if aVal < bVal: aLessOrEqual = false
  
  # Check keys in b not in a
  for nodeId, bVal in b:
    if not a.hasKey(nodeId) and bVal > 0:
      aLessOrEqual = false
  
  if aLessOrEqual and bLessOrEqual: crEqual
  elif aLessOrEqual: crBefore
  elif bLessOrEqual: crAfter
  else: crConcurrent

proc `$`*(vc: VectorClock): string =
  var parts: seq[string]
  for nodeId, val in vc:
    parts.add(&"{nodeId}:{val}")
  parts.sort()
  "{" & parts.join(", ") & "}"

# ตัวอย่าง: Distributed key-value store พร้อม vector clock
type
  VersionedValue* = object
    value*: string
    clock*: VectorClock

  CRDTStore* = ref object
    nodeId*: string
    data*: Table[string, VersionedValue]

proc newCRDTStore*(nodeId: string): CRDTStore =
  CRDTStore(nodeId: nodeId, data: initTable[string, VersionedValue]())

proc put*(store: CRDTStore, key, value: string) =
  var clock = if store.data.hasKey(key):
    store.data[key].clock
  else:
    newVectorClock()
  clock.tick(store.nodeId)
  store.data[key] = VersionedValue(value: value, clock: clock)
  echo &"[{store.nodeId}] PUT {key}={value} clock={clock}"

proc get*(store: CRDTStore, key: string): string =
  if store.data.hasKey(key):
    store.data[key].value
  else:
    ""

proc merge*(store: CRDTStore, key: string, incoming: VersionedValue) =
  ## Merge incoming value — Last Write Wins based on vector clock
  if not store.data.hasKey(key):
    store.data[key] = incoming
    echo &"[{store.nodeId}] MERGE {key}: new value '{incoming.value}'"
    return
  
  let existing = store.data[key]
  let relation = compare(incoming.clock, existing.clock)
  
  case relation
  of crAfter, crConcurrent:  # incoming is newer or concurrent (LWW: accept concurrent)
    store.data[key] = incoming
    echo &"[{store.nodeId}] MERGE {key}: updated to '{incoming.value}' ({relation})"
  of crBefore:  # incoming is older, ignore
    echo &"[{store.nodeId}] MERGE {key}: ignored stale value ({relation})"
  of crEqual:
    discard

when isMainModule:
  # Demonstrate vector clocks
  var clockA = newVectorClock()
  var clockB = newVectorClock()
  
  clockA.tick("A")
  clockA.tick("A")
  clockB.tick("B")
  
  echo &"Clock A: {clockA}"
  echo &"Clock B: {clockB}"
  echo &"Relation: {compare(clockA, clockB)}"  # concurrent
  
  clockB.merge(clockA)
  clockB.tick("B")
  echo &"Clock B after merge+tick: {clockB}"
  echo &"Relation: {compare(clockA, clockB)}"  # A before B
```

---

## 6. Leader Election (Bully Algorithm)

```nim
# leader_election.nim
# Bully Algorithm สำหรับ leader election ใน distributed cluster

import std/[asyncdispatch, asyncnet, json, tables, strformat, algorithm]
import std/[sequtils, options, times]

type
  NodeRole* = enum
    nrFollower
    nrCandidate
    nrLeader

  ElectionMessage* = object
    kind*: string     # "election", "ok", "coordinator"
    senderId*: int
    term*: int

  ClusterNode* = ref object
    id*: int
    role*: NodeRole
    currentLeader*: Option[int]
    peers*: seq[int]  # IDs of all peers
    alive*: bool
    term*: int
    electionInProgress*: bool
    lastHeartbeat*: Table[int, Time]

proc newClusterNode*(id: int, peers: seq[int]): ClusterNode =
  ClusterNode(
    id: id,
    role: nrFollower,
    peers: peers,
    alive: true,
    term: 0,
    lastHeartbeat: initTable[int, Time]()
  )

proc higherPeers*(node: ClusterNode): seq[int] =
  ## Peers with higher ID (higher priority in bully algorithm)
  node.peers.filterIt(it > node.id)

proc lowerPeers*(node: ClusterNode): seq[int] =
  ## Peers with lower ID
  node.peers.filterIt(it < node.id)

proc startElection*(node: ClusterNode, cluster: Table[int, ClusterNode]) =
  ## Start election: send ELECTION to higher-priority nodes
  if node.electionInProgress:
    return
  
  node.electionInProgress = true
  inc node.term
  echo &"[Node {node.id}] Starting election for term {node.term}"
  
  let higher = node.higherPeers()
  
  if higher.len == 0:
    # No higher peers — this node wins immediately
    node.becomeLeader(cluster)
    return
  
  # Send ELECTION to all higher peers
  var receivedOK = false
  for peerId in higher:
    if cluster.hasKey(peerId) and cluster[peerId].alive:
      echo &"[Node {node.id}] -> ELECTION to Node {peerId}"
      # Simulate: if peer is alive, it responds with OK
      receivedOK = true
      cluster[peerId].receiveElection(node.id, node.term, cluster)
  
  if not receivedOK:
    # All higher peers are down
    node.becomeLeader(cluster)

proc receiveElection*(node: ClusterNode, fromId, term: int, cluster: Table[int, ClusterNode]) =
  ## Received ELECTION from lower-priority node
  echo &"[Node {node.id}] Received ELECTION from Node {fromId}"
  # Send OK back
  echo &"[Node {node.id}] -> OK to Node {fromId}"
  # Start own election since we have higher priority
  if not node.electionInProgress:
    node.startElection(cluster)

proc becomeLeader*(node: ClusterNode, cluster: Table[int, ClusterNode]) =
  node.role = nrLeader
  node.currentLeader = some(node.id)
  node.electionInProgress = false
  echo &"[Node {node.id}] *** ELECTED LEADER for term {node.term} ***"
  
  # Broadcast COORDINATOR message to all peers
  for peerId in node.peers:
    if cluster.hasKey(peerId) and cluster[peerId].alive:
      cluster[peerId].receiveCoordinator(node.id, node.term)

proc receiveCoordinator*(node: ClusterNode, leaderId, term: int) =
  node.role = nrFollower
  node.currentLeader = some(leaderId)
  node.electionInProgress = false
  node.term = term
  echo &"[Node {node.id}] Acknowledged leader: Node {leaderId} (term {term})"

# Raft-inspired heartbeat mechanism  
proc sendHeartbeat*(leader: ClusterNode, cluster: Table[int, ClusterNode]) =
  if leader.role != nrLeader:
    return
  for peerId in leader.peers:
    if cluster.hasKey(peerId):
      cluster[peerId].lastHeartbeat[leader.id] = getTime()

proc checkLeaderAlive*(node: ClusterNode, cluster: Table[int, ClusterNode],
    timeoutSecs = 5): bool =
  if node.currentLeader.isNone:
    return false
  let leaderId = node.currentLeader.get()
  if not node.lastHeartbeat.hasKey(leaderId):
    return false
  getTime() - node.lastHeartbeat[leaderId] < initDuration(seconds = timeoutSecs)

when isMainModule:
  # Simulate 5-node cluster
  var cluster: Table[int, ClusterNode]
  let allIds = @[1, 2, 3, 4, 5]
  
  for id in allIds:
    let peers = allIds.filterIt(it != id)
    cluster[id] = newClusterNode(id, peers)
  
  echo "=== Initial cluster ==="
  
  # Simulate node 5 (highest) crashing
  echo "\n=== Node 5 (leader) crashes ==="
  cluster[5].alive = false
  
  # Node 1 detects leader failure and starts election
  echo "\n=== Node 1 starts election ==="
  cluster[1].startElection(cluster)
  
  echo "\n=== Final state ==="
  for id, node in cluster:
    if node.alive:
      echo &"Node {id}: role={node.role} leader={node.currentLeader}"
```

---

## 7. Distributed Rate Limiter

```nim
# distributed_rate_limiter.nim
# Sliding window rate limiter ที่ใช้ Redis

import std/[asyncdispatch, asyncnet, strformat, times, strutils]

type
  RateLimiter* = ref object
    client*: RedisClient  # จาก distributed_lock.nim
    keyPrefix*: string
    maxRequests*: int
    windowSecs*: int

  RateLimitResult* = object
    allowed*: bool
    remaining*: int
    resetAt*: Time
    retryAfter*: int  # seconds

proc newRateLimiter*(client: RedisClient, keyPrefix = "ratelimit",
    maxRequests = 100, windowSecs = 60): RateLimiter =
  RateLimiter(
    client: client,
    keyPrefix: keyPrefix,
    maxRequests: maxRequests,
    windowSecs: windowSecs
  )

proc checkLimit*(rl: RateLimiter, identifier: string): Future[RateLimitResult] {.async.} =
  ## Sliding window rate limit check
  ## ใช้ Redis sorted set: member=timestamp, score=timestamp
  let key = &"{rl.keyPrefix}:{identifier}"
  let now = epochTime()
  let windowStart = now - float(rl.windowSecs)
  
  # Lua script สำหรับ atomic sliding window check
  let luaScript = """
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window_start = tonumber(ARGV[2])
local max_requests = tonumber(ARGV[3])
local window_secs = tonumber(ARGV[4])

-- Remove old entries outside window
redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)

-- Count current requests in window
local count = redis.call('ZCARD', key)

if count < max_requests then
    -- Add current request
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, window_secs)
    return {1, max_requests - count - 1, 0}
else
    -- Get oldest entry to calculate retry-after
    local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
    local retry_after = 0
    if #oldest > 0 then
        retry_after = math.ceil(tonumber(oldest[2]) + window_secs - now)
    end
    return {0, 0, retry_after}
end
"""
  
  # In real code, use actual Redis Lua eval
  # Simplified simulation:
  let count = rl.maxRequests - 10  # placeholder
  
  if count < rl.maxRequests:
    return RateLimitResult(
      allowed: true,
      remaining: rl.maxRequests - count - 1,
      resetAt: fromUnix(int64(now) + rl.windowSecs)
    )
  else:
    return RateLimitResult(
      allowed: false,
      remaining: 0,
      resetAt: fromUnix(int64(now) + rl.windowSecs),
      retryAfter: rl.windowSecs
    )

# HTTP Middleware wrapper
proc rateLimitMiddleware*(rl: RateLimiter, identifier: string,
    onLimited: proc()): Future[bool] {.async.} =
  let result = await rl.checkLimit(identifier)
  if not result.allowed:
    onLimited()
    return false
  return true
```

---

## 8. สรุป

| Pattern | Use Case | Trade-off |
|---------|----------|-----------|
| Circuit Breaker | ป้องกัน cascading failures | Latency ช่วง half-open |
| Distributed Lock | Critical section protection | Single point of failure (Redis) |
| Service Registry | Service discovery | Registry must be highly available |
| Vector Clock | Causal consistency | Overhead per message |
| Leader Election | Coordination | Election latency |
| Rate Limiter | API protection | Redis dependency |

---

**Next**: [Part 66 - Message Queues & Event-Driven Architecture](../advanced/part66_message_queues.md)
