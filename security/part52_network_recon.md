# Part 52: Network Reconnaissance Tools ใน Nim

> เนื้อหานี้เป็นการสร้างเครื่องมือ network security
> สำหรับ authorized pentesting และ defensive security

## Port Scanner

```nim
# port_scanner.nim
# TCP port scanner

import asyncnet, asyncdispatch, net, strutils, strformat, times, sequtils

type
  PortResult = object
    port: int
    open: bool
    banner: string
    service: string

const commonPorts = {
  21: "FTP", 22: "SSH", 23: "Telnet", 25: "SMTP",
  53: "DNS", 80: "HTTP", 110: "POP3", 143: "IMAP",
  443: "HTTPS", 445: "SMB", 993: "IMAPS", 995: "POP3S",
  1433: "MSSQL", 3306: "MySQL", 3389: "RDP",
  5432: "PostgreSQL", 5900: "VNC", 6379: "Redis",
  8080: "HTTP-Alt", 8443: "HTTPS-Alt", 27017: "MongoDB"
}.toTable()

proc scanPort(host: string, port: int, timeoutMs: int = 2000): Future[PortResult] {.async.} =
  result = PortResult(port: port, open: false)
  
  let sock = newAsyncSocket()
  
  try:
    # Connect with timeout
    let connectFuture = sock.connect(host, Port(port))
    let timedOut = await withTimeout(connectFuture, timeoutMs)
    
    if not timedOut:
      result.open = true
      if port in commonPorts:
        result.service = commonPorts[port]
      
      # Grab banner (optional)
      try:
        let bannerFuture = sock.recv(256)
        let bannerResult = await withTimeout(bannerFuture, 1000)
        if not bannerResult:
          result.banner = sock.recv(256).read()[0..<min(256, sock.recv(256).read().len)]
      except: discard
  except: discard
  finally:
    sock.close()

proc scanHost(host: string, ports: seq[int], concurrency: int = 100): Future[seq[PortResult]] {.async.} =
  result = @[]
  var tasks: seq[Future[PortResult]] = @[]
  var idx = 0
  
  while idx < ports.len:
    # Batch by concurrency
    let batch = ports[idx..<min(idx + concurrency, ports.len)]
    let batchTasks = batch.mapIt(scanPort(host, it))
    
    let results = await all(batchTasks)
    for r in results:
      if r.open:
        result.add(r)
    
    idx += concurrency

proc printResults(host: string, results: seq[PortResult]) =
  echo &"\n=== Scan Results for {host} ==="
  echo &"Open ports: {results.len}"
  echo "-".repeat(50)
  for r in results.sortedByIt(it.port):
    let service = if r.service.len > 0: r.service else: "unknown"
    let banner = if r.banner.len > 0: " | " & r.banner[0..min(30, r.banner.len-1)] else: ""
    echo &"  {r.port:<6} {service:<15} {banner}"

# Common port ranges
let commonPortList = toSeq(1..1024) & @[3306, 3389, 5432, 5900, 6379, 8080, 8443, 27017]

proc main() {.async.} =
  let target = "127.0.0.1"  # scan localhost
  echo &"Scanning {target}..."
  let startTime = now()
  
  let results = await scanHost(target, commonPortList)
  printResults(target, results)
  
  let elapsed = (now() - startTime).inMilliseconds
  echo &"\nTime: {elapsed}ms"

waitFor main()
```

## DNS Enumeration

```nim
# dns_enum.nim
# DNS reconnaissance

import asyncnet, asyncdispatch, strutils, strformat, sequtils

proc resolveHost(hostname: string): seq[string] =
  import net
  try:
    result = @[getHostByName(hostname).addrList[0]]
  except:
    result = @[]

proc bruteforceSubdomains(domain: string, wordlist: seq[string]): seq[(string, string)] =
  result = @[]
  for word in wordlist:
    let subdomain = word & "." & domain
    let ips = resolveHost(subdomain)
    if ips.len > 0:
      result.add((subdomain, ips.join(", ")))
      echo &"  [+] {subdomain} -> {ips.join(", ")}"

proc checkCommonSubdomains(domain: string): seq[(string, string)] =
  let common = @[
    "www", "mail", "smtp", "pop", "imap",
    "ftp", "vpn", "remote", "api", "dev",
    "test", "staging", "admin", "portal",
    "blog", "shop", "app", "ns1", "ns2",
    "cdn", "static", "assets", "img"
  ]
  bruteforceSubdomains(domain, common)

proc reverseDns(ip: string): string =
  import net
  try:
    getHostByAddr(ip).name
  except:
    "unknown"

proc checkDnsRecords(domain: string) =
  # Check various DNS record types
  import osproc
  for recordType in ["A", "AAAA", "MX", "NS", "TXT", "CNAME", "SOA"]:
    let (output, exitCode) = execCmdEx(&"nslookup -type={recordType} {domain}")
    if exitCode == 0 and "Non-authoritative" notin output:
      echo &"[{recordType}] {output.strip()[0..min(100, output.len-1)]}..."

# Zone transfer attempt (AXFR)
proc tryZoneTransfer(domain, nameserver: string): seq[string] =
  import osproc
  result = @[]
  let (output, exitCode) = execCmdEx(&"nslookup -type=AXFR {domain} {nameserver}")
  if exitCode == 0 and "Address" in output:
    result = output.splitLines().filterIt(it.len > 0)

echo "=== DNS Enumeration ==="
let domain = "example.com"  # Change to target
echo &"Target: {domain}"

echo "\nCommon subdomains:"
let subdomains = checkCommonSubdomains(domain)
echo &"Found: {subdomains.len} subdomains"
```

## Network Service Banner Grabber

```nim
# banner_grabber.nim
# ผนข้อมูล service version

import asyncnet, asyncdispatch, strutils, strformat, tables

type ServiceProbe = object
  probe: string   # ส่งสิ่งนี้เพื่อ provoke response
  pattern: string # จำแนก service
  ports: seq[int]

const serviceProbes = [
  ServiceProbe(probe: "", pattern: "SSH", ports: @[22]),
  ServiceProbe(probe: "HEAD / HTTP/1.0\r\n\r\n", pattern: "HTTP", ports: @[80, 8080]),
  ServiceProbe(probe: "", pattern: "FTP", ports: @[21]),
  ServiceProbe(probe: "", pattern: "SMTP", ports: @[25, 587]),
  ServiceProbe(probe: "\x00\x00\x00\x0a\xff", pattern: "MySQL", ports: @[3306]),
  ServiceProbe(probe: "+client_encoding utf8\r\n", pattern: "PostgreSQL", ports: @[5432]),
  ServiceProbe(probe: "PING\r\n", pattern: "Redis", ports: @[6379]),
]

proc grabBanner(host: string, port: int, probeData: string = "",
                timeoutMs: int = 3000): Future[string] {.async.} =
  let sock = newAsyncSocket()
  
  try:
    let connected = await withTimeout(sock.connect(host, Port(port)), timeoutMs)
    if not connected: return ""
    
    # Send probe
    if probeData.len > 0:
      await sock.send(probeData)
    
    # Receive banner
    let bannerFuture = sock.recv(1024)
    let received = await withTimeout(bannerFuture, timeoutMs)
    if received:
      return (await bannerFuture).strip()
    
    return ""
  except:
    return ""
  finally:
    sock.close()

proc identifyService(host: string, port: int): Future[string] {.async.} =
  # Try service-specific probes
  for probe in serviceProbes:
    if port in probe.ports:
      let banner = await grabBanner(host, port, probe.probe)
      if banner.len > 0:
        return &"{probe.pattern}: {banner[0..min(60, banner.len-1)]}"
  
  # Generic banner grab
  let banner = await grabBanner(host, port)
  if banner.len > 0:
    return banner[0..min(60, banner.len-1)]
  
  return "no banner"

proc main() {.async.} =
  let target = "127.0.0.1"
  let ports = @[21, 22, 25, 80, 443, 3306, 5432, 6379, 8080]
  
  echo &"=== Banner Grabber: {target} ==="
  for port in ports:
    let info = await identifyService(target, port)
    if info != "no banner":
      echo &"  {port}: {info}"

waitFor main()
```

## สรุป Part 52

- ␅ Async port scanner
- ␅ DNS enumeration และ subdomain brute force
- ␅ Service banner grabber
- ␅ Network recon tools

---
**Next**: [Part 53 - Reverse Engineering Tools](../advanced/part53_reverse_engineering.md)
