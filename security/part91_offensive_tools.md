# Part 91 - Offensive Security Tools in Nim

> **คำเตือน**: เนื้อหาในส่วนนี้มีไว้สำหรับ **การศึกษา, CTF competitions, authorized penetration testing, และ defensive security research เท่านั้น** ห้ามนำไปใช้กับระบบที่ไม่ได้รับอนุญาต การกระทำดังกล่าวผิดกฎหมาย

---

## 1. Network Scanner

```nim
# port_scanner.nim
# Port scanner สำหรับ authorized network testing

import std/[asyncdispatch, asyncnet, strformat, sequtils, algorithm, times]

type
  ScanResult* = object
    host*: string
    port*: int
    open*: bool
    banner*: string
    latencyMs*: float

proc scanPort*(host: string, port: int, timeoutMs = 1000): Future[ScanResult] {.async.} =
  let start = cpuTime()
  let sock = newAsyncSocket()
  sock.setSockOpt(OptReuseAddr, true)
  
  try:
    # Attempt connection with timeout
    let connectFut = sock.connect(host, Port(port))
    let timeoutFut = sleepAsync(timeoutMs)
    
    let idx = await race(connectFut, timeoutFut)
    
    if idx == 1:  # Timeout
      sock.close()
      return ScanResult(host: host, port: port, open: false)
    
    let latency = (cpuTime() - start) * 1000
    
    # Grab banner (first 256 bytes)
    var banner = ""
    try:
      let bannerFut = sock.recv(256)
      let bannerTimeout = sleepAsync(500)
      let bannerIdx = await race(bannerFut, bannerTimeout)
      if bannerIdx == 0:
        banner = await bannerFut
    except: discard
    
    sock.close()
    ScanResult(host: host, port: port, open: true,
      banner: banner.strip(), latencyMs: latency)
  
  except:
    sock.close()
    ScanResult(host: host, port: port, open: false)

proc scanRange*(host: string, portStart, portEnd: int,
    concurrency = 100): Future[seq[ScanResult]] {.async.} =
  ## Scan port range with limited concurrency
  var results: seq[ScanResult]
  var pending: seq[Future[ScanResult]]
  
  for port in portStart..portEnd:
    pending.add(scanPort(host, port))
    
    if pending.len >= concurrency:
      let batch = await all(pending)
      for r in batch:
        if r.open: results.add(r)
      pending.setLen(0)
  
  if pending.len > 0:
    let batch = await all(pending)
    for r in batch:
      if r.open: results.add(r)
  
  results.sortIt(it.port)
  results

# Common service fingerprinting
proc identifyService*(port: int, banner: string): string =
  case port
  of 21: "FTP"
  of 22: "SSH"
  of 23: "Telnet"
  of 25: "SMTP"
  of 53: "DNS"
  of 80, 8080, 8000: "HTTP"
  of 443, 8443: "HTTPS"
  of 3306: "MySQL"
  of 5432: "PostgreSQL"
  of 6379: "Redis"
  of 27017: "MongoDB"
  else:
    if banner.contains("SSH"): "SSH"
    elif banner.contains("HTTP"): "HTTP"
    elif banner.contains("220 "): "FTP/SMTP"
    else: "Unknown"

when isMainModule:
  proc main() {.async.} =
    let target = "127.0.0.1"  # Only scan authorized targets!
    echo &"Scanning {target}..."
    
    let results = await scanRange(target, 1, 1024, 200)
    echo &"\nOpen ports on {target}:"
    
    for r in results:
      let service = identifyService(r.port, r.banner)
      echo &"  {r.port}/tcp  {service:<12} {r.latencyMs:.1f}ms"
      if r.banner.len > 0:
        echo &"    Banner: {r.banner[0..min(80, r.banner.len-1)]}"
  
  waitFor main()
```

---

## 2. HTTP Directory Fuzzer

```nim
# dir_fuzzer.nim
# Web directory discovery สำหรับ authorized testing

import std/[asyncdispatch, asynchttpserver, httpclient, strformat, sequtils]
import std/[os, times, strutils, tables]

type
  FuzzResult* = object
    url*: string
    statusCode*: int
    contentLength*: int
    contentType*: string
    redirectTo*: string

proc fuzzDirectory*(baseUrl: string, wordlist: seq[string],
    extensions = @["", ".php", ".html", ".txt", ".bak"],
    concurrency = 20): Future[seq[FuzzResult]] {.async.} =
  
  var results: seq[FuzzResult]
  var pending: seq[Future[(string, int, int, string, string)]]
  
  let client = newAsyncHttpClient()
  client.headers = newHttpHeaders({"User-Agent": "Mozilla/5.0"})
  
  proc checkPath(path: string): Future[(string, int, int, string, string)] {.async.} =
    let url = baseUrl.strip(trailing = true, chars = {'/'}) & "/" & path
    try:
      let resp = await client.get(url)
      let body = await resp.body
      let ct = resp.headers.getOrDefault("content-type", "")
      let location = resp.headers.getOrDefault("location", "")
      result = (url, resp.status.parseInt(), body.len, ct, location)
    except:
      result = (url, 0, 0, "", "")
  
  for word in wordlist:
    for ext in extensions:
      let path = word & ext
      pending.add(checkPath(path))
      
      if pending.len >= concurrency:
        let batch = await all(pending)
        for (url, status, length, ct, location) in batch:
          if status notin {0, 404}:
            results.add(FuzzResult(
              url: url, statusCode: status,
              contentLength: length, contentType: ct,
              redirectTo: location
            ))
        pending.setLen(0)
  
  if pending.len > 0:
    let batch = await all(pending)
    for (url, status, length, ct, location) in batch:
      if status notin {0, 404}:
        results.add(FuzzResult(
          url: url, statusCode: status,
          contentLength: length, contentType: ct,
          redirectTo: location
        ))
  
  client.close()
  results

proc loadWordlist*(path: string): seq[string] =
  if not fileExists(path):
    # Common paths for demo
    return @["admin", "login", "api", "v1", "v2", "config",
             "backup", "test", "dev", "staging", "robots",
             "sitemap", ".env", "phpinfo", "wp-admin"]
  
  readFile(path).splitLines().filterIt(it.len > 0 and not it.startsWith('#'))

when isMainModule:
  proc main() {.async.} =
    let target = "http://localhost:8080"  # Authorized target only!
    let wordlist = loadWordlist("common.txt")
    
    echo &"Fuzzing {target} with {wordlist.len} words..."
    
    let results = await fuzzDirectory(target, wordlist)
    
    echo "\nDiscovered paths:"
    for r in results:
      let redirect = if r.redirectTo.len > 0: &" → {r.redirectTo}" else: ""
      echo &"  [{r.statusCode}] {r.url} ({r.contentLength} bytes){redirect}"
  
  waitFor main()
```

---

## 3. Credential Testing Tool

```nim
# cred_tester.nim
# Credential testing สำหรับ authorized security testing

import std/[asyncdispatch, httpclient, json, strformat, sequtils, times]

type
  Credential* = object
    username*: string
    password*: string

  TestResult* = object
    cred*: Credential
    success*: bool
    statusCode*: int
    responseTime*: float

proc testHttpBasicAuth*(url, username, password: string): Future[TestResult] {.async.} =
  let start = cpuTime()
  let client = newAsyncHttpClient()
  client.headers = newHttpHeaders({
    "Authorization": "Basic " & (username & ":" & password).encode()
  })
  
  try:
    let resp = await client.get(url)
    let elapsed = (cpuTime() - start) * 1000
    client.close()
    
    TestResult(
      cred: Credential(username: username, password: password),
      success: resp.status.parseInt() in {200, 201, 302},
      statusCode: resp.status.parseInt(),
      responseTime: elapsed
    )
  except:
    client.close()
    TestResult(cred: Credential(username: username, password: password),
      success: false, statusCode: 0)

proc testFormLogin*(url, usernameField, passwordField,
    username, password: string,
    successIndicator: string): Future[TestResult] {.async.} =
  let start = cpuTime()
  let client = newAsyncHttpClient()
  client.headers = newHttpHeaders({
    "Content-Type": "application/x-www-form-urlencoded"
  })
  
  let body = &"{usernameField}={username}&{passwordField}={password}"
  
  try:
    let resp = await client.post(url, body = body)
    let respBody = await resp.body
    let elapsed = (cpuTime() - start) * 1000
    client.close()
    
    TestResult(
      cred: Credential(username: username, password: password),
      success: successIndicator in respBody,
      statusCode: resp.status.parseInt(),
      responseTime: elapsed
    )
  except:
    client.close()
    TestResult(cred: Credential(username: username, password: password),
      success: false, statusCode: 0)

proc loadCredentials*(userFile, passFile: string): seq[Credential] =
  let users = if fileExists(userFile): readFile(userFile).splitLines()
    else: @["admin", "root", "user", "test"]
  let passes = if fileExists(passFile): readFile(passFile).splitLines()
    else: @["admin", "password", "123456", "test"]
  
  for u in users:
    for p in passes:
      if u.len > 0 and p.len > 0:
        result.add(Credential(username: u, password: p))
```

---

## 4. Packet Analysis (Pcap)

```nim
# packet_analysis.nim
# Network packet analysis ด้วย libpcap FFI

import std/[strformat, tables, strutils, net]

# libpcap FFI types
type
  PcapT* = ptr object  # pcap_t handle
  PcapPktHdr* = object
    tvSec*: int32
    tvUsec*: int32
    caplen*: uint32
    len*: uint32
  
  PcapHandler* = proc(user: pointer, hdr: ptr PcapPktHdr, 
    data: ptr UncheckedArray[uint8]) {.cdecl.}

# libpcap bindings
proc pcapOpenLive*(device: cstring, snaplen, promisc, toMs: cint,
    errbuf: cstring): PcapT {.importc: "pcap_open_live", dynlib: "libpcap.so.1".}

proc pcapLoop*(p: PcapT, cnt: cint, callback: PcapHandler,
    user: pointer): cint {.importc: "pcap_loop", dynlib: "libpcap.so.1".}

proc pcapClose*(p: PcapT) {.importc: "pcap_close", dynlib: "libpcap.so.1".}

proc pcapSetFilter*(p: PcapT, fp: pointer): cint 
    {.importc: "pcap_setfilter", dynlib: "libpcap.so.1".}

# Ethernet frame parsing
type
  EtherHeader* = object
    dest*: array[6, uint8]
    src*: array[6, uint8]
    etherType*: uint16

  IPv4Header* = object
    versionIHL*: uint8
    tos*: uint8
    totalLen*: uint16
    id*: uint16
    flagsFragment*: uint16
    ttl*: uint8
    protocol*: uint8
    checksum*: uint16
    src*: array[4, uint8]
    dst*: array[4, uint8]

proc macToStr*(mac: array[6, uint8]): string =
  mac.mapIt(it.toHex(2)).join(":")

proc ipToStr*(ip: array[4, uint8]): string =
  ip.mapIt($it).join(".")

# Packet statistics
type
  PacketStats* = ref object
    total*: int
    byProtocol*: Table[int, int]
    bySourceIP*: Table[string, int]
    byDestIP*: Table[string, int]
    bytesTotal*: int64

var stats* = PacketStats(
  byProtocol: initTable[int, int](),
  bySourceIP: initTable[string, int](),
  byDestIP: initTable[string, int]()
)

proc packetHandler*(user: pointer, hdr: ptr PcapPktHdr,
    data: ptr UncheckedArray[uint8]) {.cdecl.} =
  inc stats.total
  stats.bytesTotal += hdr.caplen

  if hdr.caplen < 14: return  # Too short for ethernet
  
  let etherType = (data[12].int shl 8) or data[13].int
  
  # IPv4
  if etherType == 0x0800 and hdr.caplen >= 34:
    let ipOff = 14
    let proto = data[ipOff + 9].int
    
    var srcIP: array[4, uint8]
    var dstIP: array[4, uint8]
    copyMem(addr srcIP[0], addr data[ipOff + 12], 4)
    copyMem(addr dstIP[0], addr data[ipOff + 16], 4)
    
    let src = ipToStr(srcIP)
    let dst = ipToStr(dstIP)
    
    stats.byProtocol[proto] = stats.byProtocol.getOrDefault(proto, 0) + 1
    stats.bySourceIP[src] = stats.bySourceIP.getOrDefault(src, 0) + 1
    stats.byDestIP[dst] = stats.byDestIP.getOrDefault(dst, 0) + 1

proc printStats*() =
  echo &"\nPacket Statistics:"
  echo &"  Total: {stats.total} packets ({stats.bytesTotal} bytes)"
  echo "\nBy Protocol:"
  for proto, count in stats.byProtocol:
    let name = case proto
      of 6: "TCP"
      of 17: "UDP"
      of 1: "ICMP"
      else: $proto
    echo &"  {name}: {count}"
  echo "\nTop Source IPs:"
  var srcList = stats.bySourceIP.pairs.toSeq()
  srcList.sort(proc(a, b: auto): int = -cmp(a[1], b[1]))
  for (ip, count) in srcList[0..min(5, srcList.len-1)]:
    echo &"  {ip}: {count}"
```

---

## สรุป

| Tool | ใช้เมื่อ (Authorized Only) |
|------|--------------------------|
| Port Scanner | Network asset discovery |
| Dir Fuzzer | Web application assessment |
| Cred Tester | Password policy testing |
| Packet Analysis | Network traffic analysis |

> **จริยธรรม**: เครื่องมือเหล่านี้ต้องใช้กับการทดสอบที่ได้รับอนุญาตเท่านั้น การใช้กับระบบที่ไม่ได้รับอนุญาตถือเป็นอาชญากรรมทางคอมพิวเตอร์

---

**Next**: [Part 92 - Defensive Security & Blue Team Tools](../security/part92_defensive.md)
