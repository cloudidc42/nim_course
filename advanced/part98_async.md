# Part 98 - Advanced Async Patterns

## บทนำ

Advanced async patterns ใน Nim:
- Structured concurrency
- Cancellation
- Async streams
- Backpressure

---

## 1. Structured Concurrency

```nim
# structured_concurrency.nim
# Scoped concurrent tasks

import std/[asyncdispatch, asyncfutures, times, strformat, sequtils]

# Task group — all tasks must complete/cancel before group exits
type
  TaskError* = object of CatchableError
  
  TaskResult*[T] = object
    case succeeded*: bool
    of true: value*: T
    of false: error*: string

  TaskGroup*[T] = ref object
    tasks*: seq[Future[T]]
    cancelled*: bool
    timeout*: Duration
    errors*: seq[string]

proc newTaskGroup*[T](timeout = initDuration(seconds = 30)): TaskGroup[T] =
  TaskGroup[T](timeout: timeout)

proc spawn*[T](tg: TaskGroup[T], fn: proc(): Future[T]) =
  if not tg.cancelled:
    tg.tasks.add(fn())

proc cancelAll*[T](tg: TaskGroup[T]) =
  tg.cancelled = true
  for t in tg.tasks:
    if not t.finished:
      t.fail(newException(TaskError, "Cancelled"))

proc waitAll*[T](tg: TaskGroup[T]): Future[seq[TaskResult[T]]] {.async.} =
  let timeoutFut = sleepAsync(tg.timeout.inMilliseconds.int)
  let allFut = all(tg.tasks)
  
  await allFut or timeoutFut
  
  if not allFut.finished:
    tg.cancelAll()
    raise newException(TaskError, "Task group timed out")
  
  result = newSeq[TaskResult[T]](tg.tasks.len)
  for i, t in tg.tasks:
    if t.failed:
      result[i] = TaskResult[T](succeeded: false, error: t.error.msg)
    else:
      result[i] = TaskResult[T](succeeded: true, value: t.read())

# Nursery pattern — fail-fast on any child failure
type
  Nursery*[T] = ref object
    tasks*: seq[Future[T]]
    failFast*: bool

proc newNursery*[T](failFast = true): Nursery[T] =
  Nursery[T](failFast: failFast)

proc spawn*[T](n: Nursery[T], fn: proc(): Future[T]) =
  n.tasks.add(fn())

proc join*[T](n: Nursery[T]): Future[seq[T]] {.async.} =
  if n.failFast:
    # Cancel all if any fails
    let combined = all(n.tasks)
    try:
      result = await combined
    except Exception as e:
      for t in n.tasks:
        if not t.finished: t.fail(e)
      raise
  else:
    result = await all(n.tasks)

# Semaphore for limiting concurrency
type
  AsyncSemaphore* = ref object
    permits*: int
    waiting*: seq[Future[void]]

proc newAsyncSemaphore*(permits: int): AsyncSemaphore =
  AsyncSemaphore(permits: permits)

proc acquire*(sem: AsyncSemaphore): Future[void] {.async.} =
  if sem.permits > 0:
    dec sem.permits
    return
  
  let waiter = newFuture[void]("semaphore.acquire")
  sem.waiting.add(waiter)
  await waiter

proc release*(sem: AsyncSemaphore) =
  if sem.waiting.len > 0:
    let next = sem.waiting[0]
    sem.waiting.delete(0)
    next.complete()
  else:
    inc sem.permits

template withSemaphore*(sem: AsyncSemaphore, body: untyped) =
  await sem.acquire()
  try:
    body
  finally:
    sem.release()

when isMainModule:
  # Task group example
  let group = newTaskGroup[string](timeout = initDuration(seconds = 5))
  
  group.spawn(proc(): Future[string] {.async.} =
    await sleepAsync(100)
    return "task1"
  )
  group.spawn(proc(): Future[string] {.async.} =
    await sleepAsync(50)
    return "task2"
  )
  group.spawn(proc(): Future[string] {.async.} =
    return "task3"
  )
  
  let results = waitFor group.waitAll()
  for r in results:
    if r.succeeded: echo "Success: ", r.value
    else: echo "Error: ", r.error
  
  # Semaphore limiting
  let sem = newAsyncSemaphore(3)  # Max 3 concurrent
  
  proc limitedWork(i: int): Future[void] {.async.} =
    withSemaphore(sem):
      echo &"Working {i}"
      await sleepAsync(100)
  
  var workFuts: seq[Future[void]]
  for i in 0..9: workFuts.add(limitedWork(i))
  waitFor all(workFuts)
  echo "All limited work done"
```

---

## 2. Async Channels & Streams

```nim
# async_streams.nim
# Async channels and data streams

import std/[asyncdispatch, asyncfutures, deques, strformat, options]

# Buffered async channel
type
  ChannelClosed* = object of CatchableError
  
  AsyncChannel*[T] = ref object
    buffer*: Deque[T]
    capacity*: int
    closed*: bool
    readWaiters*: seq[Future[T]]
    writeWaiters*: seq[Future[void]]

proc newAsyncChannel*[T](capacity = 16): AsyncChannel[T] =
  AsyncChannel[T](buffer: initDeque[T](), capacity: capacity)

proc send*[T](ch: AsyncChannel[T], value: T): Future[void] {.async.} =
  if ch.closed:
    raise newException(ChannelClosed, "Channel is closed")
  
  if ch.buffer.len < ch.capacity:
    ch.buffer.addLast(value)
    # Wake up a reader
    if ch.readWaiters.len > 0:
      let waiter = ch.readWaiters[0]
      ch.readWaiters.delete(0)
      waiter.complete(ch.buffer.popFirst())
    return
  
  # Buffer full — wait
  let waiter = newFuture[void]("channel.send")
  ch.writeWaiters.add(waiter)
  await waiter
  ch.buffer.addLast(value)

proc recv*[T](ch: AsyncChannel[T]): Future[T] {.async.} =
  if ch.buffer.len > 0:
    result = ch.buffer.popFirst()
    # Wake up a writer
    if ch.writeWaiters.len > 0:
      let waiter = ch.writeWaiters[0]
      ch.writeWaiters.delete(0)
      waiter.complete()
    return
  
  if ch.closed:
    raise newException(ChannelClosed, "Channel is closed and empty")
  
  let waiter = newFuture[T]("channel.recv")
  ch.readWaiters.add(waiter)
  result = await waiter

proc close*[T](ch: AsyncChannel[T]) =
  ch.closed = true
  for w in ch.readWaiters:
    w.fail(newException(ChannelClosed, "Channel closed"))
  ch.readWaiters = @[]

proc len*[T](ch: AsyncChannel[T]): int = ch.buffer.len
proc isClosed*[T](ch: AsyncChannel[T]): bool = ch.closed

# Async stream
type
  AsyncStream*[T] = ref object
    channel*: AsyncChannel[T]
    done*: bool

proc newAsyncStream*[T](bufSize = 32): AsyncStream[T] =
  AsyncStream[T](channel: newAsyncChannel[T](bufSize))

proc emit*[T](stream: AsyncStream[T], value: T): Future[void] =
  stream.channel.send(value)

proc finish*[T](stream: AsyncStream[T]) =
  stream.done = true
  stream.channel.close()

iterator items*[T](stream: AsyncStream[T]): Future[T] =
  while not (stream.done and stream.channel.len == 0):
    yield stream.channel.recv()

# Async generator using closures
type
  AsyncGenerator*[T] = proc(): Future[Option[T]]

proc generator*[T](fn: proc(emit: proc(v: T): Future[void])): AsyncGenerator[T] =
  let ch = newAsyncChannel[T](1)
  
  # Run producer in background
  asyncCheck (proc() {.async.} =
    try:
      await fn(proc(v: T): Future[void] = ch.send(v))
    finally:
      ch.close()
  )()
  
  return proc(): Future[Option[T]] {.async.} =
    try:
      return some(await ch.recv())
    except ChannelClosed:
      return none(T)

proc toSeq*[T](gen: AsyncGenerator[T]): Future[seq[T]] {.async.} =
  result = @[]
  while true:
    let val = await gen()
    if val.isNone: break
    result.add(val.get())

when isMainModule:
  # Channel example
  let ch = newAsyncChannel[int](5)
  
  proc producer() {.async.} =
    for i in 1..10:
      await ch.send(i)
    ch.close()
  
  proc consumer() {.async.} =
    while true:
      try:
        let v = await ch.recv()
        echo &"Received: {v}"
      except ChannelClosed:
        echo "Channel closed"
        break
  
  waitFor (proc() {.async.} =
    asyncCheck producer()
    await consumer()
  )()
  
  # Generator example
  let gen = generator[int](proc(emit: proc(v: int): Future[void]) {.async.} =
    for i in 0..4:
      await emit(i * i)
  )
  
  let values = waitFor gen.toSeq()
  echo "Generated: ", values  # @[0, 1, 4, 9, 16]
```

---

## 3. Backpressure

```nim
# backpressure.nim
# Flow control with backpressure

import std/[asyncdispatch, asyncfutures, times, strformat]

type
  FlowController* = ref object
    capacity*: int
    current*: int
    highWatermark*: int
    lowWatermark*: int
    paused*: bool
    resumeWaiters*: seq[Future[void]]

proc newFlowController*(capacity = 1000): FlowController =
  FlowController(
    capacity: capacity,
    highWatermark: capacity * 3 div 4,
    lowWatermark: capacity div 4
  )

proc add*(fc: FlowController, n = 1) =
  inc fc.current, n
  if fc.current >= fc.highWatermark and not fc.paused:
    fc.paused = true
    echo &"[Flow] Pausing at {fc.current}/{fc.capacity}"

proc remove*(fc: FlowController, n = 1) =
  dec fc.current, n
  if fc.current <= fc.lowWatermark and fc.paused:
    fc.paused = false
    echo &"[Flow] Resuming at {fc.current}/{fc.capacity}"
    for w in fc.resumeWaiters: w.complete()
    fc.resumeWaiters = @[]

proc waitResume*(fc: FlowController): Future[void] {.async.} =
  if not fc.paused: return
  let waiter = newFuture[void]("flow.resume")
  fc.resumeWaiters.add(waiter)
  await waiter

# Rate limiter with token bucket
type
  TokenBucket* = ref object
    tokens*: float
    maxTokens*: float
    refillRate*: float  # tokens per second
    lastRefill*: Time

proc newTokenBucket*(capacity: float, refillRate: float): TokenBucket =
  TokenBucket(
    tokens: capacity,
    maxTokens: capacity,
    refillRate: refillRate,
    lastRefill: now().toTime()
  )

proc refill*(tb: TokenBucket) =
  let now = getTime()
  let elapsed = (now - tb.lastRefill).inMilliseconds.float / 1000.0
  tb.tokens = min(tb.maxTokens, tb.tokens + elapsed * tb.refillRate)
  tb.lastRefill = now

proc tryConsume*(tb: TokenBucket, tokens = 1.0): bool =
  tb.refill()
  if tb.tokens >= tokens:
    tb.tokens -= tokens
    return true
  false

proc consume*(tb: TokenBucket, tokens = 1.0): Future[void] {.async.} =
  while not tb.tryConsume(tokens):
    let waitMs = int((tokens - tb.tokens) / tb.refillRate * 1000.0)
    await sleepAsync(max(1, waitMs))

# Async pipeline with backpressure
type
  Stage*[I, O] = proc(input: I): Future[O]

proc pipeline*[A, B, C](
  source: AsyncChannel[A],
  stage1: Stage[A, B],
  stage2: Stage[B, C],
  sink: AsyncChannel[C],
  workers = 4
): Future[void] {.async.} =
  let intermediate = newAsyncChannel[B](workers * 2)
  
  # Stage 1 workers
  var s1Workers: seq[Future[void]]
  for _ in 0..<workers:
    s1Workers.add((proc() {.async.} =
      while true:
        try:
          let input = await source.recv()
          let output = await stage1(input)
          await intermediate.send(output)
        except ChannelClosed: break
    )())
  
  # Stage 2 workers
  var s2Workers: seq[Future[void]]
  for _ in 0..<workers:
    s2Workers.add((proc() {.async.} =
      while true:
        try:
          let input = await intermediate.recv()
          let output = await stage2(input)
          await sink.send(output)
        except ChannelClosed: break
    )())
  
  await all(s1Workers)
  intermediate.close()
  await all(s2Workers)
  sink.close()

when isMainModule:
  # Flow control
  let fc = newFlowController(100)
  for i in 0..80: fc.add()
  echo &"Current: {fc.current}, Paused: {fc.paused}"
  for i in 0..70: fc.remove()
  echo &"Current: {fc.current}, Paused: {fc.paused}"
  
  # Token bucket rate limiting
  let bucket = newTokenBucket(10.0, 5.0)  # 5 RPS
  var processed = 0
  for i in 0..4:
    waitFor bucket.consume()
    inc processed
    echo &"Request {i} processed"
  
  echo &"Total processed: {processed}"
```

---

## สรุป

| Pattern | Use Case |
|---------|---------|
| Task Group | Scoped parallel tasks |
| Nursery | Fail-fast child tasks |
| Async Channel | Producer-consumer |
| Async Generator | Lazy sequences |
| Token Bucket | Rate limiting |
| Flow Controller | Backpressure |

---

**Next**: [Part 99 - Nim Ecosystem](../advanced/part99_ecosystem.md)
