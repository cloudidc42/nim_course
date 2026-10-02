# Part 78 - Production Deployment & Monitoring

## บทนำ

การ deploy Nim applications สู่ production ต้องการ:
- Docker containerization
- Health checks และ graceful shutdown
- Metrics (Prometheus)
- Structured logging
- Configuration management
- Circuit breakers สำหรับ external dependencies

---

## 1. Docker + Multi-Stage Build

```dockerfile
# Dockerfile สำหรับ Nim web service
FROM nimlang/nim:2.0.0 AS builder

WORKDIR /app
COPY *.nimble ./
COPY src/ ./src/

# Install dependencies
RUN nimble install -y --depsOnly

# Build release binary
RUN nim c \
    -d:release \
    -d:danger \
    --opt:speed \
    --passC:-march=x86-64 \
    --mm:orc \
    -o:server \
    src/server.nim

# Runtime image (minimal)
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    libssl3 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /app/server ./server
COPY config/ ./config/

# Non-root user
RUN useradd -r -s /bin/false appuser
USER appuser

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1

CMD ["./server"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - PORT=8080
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
      - REDIS_URL=redis://redis:6379
      - LOG_LEVEL=info
      - METRICS_ENABLED=true
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 256M

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 64mb --maxmemory-policy allkeys-lru

  prometheus:
    image: prom/prometheus:v2.48.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:10.2.0
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

volumes:
  pgdata:
```

---

## 2. Graceful Shutdown

```nim
# graceful_shutdown.nim
# Graceful shutdown สำหรับ production server

import std/[asyncdispatch, asynchttpserver, locks, atomics, strformat, times, os, posix]

type
  ShutdownManager* = ref object
    shutdownFlag*: Atomic[bool]
    activeRequests*: Atomic[int]
    drainTimeout*: Duration
    lock*: Lock
    onShutdown*: seq[proc()]

proc newShutdownManager*(drainTimeout = initDuration(seconds=30)): ShutdownManager =
  result = ShutdownManager(drainTimeout: drainTimeout)
  initLock(result.lock)

proc isShuttingDown*(sm: ShutdownManager): bool =
  sm.shutdownFlag.load()

proc requestStart*(sm: ShutdownManager): bool =
  ## Returns false if shutting down
  if sm.isShuttingDown():
    return false
  discard sm.activeRequests.fetchAdd(1)
  true

proc requestEnd*(sm: ShutdownManager) =
  discard sm.activeRequests.fetchSub(1)

proc shutdown*(sm: ShutdownManager) =
  sm.shutdownFlag.store(true)
  echo "Shutdown initiated, draining requests..."
  
  # Wait for active requests to complete
  let deadline = getTime() + sm.drainTimeout
  while sm.activeRequests.load() > 0 and getTime() < deadline:
    sleep(100)
  
  if sm.activeRequests.load() > 0:
    echo &"Warning: {sm.activeRequests.load()} requests still active after drain timeout"
  
  # Run cleanup hooks
  acquire(sm.lock)
  for hook in sm.onShutdown:
    try:
      hook()
    except CatchableError as e:
      echo &"Shutdown hook error: {e.msg}"
  release(sm.lock)
  
  echo "Shutdown complete"

proc onShutdown*(sm: ShutdownManager, fn: proc()) =
  acquire(sm.lock)
  sm.onShutdown.add(fn)
  release(sm.lock)

# Signal handling
var globalShutdown*: ShutdownManager

proc handleSignal(sig: cint) {.noconv.} =
  echo &"\nReceived signal {sig}, starting graceful shutdown..."
  if not globalShutdown.isNil:
    globalShutdown.shutdown()
  quit(0)

proc setupSignalHandlers*(sm: ShutdownManager) =
  globalShutdown = sm
  signal(SIGTERM, handleSignal)
  signal(SIGINT, handleSignal)

# Server with graceful shutdown
type
  ProductionServer* = ref object
    server*: AsyncHttpServer
    shutdown*: ShutdownManager
    port*: Port

proc newProductionServer*(port: int): ProductionServer =
  result = ProductionServer(
    server: newAsyncHttpServer(),
    shutdown: newShutdownManager(),
    port: Port(port)
  )
  setupSignalHandlers(result.shutdown)

proc handleRequest*(srv: ProductionServer, handler: proc(req: Request): Future[void]): 
    proc(req: Request): Future[void] =
  proc wrappedHandler(req: Request) {.async.} =
    if not srv.shutdown.requestStart():
      await req.respond(Http503, "Service Unavailable - shutting down")
      return
    defer: srv.shutdown.requestEnd()
    await handler(req)
  wrappedHandler

proc serve*(srv: ProductionServer, handler: proc(req: Request): Future[void]) {.async.} =
  echo &"Server listening on port {srv.port.int}"
  await srv.server.serve(srv.port, srv.handleRequest(handler))
```

---

## 3. Prometheus Metrics

```nim
# metrics.nim
# Prometheus metrics สำหรับ Nim applications

import std/[tables, locks, strformat, times, atomics, sequtils, algorithm, math]

type
  MetricType* = enum
    mtCounter
    mtGauge
    mtHistogram
    mtSummary

  LabelSet* = Table[string, string]

  MetricValue* = object
    value*: float64
    labels*: LabelSet
    timestamp*: int64

  Metric* = ref object
    name*: string
    help*: string
    metricType*: MetricType
    values*: seq[MetricValue]
    lock*: Lock

  HistogramMetric* = ref object
    base*: Metric
    buckets*: seq[float64]
    counts*: Table[string, seq[int64]]  # label_key -> bucket counts
    sums*: Table[string, float64]
    totalCounts*: Table[string, int64]

  Registry* = ref object
    metrics*: Table[string, Metric]
    histograms*: Table[string, HistogramMetric]
    lock*: Lock

proc newRegistry*(): Registry =
  result = Registry(
    metrics: initTable[string, Metric](),
    histograms: initTable[string, HistogramMetric]()
  )
  initLock(result.lock)

var defaultRegistry* = newRegistry()

proc labelKey*(labels: LabelSet): string =
  var pairs: seq[string]
  for k, v in labels:
    pairs.add(&"{k}={v}")
  pairs.sort()
  pairs.join(",")

# Counter
proc counter*(registry: Registry, name, help: string): Metric =
  acquire(registry.lock)
  defer: release(registry.lock)
  
  if name in registry.metrics:
    return registry.metrics[name]
  
  let m = Metric(name: name, help: help, metricType: mtCounter)
  initLock(m.lock)
  registry.metrics[name] = m
  m

proc inc*(m: Metric, amount = 1.0, labels: LabelSet = initTable[string, string]()) =
  acquire(m.lock)
  defer: release(m.lock)
  
  let key = labels.labelKey()
  for i in 0..<m.values.len:
    if m.values[i].labels.labelKey() == key:
      m.values[i].value += amount
      return
  
  m.values.add(MetricValue(value: amount, labels: labels,
    timestamp: getTime().toUnixFloat().int64))

# Gauge
proc gauge*(registry: Registry, name, help: string): Metric =
  acquire(registry.lock)
  defer: release(registry.lock)
  
  if name in registry.metrics:
    return registry.metrics[name]
  
  let m = Metric(name: name, help: help, metricType: mtGauge)
  initLock(m.lock)
  registry.metrics[name] = m
  m

proc set*(m: Metric, value: float64, labels: LabelSet = initTable[string, string]()) =
  acquire(m.lock)
  defer: release(m.lock)
  
  let key = labels.labelKey()
  for i in 0..<m.values.len:
    if m.values[i].labels.labelKey() == key:
      m.values[i].value = value
      return
  
  m.values.add(MetricValue(value: value, labels: labels,
    timestamp: getTime().toUnixFloat().int64))

# Histogram
proc histogram*(registry: Registry, name, help: string,
    buckets = @[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]): HistogramMetric =
  acquire(registry.lock)
  defer: release(registry.lock)
  
  if name in registry.histograms:
    return registry.histograms[name]
  
  let base = Metric(name: name, help: help, metricType: mtHistogram)
  initLock(base.lock)
  
  let h = HistogramMetric(
    base: base,
    buckets: buckets & @[Inf],
    counts: initTable[string, seq[int64]](),
    sums: initTable[string, float64](),
    totalCounts: initTable[string, int64]()
  )
  registry.histograms[name] = h
  h

proc observe*(h: HistogramMetric, value: float64, labels: LabelSet = initTable[string, string]()) =
  acquire(h.base.lock)
  defer: release(h.base.lock)
  
  let key = labels.labelKey()
  
  if key notin h.counts:
    h.counts[key] = newSeq[int64](h.buckets.len)
    h.sums[key] = 0.0
    h.totalCounts[key] = 0
  
  for i, bucket in h.buckets:
    if value <= bucket:
      inc h.counts[key][i]
  
  h.sums[key] += value
  inc h.totalCounts[key]

# Timer helper
template timeIt*(h: HistogramMetric, labels: LabelSet, body: untyped) =
  let start = cpuTime()
  body
  h.observe(cpuTime() - start, labels)

# Prometheus text format export
proc toPrometheus*(registry: Registry): string =
  var lines: seq[string]
  
  acquire(registry.lock)
  defer: release(registry.lock)
  
  for name, metric in registry.metrics:
    lines.add(&"# HELP {name} {metric.help}")
    lines.add(&"# TYPE {name} {($metric.metricType).toLowerAscii()}")
    
    for val in metric.values:
      var labelStr = ""
      if val.labels.len > 0:
        var pairs: seq[string]
        for k, v in val.labels:
          pairs.add(&"{k}=\"{v}\"")
        labelStr = &"{{{pairs.join(\",\")}}}"
      
      lines.add(&"{name}{labelStr} {val.value}")
  
  for name, hist in registry.histograms:
    lines.add(&"# HELP {name} {hist.base.help}")
    lines.add(&"# TYPE {name} histogram")
    
    for key, counts in hist.counts:
      for i, bucket in hist.buckets:
        let bucketStr = if bucket == Inf: "+Inf" else: $bucket
        lines.add(&"{name}_bucket{{le=\"{bucketStr}\"}} {counts[i]}")
      
      lines.add(&"{name}_sum {hist.sums[key]}")
      lines.add(&"{name}_count {hist.totalCounts[key]}")
  
  lines.join("\n") & "\n"
```

---

## 4. Structured Logging

```nim
# structured_log.nim
# Structured JSON logging สำหรับ production

import std/[json, times, tables, strformat, locks, os, strutils]

type
  LogLevel* = enum
    llDebug = "DEBUG"
    llInfo = "INFO"
    llWarn = "WARN"
    llError = "ERROR"
    llFatal = "FATAL"

  LogEntry* = object
    timestamp*: string
    level*: string
    message*: string
    service*: string
    fields*: Table[string, JsonNode]
    traceId*: string
    spanId*: string

  Logger* = ref object
    level*: LogLevel
    service*: string
    output*: File
    lock*: Lock
    fields*: Table[string, JsonNode]  # Default fields

proc newLogger*(service: string, level = llInfo, output = stdout): Logger =
  result = Logger(
    level: level,
    service: service,
    output: output,
    fields: initTable[string, JsonNode]()
  )
  initLock(result.lock)

proc withField*(logger: Logger, key: string, value: JsonNode): Logger =
  ## Create child logger with extra field
  result = Logger(
    level: logger.level,
    service: logger.service,
    output: logger.output,
    fields: logger.fields
  )
  initLock(result.lock)
  result.fields[key] = value

proc log*(logger: Logger, level: LogLevel, msg: string,
    fields: Table[string, JsonNode] = initTable[string, JsonNode](),
    traceId = "", spanId = "") =
  
  if level < logger.level:
    return
  
  var entry = %*{
    "timestamp": getTime().format("yyyy-MM-dd'T'HH:mm:ss'.'fff'Z'"),
    "level": $level,
    "message": msg,
    "service": logger.service
  }
  
  if traceId.len > 0:
    entry["trace_id"] = %traceId
  if spanId.len > 0:
    entry["span_id"] = %spanId
  
  # Merge default fields
  for k, v in logger.fields:
    entry[k] = v
  
  # Merge call-site fields
  for k, v in fields:
    entry[k] = v
  
  acquire(logger.lock)
  logger.output.writeLine($entry)
  logger.output.flushFile()
  release(logger.lock)

proc debug*(logger: Logger, msg: string, fields = initTable[string, JsonNode]()) =
  logger.log(llDebug, msg, fields)

proc info*(logger: Logger, msg: string, fields = initTable[string, JsonNode]()) =
  logger.log(llInfo, msg, fields)

proc warn*(logger: Logger, msg: string, fields = initTable[string, JsonNode]()) =
  logger.log(llWarn, msg, fields)

proc error*(logger: Logger, msg: string, fields = initTable[string, JsonNode]()) =
  logger.log(llError, msg, fields)

proc fatal*(logger: Logger, msg: string, fields = initTable[string, JsonNode]()) =
  logger.log(llFatal, msg, fields)
  quit(1)

# Convenience: log with typed fields
template logf*(logger: Logger, level: LogLevel, msg: string, body: untyped) =
  var fields = initTable[string, JsonNode]()
  template field(k: string, v: untyped) = fields[k] = %v
  body
  logger.log(level, msg, fields)

# Example usage
when isMainModule:
  let log = newLogger("myservice", llInfo)
  
  log.info("Server started", {"port": %8080, "env": %"production"}.toTable)
  
  logf(log, llError, "Request failed"):
    field("path", "/api/users")
    field("status", 500)
    field("latency_ms", 234)
    field("user_id", "usr_123")
```

---

## 5. Configuration Management

```nim
# config.nim
# Type-safe configuration management

import std/[os, strutils, json, tables, options, strformat]

type
  ConfigError* = object of CatchableError

  DatabaseConfig* = object
    host*: string
    port*: int
    name*: string
    user*: string
    password*: string
    maxConnections*: int
    sslMode*: string

  RedisConfig* = object
    host*: string
    port*: int
    password*: string
    db*: int
    maxConnections*: int

  ServerConfig* = object
    port*: int
    host*: string
    readTimeout*: int    # milliseconds
    writeTimeout*: int
    maxBodySize*: int64  # bytes

  AppConfig* = object
    env*: string
    logLevel*: string
    server*: ServerConfig
    database*: DatabaseConfig
    redis*: RedisConfig
    jwtSecret*: string
    metricsEnabled*: bool

proc getEnv*(key, default: string): string =
  let val = os.getEnv(key)
  if val.len == 0: default else: val

proc getEnvInt*(key: string, default: int): int =
  let val = os.getEnv(key)
  if val.len == 0: return default
  try: parseInt(val)
  except ValueError:
    raise newException(ConfigError, &"Invalid integer for {key}: {val}")

proc getEnvBool*(key: string, default: bool): bool =
  let val = os.getEnv(key).toLowerAscii()
  case val
  of "true", "1", "yes", "on": true
  of "false", "0", "no", "off": false
  of "": default
  else:
    raise newException(ConfigError, &"Invalid bool for {key}: {val}")

proc getEnvRequired*(key: string): string =
  let val = os.getEnv(key)
  if val.len == 0:
    raise newException(ConfigError, &"Required environment variable {key} not set")
  val

proc loadConfig*(): AppConfig =
  result = AppConfig(
    env: getEnv("APP_ENV", "development"),
    logLevel: getEnv("LOG_LEVEL", "info"),
    metricsEnabled: getEnvBool("METRICS_ENABLED", true),
    jwtSecret: getEnv("JWT_SECRET", "dev-secret-change-in-production"),
    
    server: ServerConfig(
      port: getEnvInt("PORT", 8080),
      host: getEnv("HOST", "0.0.0.0"),
      readTimeout: getEnvInt("READ_TIMEOUT_MS", 30000),
      writeTimeout: getEnvInt("WRITE_TIMEOUT_MS", 30000),
      maxBodySize: getEnvInt("MAX_BODY_SIZE", 10 * 1024 * 1024)
    ),
    
    database: DatabaseConfig(
      host: getEnv("DB_HOST", "localhost"),
      port: getEnvInt("DB_PORT", 5432),
      name: getEnv("DB_NAME", "app"),
      user: getEnv("DB_USER", "postgres"),
      password: getEnv("DB_PASSWORD", ""),
      maxConnections: getEnvInt("DB_MAX_CONNECTIONS", 10),
      sslMode: getEnv("DB_SSL_MODE", "disable")
    ),
    
    redis: RedisConfig(
      host: getEnv("REDIS_HOST", "localhost"),
      port: getEnvInt("REDIS_PORT", 6379),
      password: getEnv("REDIS_PASSWORD", ""),
      db: getEnvInt("REDIS_DB", 0),
      maxConnections: getEnvInt("REDIS_MAX_CONNECTIONS", 10)
    )
  )
  
  # Validate in production
  if result.env == "production":
    if result.jwtSecret == "dev-secret-change-in-production":
      raise newException(ConfigError, "JWT_SECRET must be set in production")
    if result.database.password.len == 0:
      raise newException(ConfigError, "DB_PASSWORD must be set in production")

proc databaseUrl*(cfg: DatabaseConfig): string =
  &"postgresql://{cfg.user}:{cfg.password}@{cfg.host}:{cfg.port}/{cfg.name}?sslmode={cfg.sslMode}"

when isMainModule:
  try:
    let cfg = loadConfig()
    echo &"Loaded config for env: {cfg.env}"
    echo &"Server: {cfg.server.host}:{cfg.server.port}"
    echo &"DB: {cfg.database.host}:{cfg.database.port}/{cfg.database.name}"
  except ConfigError as e:
    echo &"Config error: {e.msg}"
    quit(1)
```

---

## สรุป

| Component | Pattern |
|-----------|---------|
| Docker | Multi-stage build ลด image size |
| Shutdown | Drain active requests → run hooks → exit |
| Metrics | Prometheus counter/gauge/histogram |
| Logging | Structured JSON ด้วย trace context |
| Config | Environment variables + validation |

---

**Next**: [Part 79 - Writing a Compiler/Interpreter in Nim](../advanced/part79_compiler.md)
