# Part 20: Threads และ Channels - การทำงานแบบ Concurrent

## Threads เบื้องต้น

```nim
import os, locks

# Thread เบื้องต้น
proc worker(id: int) {.thread.} =
  echo "Worker ", id, " started (thread ", getThreadId(), ")"
  sleep(100 * id)
  echo "Worker ", id, " done"

var threads: array[4, Thread[int]]
for i in 0..<4:
  createThread(threads[i], worker, i)

joinThreads(threads)
echo "All threads done"

# Thread กับ shared data (ต้องระวัง!)
var sharedCounter {.global.} = 0
var counterLock: Lock

proc increment() {.thread.} =
  for i in 0..<1000:
    acquire(counterLock)
    inc sharedCounter
    release(counterLock)

initLock(counterLock)
var incThreads: array[4, Thread[void]]
for t in incThreads.mitems:
  createThread(t, increment)
joinThreads(incThreads)
echo "Counter: ", sharedCounter  # Should be 4000
deinitLock(counterLock)
```

## Channels

```nim
import os

# Channel สำหรับส่งข้อมูลระหว่าง threads
var ch: Channel[string]
ch.open()

proc producer() {.thread.} =
  for i in 0..<5:
    sleep(100)
    let msg = "Message " & $i
    ch.send(msg)
    echo "Sent: ", msg
  ch.send("DONE")

proc consumer() {.thread.} =
  while true:
    let (success, msg) = ch.tryRecv()
    if success:
      if msg == "DONE":
        break
      echo "Received: ", msg
    else:
      sleep(10)

var prodThread, consThread: Thread[void]
createThread(prodThread, producer)
createThread(consThread, consumer)
joinThreads([prodThread, consThread])
ch.close()

# Channel แบบ buffered
var bufferedCh: Channel[int]
bufferedCh.open(10)  # buffer size 10

proc sendNumbers() {.thread.} =
  for i in 1..20:
    bufferedCh.send(i)
  bufferedCh.send(-1)  # sentinel

proc receiveNumbers() {.thread.} =
  var sum = 0
  while true:
    let n = bufferedCh.recv()
    if n == -1: break
    sum += n
  echo "Sum = ", sum  # 210

var sThread, rThread: Thread[void]
createThread(sThread, sendNumbers)
createThread(rThread, receiveNumbers)
joinThreads([sThread, rThread])
bufferedCh.close()
```

## Thread Pool Pattern

```nim
import os, locks, sequtils

type
  Task = proc() {.thread.}
  
  ThreadPool = object
    threads: seq[Thread[void]]
    taskQueue: Channel[Task]
    running: bool

proc workerLoop(pool: ptr ThreadPool) {.thread.} =
  while pool.running:
    let (ok, task) = pool.taskQueue.tryRecv()
    if ok:
      task()
    else:
      sleep(5)

proc newThreadPool(size: int): ptr ThreadPool =
  result = cast[ptr ThreadPool](allocShared0(sizeof(ThreadPool)))
  result.running = true
  result.taskQueue.open(100)
  result.threads = newSeq[Thread[void]](size)
  for i in 0..<size:
    createThread(result.threads[i], workerLoop, result)

proc submit(pool: ptr ThreadPool, task: Task) =
  pool.taskQueue.send(task)

proc stop(pool: ptr ThreadPool) =
  pool.running = false
  joinThreads(pool.threads)
  pool.taskQueue.close()
  deallocShared(pool)

# Usage
let pool = newThreadPool(4)
var resultLock: Lock
var results: seq[int] = @[]
initLock(resultLock)

for i in 0..9:
  let n = i  # capture
  pool.submit(proc() {.thread.} =
    let val = n * n
    acquire(resultLock)
    results.add(val)
    release(resultLock)
  )

sleep(200)
pool.stop()
deinitLock(resultLock)
echo "Results: ", results.sorted()
```

## Parallel Algorithms

```nim
import os, math, sequtils

# Parallel sum
proc parallelSum(arr: seq[int], numThreads: int): int =
  type WorkerArg = tuple[arr: ptr seq[int], start, finish: int, result: ptr int]
  
  var partialSums = newSeq[int](numThreads)
  var threads = newSeq[Thread[tuple[data: seq[int], s, e: int, res: ptr int]]](numThreads)
  
  let chunkSize = arr.len div numThreads
  
  proc sumWorker(args: tuple[data: seq[int], s, e: int, res: ptr int]) {.thread.} =
    var total = 0
    for i in args.s..<args.e:
      total += args.data[i]
    args.res[] = total
  
  for i in 0..<numThreads:
    let start = i * chunkSize
    let finish = if i == numThreads - 1: arr.len else: start + chunkSize
    createThread(threads[i], sumWorker, (arr, start, finish, addr partialSums[i]))
  
  joinThreads(threads)
  return partialSums.foldl(a + b)

# Test
let data = toSeq(1..1000)
echo "Sequential: ", data.foldl(a + b)  # 500500
echo "Parallel:   ", parallelSum(data, 4)  # 500500

# Parallel map
proc parallelMap[T, U](arr: seq[T], f: proc(x: T): U {.thread.}): seq[U] =
  let n = arr.len
  result = newSeq[U](n)
  
  type Arg = tuple[input: T, output: ptr U]
  
  proc worker(arg: Arg) {.thread.} =
    arg.output[] = f(arg.input)
  
  var threads = newSeq[Thread[Arg]](n)
  for i in 0..<n:
    createThread(threads[i], worker, (arr[i], addr result[i]))
  joinThreads(threads)
```

## Async vs Threads

```nim
# Async: I/O bound tasks (HTTP, DB, file)
# Threads: CPU bound tasks (computation)

# สำหรับ CPU-bound tasks, ใช้ threads
import threadpool  # nim's built-in thread pool

proc heavyComputation(n: int): int {.thread.} =
  # expensive computation
  var sum = 0
  for i in 1..n:
    sum += i
  sum

# spawn ใช้ threadpool
let f1 = spawn heavyComputation(1_000_000)
let f2 = spawn heavyComputation(2_000_000)
let f3 = spawn heavyComputation(3_000_000)

echo ^f1  # wait and get result
echo ^f2
echo ^f3

# parallel for loop
import threadpool

proc parallelFor() =
  var results = newSeq[int](100)
  
  parallel:
    for i in 0..<100:
      results[i] = spawn (proc(n: int): int = n * n)(i)

parallelFor()
```

## Practical: Producer-Consumer Pipeline

```nim
import os, locks, tables

type
  RawData = object
    id: int
    content: string
  
  ProcessedData = object
    id: int
    result: string
    processingTime: float

var (rawCh, processedCh): (Channel[RawData], Channel[ProcessedData])
rawCh.open(20)
processedCh.open(20)

# Producer
proc dataProducer() {.thread.} =
  for i in 0..19:
    sleep(10)
    rawCh.send(RawData(id: i, content: "data_" & $i))
  # Send sentinels for each consumer
  for i in 0..1:
    rawCh.send(RawData(id: -1))

# Consumer/Processor
proc dataProcessor(id: int) {.thread.} =
  while true:
    let raw = rawCh.recv()
    if raw.id == -1: break
    
    let start = cpuTime()
    sleep(20)  # simulate work
    let processed = ProcessedData(
      id: raw.id,
      result: raw.content.toUpper(),
      processingTime: cpuTime() - start
    )
    processedCh.send(processed)
  
  processedCh.send(ProcessedData(id: -1))  # sentinel

# Aggregator  
proc resultAggregator() {.thread.} =
  var count = 0
  var done = 0
  while done < 2:  # wait for 2 sentinels
    let (ok, data) = processedCh.tryRecv()
    if ok:
      if data.id == -1:
        inc done
      else:
        inc count
        echo "Processed [", data.id, "]: ", data.result
  echo "Total processed: ", count

var threads: array[4, Thread[void]]
var prod = threads[0]
var cons1, cons2 = threads[1]
var agg = threads[3]

createThread(threads[0], dataProducer)
createThread(threads[1], dataProcessor, 1)
createThread(threads[2], dataProcessor, 2)
createThread(threads[3], resultAggregator)

joinThreads(threads)
rawCh.close()
processedCh.close()
```

## สรุป Part 20

ในบทนี้เราได้เรียนรู้:
- ✅ Thread creation และ joining
- ✅ Locks สำหรับ shared data
- ✅ Channels สำหรับ message passing
- ✅ Thread pool pattern
- ✅ Parallel algorithms
- ✅ threadpool และ spawn
- ✅ Practical: Producer-Consumer pipeline

---

**Previous**: [Part 19 - Async/Await](part19_async.md)
**Next**: [Part 21 - FFI with C](part21_ffi.md)
