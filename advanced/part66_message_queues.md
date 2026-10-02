# Part 66 - Message Queues & Event-Driven Architecture

## บทนำ

Message Queue คือ middleware ที่ช่วยให้ components สื่อสารกันแบบ asynchronous
แยก producer ออกจาก consumer — ทำให้ระบบ scale ได้ง่าย และ fault-tolerant มากขึ้น

---

## 1. In-Memory Queue ด้วย AsyncChannel

```nim
# async_queue.nim
# Simple async queue สำหรับ producer/consumer pattern

import std/[asyncdispatch, asyncfutures, deques, strformat, times]
import std/[options, locks, atomics]

type
  Message*[T] = object
    id*: string
    payload*: T
    timestamp*: Time
    retries*: int
    headers*: seq[(string, string)]

  QueueState* = enum
    qsRunning
    qsStopped
    qsDraining  # หยุดรับ message ใหม่ แต่ flush ที่เหลือ

  AsyncQueue*[T] = ref object
    name*: string
    buffer*: Deque[Message[T]]
    maxSize*: int
    state*: QueueState
    waiters*: seq[Future[Message[T]]]
    msgCounter*: Atomic[int]

proc newAsyncQueue*[T](name: string, maxSize = 1000): AsyncQueue[T] =
  result = AsyncQueue[T](
    name: name,
    buffer: initDeque[Message[T]](),
    maxSize: maxSize,
    state: qsRunning
  )
  result.msgCounter.store(0)

proc publish*[T](q: AsyncQueue[T], payload: T,
    headers: seq[(string, string)] = @[]): Future[bool] {.async.} =
  ## Publish message — returns false if queue is full or stopped
  if q.state != qsRunning:
    return false
  
  if q.buffer.len >= q.maxSize:
    return false  # Backpressure: caller should retry
  
  let id = $q.msgCounter.fetchAdd(1)
  let msg = Message[T](
    id: id,
    payload: payload,
    timestamp: getTime(),
    headers: headers
  )
  
  # If there are waiting consumers, deliver directly
  if q.waiters.len > 0:
    let waiter = q.waiters.pop()
    waiter.complete(msg)
    return true
  
  q.buffer.addLast(msg)
  return true

proc consume*[T](q: AsyncQueue[T]): Future[Message[T]] {.async.} =
  ## Wait for next message
  if q.buffer.len > 0:
    return q.buffer.popFirst()
  
  # No messages available — register as waiter
  let fut = newFuture[Message[T]]("AsyncQueue.consume")
  q.waiters.add(fut)
  return await fut

proc len*[T](q: AsyncQueue[T]): int = q.buffer.len
proc isEmpty*[T](q: AsyncQueue[T]): bool = q.buffer.len == 0

proc drain*[T](q: AsyncQueue[T]) =
  ## Stop accepting new messages, signal drain mode
  q.state = qsDraining
  echo &"[Queue:{q.name}] Draining..."

# Worker pool pattern
proc startWorkers*[T](q: AsyncQueue[T], numWorkers: int,
    handler: proc(msg: Message[T]): Future[void]) {.async.} =
  var workers: seq[Future[void]]
  
  for i in 0..<numWorkers:
    workers.add(
      (proc(workerId: int): Future[void] {.async.} =
        echo &"[Worker {workerId}] Started"
        while q.state == qsRunning or not q.isEmpty():
          let msg = await q.consume()
          try:
            await handler(msg)
          except CatchableError as e:
            echo &"[Worker {workerId}] Error processing msg {msg.id}: {e.msg}"
      )(i)
    )
  
  await all(workers)

when isMainModule:
  proc main() {.async.} =
    let q = newAsyncQueue[string]("tasks", maxSize = 100)
    
    # Start 3 workers
    asyncCheck q.startWorkers(3, proc(msg: Message[string]): Future[void] {.async.} =
      echo &"Processing: {msg.payload} (id={msg.id})"
      await sleepAsync(100)
    )
    
    # Publish 10 messages
    for i in 1..10:
      discard await q.publish(&"task-{i}")
      await sleepAsync(50)
    
    await sleepAsync(2000)
  
  waitFor main()
```

---

## 2. Dead Letter Queue (DLQ)

```nim
# dlq.nim
# Dead letter queue สำหรับ messages ที่ process ไม่ได้

import std/[asyncdispatch, deques, strformat, times, tables, json]

type
  ProcessResult* = enum
    prSuccess
    prRetry
    prDeadLetter  # ส่งไป DLQ ทันที ไม่ retry

  DeadLetterReason* = enum
    dlrMaxRetries
    dlrPoisonMessage
    dlrExpired
    dlrManual

  DLQEntry*[T] = object
    originalMsg*: Message[T]
    reason*: DeadLetterReason
    errorMsg*: string
    deadAt*: Time
    originalQueue*: string

  QueueWithDLQ*[T] = ref object
    mainQueue*: AsyncQueue[T]
    dlq*: Deque[DLQEntry[T]]
    maxRetries*: int
    msgTtl*: Duration  # Message time-to-live

proc newQueueWithDLQ*[T](name: string, maxRetries = 3,
    ttlSecs = 3600): QueueWithDLQ[T] =
  QueueWithDLQ[T](
    mainQueue: newAsyncQueue[T](name),
    dlq: initDeque[DLQEntry[T]](),
    maxRetries: maxRetries,
    msgTtl: initDuration(seconds = ttlSecs)
  )

proc sendToDLQ*[T](q: QueueWithDLQ[T], msg: Message[T],
    reason: DeadLetterReason, errorMsg = "") =
  q.dlq.addLast(DLQEntry[T](
    originalMsg: msg,
    reason: reason,
    errorMsg: errorMsg,
    deadAt: getTime(),
    originalQueue: q.mainQueue.name
  ))
  echo &"[DLQ] Message {msg.id} moved to dead letter: {reason}"

proc processWithRetry*[T](q: QueueWithDLQ[T],
    handler: proc(msg: Message[T]): Future[ProcessResult]): Future[void] {.async.} =
  while true:
    var msg = await q.mainQueue.consume()
    
    # Check TTL
    if getTime() - msg.timestamp > q.msgTtl:
      q.sendToDLQ(msg, dlrExpired, "Message TTL exceeded")
      continue
    
    # Try processing
    var result: ProcessResult
    try:
      result = await handler(msg)
    except CatchableError as e:
      result = prRetry
      msg.retries.inc
      echo &"[Queue] Error processing {msg.id}: {e.msg} (retry {msg.retries}/{q.maxRetries})"
    
    case result
    of prSuccess:
      discard  # Done
    of prRetry:
      if msg.retries >= q.maxRetries:
        q.sendToDLQ(msg, dlrMaxRetries, &"Exceeded {q.maxRetries} retries")
      else:
        # Re-queue with backoff
        let delayMs = 1000 * (1 shl msg.retries)  # 2^retries seconds
        await sleepAsync(delayMs)
        discard await q.mainQueue.publish(msg.payload, msg.headers)
    of prDeadLetter:
      q.sendToDLQ(msg, dlrPoisonMessage, "Handler rejected message")

proc replayDLQ*[T](q: QueueWithDLQ[T], filter: proc(e: DLQEntry[T]): bool = nil) =
  ## Replay messages from DLQ back to main queue
  var toReplay: seq[DLQEntry[T]]
  while q.dlq.len > 0:
    let entry = q.dlq.popFirst()
    if filter == nil or filter(entry):
      toReplay.add(entry)
    else:
      q.dlq.addLast(entry)  # Keep it
  
  echo &"[DLQ] Replaying {toReplay.len} messages"
  for entry in toReplay:
    var msg = entry.originalMsg
    msg.retries = 0
    asyncCheck q.mainQueue.publish(msg.payload, msg.headers)
```

---

## 3. Kafka-like Log-Structured Storage

```nim
# kafka_log.nim
# Append-only log — basis ของ Kafka partitions

import std/[streams, strformat, times, os, strutils, endians]
import std/[options, json, asyncdispatch, sequtils]

# Log format: [8-byte offset][4-byte timestamp][4-byte length][payload bytes]
const
  OFFSET_SIZE  = 8
  TS_SIZE      = 4
  LENGTH_SIZE  = 4
  HEADER_SIZE  = OFFSET_SIZE + TS_SIZE + LENGTH_SIZE

type
  LogRecord* = object
    offset*: int64
    timestamp*: uint32
    payload*: string

  LogSegment* = ref object
    path*: string
    baseOffset*: int64
    stream*: FileStream
    currentOffset*: int64
    size*: int64

  CommitLog* = ref object
    dir*: string
    topic*: string
    partition*: int
    segments*: seq[LogSegment]
    maxSegmentBytes*: int64
    retentionMs*: int64
    consumerOffsets*: Table[string, int64]  # groupId -> offset

proc openSegment*(path: string, baseOffset: int64): LogSegment =
  result = LogSegment(
    path: path,
    baseOffset: baseOffset,
    stream: newFileStream(path, fmAppend),
    currentOffset: baseOffset
  )
  if fileExists(path):
    result.size = getFileSize(path)

proc append*(seg: LogSegment, payload: string): int64 =
  ## Append record, return offset
  let offset = seg.currentOffset
  let ts = uint32(epochTime().int)
  let length = uint32(payload.len)
  
  # Write header
  var buf: array[HEADER_SIZE, byte]
  bigEndian64(addr buf[0], unsafeAddr offset)
  bigEndian32(addr buf[8], unsafeAddr ts)
  bigEndian32(addr buf[12], unsafeAddr length)
  seg.stream.writeData(addr buf[0], HEADER_SIZE)
  seg.stream.write(payload)
  seg.stream.flush()
  
  inc seg.currentOffset
  seg.size += int64(HEADER_SIZE + payload.len)
  return offset

proc newCommitLog*(dir, topic: string, partition = 0,
    maxSegmentBytes = 1024 * 1024 * 64'i64,  # 64MB
    retentionMs = 7 * 24 * 3600 * 1000'i64): CommitLog =  # 7 days
  
  createDir(&"{dir}/{topic}/{partition}")
  result = CommitLog(
    dir: dir,
    topic: topic,
    partition: partition,
    maxSegmentBytes: maxSegmentBytes,
    retentionMs: retentionMs,
    consumerOffsets: initTable[string, int64]()
  )
  
  # Load existing segments or create first one
  let segPath = &"{dir}/{topic}/{partition}/00000000000000000000.log"
  result.segments.add(openSegment(segPath, 0))

proc activeSegment*(log: CommitLog): LogSegment =
  log.segments[^1]

proc produce*(log: CommitLog, messages: seq[string]): seq[int64] =
  ## Produce messages to log, return their offsets
  result = @[]
  let seg = log.activeSegment()
  
  for msg in messages:
    # Roll segment if too large
    if seg.size >= log.maxSegmentBytes:
      let newBase = seg.currentOffset
      let newPath = &"{log.dir}/{log.topic}/{log.partition}/{newBase:020d}.log"
      log.segments.add(openSegment(newPath, newBase))
    
    result.add(seg.append(msg))

proc read*(log: CommitLog, offset: int64, maxCount = 100): seq[LogRecord] =
  ## Read messages starting from offset
  result = @[]
  
  # Find the right segment (binary search by baseOffset)
  var segIdx = 0
  for i, seg in log.segments:
    if seg.baseOffset <= offset:
      segIdx = i
  
  let seg = log.segments[segIdx]
  let stream = newFileStream(seg.path, fmRead)
  if stream == nil:
    return
  
  defer: stream.close()
  
  while result.len < maxCount and not stream.atEnd():
    var buf: array[HEADER_SIZE, byte]
    if stream.readData(addr buf[0], HEADER_SIZE) < HEADER_SIZE:
      break
    
    var recordOffset: int64
    var ts: uint32
    var length: uint32
    bigEndian64(addr recordOffset, addr buf[0])
    bigEndian32(addr ts, addr buf[8])
    bigEndian32(addr length, addr buf[12])
    
    let payload = stream.readStr(int(length))
    
    if recordOffset >= offset:
      result.add(LogRecord(
        offset: recordOffset,
        timestamp: ts,
        payload: payload
      ))

proc commit*(log: CommitLog, groupId: string, offset: int64) =
  ## Consumer commits processed offset
  log.consumerOffsets[groupId] = offset
  echo &"[Log] Group '{groupId}' committed offset {offset}"

proc latestOffset*(log: CommitLog): int64 =
  log.activeSegment().currentOffset

# Consumer group pattern
type
  Consumer* = ref object
    groupId*: string
    log*: CommitLog
    pollIntervalMs*: int

proc newConsumer*(groupId: string, log: CommitLog, pollIntervalMs = 500): Consumer =
  Consumer(groupId: groupId, log: log, pollIntervalMs: pollIntervalMs)

proc poll*(c: Consumer, maxCount = 10): seq[LogRecord] =
  let offset = c.log.consumerOffsets.getOrDefault(c.groupId, 0)
  let records = c.log.read(offset, maxCount)
  if records.len > 0:
    let newOffset = records[^1].offset + 1
    c.log.commit(c.groupId, newOffset)
  return records

proc startConsuming*(c: Consumer, handler: proc(rec: LogRecord)) {.async.} =
  while true:
    let records = c.poll()
    for rec in records:
      handler(rec)
    if records.len == 0:
      await sleepAsync(c.pollIntervalMs)
```

---

## 4. Event Sourcing Pattern

```nim
# event_sourcing.nim
# Events เป็น source of truth — rebuild state จาก event history

import std/[json, times, strformat, sequtils, options, tables, strutils]

type
  EventType* = string

  DomainEvent* = object
    id*: string
    aggregateId*: string
    eventType*: EventType
    version*: int
    timestamp*: Time
    data*: JsonNode
    metadata*: JsonNode

  EventStore* = ref object
    events*: Table[string, seq[DomainEvent]]  # aggregateId -> events
    globalLog*: seq[DomainEvent]
    handlers*: Table[EventType, seq[proc(e: DomainEvent)]]

proc newEventStore*(): EventStore =
  EventStore(
    events: initTable[string, seq[DomainEvent]](),
    globalLog: @[],
    handlers: initTable[EventType, seq[proc(e: DomainEvent)]]()
  )

proc append*(store: EventStore, event: DomainEvent) =
  if not store.events.hasKey(event.aggregateId):
    store.events[event.aggregateId] = @[]
  store.events[event.aggregateId].add(event)
  store.globalLog.add(event)
  
  # Publish to handlers (projections)
  if store.handlers.hasKey(event.eventType):
    for handler in store.handlers[event.eventType]:
      handler(event)

proc getEvents*(store: EventStore, aggregateId: string,
    fromVersion = 0): seq[DomainEvent] =
  if not store.events.hasKey(aggregateId):
    return @[]
  store.events[aggregateId].filterIt(it.version > fromVersion)

proc subscribe*(store: EventStore, eventType: EventType,
    handler: proc(e: DomainEvent)) =
  if not store.handlers.hasKey(eventType):
    store.handlers[eventType] = @[]
  store.handlers[eventType].add(handler)

# Example: Bank Account Aggregate
type
  AccountState* = object
    id*: string
    balance*: float
    isOpen*: bool
    version*: int

proc applyEvent*(state: var AccountState, event: DomainEvent) =
  case event.eventType
  of "AccountOpened":
    state.id = event.aggregateId
    state.balance = event.data["initialBalance"].getFloat()
    state.isOpen = true
  of "MoneyDeposited":
    state.balance += event.data["amount"].getFloat()
  of "MoneyWithdrawn":
    state.balance -= event.data["amount"].getFloat()
  of "AccountClosed":
    state.isOpen = false
  inc state.version

proc rebuildState*(store: EventStore, accountId: string): AccountState =
  let events = store.getEvents(accountId)
  var state = AccountState()
  for event in events:
    state.applyEvent(event)
  return state

# Command handlers (write side)
proc openAccount*(store: EventStore, accountId: string, initialBalance: float) =
  let event = DomainEvent(
    id: &"{accountId}-1",
    aggregateId: accountId,
    eventType: "AccountOpened",
    version: 1,
    timestamp: getTime(),
    data: %*{"initialBalance": initialBalance}
  )
  store.append(event)
  echo &"[ES] Account {accountId} opened with balance {initialBalance}"

proc deposit*(store: EventStore, accountId: string, amount: float) =
  let currentEvents = store.getEvents(accountId)
  let version = currentEvents.len + 1
  let state = store.rebuildState(accountId)
  
  if not state.isOpen:
    raise newException(ValueError, "Account is not open")
  
  let event = DomainEvent(
    id: &"{accountId}-{version}",
    aggregateId: accountId,
    eventType: "MoneyDeposited",
    version: version,
    timestamp: getTime(),
    data: %*{"amount": amount}
  )
  store.append(event)
  echo &"[ES] Deposited {amount} to {accountId}"

proc withdraw*(store: EventStore, accountId: string, amount: float) =
  let state = store.rebuildState(accountId)
  
  if not state.isOpen:
    raise newException(ValueError, "Account is not open")
  if state.balance < amount:
    raise newException(ValueError, &"Insufficient funds: {state.balance} < {amount}")
  
  let version = store.getEvents(accountId).len + 1
  let event = DomainEvent(
    id: &"{accountId}-{version}",
    aggregateId: accountId,
    eventType: "MoneyWithdrawn",
    version: version,
    timestamp: getTime(),
    data: %*{"amount": amount}
  )
  store.append(event)
  echo &"[ES] Withdrew {amount} from {accountId}"

when isMainModule:
  let store = newEventStore()
  
  # Subscribe to build read model (CQRS)
  var balances: Table[string, float]
  store.subscribe("AccountOpened", proc(e: DomainEvent) =
    balances[e.aggregateId] = e.data["initialBalance"].getFloat()
  )
  store.subscribe("MoneyDeposited", proc(e: DomainEvent) =
    balances[e.aggregateId] = balances.getOrDefault(e.aggregateId) + 
      e.data["amount"].getFloat()
  )
  store.subscribe("MoneyWithdrawn", proc(e: DomainEvent) =
    balances[e.aggregateId] = balances.getOrDefault(e.aggregateId) - 
      e.data["amount"].getFloat()
  )
  
  store.openAccount("acc-001", 1000.0)
  store.deposit("acc-001", 500.0)
  store.withdraw("acc-001", 200.0)
  
  # Query read model
  echo &"Balance (from projection): {balances[\"acc-001\"]}"
  
  # Rebuild from events
  let state = store.rebuildState("acc-001")
  echo &"Balance (from events): {state.balance}"
```

---

## 5. CQRS — Command Query Responsibility Segregation

```nim
# cqrs.nim
# แยก write model (Commands) ออกจาก read model (Queries)

import std/[tables, times, json, strformat, options, asyncdispatch]

# === Commands (Write Side) ===
type
  Command* = concept c
    c.aggregateId is string
    c.validate() is bool

  CreateOrderCmd* = object
    aggregateId*: string
    customerId*: string
    items*: seq[tuple[productId: string, qty: int, price: float]]

  UpdateOrderStatusCmd* = object
    aggregateId*: string
    newStatus*: string

proc validate*(cmd: CreateOrderCmd): bool =
  cmd.customerId.len > 0 and cmd.items.len > 0

proc validate*(cmd: UpdateOrderStatusCmd): bool =
  cmd.newStatus in ["pending", "confirmed", "shipped", "delivered", "cancelled"]

# === Read Models (Query Side — optimized for reads) ===
type
  OrderSummary* = object
    id*: string
    customerId*: string
    status*: string
    totalAmount*: float
    itemCount*: int
    createdAt*: Time
    updatedAt*: Time

  CustomerOrdersView* = object
    customerId*: string
    orders*: seq[OrderSummary]
    totalOrders*: int
    totalSpent*: float

# === Read Store (can be different DB, cache, search index) ===
type
  ReadStore* = ref object
    orders*: Table[string, OrderSummary]
    customerOrders*: Table[string, seq[string]]  # customerId -> orderIds

proc newReadStore*(): ReadStore =
  ReadStore(
    orders: initTable[string, OrderSummary](),
    customerOrders: initTable[string, seq[string]]()
  )

proc upsertOrder*(rs: ReadStore, order: OrderSummary) =
  rs.orders[order.id] = order
  if not rs.customerOrders.hasKey(order.customerId):
    rs.customerOrders[order.customerId] = @[]
  if order.id notin rs.customerOrders[order.customerId]:
    rs.customerOrders[order.customerId].add(order.id)

proc getOrder*(rs: ReadStore, id: string): Option[OrderSummary] =
  if rs.orders.hasKey(id):
    some(rs.orders[id])
  else:
    none(OrderSummary)

proc getCustomerOrders*(rs: ReadStore, customerId: string): CustomerOrdersView =
  let orderIds = rs.customerOrders.getOrDefault(customerId, @[])
  var orders: seq[OrderSummary]
  var totalSpent = 0.0
  
  for id in orderIds:
    if rs.orders.hasKey(id):
      let order = rs.orders[id]
      orders.add(order)
      totalSpent += order.totalAmount
  
  CustomerOrdersView(
    customerId: customerId,
    orders: orders,
    totalOrders: orders.len,
    totalSpent: totalSpent
  )

# === Command Bus ===
type
  CommandResult* = object
    success*: bool
    aggregateId*: string
    error*: string

  CommandHandler*[C] = proc(cmd: C): Future[CommandResult]
  
  CommandBus* = ref object
    store*: EventStore
    readStore*: ReadStore

proc handleCreateOrder*(bus: CommandBus, cmd: CreateOrderCmd): Future[CommandResult] {.async.} =
  if not cmd.validate():
    return CommandResult(success: false, error: "Invalid command")
  
  let total = cmd.items.foldl(a + b.qty.float * b.price, 0.0)
  
  # Write side: append event
  bus.store.openAccount(cmd.aggregateId, total)  # reuse for demo
  
  # Update read model (projection)
  bus.readStore.upsertOrder(OrderSummary(
    id: cmd.aggregateId,
    customerId: cmd.customerId,
    status: "pending",
    totalAmount: total,
    itemCount: cmd.items.len,
    createdAt: getTime(),
    updatedAt: getTime()
  ))
  
  return CommandResult(success: true, aggregateId: cmd.aggregateId)
```

---

## 6. Saga Pattern (Distributed Transactions)

```nim
# saga.nim
# Saga pattern สำหรับ distributed transactions โดยไม่ใช้ 2PC

import std/[asyncdispatch, sequtils, strformat, options]

type
  SagaStepStatus* = enum
    sssNotStarted
    sssCompleted
    sssFailed
    sssCompensated

  SagaStep* = ref object
    name*: string
    status*: SagaStepStatus
    action*: proc(): Future[bool]      # ทำงาน
    compensation*: proc(): Future[bool] # undo ถ้า step อื่น fail

  SagaStatus* = enum
    sagRunning
    sagCompleted
    sagCompensating
    sagFailed

  Saga* = ref object
    id*: string
    steps*: seq[SagaStep]
    status*: SagaStatus
    currentStep*: int
    completedSteps*: seq[int]

proc newSaga*(id: string): Saga =
  Saga(id: id, status: sagRunning, steps: @[])

proc addStep*(saga: Saga, name: string,
    action: proc(): Future[bool],
    compensation: proc(): Future[bool]) =
  saga.steps.add(SagaStep(
    name: name,
    status: sssNotStarted,
    action: action,
    compensation: compensation
  ))

proc execute*(saga: Saga): Future[bool] {.async.} =
  echo &"[Saga:{saga.id}] Starting with {saga.steps.len} steps"
  
  for i, step in saga.steps:
    saga.currentStep = i
    echo &"[Saga:{saga.id}] Executing step {i+1}: {step.name}"
    
    let success = await step.action()
    
    if success:
      step.status = sssCompleted
      saga.completedSteps.add(i)
      echo &"[Saga:{saga.id}] Step {step.name} completed"
    else:
      step.status = sssFailed
      echo &"[Saga:{saga.id}] Step {step.name} failed — compensating"
      saga.status = sagCompensating
      
      # Compensate completed steps in reverse order
      for j in countdown(saga.completedSteps.len - 1, 0):
        let stepIdx = saga.completedSteps[j]
        let failedStep = saga.steps[stepIdx]
        echo &"[Saga:{saga.id}] Compensating: {failedStep.name}"
        
        let compensated = await failedStep.compensation()
        failedStep.status = if compensated: sssCompensated else: sssFailed
      
      saga.status = sagFailed
      return false
  
  saga.status = sagCompleted
  echo &"[Saga:{saga.id}] Completed successfully"
  return true

# Example: Order processing saga
proc createOrderSaga*(orderId, customerId, productId: string,
    qty: int, price: float): Saga =
  let saga = newSaga(&"order-{orderId}")
  
  # Step 1: Reserve inventory
  saga.addStep("ReserveInventory",
    action = proc(): Future[bool] {.async.} =
      echo &"  Reserving {qty}x {productId}"
      # await inventoryService.reserve(productId, qty)
      return true
    ,
    compensation = proc(): Future[bool] {.async.} =
      echo &"  Releasing reserved inventory"
      # await inventoryService.release(productId, qty)
      return true
  )
  
  # Step 2: Charge customer
  saga.addStep("ChargeCustomer",
    action = proc(): Future[bool] {.async.} =
      echo &"  Charging customer {customerId}: ${price * qty.float}"
      # await paymentService.charge(customerId, price * qty.float)
      return true  # Simulate success
    ,
    compensation = proc(): Future[bool] {.async.} =
      echo &"  Refunding customer {customerId}"
      # await paymentService.refund(customerId, price * qty.float)
      return true
  )
  
  # Step 3: Create order record
  saga.addStep("CreateOrderRecord",
    action = proc(): Future[bool] {.async.} =
      echo &"  Creating order record {orderId}"
      # await orderService.create(orderId, customerId, productId, qty, price)
      return true
    ,
    compensation = proc(): Future[bool] {.async.} =
      echo &"  Deleting order record {orderId}"
      # await orderService.delete(orderId)
      return true
  )
  
  # Step 4: Send confirmation email
  saga.addStep("SendConfirmation",
    action = proc(): Future[bool] {.async.} =
      echo &"  Sending order confirmation"
      return true  # Non-critical, but part of saga
    ,
    compensation = proc(): Future[bool] {.async.} =
      echo &"  Sending cancellation email"
      return true
  )
  
  return saga

when isMainModule:
  proc main() {.async.} =
    let saga = createOrderSaga("ORD-001", "CUST-001", "PROD-001", 2, 29.99)
    let success = await saga.execute()
    echo &"Saga result: {'SUCCESS' if success else 'FAILED'}"
  
  waitFor main()
```

---

## 7. สรุป Event-Driven Architecture

```nim
# event_bus.nim
# Simple in-process event bus สำหรับ decoupled components

import std/[tables, asyncdispatch, json, strformat]

type
  EventHandler* = proc(event: JsonNode): Future[void]
  
  EventBus* = ref object
    handlers*: Table[string, seq[EventHandler]]
    middlewares*: seq[proc(eventType: string, event: JsonNode): Future[void]]

proc newEventBus*(): EventBus =
  EventBus(
    handlers: initTable[string, seq[EventHandler]]()
  )

proc on*(bus: EventBus, eventType: string, handler: EventHandler) =
  if not bus.handlers.hasKey(eventType):
    bus.handlers[eventType] = @[]
  bus.handlers[eventType].add(handler)

proc use*(bus: EventBus, middleware: proc(eventType: string, event: JsonNode): Future[void]) =
  bus.middlewares.add(middleware)

proc emit*(bus: EventBus, eventType: string, event: JsonNode) {.async.} =
  # Run middlewares (logging, metrics, tracing)
  for mw in bus.middlewares:
    await mw(eventType, event)
  
  # Dispatch to handlers
  if bus.handlers.hasKey(eventType):
    for handler in bus.handlers[eventType]:
      asyncCheck handler(event)

when isMainModule:
  proc main() {.async.} =
    let bus = newEventBus()
    
    # Logging middleware
    bus.use(proc(eventType: string, event: JsonNode): Future[void] {.async.} =
      echo &"[EventBus] {eventType}: {event}"
    )
    
    # Subscribe
    bus.on("user.created", proc(event: JsonNode): Future[void] {.async.} =
      echo &"Sending welcome email to {event[\"email\"].getStr()}"
    )
    
    bus.on("user.created", proc(event: JsonNode): Future[void] {.async.} =
      echo &"Creating default settings for user {event[\"id\"].getStr()}"
    )
    
    # Emit
    await bus.emit("user.created", %*{
      "id": "usr-001",
      "email": "user@example.com",
      "name": "John"
    })
    
    await sleepAsync(100)
  
  waitFor main()
```

---

## สรุป

| Pattern | เมื่อใช้ | Trade-off |
|---------|---------|-----------|
| Async Queue | Decouple producer/consumer | In-memory = data loss on crash |
| Dead Letter Queue | Handle poison messages | ต้องมีกระบวนการ replay |
| Commit Log | Durable, replayable stream | Disk space, compaction |
| Event Sourcing | Audit trail, time travel | Complex queries, eventual consistency |
| CQRS | High-read performance | Eventual consistency, complexity |
| Saga | Distributed transactions | Eventual consistency, compensation logic |
| Event Bus | Loose coupling | No guaranteed delivery |

---

**Next**: [Part 67 - API Security: OAuth2, JWT, mTLS](../web/part67_api_security.md)
