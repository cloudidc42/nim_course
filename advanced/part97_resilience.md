# Part 97 - Resilience Engineering

## บทนำ

Resilience Engineering — สร้างระบบที่ทนต่อความผิดพลาด:
- Circuit breaker
- Bulkhead pattern
- Chaos engineering
- Health monitoring

---

## 1. Circuit Breaker

```nim
# circuit_breaker.nim
# Circuit breaker pattern

import std/[asyncdispatch, times, tables, strformat, atomics]

type
  CircuitState* = enum
    csClosed    # Normal operation
    csOpen      # Blocking calls
    csHalfOpen  # Testing recovery

  CircuitBreaker* = ref object
    name*: string
    state*: CircuitState
    failureCount*: int
    successCount*: int
    lastFailureTime*: Time
    lastStateChange*: Time
    
    # Config
    failureThreshold*: int     # Failures before opening
    successThreshold*: int     # Successes to close from half-open
    timeout*: Duration         # Time before half-open
    
    # Stats
    totalCalls*: int
    totalFailures*: int
    totalSuccesses*: int

proc newCircuitBreaker*(
  name: string,
  failureThreshold = 5,
  successThreshold = 2,
  timeout = initDuration(seconds = 60)
): CircuitBreaker =
  CircuitBreaker(
    name: name,
    state: csClosed,
    failureThreshold: failureThreshold,
    successThreshold: successThreshold,
    timeout: timeout,
    lastStateChange: now().toTime()
  )

proc canCall*(cb: CircuitBreaker): bool =
  case cb.state
  of csClosed: true
  of csOpen:
    let elapsed = now().toTime() - cb.lastStateChange
    elapsed >= cb.timeout
  of csHalfOpen: true

proc recordSuccess*(cb: CircuitBreaker) =
  inc cb.totalCalls
  inc cb.totalSuccesses
  
  case cb.state
  of csClosed:
    cb.failureCount = 0
  of csHalfOpen:
    inc cb.successCount
    if cb.successCount >= cb.successThreshold:
      cb.state = csClosed
      cb.successCount = 0
      cb.failureCount = 0
      cb.lastStateChange = now().toTime()
      echo &"[CB:{cb.name}] Closed (recovered)"
  of csOpen: discard

proc recordFailure*(cb: CircuitBreaker) =
  inc cb.totalCalls
  inc cb.totalFailures
  inc cb.failureCount
  cb.lastFailureTime = now().toTime()
  
  case cb.state
  of csClosed:
    if cb.failureCount >= cb.failureThreshold:
      cb.state = csOpen
      cb.lastStateChange = now().toTime()
      echo &"[CB:{cb.name}] Opened (failures={cb.failureCount})"
  of csHalfOpen:
    cb.state = csOpen
    cb.successCount = 0
    cb.lastStateChange = now().toTime()
    echo &"[CB:{cb.name}] Reopened"
  of csOpen: discard

proc transition*(cb: CircuitBreaker) =
  if cb.state == csOpen:
    let elapsed = now().toTime() - cb.lastStateChange
    if elapsed >= cb.timeout:
      cb.state = csHalfOpen
      cb.successCount = 0
      cb.lastStateChange = now().toTime()
      echo &"[CB:{cb.name}] Half-open (testing)"

type CircuitOpenError* = object of CatchableError

proc call*[T](cb: CircuitBreaker, fn: proc(): Future[T]): Future[T] {.async.} =
  cb.transition()
  
  if not cb.canCall():
    raise newException(CircuitOpenError, &"Circuit {cb.name} is open")
  
  try:
    result = await fn()
    cb.recordSuccess()
  except Exception as e:
    cb.recordFailure()
    raise

proc stats*(cb: CircuitBreaker): string =
  &"CB[{cb.name}] state={cb.state} calls={cb.totalCalls} " &
  &"failures={cb.totalFailures} failRate={cb.totalFailures * 100 div max(1, cb.totalCalls)}%"

when isMainModule:
  let cb = newCircuitBreaker("database", failureThreshold = 3, timeout = initDuration(seconds = 5))
  
  proc callDb(): Future[string] {.async.} =
    # Simulate failure
    raise newException(IOError, "DB connection refused")
  
  proc callDbOk(): Future[string] {.async.} =
    return "ok"
  
  # Trip the breaker
  for i in 1..5:
    try:
      discard waitFor cb.call(callDb)
    except:
      discard
  
  echo cb.stats()
  
  # Try while open
  try:
    discard waitFor cb.call(callDbOk)
  except CircuitOpenError as e:
    echo "Blocked: ", e.msg
```

---

## 2. Bulkhead Pattern

```nim
# bulkhead.nim
# Isolate resources with bulkheads

import std/[asyncdispatch, asyncfutures, times, strformat]

type
  BulkheadConfig* = object
    maxConcurrent*: int      # Max concurrent calls
    maxWaiting*: int         # Max queued calls
    timeout*: Duration       # Call timeout

  Bulkhead* = ref object
    name*: string
    config*: BulkheadConfig
    active*: int
    waiting*: int
    totalRejected*: int
    totalTimeout*: int
    queue*: seq[Future[void]]

  BulkheadFullError* = object of CatchableError
  BulkheadTimeoutError* = object of CatchableError

proc newBulkhead*(name: string, maxConcurrent = 10, maxWaiting = 20,
                   timeout = initDuration(seconds = 30)): Bulkhead =
  Bulkhead(
    name: name,
    config: BulkheadConfig(
      maxConcurrent: maxConcurrent,
      maxWaiting: maxWaiting,
      timeout: timeout
    )
  )

proc call*[T](bh: Bulkhead, fn: proc(): Future[T]): Future[T] {.async.} =
  if bh.active >= bh.config.maxConcurrent:
    if bh.waiting >= bh.config.maxWaiting:
      inc bh.totalRejected
      raise newException(BulkheadFullError,
        &"Bulkhead {bh.name} full (active={bh.active}, waiting={bh.waiting})")
    
    # Queue the call
    inc bh.waiting
    let waitFut = newFuture[void]("bulkhead.wait")
    bh.queue.add(waitFut)
    
    try:
      let timeoutFut = sleepAsync(bh.config.timeout.inMilliseconds.int)
      await waitFut or timeoutFut
      
      if not waitFut.finished:
        inc bh.totalTimeout
        dec bh.waiting
        raise newException(BulkheadTimeoutError, &"Bulkhead {bh.name} wait timeout")
    except CancelledError:
      dec bh.waiting
      raise
    
    dec bh.waiting
  
  inc bh.active
  try:
    result = await fn()
  finally:
    dec bh.active
    # Release next waiter
    if bh.queue.len > 0:
      let next = bh.queue[0]
      bh.queue.delete(0)
      next.complete()

# Thread-pool bulkhead
type
  ThreadBulkhead* = ref object
    name*: string
    maxThreads*: int
    activeThreads*: int
    queue*: seq[proc() {.thread.}]

proc stats*(bh: Bulkhead): string =
  &"Bulkhead[{bh.name}] active={bh.active}/{bh.config.maxConcurrent} " &
  &"waiting={bh.waiting}/{bh.config.maxWaiting} " &
  &"rejected={bh.totalRejected} timeout={bh.totalTimeout}"

when isMainModule:
  let bh = newBulkhead("api", maxConcurrent = 3, maxWaiting = 5)
  
  proc slowCall(): Future[string] {.async.} =
    await sleepAsync(100)
    return "done"
  
  # Launch more calls than concurrency allows
  var futs: seq[Future[string]]
  for i in 0..4:
    futs.add(bh.call(slowCall))
  
  for f in futs:
    try:
      echo "Result: ", waitFor f
    except Exception as e:
      echo "Error: ", e.msg
  
  echo bh.stats()
```

---

## 3. Chaos Engineering

```nim
# chaos.nim
# Inject failures for resilience testing

import std/[asyncdispatch, random, times, strformat, tables]

type
  FailureType* = enum
    ftLatency    # Add delay
    ftError      # Raise exception
    ftPartial    # Return partial/corrupted data
    ftTimeout    # Never respond

  ChaosRule* = object
    name*: string
    probability*: float  # 0.0 - 1.0
    failure*: FailureType
    latencyMs*: int      # For ftLatency
    errorMsg*: string    # For ftError

  ChaosConfig* = object
    enabled*: bool
    rules*: seq[ChaosRule]
    seed*: int64

  ChaosMiddleware* = ref object
    config*: ChaosConfig
    rng*: Rand
    triggered*: Table[string, int]  # rule name -> count

proc newChaosMiddleware*(config: ChaosConfig): ChaosMiddleware =
  ChaosMiddleware(
    config: config,
    rng: initRand(config.seed),
    triggered: initTable[string, int]()
  )

proc shouldTrigger*(cm: ChaosMiddleware, rule: ChaosRule): bool =
  cm.rng.rand(1.0) < rule.probability

proc inject*[T](cm: ChaosMiddleware, fn: proc(): Future[T]): Future[T] {.async.} =
  if not cm.config.enabled:
    return await fn()
  
  # Check rules
  for rule in cm.config.rules:
    if cm.shouldTrigger(rule):
      cm.triggered.mgetOrPut(rule.name, 0).inc()
      
      case rule.failure
      of ftLatency:
        echo &"[CHAOS] Injecting {rule.latencyMs}ms latency"
        await sleepAsync(rule.latencyMs)
      
      of ftError:
        echo &"[CHAOS] Injecting error: {rule.errorMsg}"
        raise newException(IOError, rule.errorMsg)
      
      of ftPartial:
        echo "[CHAOS] Injecting partial failure"
        # For demonstration, proceed normally but could corrupt data
      
      of ftTimeout:
        echo "[CHAOS] Injecting timeout (30s)"
        await sleepAsync(30_000)
  
  result = await fn()

# Fault injection decorators
type
  FaultInjector* = ref object
    latencyProb*: float
    latencyMs*: int
    errorProb*: float
    rng*: Rand

proc newFaultInjector*(
  latencyProb = 0.1, latencyMs = 100,
  errorProb = 0.05, seed: int64 = 0
): FaultInjector =
  FaultInjector(
    latencyProb: latencyProb,
    latencyMs: latencyMs,
    errorProb: errorProb,
    rng: initRand(seed)
  )

template withFaults*(fi: FaultInjector, name: string, body: untyped): untyped =
  if fi.rng.rand(1.0) < fi.errorProb:
    raise newException(IOError, &"Injected fault in {name}")
  
  if fi.rng.rand(1.0) < fi.latencyProb:
    await sleepAsync(fi.latencyMs)
  
  body

# Resilience testing framework
type
  ResilienceTest* = ref object
    name*: string
    iterations*: int
    faultInjector*: FaultInjector
    results*: seq[bool]
    errors*: seq[string]

proc run*[T](rt: ResilienceTest, fn: proc(): Future[T]): Future[void] {.async.} =
  for i in 0..<rt.iterations:
    try:
      discard await fn()
      rt.results.add(true)
    except Exception as e:
      rt.results.add(false)
      rt.errors.add(e.msg)

proc report*(rt: ResilienceTest): string =
  let total = rt.results.len
  let successes = rt.results.filterIt(it).len
  let rate = successes * 100 div max(1, total)
  &"ResilienceTest[{rt.name}]: {successes}/{total} ({rate}%) success"

when isMainModule:
  let config = ChaosConfig(
    enabled: true,
    seed: 42,
    rules: @[
      ChaosRule(name: "latency", probability: 0.3, failure: ftLatency, latencyMs: 50),
      ChaosRule(name: "error", probability: 0.1, failure: ftError, errorMsg: "Chaos error")
    ]
  )
  
  let chaos = newChaosMiddleware(config)
  
  proc myService(): Future[string] {.async.} =
    return "service response"
  
  var successCount = 0
  for i in 0..9:
    try:
      let result = waitFor chaos.inject(myService)
      inc successCount
    except Exception as e:
      echo &"Call {i}: {e.msg}"
  
  echo &"Success: {successCount}/10"
  echo "Triggered: ", chaos.triggered
```

---

## 4. Health Monitoring

```nim
# health_monitor.nim
# System health checks

import std/[asyncdispatch, httpclient, times, strformat, tables, json]

type
  HealthStatus* = enum
    hsHealthy
    hsDegraded
    hsUnhealthy

  CheckResult* = object
    name*: string
    status*: HealthStatus
    message*: string
    duration*: Duration
    timestamp*: DateTime

  HealthCheck* = proc(): Future[CheckResult] {.async.}

  HealthMonitor* = ref object
    checks*: Table[string, HealthCheck]
    results*: Table[string, CheckResult]
    interval*: int  # milliseconds

proc newHealthMonitor*(interval = 30_000): HealthMonitor =
  HealthMonitor(interval: interval)

proc add*(hm: HealthMonitor, name: string, check: HealthCheck) =
  hm.checks[name] = check

proc runAll*(hm: HealthMonitor): Future[Table[string, CheckResult]] {.async.} =
  for name, check in hm.checks:
    let start = now()
    try:
      let result = await check()
      hm.results[name] = CheckResult(
        name: name,
        status: result.status,
        message: result.message,
        duration: now() - start,
        timestamp: now()
      )
    except Exception as e:
      hm.results[name] = CheckResult(
        name: name,
        status: hsUnhealthy,
        message: e.msg,
        duration: now() - start,
        timestamp: now()
      )
  
  result = hm.results

proc overallStatus*(hm: HealthMonitor): HealthStatus =
  var hasUnhealthy = false
  var hasDegraded = false
  
  for _, r in hm.results:
    case r.status
    of hsUnhealthy: hasUnhealthy = true
    of hsDegraded: hasDegraded = true
    of hsHealthy: discard
  
  if hasUnhealthy: hsUnhealthy
  elif hasDegraded: hsDegraded
  else: hsHealthy

proc toJson*(hm: HealthMonitor): JsonNode =
  result = %*{
    "status": $hm.overallStatus(),
    "checks": {}
  }
  
  for name, r in hm.results:
    result["checks"][name] = %*{
      "status": $r.status,
      "message": r.message,
      "durationMs": r.duration.inMilliseconds
    }

# Built-in health checks
proc httpCheck*(url: string, expectedStatus = 200): HealthCheck =
  return proc(): Future[CheckResult] {.async.} =
    let client = newAsyncHttpClient()
    defer: client.close()
    
    try:
      let resp = await client.get(url)
      let status = if resp.code.int == expectedStatus: hsHealthy else: hsDegraded
      return CheckResult(
        status: status,
        message: &"HTTP {resp.code.int}"
      )
    except Exception as e:
      return CheckResult(status: hsUnhealthy, message: e.msg)

proc memoryCheck*(maxUsageMb = 512): HealthCheck =
  return proc(): Future[CheckResult] {.async.} =
    when defined(linux):
      try:
        let content = readFile("/proc/self/status")
        for line in content.splitLines():
          if line.startsWith("VmRSS:"):
            let parts = line.split()
            if parts.len >= 2:
              let kb = parseInt(parts[1])
              let mb = kb div 1024
              let status = if mb < maxUsageMb: hsHealthy
                          elif mb < maxUsageMb * 2: hsDegraded
                          else: hsUnhealthy
              return CheckResult(
                status: status,
                message: &"{mb}MB / {maxUsageMb}MB"
              )
      except: discard
    
    return CheckResult(status: hsHealthy, message: "Memory check unavailable")

when isMainModule:
  let monitor = newHealthMonitor(interval = 10_000)
  
  monitor.add("memory", memoryCheck(maxUsageMb = 256))
  
  # Add custom check
  monitor.add("custom", proc(): Future[CheckResult] {.async.} =
    CheckResult(status: hsHealthy, message: "All good")
  )
  
  let results = waitFor monitor.runAll()
  
  echo "Health check results:"
  for name, r in results:
    let sym = case r.status
              of hsHealthy: "[OK]"
              of hsDegraded: "[WARN]"
              of hsUnhealthy: "[FAIL]"
    echo &"  {sym} {name}: {r.message} ({r.duration.inMilliseconds}ms)"
  
  echo "\nOverall: ", monitor.overallStatus()
  echo "\nJSON:\n", monitor.toJson().pretty()
```

---

## สรุป

| Pattern | Purpose |
|---------|---------|
| Circuit Breaker | Stop cascade failures |
| Bulkhead | Resource isolation |
| Chaos Engineering | Test failure scenarios |
| Health Monitoring | Detect degradation early |

---

**Next**: [Part 98 - Async Patterns](../advanced/part98_async.md)
