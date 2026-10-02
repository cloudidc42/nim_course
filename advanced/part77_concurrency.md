# Part 77 - Advanced Concurrency Patterns in Nim

## บทนำ

Nim รองรับ concurrency หลายรูปแบบ:
- Async/await (single-thread cooperative)
- Threads + channels (OS threads)
- Threadpool
- Actor model ด้วย channels
- Lock-free data structures

---

## 1. Thread-Safe Data Structures

```nim
# thread_safe.nim
# Lock-free และ thread-safe data structures

import std/[atomics, locks, strformat, times, sequtils]

# Atomic counter
type
  AtomicCounter* = object
    value*: Atomic[int]

proc inc*(counter: var AtomicCounter, amount = 1) =
  discard counter.value.fetchAdd(amount)

proc dec*(counter: var AtomicCounter, amount = 1) =
  discard counter.value.fetchSub(amount)

proc get*(counter: var AtomicCounter): int =
  counter.value.load()

proc reset*(counter: var AtomicCounter) =
  counter.value.store(0)

# Lock-free stack (Treiber stack)
type
  StackNode*[T] = ref object
    value*: T
    next*: ptr StackNode[T]

  LockFreeStack*[T] = object
    head*: Atomic[pointer]  # ptr StackNode[T]

proc push*[T](stack: var LockFreeStack[T], value: T) =
  let node = new StackNode[T]
  node.value = value
  
  while true:
    let oldHead = stack.head.load()
    node.next = cast[ptr StackNode[T]](oldHead)
    
    if stack.head.compareExchange(
      expected = oldHead,
      desired = cast[pointer](node)
    ):
      break

proc pop*[T](stack: var LockFreeStack[T]): Option[T] =
  while true:
    let head = stack.head.load()
    if head == nil:
      return none(T)
    
    let node = cast[ptr StackNode[T]](head)
    let newHead = cast[pointer](node.next)
    
    if stack.head.compareExchange(
      expected = head,
      desired = newHead
    ):
      return some(node.value)

# MPMC Queue (Michael-Scott queue concept — simplified)
type
  MPMCQueue*[T] = ref object
    lock*: Lock
    data*: seq[T]
    maxSize*: int
    notEmpty*: Cond
    notFull*: Cond

proc newMPMCQueue*[T](maxSize = 1000): MPMCQueue[T] =
  result = MPMCQueue[T](maxSize: maxSize, data: @[])
  initLock(result.lock)
  initCond(result.notEmpty)
  initCond(result.notFull)

proc enqueue*[T](q: MPMCQueue[T], item: T) =
  acquire(q.lock)
  while q.data.len >= q.maxSize:
    wait(q.notFull, q.lock)
  q.data.add(item)
  signal(q.notEmpty)
  release(q.lock)

proc dequeue*[T](q: MPMCQueue[T]): T =
  acquire(q.lock)
  while q.data.len == 0:
    wait(q.notEmpty, q.lock)
  result = q.data[0]
  q.data.delete(0)
  signal(q.notFull)
  release(q.lock)

proc tryEnqueue*[T](q: MPMCQueue[T], item: T): bool =
  acquire(q.lock)
  defer: release(q.lock)
  
  if q.data.len >= q.maxSize:
    return false
  
  q.data.add(item)
  signal(q.notEmpty)
  true

proc tryDequeue*[T](q: MPMCQueue[T]): Option[T] =
  acquire(q.lock)
  defer: release(q.lock)
  
  if q.data.len == 0:
    return none(T)
  
  let item = q.data[0]
  q.data.delete(0)
  signal(q.notFull)
  some(item)

# Read-Write Lock
type
  RWLock* = object
    mutex*: Lock
    readCond*, writeCond*: Cond
    readers*: int
    writers*: int
    waitingWriters*: int

proc initRWLock*(rw: var RWLock) =
  initLock(rw.mutex)
  initCond(rw.readCond)
  initCond(rw.writeCond)

proc readLock*(rw: var RWLock) =
  acquire(rw.mutex)
  while rw.writers > 0 or rw.waitingWriters > 0:
    wait(rw.readCond, rw.mutex)
  inc rw.readers
  release(rw.mutex)

proc readUnlock*(rw: var RWLock) =
  acquire(rw.mutex)
  dec rw.readers
  if rw.readers == 0:
    signal(rw.writeCond)
  release(rw.mutex)

proc writeLock*(rw: var RWLock) =
  acquire(rw.mutex)
  inc rw.waitingWriters
  while rw.readers > 0 or rw.writers > 0:
    wait(rw.writeCond, rw.mutex)
  dec rw.waitingWriters
  inc rw.writers
  release(rw.mutex)

proc writeUnlock*(rw: var RWLock) =
  acquire(rw.mutex)
  dec rw.writers
  if rw.waitingWriters > 0:
    signal(rw.writeCond)
  else:
    broadcast(rw.readCond)
  release(rw.mutex)

template withReadLock*(rw: var RWLock, body: untyped) =
  rw.readLock()
  try:
    body
  finally:
    rw.readUnlock()

template withWriteLock*(rw: var RWLock, body: untyped) =
  rw.writeLock()
  try:
    body
  finally:
    rw.writeUnlock()
```

---

## 2. Actor Model

```nim
# actor.nim
# Actor model ด้วย threads + channels

import std/[channels, locks, tables, strformat, asyncdispatch, times]
import std/[options, typetraits]

type
  ActorId* = int
  
  ActorMessage*[T] = object
    from*: ActorId
    payload*: T
    replyTo*: Option[ptr Channel[string]]  # For ask pattern

  ActorState* = enum
    asRunning
    asStopped
    asRestarting

  ActorBehavior*[T] = proc(self: ActorId, msg: ActorMessage[T])

  Actor*[T] = ref object
    id*: ActorId
    mailbox*: ptr Channel[ActorMessage[T]]
    behavior*: ActorBehavior[T]
    state*: ActorState
    thread*: Thread[Actor[T]]

  ActorSystem* = ref object
    actors*: Table[ActorId, pointer]  # ActorId -> Actor[T]
    nextId*: ActorId
    lock*: Lock

var globalSystem* = ActorSystem(actors: initTable[ActorId, pointer]())
initLock(globalSystem.lock)

proc spawn*[T](system: ActorSystem, behavior: ActorBehavior[T]): ActorId =
  acquire(system.lock)
  let id = system.nextId
  inc system.nextId
  release(system.lock)
  
  let actor = Actor[T](
    id: id,
    mailbox: create(Channel[ActorMessage[T]]),
    behavior: behavior,
    state: asRunning
  )
  actor.mailbox[].open(100)  # Buffer size 100
  
  proc actorLoop(a: Actor[T]) {.thread.} =
    while a.state == asRunning:
      let msg = a.mailbox[].recv()
      try:
        a.behavior(a.id, msg)
      except CatchableError as e:
        echo &"Actor {a.id} error: {e.msg}"
  
  createThread(actor.thread, actorLoop, actor)
  
  acquire(system.lock)
  system.actors[id] = cast[pointer](actor)
  release(system.lock)
  
  id

proc send*[T](system: ActorSystem, to: ActorId, msg: T, from: ActorId = -1) =
  acquire(system.lock)
  if system.actors.hasKey(to):
    let actor = cast[Actor[T]](system.actors[to])
    release(system.lock)
    actor.mailbox[].send(ActorMessage[T](from: from, payload: msg))
  else:
    release(system.lock)
    echo &"Actor {to} not found"

proc stop*(system: ActorSystem, id: ActorId) =
  acquire(system.lock)
  if system.actors.hasKey(id):
    let actor = cast[Actor[pointer]](system.actors[id])
    actor.state = asStopped
    system.actors.del(id)
  release(system.lock)

# Example: Bank account actor
type
  AccountMsg* = object
    case kind*: enum
      amDeposit, amWithdraw, amGetBalance
    amount*: float

proc accountBehavior*(self: ActorId, msg: ActorMessage[AccountMsg]) =
  {.global.}:
    var balance {.threadvar.}: float
  
  case msg.payload.kind
  of amDeposit:
    balance += msg.payload.amount
    echo &"[Account {self}] Deposited {msg.payload.amount}. Balance: {balance}"
  of amWithdraw:
    if balance >= msg.payload.amount:
      balance -= msg.payload.amount
      echo &"[Account {self}] Withdrew {msg.payload.amount}. Balance: {balance}"
    else:
      echo &"[Account {self}] Insufficient funds"
  of amGetBalance:
    echo &"[Account {self}] Balance: {balance}"
```

---

## 3. Thread Pool

```nim
# threadpool.nim
# Custom thread pool สำหรับ CPU-intensive tasks

import std/[locks, channels, times, strformat, math, sequtils]
import std/[atomics, options]

type
  WorkItem* = object
    fn*: proc()
    id*: int

  ThreadPool* = ref object
    workers*: seq[Thread[ThreadPool]]
    queue*: ptr Channel[Option[WorkItem]]
    activeWorkers*: Atomic[int]
    completedTasks*: Atomic[int64]
    running*: bool
    size*: int

proc worker*(pool: ThreadPool) {.thread.} =
  while true:
    let item = pool.queue[].recv()
    if item.isNone:
      break  # Poison pill — shutdown signal
    
    discard pool.activeWorkers.fetchAdd(1)
    try:
      item.get().fn()
    except CatchableError as e:
      echo &"Worker error: {e.msg}"
    finally:
      discard pool.activeWorkers.fetchSub(1)
      discard pool.completedTasks.fetchAdd(1)

proc newThreadPool*(size: int): ThreadPool =
  result = ThreadPool(
    size: size,
    queue: create(Channel[Option[WorkItem]]),
    running: true,
    workers: newSeq[Thread[ThreadPool]](size)
  )
  result.queue[].open(1000)
  
  for i in 0..<size:
    createThread(result.workers[i], worker, result)

proc submit*(pool: ThreadPool, fn: proc()) =
  {.global.}: var taskId: Atomic[int]
  let id = taskId.fetchAdd(1)
  pool.queue[].send(some(WorkItem(fn: fn, id: id)))

proc submitAll*(pool: ThreadPool, tasks: seq[proc()]) =
  for task in tasks:
    pool.submit(task)

proc shutdown*(pool: ThreadPool) =
  pool.running = false
  # Send poison pill to each worker
  for _ in 0..<pool.size:
    pool.queue[].send(none(WorkItem))
  
  for thread in pool.workers:
    joinThread(thread)

proc wait*(pool: ThreadPool) =
  ## Wait until queue is empty and all workers idle
  while pool.queue[].peek() > 0 or pool.activeWorkers.load() > 0:
    sleep(10)

# Parallel map using thread pool
proc parallelMap*[T, R](pool: ThreadPool, items: seq[T],
    fn: proc(x: T): R): seq[R] =
  var results = newSeq[R](items.len)
  var resultsLock: Lock
  initLock(resultsLock)
  
  var pending: Atomic[int]
  pending.store(items.len)
  
  for i, item in items:
    let idx = i
    let val = item
    pool.submit(proc() =
      let r = fn(val)
      acquire(resultsLock)
      results[idx] = r
      release(resultsLock)
      discard pending.fetchSub(1)
    )
  
  while pending.load() > 0:
    sleep(1)
  
  results

# Work-stealing deque (basis of work-stealing thread pool)
type
  WSDeque*[T] = object
    buffer*: seq[T]
    top*: Atomic[int]
    bottom*: int
    lock*: Lock

proc newWSDeque*[T](capacity = 256): WSDeque[T] =
  result = WSDeque[T](buffer: newSeq[T](capacity))
  initLock(result.lock)

proc pushBottom*[T](deque: var WSDeque[T], item: T) =
  deque.buffer[deque.bottom mod deque.buffer.len] = item
  inc deque.bottom

proc popBottom*[T](deque: var WSDeque[T]): Option[T] =
  let b = deque.bottom - 1
  deque.bottom = b
  
  let t = deque.top.load()
  if t > b:
    deque.bottom = t
    return none(T)
  
  let item = deque.buffer[b mod deque.buffer.len]
  
  if t == b:
    if not deque.top.compareExchange(expected = t, desired = t + 1):
      deque.bottom = t + 1
      return none(T)
    deque.bottom = t + 1
  
  some(item)

proc steal*[T](deque: var WSDeque[T]): Option[T] =
  ## Steal from top (used by other threads)
  let t = deque.top.load()
  let b = deque.bottom
  
  if t >= b:
    return none(T)
  
  let item = deque.buffer[t mod deque.buffer.len]
  
  if deque.top.compareExchange(expected = t, desired = t + 1):
    return some(item)
  
  none(T)
```

---

## 4. Async + Threads Hybrid

```nim
# hybrid_async.nim
# CPU-intensive work ใน thread, async I/O บน main thread

import std/[asyncdispatch, asyncfutures, channels, locks, strformat, math]

type
  ComputeTask* = object
    data*: seq[float64]
    resultChan*: ptr Channel[seq[float64]]

var computeChannel*: ptr Channel[ComputeTask]

proc computeWorker() {.thread.} =
  ## Background thread สำหรับ CPU-intensive computation
  while true:
    let task = computeChannel[].recv()
    
    # Expensive computation (e.g., FFT, matrix multiply)
    var result = newSeq[float64](task.data.len)
    for i, x in task.data:
      result[i] = sqrt(abs(sin(x) * cos(x) * x))  # CPU intensive
    
    task.resultChan[].send(result)

var workerThread: Thread[void]

proc initComputeWorker*() =
  computeChannel = create(Channel[ComputeTask])
  computeChannel[].open(10)
  createThread(workerThread, computeWorker)

proc computeAsync*(data: seq[float64]): Future[seq[float64]] =
  ## Offload computation to worker thread, return async future
  let resultChan = create(Channel[seq[float64]])
  resultChan[].open(1)
  
  computeChannel[].send(ComputeTask(data: data, resultChan: resultChan))
  
  var future = newFuture[seq[float64]]("computeAsync")
  
  # Poll for result (simplified — real impl uses callback)
  asyncCheck (proc() {.async.} =
    while resultChan[].peek() == 0:
      await sleepAsync(1)
    let result = resultChan[].recv()
    dealloc(resultChan)
    future.complete(result)
  )()
  
  future

when isMainModule:
  proc main() {.async.} =
    initComputeWorker()
    
    echo "Submitting CPU-intensive tasks..."
    
    let tasks = @[
      computeAsync(@[1.0, 2.0, 3.0, 4.0, 5.0]),
      computeAsync(@[6.0, 7.0, 8.0, 9.0, 10.0])
    ]
    
    let results = await all(tasks)
    
    for i, r in results:
      echo &"Task {i}: {r}"
    
    echo "All tasks complete"
  
  waitFor main()
```

---

## 5. Barrier Synchronization

```nim
# barrier.nim
# Barrier สำหรับ synchronize หลาย threads ณ จุดหนึ่ง

import std/[locks, atomics, strformat]

type
  Barrier* = object
    mutex*: Lock
    cond*: Cond
    count*: int         # Threads ที่ต้องรอ
    totalCount*: int    # Thread count ทั้งหมด
    generation*: int    # เพื่อป้องกัน spurious wakeup

proc initBarrier*(b: var Barrier, count: int) =
  initLock(b.mutex)
  initCond(b.cond)
  b.count = count
  b.totalCount = count

proc wait*(b: var Barrier) =
  ## Block until all threads reach barrier
  acquire(b.mutex)
  let gen = b.generation
  dec b.count
  
  if b.count == 0:
    # Last thread: wake everyone
    inc b.generation
    b.count = b.totalCount
    broadcast(b.cond)
  else:
    # Wait for others
    while b.generation == gen:
      wait(b.cond, b.mutex)
  
  release(b.mutex)

# Example: Parallel matrix multiply with barrier
proc parallelMatMul*(a, b: seq[seq[float64]], numThreads: int): seq[seq[float64]] =
  let n = a.len
  var result = newSeq[seq[float64]](n)
  for i in 0..<n:
    result[i] = newSeq[float64](n)
  
  var barrier: Barrier
  initBarrier(barrier, numThreads)
  
  var threads = newSeq[Thread[(seq[seq[float64]], seq[seq[float64]], ptr seq[seq[float64]], int, int, int, ptr Barrier)]](numThreads)
  
  proc threadWork(args: (seq[seq[float64]], seq[seq[float64]], ptr seq[seq[float64]], int, int, int, ptr Barrier)) {.thread.} =
    let (a, b, result, n, start, step, barrier) = args
    
    for i in countup(start, n - 1, step):
      for j in 0..<n:
        var sum = 0.0
        for k in 0..<n:
          sum += a[i][k] * b[k][j]
        result[][i][j] = sum
    
    barrier[].wait()
  
  for t in 0..<numThreads:
    createThread(threads[t], threadWork, (a, b, addr result, n, t, numThreads, addr barrier))
  
  for t in threads:
    joinThread(t)
  
  result
```

---

## สรุป

| Pattern | Use Case |
|---------|---------|
| Atomic | Lock-free counters, flags |
| Lock-free Stack | Concurrent push/pop |
| RW Lock | Mostly-read data structures |
| Actor Model | Independent concurrent entities |
| Thread Pool | CPU parallelism |
| Work Stealing | Load-balanced parallelism |
| Async + Threads | I/O + CPU combo |
| Barrier | Synchronization points |

---

**Next**: [Part 78 - Production Deployment & Monitoring](../ops/part78_deployment.md)
