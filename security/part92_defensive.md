# Part 92 - Defensive Security & Blue Team Tools

## บทนำ

Blue team tools ช่วยป้องกันและตรวจจับการโจมตี:
- Intrusion Detection System (IDS)
- Log analysis
- Anomaly detection
- Honeypot

---

## 1. Intrusion Detection System

```nim
# ids.nim
# Rule-based IDS สำหรับ network monitoring

import std/[asyncdispatch, asyncnet, tables, strformat, times, sequtils, re, json]

type
  RuleSeverity* = enum
    rsInfo = "INFO"
    rsLow = "LOW"
    rsMedium = "MEDIUM"
    rsHigh = "HIGH"
    rsCritical = "CRITICAL"

  IDSRule* = object
    id*: int
    name*: string
    severity*: RuleSeverity
    pattern*: string     # Regex pattern
    description*: string
    action*: string      # "alert" | "block" | "log"

  Alert* = object
    ruleId*: int
    ruleName*: string
    severity*: RuleSeverity
    timestamp*: DateTime
    srcIP*: string
    dstIP*: string
    payload*: string
    description*: string

  IDS* = ref object
    rules*: seq[IDSRule]
    alerts*: seq[Alert]
    alertHandlers*: seq[proc(alert: Alert)]
    stats*: IDSStats

  IDSStats* = object
    totalInspected*: int64
    totalAlerts*: int64
    byRule*: Table[int, int64]
    bySeverity*: Table[string, int64]

# Common attack signatures
proc defaultRules*(): seq[IDSRule] = @[
  IDSRule(id: 1, name: "SQL_INJECTION", severity: rsHigh,
    pattern: r"(?i)(union\s+select|' or '1'='1|--\s*$|;\s*drop\s+table)",
    description: "SQL injection attempt detected"),
  
  IDSRule(id: 2, name: "XSS_ATTEMPT", severity: rsMedium,
    pattern: r"(?i)(<script[^>]*>|javascript:|onerror\s*=|onload\s*=)",
    description: "Cross-site scripting attempt"),
  
  IDSRule(id: 3, name: "PATH_TRAVERSAL", severity: rsMedium,
    pattern: r"(\.\./){2,}|%2e%2e%2f|%252e%252e",
    description: "Path traversal attempt"),
  
  IDSRule(id: 4, name: "CMD_INJECTION", severity: rsCritical,
    pattern: r"(?i)(;\s*(ls|cat|id|whoami|uname|wget|curl)\s|&&\s*(ls|cat|id)|`[^`]+`)",
    description: "Command injection attempt"),
  
  IDSRule(id: 5, name: "BRUTE_FORCE_INDICATOR", severity: rsHigh,
    pattern: r"(password|passwd|pwd|secret).*?(wrong|invalid|failed|incorrect)",
    description: "Possible brute force indicator"),
  
  IDSRule(id: 6, name: "SENSITIVE_FILE_ACCESS", severity: rsHigh,
    pattern: r"(?i)/(etc/passwd|etc/shadow|\.env|\.git/config|wp-config\.php|web\.config)",
    description: "Sensitive file access attempt"),
  
  IDSRule(id: 7, name: "SCANNER_FINGERPRINT", severity: rsInfo,
    pattern: r"(?i)(sqlmap|nmap|nikto|masscan|dirb|gobuster|nuclei)",
    description: "Security scanner detected"),
  
  IDSRule(id: 8, name: "SHELLSHOCK", severity: rsCritical,
    pattern: r"\(\)\s*\{\s*:;\s*\}",
    description: "Shellshock exploit attempt")
]

proc newIDS*(): IDS =
  IDS(
    rules: defaultRules(),
    stats: IDSStats(
      byRule: initTable[int, int64](),
      bySeverity: initTable[string, int64]()
    )
  )

proc onAlert*(ids: IDS, handler: proc(alert: Alert)) =
  ids.alertHandlers.add(handler)

proc inspect*(ids: IDS, payload, srcIP = "", dstIP = "") =
  inc ids.stats.totalInspected
  
  for rule in ids.rules:
    if payload.find(re(rule.pattern)) != -1:
      let alert = Alert(
        ruleId: rule.id,
        ruleName: rule.name,
        severity: rule.severity,
        timestamp: now(),
        srcIP: srcIP,
        dstIP: dstIP,
        payload: payload[0..min(200, payload.len-1)],
        description: rule.description
      )
      
      ids.alerts.add(alert)
      inc ids.stats.totalAlerts
      ids.stats.byRule[rule.id] = ids.stats.byRule.getOrDefault(rule.id, 0) + 1
      ids.stats.bySeverity[$rule.severity] = 
        ids.stats.bySeverity.getOrDefault($rule.severity, 0) + 1
      
      for handler in ids.alertHandlers:
        handler(alert)

proc printStats*(ids: IDS) =
  echo &"\nIDS Statistics:"
  echo &"  Inspected: {ids.stats.totalInspected}"
  echo &"  Alerts: {ids.stats.totalAlerts}"
  echo "\nBy Severity:"
  for sev, count in ids.stats.bySeverity:
    echo &"  {sev}: {count}"
```

---

## 2. Log Analyzer

```nim
# log_analyzer.nim
# SIEM-style log analysis

import std/[strformat, strutils, tables, times, sequtils, algorithm, re, json]

type
  LogLevel* = enum
    llDebug, llInfo, llWarn, llError, llCritical

  LogEntry* = object
    timestamp*: DateTime
    level*: LogLevel
    service*: string
    message*: string
    fields*: Table[string, string]
    sourceIP*: string

  LogPattern* = object
    name*: string
    regex*: string
    level*: LogLevel
    extract*: seq[string]  # Named capture groups

  Anomaly* = object
    name*: string
    description*: string
    severity*: string
    count*: int
    examples*: seq[string]

  LogAnalyzer* = ref object
    entries*: seq[LogEntry]
    patterns*: seq[LogPattern]
    ipStats*: Table[string, int]
    errorStats*: Table[string, int]
    timeWindow*: Duration

proc newLogAnalyzer*(): LogAnalyzer =
  LogAnalyzer(
    timeWindow: initDuration(minutes = 5),
    ipStats: initTable[string, int](),
    errorStats: initTable[string, int]()
  )

proc parseApacheLog*(line: string): Option[LogEntry] =
  # Common Log Format: IP - - [date] "METHOD /path HTTP/ver" status bytes
  let pattern = re(r'^(\S+) \S+ \S+ \[([^\]]+)\] "(\w+) ([^\s"]+)[^"]*" (\d+) (\d+)')
  var m: RegexMatch
  if not line.match(pattern, m): return none(LogEntry)
  
  let ip = line[m.captures[0][0]]
  let method = line[m.captures[2][0]]
  let path = line[m.captures[3][0]]
  let status = parseInt(line[m.captures[4][0]])
  
  let level = case status
    of 200..299: llInfo
    of 300..399: llInfo
    of 400..499: llWarn
    of 500..599: llError
    else: llInfo
  
  some(LogEntry(
    level: level,
    service: "apache",
    message: &"{method} {path} -> {status}",
    sourceIP: ip,
    fields: {"method": method, "path": path, "status": $status}.toTable
  ))

proc analyze*(analyzer: LogAnalyzer): seq[Anomaly] =
  # Count requests per IP
  for entry in analyzer.entries:
    if entry.sourceIP.len > 0:
      analyzer.ipStats[entry.sourceIP] = 
        analyzer.ipStats.getOrDefault(entry.sourceIP, 0) + 1
    
    if entry.level in {llError, llCritical}:
      analyzer.errorStats[entry.message] = 
        analyzer.errorStats.getOrDefault(entry.message, 0) + 1
  
  # Detect anomalies
  var anomalies: seq[Anomaly]
  
  # High request rate (potential DDoS/scraping)
  for ip, count in analyzer.ipStats:
    if count > 1000:
      anomalies.add(Anomaly(
        name: "HIGH_REQUEST_RATE",
        description: &"IP {ip} made {count} requests",
        severity: "HIGH",
        count: count,
        examples: @[ip]
      ))
  
  # Repeated errors
  for msg, count in analyzer.errorStats:
    if count > 50:
      anomalies.add(Anomaly(
        name: "REPEATED_ERRORS",
        description: &"Error repeated {count} times: {msg}",
        severity: "MEDIUM",
        count: count,
        examples: @[msg]
      ))
  
  anomalies

# Time-series analysis for anomaly detection
proc detectSpike*(timeSeries: seq[(DateTime, int)], windowSize = 5): seq[DateTime] =
  ## Detect spikes using Z-score
  if timeSeries.len < windowSize * 2: return @[]
  
  var anomalies: seq[DateTime]
  let values = timeSeries.mapIt(it[1].float)
  let mean = values.sum() / values.len.float
  let variance = values.mapIt((it - mean) ^ 2).sum() / values.len.float
  let stddev = sqrt(variance)
  
  for i, (ts, count) in timeSeries:
    let zScore = if stddev > 0: abs(count.float - mean) / stddev else: 0.0
    if zScore > 3.0:  # 3 sigma threshold
      anomalies.add(ts)
  
  anomalies
```

---

## 3. Honeypot

```nim
# honeypot.nim
# Honeypot สำหรับ attract และ detect attackers

import std/[asyncdispatch, asyncnet, asynchttpserver, json, strformat, times, tables]

type
  HoneypotEvent* = object
    timestamp*: DateTime
    eventType*: string
    srcIP*: string
    srcPort*: int
    data*: string
    service*: string

  Honeypot* = ref object
    events*: seq[HoneypotEvent]
    onEvent*: proc(event: HoneypotEvent)
    attackerProfiles*: Table[string, AttackerProfile]

  AttackerProfile* = object
    ip*: string
    firstSeen*: DateTime
    lastSeen*: DateTime
    eventCount*: int
    services*: seq[string]
    payloads*: seq[string]

proc newHoneypot*(onEvent: proc(event: HoneypotEvent) = nil): Honeypot =
  Honeypot(
    attackerProfiles: initTable[string, AttackerProfile](),
    onEvent: onEvent
  )

proc recordEvent*(hp: Honeypot, event: HoneypotEvent) =
  hp.events.add(event)
  
  # Update attacker profile
  if event.srcIP notin hp.attackerProfiles:
    hp.attackerProfiles[event.srcIP] = AttackerProfile(
      ip: event.srcIP,
      firstSeen: event.timestamp
    )
  
  var profile = hp.attackerProfiles[event.srcIP]
  profile.lastSeen = event.timestamp
  inc profile.eventCount
  if event.service notin profile.services:
    profile.services.add(event.service)
  if event.data.len > 0:
    profile.payloads.add(event.data[0..min(100, event.data.len-1)])
  hp.attackerProfiles[event.srcIP] = profile
  
  if not hp.onEvent.isNil:
    hp.onEvent(event)

# Fake SSH honeypot
proc startSSHHoneypot*(hp: Honeypot, port = 2222) {.async.} =
  let server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(port))
  server.listen()
  echo &"[SSH Honeypot] Listening on :{port}"
  
  while true:
    let (client, address) = await server.acceptAddr()
    let ip = address
    
    asyncCheck (proc() {.async.} =
      defer: client.close()
      
      # Send fake SSH banner
      await client.send("SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.4\r\n")
      
      # Wait for client data
      try:
        let data = await client.recv(1024)
        hp.recordEvent(HoneypotEvent(
          timestamp: now(),
          eventType: "SSH_PROBE",
          srcIP: ip,
          data: data,
          service: "SSH"
        ))
      except: discard
    )()

# Fake HTTP honeypot with fake admin panel
proc startHTTPHoneypot*(hp: Honeypot, port = 8888) {.async.} =
  let server = newAsyncHttpServer()
  
  proc handler(req: Request) {.async.} =
    let ip = req.hostname
    let path = req.url.path
    let method = $req.reqMethod
    
    hp.recordEvent(HoneypotEvent(
      timestamp: now(),
      eventType: "HTTP_REQUEST",
      srcIP: ip,
      data: &"{method} {path}\n{req.body[0..min(500, req.body.len-1)]}",
      service: "HTTP"
    ))
    
    # Respond with fake content to encourage exploration
    case path
    of "/", "/admin", "/login":
      await req.respond(Http200, """
        <html><body>
        <form method="POST" action="/login">
          <input name="username"><input name="password" type="password">
          <button type="submit">Login</button>
        </form>
        </body></html>
      """)
    of "/wp-admin", "/admin/login":
      await req.respond(Http200, "<html><body>WordPress Admin</body></html>")
    else:
      await req.respond(Http404, "Not Found")
  
  echo &"[HTTP Honeypot] Listening on :{port}"
  await server.serve(Port(port), handler)

proc generateReport*(hp: Honeypot): JsonNode =
  let profiles = newJArray()
  for ip, profile in hp.attackerProfiles:
    profiles.add(%*{
      "ip": ip,
      "firstSeen": profile.firstSeen.format("yyyy-MM-dd'T'HH:mm:ss'Z'"),
      "lastSeen": profile.lastSeen.format("yyyy-MM-dd'T'HH:mm:ss'Z'"),
      "eventCount": profile.eventCount,
      "services": profile.services,
      "samplePayloads": profile.payloads[0..min(3, profile.payloads.len-1)]
    })
  
  %*{
    "totalEvents": hp.events.len,
    "uniqueAttackers": hp.attackerProfiles.len,
    "attackerProfiles": profiles
  }
```

---

## สรุป

| Tool | ป้องกันอะไร |
|------|------------|
| IDS | ตรวจจับ attack signatures ใน traffic |
| Log Analyzer | Anomaly detection ใน logs |
| Honeypot | Detect และ track attackers |

---

**Next**: [Part 93 - Advanced Nim Patterns](../advanced/part93_patterns.md)
