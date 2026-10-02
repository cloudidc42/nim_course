# Part 76 - Real-Time Data Processing Pipelines

## บทนำ

Real-time data processing คือการประมวลผลข้อมูลทันทีที่เข้ามา
เรียนรู้: streaming pipelines, windowed aggregations, backpressure,
time-series processing, และ reactive programming patterns ด้วย Nim

---

## 1. Streaming Pipeline

```nim
# pipeline.nim
# Composable streaming pipeline สำหรับ data transformation

import std/[asyncdispatch, asyncfutures, deques, strformat, times, options]
import std/[sequtils, algorithm, tables, locks]

type
  PipelineItem*[T] = object
    value*: T
    timestamp*: Time
    key*: string      # สำหรับ partitioning
    metadata*: seq[(string, string)]

  Stage*[A, B] = ref object
    transform*: proc(item: PipelineItem[A]): Future[Option[PipelineItem[B]]]
    name*: string
    processed*: int64
    dropped*: int64

  Pipeline*[A, B] = ref object
    stages*: seq[pointer]  # Heterogeneous stages
    inputQueue*: Deque[PipelineItem[A]]
    outputQueue*: Deque[PipelineItem[B]]
    running*: bool
    backpressureThreshold*: int

proc newStage*[A, B](name: string,
    transform: proc(item: PipelineItem[A]): Future[Option[PipelineItem[B]]]): Stage[A, B] =
  Stage[A, B](transform: transform, name: name)

# Map stage
proc mapStage*[A, B](name: string, fn: proc(x: A): B): Stage[A, B] =
  newStage[A, B](name, proc(item: PipelineItem[A]): Future[Option[PipelineItem[B]]] {.async.} =
    let transformed = PipelineItem[B](
      value: fn(item.value),
      timestamp: item.timestamp,
      key: item.key,
      metadata: item.metadata
    )
    return some(transformed)
  )

# Filter stage
proc filterStage*[A](name: string, predicate: proc(x: A): bool): Stage[A, A] =
  newStage[A, A](name, proc(item: PipelineItem[A]): Future[Option[PipelineItem[A]]] {.async.} =
    if predicate(item.value):
      return some(item)
    return none(PipelineItem[A])
  )

# Flat map stage
proc flatMapStage*[A, B](name: string,
    fn: proc(x: A): seq[B]): Stage[A, B] =
  # Note: simplified — real flatMap needs queue buffering
  newStage[A, B](name, proc(item: PipelineItem[A]): Future[Option[PipelineItem[B]]] {.async.} =
    let results = fn(item.value)
    if results.len == 0:
      return none(PipelineItem[B])
    return some(PipelineItem[B](
      value: results[0],
      timestamp: item.timestamp,
      key: item.key
    ))
  )

# Enrichment stage (async lookup)
proc enrichStage*[A, B](name: string,
    enrich: proc(x: A): Future[B]): Stage[A, B] =
  newStage[A, B](name, proc(item: PipelineItem[A]): Future[Option[PipelineItem[B]]] {.async.} =
    let enriched = await enrich(item.value)
    return some(PipelineItem[B](
      value: enriched,
      timestamp: item.timestamp,
      key: item.key
    ))
  )

# Example pipeline stages for IoT data
type
  RawSensorData* = object
    deviceId*: string
    sensorType*: string
    rawValue*: float64
    timestamp*: int64

  NormalizedData* = object
    deviceId*: string
    sensorType*: string
    value*: float64
    unit*: string
    timestamp*: int64

  AnomalyAlert* = object
    deviceId*: string
    message*: string
    value*: float64
    severity*: string

proc normalizeSensor*(raw: RawSensorData): NormalizedData =
  let (value, unit) = case raw.sensorType
    of "temperature": (raw.rawValue / 10.0, "celsius")
    of "humidity":    (raw.rawValue / 100.0, "percent")
    of "pressure":    (raw.rawValue * 0.01, "hPa")
    else:             (raw.rawValue, "unknown")
  
  NormalizedData(
    deviceId: raw.deviceId,
    sensorType: raw.sensorType,
    value: value,
    unit: unit,
    timestamp: raw.timestamp
  )

proc detectAnomaly*(data: NormalizedData): bool =
  case data.sensorType
  of "temperature": data.value < -50.0 or data.value > 150.0
  of "humidity":    data.value < 0.0 or data.value > 100.0
  else:             false

when isMainModule:
  let normalizeStage = mapStage[RawSensorData, NormalizedData](
    "normalize",
    normalizeSensor
  )
  
  let filterAnomalies = filterStage[NormalizedData](
    "filter-anomalies",
    proc(d: NormalizedData): bool = not detectAnomaly(d)
  )
  
  # Process sample data
  let raw = RawSensorData(
    deviceId: "sensor-001",
    sensorType: "temperature",
    rawValue: 250.0,
    timestamp: epochTime().int64
  )
  
  let item = PipelineItem[RawSensorData](
    value: raw,
    timestamp: getTime(),
    key: raw.deviceId
  )
  
  proc run() {.async.} =
    let normalized = await normalizeStage.transform(item)
    if normalized.isSome:
      echo &"Normalized: {normalized.get().value.value} {normalized.get().value.unit}"
      
      let filtered = await filterAnomalies.transform(normalized.get())
      echo &"Passed filter: {filtered.isSome}"
  
  waitFor run()
```

---

## 2. Windowed Aggregations

```nim
# windowing.nim
# Tumbling, sliding, and session windows สำหรับ streaming

import std/[times, tables, sequtils, math, strformat, deques, options]

type
  WindowType* = enum
    wTumbling   # Fixed-size non-overlapping windows
    wSliding    # Fixed-size overlapping windows
    wSession    # Dynamic session windows

  WindowConfig* = object
    windowType*: WindowType
    size*: Duration       # Window size (tumbling/sliding)
    slide*: Duration      # Slide interval (sliding only)
    sessionGap*: Duration # Session gap (session only)

  Window*[T] = object
    start*: Time
    `end`*: Time
    items*: seq[T]
    key*: string

  WindowedAggregator*[T, A] = ref object
    config*: WindowConfig
    windows*: Table[string, seq[Window[T]]]  # key -> windows
    aggregate*: proc(items: seq[T]): A
    onComplete*: proc(key: string, window: Window[T], result: A)

proc newTumblingWindow*[T, A](size: Duration,
    aggregate: proc(items: seq[T]): A,
    onComplete: proc(key: string, window: Window[T], result: A)): WindowedAggregator[T, A] =
  WindowedAggregator[T, A](
    config: WindowConfig(windowType: wTumbling, size: size),
    windows: initTable[string, seq[Window[T]]](),
    aggregate: aggregate,
    onComplete: onComplete
  )

proc addItem*[T, A](agg: WindowedAggregator[T, A], item: T,
    timestamp: Time, key = "default") =
  if not agg.windows.hasKey(key):
    agg.windows[key] = @[]
  
  let windows = agg.windows[key]
  
  case agg.config.windowType
  of wTumbling:
    # Find or create window for this timestamp
    let windowStart = Time(
      int(timestamp.toUnix()) div int(agg.config.size.inSeconds) * int(agg.config.size.inSeconds)
    ).fromUnix(0)
    let windowEnd = windowStart + agg.config.size
    
    var found = false
    for i in 0..<windows.len:
      if windows[i].start == windowStart:
        agg.windows[key][i].items.add(item)
        found = true
        break
    
    if not found:
      agg.windows[key].add(Window[T](
        start: windowStart,
        `end`: windowEnd,
        items: @[item],
        key: key
      ))
  
  of wSliding:
    # Item belongs to multiple windows
    let slide = agg.config.slide
    let size = agg.config.size
    # Calculate all windows that contain this timestamp
    discard  # Simplified
  
  of wSession:
    # Extend or create session window
    if windows.len == 0:
      agg.windows[key].add(Window[T](
        start: timestamp,
        `end`: timestamp + agg.config.sessionGap,
        items: @[item],
        key: key
      ))
    else:
      var lastWindow = agg.windows[key][^1]
      if timestamp <= lastWindow.`end`:
        # Extend session
        agg.windows[key][^1].items.add(item)
        agg.windows[key][^1].`end` = timestamp + agg.config.sessionGap
      else:
        # New session
        agg.windows[key].add(Window[T](
          start: timestamp,
          `end`: timestamp + agg.config.sessionGap,
          items: @[item],
          key: key
        ))

proc flush*[T, A](agg: WindowedAggregator[T, A], upTo: Time) =
  ## Emit completed windows
  for key, windows in agg.windows.mpairs:
    var remaining: seq[Window[T]]
    
    for window in windows:
      if window.`end` <= upTo:
        let result = agg.aggregate(window.items)
        agg.onComplete(key, window, result)
      else:
        remaining.add(window)
    
    agg.windows[key] = remaining

# Statistical aggregations
proc mean*(values: seq[float64]): float64 =
  if values.len == 0: return 0.0
  values.foldl(a + b) / float(values.len)

proc stddev*(values: seq[float64]): float64 =
  if values.len < 2: return 0.0
  let m = values.mean()
  let variance = values.mapIt((it - m) * (it - m)).mean()
  sqrt(variance)

proc percentile*(values: seq[float64], p: float64): float64 =
  if values.len == 0: return 0.0
  var sorted = values
  sorted.sort()
  let idx = int(p / 100.0 * float(sorted.len - 1))
  sorted[min(idx, sorted.len - 1)]

type
  WindowStats* = object
    count*: int
    sum*: float64
    mean*: float64
    min*: float64
    max*: float64
    stddev*: float64
    p50*, p95*, p99*: float64

proc computeStats*(values: seq[float64]): WindowStats =
  if values.len == 0:
    return WindowStats()
  
  WindowStats(
    count: values.len,
    sum: values.foldl(a + b),
    mean: values.mean(),
    min: values.foldl(min(a, b)),
    max: values.foldl(max(a, b)),
    stddev: values.stddev(),
    p50: values.percentile(50),
    p95: values.percentile(95),
    p99: values.percentile(99)
  )
```

---

## 3. Time-Series Database

```nim
# timeseries.nim
# Simple in-memory time-series database with compression

import std/[times, tables, strformat, math, algorithm, sequtils]

type
  Sample* = object
    timestamp*: int64  # Unix timestamp milliseconds
    value*: float64

  MetricSeries* = ref object
    name*: string
    labels*: Table[string, string]  # e.g. {host: "server1", env: "prod"}
    samples*: seq[Sample]
    retention*: Duration

  TSDB* = ref object
    series*: Table[string, MetricSeries]
    retention*: Duration

proc seriesKey*(name: string, labels: Table[string, string]): string =
  ## Create unique key from name + sorted labels
  var labelParts: seq[string]
  for k, v in labels:
    labelParts.add(&"{k}={v}")
  labelParts.sort()
  name & "{" & labelParts.join(",") & "}"

proc newTSDB*(retention = initDuration(hours = 24)): TSDB =
  TSDB(series: initTable[string, MetricSeries](), retention: retention)

proc write*(db: TSDB, name: string, value: float64,
    labels: Table[string, string] = initTable[string, string](),
    timestamp: int64 = 0) =
  let ts = if timestamp > 0: timestamp else: (epochTime() * 1000).int64
  let key = seriesKey(name, labels)
  
  if not db.series.hasKey(key):
    db.series[key] = MetricSeries(
      name: name,
      labels: labels,
      samples: @[],
      retention: db.retention
    )
  
  db.series[key].samples.add(Sample(timestamp: ts, value: value))

proc query*(db: TSDB, name: string, 
    labels: Table[string, string] = initTable[string, string](),
    fromMs, toMs: int64): seq[Sample] =
  let key = seriesKey(name, labels)
  
  if not db.series.hasKey(key):
    return @[]
  
  db.series[key].samples.filterIt(
    it.timestamp >= fromMs and it.timestamp <= toMs
  )

proc aggregate*(samples: seq[Sample], interval: int64,
    fn: proc(values: seq[float64]): float64): seq[Sample] =
  ## Downsample: aggregate samples into larger intervals
  if samples.len == 0: return @[]
  
  let firstTs = samples[0].timestamp
  let lastTs = samples[^1].timestamp
  
  var result: seq[Sample]
  var ts = firstTs - (firstTs mod interval)
  
  while ts <= lastTs:
    let bucketSamples = samples.filterIt(
      it.timestamp >= ts and it.timestamp < ts + interval
    )
    
    if bucketSamples.len > 0:
      let values = bucketSamples.mapIt(it.value)
      result.add(Sample(timestamp: ts, value: fn(values)))
    
    ts += interval
  
  result

# Delta compression for storage efficiency
proc deltaEncode*(samples: seq[Sample]): tuple[base: int64, deltas: seq[int32], values: seq[float64]] =
  if samples.len == 0:
    return (base: 0, deltas: @[], values: @[])
  
  let base = samples[0].timestamp
  var deltas: seq[int32]
  var values: seq[float64]
  
  deltas.add(0)
  values.add(samples[0].value)
  
  for i in 1..<samples.len:
    deltas.add(int32(samples[i].timestamp - samples[i-1].timestamp))
    values.add(samples[i].value)
  
  (base: base, deltas: deltas, values: values)

proc deltaDecode*(base: int64, deltas: seq[int32], values: seq[float64]): seq[Sample] =
  result = @[]
  var ts = base
  
  for i in 0..<deltas.len:
    ts += deltas[i]
    result.add(Sample(timestamp: ts, value: values[i]))

# Range query with aggregation
proc rangeQuery*(db: TSDB, name: string, 
    fromMs, toMs, stepMs: int64,
    labels: Table[string, string] = initTable[string, string]()): seq[Sample] =
  let raw = db.query(name, labels, fromMs, toMs)
  raw.aggregate(stepMs, mean)

# Retention cleanup
proc evict*(db: TSDB) =
  let cutoff = int64((epochTime() - db.retention.inSeconds.float) * 1000)
  for key, series in db.series.mpairs:
    series.samples = series.samples.filterIt(it.timestamp > cutoff)

when isMainModule:
  let db = newTSDB(retention = initDuration(hours = 1))
  
  let now = int64(epochTime() * 1000)
  
  # Simulate writing metrics
  let labels = {"host": "server1", "env": "prod"}.toTable
  
  for i in 0..59:
    let ts = now - int64(59 - i) * 1000  # One per second
    let value = 20.0 + sin(float(i) / 10.0) * 5.0  # Sinusoidal pattern
    db.write("cpu_temp", value, labels, ts)
  
  # Query last 30 seconds
  let samples = db.query("cpu_temp", labels, now - 30000, now)
  echo &"Found {samples.len} samples in last 30s"
  
  # Aggregate to 5-second buckets
  let aggregated = samples.aggregate(5000, mean)
  for s in aggregated:
    echo &"  {s.timestamp}: {s.value:.2f}°C"
```

---

## 4. Reactive Event Stream

```nim
# reactive.nim
# Observable/Observer pattern สำหรับ reactive programming

import std/[asyncdispatch, asyncfutures, sequtils, options, strformat]
import std/[tables, times]

type
  Observer*[T] = ref object
    onNext*: proc(value: T)
    onError*: proc(err: ref Exception)
    onComplete*: proc()

  Observable*[T] = ref object
    subscribe*: proc(observer: Observer[T])
    name*: string

  Subject*[T] = ref object
    observers*: seq[Observer[T]]
    completed*: bool
    error*: ref Exception

proc newObserver*[T](onNext: proc(value: T),
    onError: proc(err: ref Exception) = nil,
    onComplete: proc() = nil): Observer[T] =
  Observer[T](onNext: onNext, onError: onError, onComplete: onComplete)

proc newSubject*[T](): Subject[T] =
  Subject[T](observers: @[])

proc subscribe*[T](subject: Subject[T], observer: Observer[T]) =
  subject.observers.add(observer)

proc next*[T](subject: Subject[T], value: T) =
  if subject.completed or subject.error != nil: return
  for obs in subject.observers:
    if obs.onNext != nil:
      obs.onNext(value)

proc complete*[T](subject: Subject[T]) =
  if subject.completed: return
  subject.completed = true
  for obs in subject.observers:
    if obs.onComplete != nil:
      obs.onComplete()

proc error*[T](subject: Subject[T], err: ref Exception) =
  subject.error = err
  for obs in subject.observers:
    if obs.onError != nil:
      obs.onError(err)

proc toObservable*[T](subject: Subject[T]): Observable[T] =
  Observable[T](
    name: "subject",
    subscribe: proc(observer: Observer[T]) =
      subject.subscribe(observer)
  )

# Operators
proc map*[A, B](source: Observable[A], fn: proc(x: A): B): Observable[B] =
  Observable[B](
    name: "map",
    subscribe: proc(observer: Observer[B]) =
      source.subscribe(newObserver[A](
        onNext = proc(x: A) = observer.onNext(fn(x)),
        onError = observer.onError,
        onComplete = observer.onComplete
      ))
  )

proc filter*[T](source: Observable[T], predicate: proc(x: T): bool): Observable[T] =
  Observable[T](
    name: "filter",
    subscribe: proc(observer: Observer[T]) =
      source.subscribe(newObserver[T](
        onNext = proc(x: T) =
          if predicate(x): observer.onNext(x),
        onError = observer.onError,
        onComplete = observer.onComplete
      ))
  )

proc debounce*[T](source: Observable[T], durationMs: int): Observable[T] =
  ## Emit only after no events for durationMs
  Observable[T](
    name: "debounce",
    subscribe: proc(observer: Observer[T]) =
      var lastTimer: Future[void]
      source.subscribe(newObserver[T](
        onNext = proc(x: T) =
          if lastTimer != nil and not lastTimer.finished:
            lastTimer.cancel()
          lastTimer = sleepAsync(durationMs)
          asyncCheck lastTimer.then(proc(): Future[void] {.async.} =
            observer.onNext(x)
          ),
        onError = observer.onError,
        onComplete = observer.onComplete
      ))
  )

proc throttle*[T](source: Observable[T], durationMs: int): Observable[T] =
  ## Emit at most once per durationMs
  Observable[T](
    name: "throttle",
    subscribe: proc(observer: Observer[T]) =
      var lastEmit = 0'i64
      source.subscribe(newObserver[T](
        onNext = proc(x: T) =
          let now = int64(epochTime() * 1000)
          if now - lastEmit >= durationMs:
            lastEmit = now
            observer.onNext(x),
        onError = observer.onError,
        onComplete = observer.onComplete
      ))
  )

proc take*[T](source: Observable[T], n: int): Observable[T] =
  Observable[T](
    name: "take",
    subscribe: proc(observer: Observer[T]) =
      var count = 0
      source.subscribe(newObserver[T](
        onNext = proc(x: T) =
          if count < n:
            inc count
            observer.onNext(x)
            if count >= n:
              observer.onComplete(),
        onError = observer.onError,
        onComplete = observer.onComplete
      ))
  )

when isMainModule:
  proc main() {.async.} =
    let subject = newSubject[int]()
    
    # Build pipeline
    let pipeline = subject.toObservable()
      .filter(proc(x: int): bool = x mod 2 == 0)
      .map(proc(x: int): string = &"Even: {x}")
      .take(3)
    
    pipeline.subscribe(newObserver[string](
      onNext = proc(s: string) = echo s,
      onComplete = proc() = echo "Stream complete"
    ))
    
    for i in 1..10:
      subject.next(i)
      await sleepAsync(100)
    
    subject.complete()
  
  waitFor main()
```

---

## สรุป

| Pattern | Use Case |
|---------|---------|
| Pipeline | ETL, data transformation |
| Tumbling Window | Per-minute statistics |
| Sliding Window | Moving average |
| Session Window | User activity sessions |
| TSDB | Metrics, monitoring |
| Observable | Event-driven data flows |
| Debounce/Throttle | Rate limiting events |

---

**Next**: [Part 77 - Advanced Concurrency Patterns](../advanced/part77_concurrency.md)
