# Part 60 - Network Attack Defense Concepts in Nim

## คำเตือน / Disclaimer

> เนื้อหานี้มุ่งเน้นเป็น Defensive Security สร้าง detection tools และ understanding attacks
> เพื่อป้องกันระบบเป็นหลัก

---

## บทนำ Network Attack Defense

Network defenders ต้องเข้าใจวิธีการโจมตีเพื่อสร้าง detection rules ที่มีประสิทธิภาพ

---

## 1. Packet Capture and Analysis

```nim
# packet_analyzer.nim
# Analyze network packets for threat detection
# Requires: libpcap on Linux, WinPcap/Npcap on Windows

import std/[strformat, tables, sets, json, times, strutils, sequtils]

type
  PacketType* = enum
    ptTCP = "TCP"
    ptUDP = "UDP"
    ptICMP = "ICMP"
    ptOther = "Other"

  PacketInfo* = object
    timestamp*: float
    srcIp*: string
    dstIp*: string
    srcPort*: int
    dstPort*: int
    protocol*: PacketType
    payloadSize*: int
    flags*: set[char]  # TCP flags: S=SYN, A=ACK, F=FIN, R=RST
    payload*: seq[byte]

  ConnectionTracker* = object
    connections*: Table[string, ConnectionState]
    suspiciousConns*: seq[string]

  ConnectionState* = object
    firstSeen*: float
    lastSeen*: float
    packetCount*: int
    byteCount*: int
    srcIp*: string
    dstIp*: string
    dstPort*: int
    synCount*: int
    flags*: set[char]
    isEstablished*: bool

# Parse raw Ethernet frame
proc parseEthernetFrame*(raw: seq[byte]): tuple[srcMac, dstMac: string, etherType: uint16, payload: seq[byte]] =
  if raw.len < 14: return
  
  result.dstMac = raw[0..5].mapIt(&"{it:02X}").join(":")
  result.srcMac = raw[6..11].mapIt(&"{it:02X}").join(":")
  result.etherType = (raw[12].uint16 shl 8) or raw[13].uint16
  result.payload = raw[14..^1]

# Parse IPv4 header
proc parseIPv4Header*(data: seq[byte]): tuple[srcIp, dstIp: string, proto: byte, payload: seq[byte]] =
  if data.len < 20: return
  
  let ihl = (data[0] and 0x0F).int * 4
  result.proto = data[9]
  
  result.srcIp = &"{data[12]}.{data[13]}.{data[14]}.{data[15]}"
  result.dstIp = &"{data[16]}.{data[17]}.{data[18]}.{data[19]}"
  result.payload = data[ihl..^1]

# Parse TCP header
proc parseTCPHeader*(data: seq[byte]): tuple[srcPort, dstPort: int, flags: uint8, seqNum: uint32, payload: seq[byte]] =
  if data.len < 20: return
  
  result.srcPort = (data[0].int shl 8) or data[1].int
  result.dstPort = (data[2].int shl 8) or data[3].int
  result.seqNum = (data[4].uint32 shl 24) or (data[5].uint32 shl 16) or
                  (data[6].uint32 shl 8) or data[7].uint32
  result.flags = data[13]
  
  let dataOffset = ((data[12] shr 4) * 4).int
  if data.len > dataOffset:
    result.payload = data[dataOffset..^1]

# Connection tracking
proc newConnectionTracker*(): ConnectionTracker =
  result.connections = initTable[string, ConnectionState]()
  result.suspiciousConns = @[]

proc trackPacket*(tracker: var ConnectionTracker, pkt: PacketInfo) =
  let key = &"{pkt.srcIp}:{pkt.srcPort}->{pkt.dstIp}:{pkt.dstPort}"
  
  if key notin tracker.connections:
    tracker.connections[key] = ConnectionState(
      firstSeen: pkt.timestamp,
      lastSeen: pkt.timestamp,
      srcIp: pkt.srcIp,
      dstIp: pkt.dstIp,
      dstPort: pkt.dstPort
    )
  
  var conn = tracker.connections[key]
  conn.lastSeen = pkt.timestamp
  inc conn.packetCount
  conn.byteCount += pkt.payloadSize
  
  if 'S' in pkt.flags:
    inc conn.synCount
  if 'A' in pkt.flags and 'S' in pkt.flags:
    conn.isEstablished = true
  
  tracker.connections[key] = conn

# --- Threat Detection Rules ---

type
  ThreatSignature* = object
    name*: string
    severity*: string
    description*: string
    mitreTechnique*: string

proc detectPortScan*(tracker: ConnectionTracker, srcIp: string, windowSeconds = 60.0): seq[ThreatSignature] =
  ## Detect port scanning from an IP
  result = @[]
  
  var uniqueDestPorts: HashSet[int]
  var synCount = 0
  let now = epochTime()
  
  for key, conn in tracker.connections:
    if conn.srcIp != srcIp: continue
    if now - conn.firstSeen > windowSeconds: continue
    
    uniqueDestPorts.incl(conn.dstPort)
    synCount += conn.synCount
  
  # Horizontal scan: many ports on few hosts
  if uniqueDestPorts.len > 20:
    result.add(ThreatSignature(
      name: "PORT_SCAN_HORIZONTAL",
      severity: "medium",
      description: &"{srcIp} scanned {uniqueDestPorts.len} unique ports in {windowSeconds:.0f}s",
      mitreTechnique: "T1046 - Network Service Scanning"
    ))
  
  # SYN flood: many SYNs without ACKs
  if synCount > 100:
    result.add(ThreatSignature(
      name: "SYN_FLOOD_POSSIBLE",
      severity: "high",
      description: &"{srcIp} sent {synCount} SYNs, possible SYN flood",
      mitreTechnique: "T1498 - Network Denial of Service"
    ))

proc detectDataExfiltration*(tracker: ConnectionTracker, internalCidr: string): seq[ThreatSignature] =
  ## Detect large outbound data transfers
  result = @[]
  
  var outboundBytes: Table[string, int]
  
  for key, conn in tracker.connections:
    # Check if source is internal
    if conn.srcIp.startsWith("192.168.") or
       conn.srcIp.startsWith("10.") or
       conn.srcIp.startsWith("172.16."):
      outboundBytes[conn.srcIp] = outboundBytes.getOrDefault(conn.srcIp, 0) + conn.byteCount
  
  for ip, bytes in outboundBytes:
    if bytes > 100 * 1024 * 1024:  # > 100MB
      result.add(ThreatSignature(
        name: "LARGE_OUTBOUND_TRANSFER",
        severity: "high",
        description: &"{ip} transferred {bytes div 1024 div 1024}MB outbound",
        mitreTechnique: "T1048 - Exfiltration Over Alternative Protocol"
      ))

proc detectDNSTunneling*(dnsQueries: seq[string]): seq[ThreatSignature] =
  ## Detect DNS tunneling by analyzing query patterns
  result = @[]
  
  var domainCounts: Table[string, int]
  var longQueryCount = 0
  
  for query in dnsQueries:
    let parts = query.split(".")
    
    # Long subdomains (> 40 chars) indicate encoding
    for part in parts:
      if part.len > 40:
        inc longQueryCount
    
    # Count queries per domain
    if parts.len >= 2:
      let domain = parts[^2..^1].join(".")
      domainCounts[domain] = domainCounts.getOrDefault(domain, 0) + 1
  
  if longQueryCount > 10:
    result.add(ThreatSignature(
      name: "DNS_TUNNEL_LONG_LABELS",
      severity: "high",
      description: &"{longQueryCount} DNS queries with suspicious long subdomains",
      mitreTechnique: "T1071.004 - Application Layer Protocol: DNS"
    ))
  
  for domain, count in domainCounts:
    if count > 100:  # High query volume to single domain
      result.add(ThreatSignature(
        name: "DNS_TUNNEL_HIGH_VOLUME",
        severity: "medium",
        description: &"High DNS query volume to {domain}: {count} queries",
        mitreTechnique: "T1071.004 - Application Layer Protocol: DNS"
      ))

# --- Network IDS Rule Engine ---

type
  IDSRule* = object
    id*: int
    name*: string
    priority*: int
    action*: string  # "alert", "drop", "log"
    conditions*: seq[proc(pkt: PacketInfo): bool]
    mitre*: string

  IDSEngine* = object
    rules*: seq[IDSRule]
    alerts*: seq[tuple[ruleId: int, pkt: PacketInfo]]
    packetCount*: int

proc newIDSEngine*(): IDSEngine =
  result.rules = @[]
  result.alerts = @[]

proc addRule*(engine: var IDSEngine, rule: IDSRule) =
  engine.rules.add(rule)
  # Sort by priority
  engine.rules.sort do (a, b: IDSRule) -> int: cmp(a.priority, b.priority)

proc processPacket*(engine: var IDSEngine, pkt: PacketInfo) =
  inc engine.packetCount
  
  for rule in engine.rules:
    var matches = true
    for condition in rule.conditions:
      if not condition(pkt):
        matches = false
        break
    
    if matches:
      engine.alerts.add((rule.id, pkt))
      echo &"[{rule.action.toUpperAscii}] Rule {rule.id}: {rule.name}"
      echo &"  {pkt.srcIp}:{pkt.srcPort} -> {pkt.dstIp}:{pkt.dstPort} ({pkt.protocol})"

proc loadDefaultRules*(engine: var IDSEngine) =
  ## Load common detection rules (Snort/Suricata-inspired)
  
  # Rule 1: Telnet connection attempt
  engine.addRule(IDSRule(
    id: 1001,
    name: "TELNET_ATTEMPT",
    priority: 2,
    action: "alert",
    conditions: @[
      proc(pkt: PacketInfo): bool = pkt.dstPort == 23 and pkt.protocol == ptTCP
    ],
    mitre: "T1021 - Remote Services"
  ))
  
  # Rule 2: SSH brute force indicator
  # (Would need connection tracking for full detection)
  engine.addRule(IDSRule(
    id: 1002,
    name: "SSH_CONNECTION",
    priority: 3,
    action: "log",
    conditions: @[
      proc(pkt: PacketInfo): bool = pkt.dstPort == 22 and pkt.protocol == ptTCP
    ],
    mitre: "T1110 - Brute Force"
  ))
  
  # Rule 3: Common malware port
  engine.addRule(IDSRule(
    id: 1003,
    name: "SUSPICIOUS_OUTBOUND_PORT",
    priority: 1,
    action: "alert",
    conditions: @[
      proc(pkt: PacketInfo): bool =
        pkt.dstPort in {4444, 4445, 31337, 1337, 8888} and pkt.protocol == ptTCP
    ],
    mitre: "T1571 - Non-Standard Port"
  ))
  
  # Rule 4: ICMP large payload (possible tunnel)
  engine.addRule(IDSRule(
    id: 1004,
    name: "ICMP_LARGE_PAYLOAD",
    priority: 2,
    action: "alert",
    conditions: @[
      proc(pkt: PacketInfo): bool = pkt.protocol == ptICMP and pkt.payloadSize > 1000
    ],
    mitre: "T1095 - Non-Application Layer Protocol"
  ))

# --- Firewall Rule Generator ---

type
  FirewallRule* = object
    action*: string  # "ACCEPT", "DROP", "REJECT"
    protocol*: string
    srcIp*: string
    dstIp*: string
    dstPort*: int
    comment*: string

proc generateIptablesRules*(rules: seq[FirewallRule]): string =
  ## Generate iptables commands from rules
  result = "#!/bin/bash\n# Generated firewall rules\n\n"
  result.add("# Flush existing rules\n")
  result.add("iptables -F\n")
  result.add("iptables -P INPUT DROP\n")
  result.add("iptables -P FORWARD DROP\n")
  result.add("iptables -P OUTPUT ACCEPT\n\n")
  
  for rule in rules:
    var cmd = &"iptables -A INPUT"
    if rule.protocol != "": cmd.add(&" -p {rule.protocol}")
    if rule.srcIp != "": cmd.add(&" -s {rule.srcIp}")
    if rule.dstPort > 0: cmd.add(&" --dport {rule.dstPort}")
    cmd.add(&" -j {rule.action}")
    if rule.comment != "": cmd.add(&" -m comment --comment \"{rule.comment}\"")
    result.add(cmd & "\n")

proc generateFromAlerts*(alerts: seq[tuple[ruleId: int, srcIp: string]]): seq[FirewallRule] =
  ## Auto-generate firewall rules from IDS alerts
  result = @[]
  
  var alertedIps: Table[string, int]
  for (ruleId, ip) in alerts:
    alertedIps[ip] = alertedIps.getOrDefault(ip, 0) + 1
  
  for ip, count in alertedIps:
    if count > 5:  # Block IPs with multiple alerts
      result.add(FirewallRule(
        action: "DROP",
        protocol: "",
        srcIp: ip,
        dstIp: "",
        dstPort: 0,
        comment: &"Auto-blocked: {count} security alerts"
      ))

when isMainModule:
  echo "Network Defense Tool"
  echo "===================="
  
  # Demo IDS engine
  var ids = newIDSEngine()
  ids.loadDefaultRules()
  echo &"Loaded {ids.rules.len} detection rules"
  
  # Simulate some packets
  let testPackets = @[
    PacketInfo(srcIp: "192.168.1.100", dstIp: "10.0.0.1", srcPort: 12345, dstPort: 4444,
               protocol: ptTCP, payloadSize: 200, timestamp: epochTime()),
    PacketInfo(srcIp: "192.168.1.200", dstIp: "10.0.0.2", srcPort: 45678, dstPort: 22,
               protocol: ptTCP, payloadSize: 100, timestamp: epochTime()),
  ]
  
  echo &"\nProcessing {testPackets.len} test packets..."
  for pkt in testPackets:
    ids.processPacket(pkt)
  
  echo &"\nTotal alerts: {ids.alerts.len}"
  
  # DNS tunneling detection demo
  let dnsQueries = @[
    "aGVsbG8gd29ybGQ.c2.attacker.com",
    "dGhpcyBpcyBhIHRlc3Q.c2.attacker.com",
    "shortquery.google.com",
    "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIII.tunnel.evil.com",
  ]
  
  echo "\nDNS Analysis:"
  let dnsThreat = detectDNSTunneling(dnsQueries)
  for t in dnsThreat:
    echo &"  [{t.severity.toUpperAscii}] {t.name}: {t.description}"
  
  # Generate firewall rules
  let alertedIps = ids.alerts.mapIt((it[0], it[1].srcIp))
  let fwRules = generateFromAlerts(alertedIps)
  if fwRules.len > 0:
    echo "\n\nGenerated Firewall Rules:"
    echo generateIptablesRules(fwRules)
```

---

## 2. SIEM Integration

```nim
# siem_connector.nim
# Send security events to SIEM systems (Elastic, Splunk, etc.)

import std/[asyncdispatch, httpclient, json, strformat, times, strutils]

type
  SiemEvent* = object
    timestamp*: DateTime
    severity*: string  # "low", "medium", "high", "critical"
    category*: string
    action*: string
    outcome*: string   # "success", "failure", "unknown"
    srcIp*: string
    dstIp*: string
    srcPort*: int
    dstPort*: int
    protocol*: string
    message*: string
    tags*: seq[string]
    mitre*: string
    rawEvent*: JsonNode

  ElasticConnector* = object
    url*: string
    index*: string
    apiKey*: string

  SplunkConnector* = object
    url*: string
    token*: string
    sourcetype*: string

proc toECS*(event: SiemEvent): JsonNode =
  ## Convert to Elastic Common Schema format
  %*{
    "@timestamp": event.timestamp.format("yyyy-MM-dd'T'HH:mm:sszzz"),
    "event": {
      "kind": "alert",
      "category": [event.category],
      "action": event.action,
      "outcome": event.outcome,
      "severity": case event.severity
        of "low": 1
        of "medium": 2
        of "high": 3
        of "critical": 4
        else: 0,
    },
    "message": event.message,
    "source": {
      "ip": event.srcIp,
      "port": event.srcPort
    },
    "destination": {
      "ip": event.dstIp,
      "port": event.dstPort
    },
    "network": {
      "transport": event.protocol.toLower()
    },
    "threat": {
      "technique": {
        "id": event.mitre
      }
    },
    "labels": event.tags.mapIt(%it)
  }

proc sendToElastic*(conn: ElasticConnector, event: SiemEvent): Future[bool] {.async.} =
  let client = newAsyncHttpClient()
  defer: client.close()
  
  var headers = newHttpHeaders()
  headers["Content-Type"] = "application/json"
  if conn.apiKey != "":
    headers["Authorization"] = &"ApiKey {conn.apiKey}"
  
  let body = $event.toECS()
  let url = &"{conn.url}/{conn.index}/_doc"
  
  try:
    let resp = await client.request(url, HttpPost, body, headers)
    result = resp.code.int in {200, 201}
  except:
    result = false

proc sendToSplunkHEC*(conn: SplunkConnector, event: SiemEvent): Future[bool] {.async.} =
  ## Send to Splunk HTTP Event Collector
  let client = newAsyncHttpClient()
  defer: client.close()
  
  var headers = newHttpHeaders()
  headers["Content-Type"] = "application/json"
  headers["Authorization"] = &"Splunk {conn.token}"
  
  let body = $(%*{
    "time": event.timestamp.toTime().toUnix(),
    "sourcetype": conn.sourcetype,
    "event": {
      "severity": event.severity,
      "message": event.message,
      "src_ip": event.srcIp,
      "dst_ip": event.dstIp,
      "mitre": event.mitre
    }
  })
  
  try:
    let resp = await client.request(&"{conn.url}/services/collector",
      HttpPost, body, headers)
    result = resp.code.int == 200
  except:
    result = false

proc createThreatAlert*(name, srcIp, dstIp, mitre, description: string,
                         severity = "medium"): SiemEvent =
  SiemEvent(
    timestamp: now(),
    severity: severity,
    category: "intrusion_detection",
    action: name,
    outcome: "unknown",
    srcIp: srcIp,
    dstIp: dstIp,
    message: description,
    mitre: mitre,
    tags: @["automated", "nim-ids"]
  )

when isMainModule:
  echo "SIEM Connector Demo"
  echo "==================="
  
  let alert = createThreatAlert(
    "PORT_SCAN",
    "192.168.1.100",
    "10.0.0.50",
    "T1046",
    "Host scanned 50 ports in 30 seconds",
    "medium"
  )
  
  echo "ECS-formatted event:"
  echo $alert.toECS()
  
  # To actually send:
  # let elastic = ElasticConnector(url: "http://elastic:9200", index: "security-*", apiKey: "...")
  # let ok = waitFor elastic.sendToElastic(alert)
  # echo if ok: "[+] Event sent" else: "[-] Failed"
```

---

## สรุป Part 60

| เครื่องมือ | ฟังก์ชัน |
|-----------|----------|
| `packet_analyzer.nim` | Packet parsing + connection tracking |
| IDS Engine | Rule-based threat detection |
| `siem_connector.nim` | ECS format + Elastic/Splunk integration |
| Firewall generator | Auto-generate block rules from alerts |

### MITRE ATT&CK Coverage

| Technique | Detection Method |
|-----------|------------------|
| T1046 - Port Scan | Unique port count per source IP |
| T1071.004 - DNS | Long subdomain labels, high query volume |
| T1048 - Exfiltration | Large outbound byte counts |
| T1571 - Non-std Port | Port whitelist/blacklist |
| T1498 - DoS | SYN count tracking |

**Next**: [Part 61 - Nim Compiler Internals Deep Dive](../advanced/part61_nim_deep_internals.md)
