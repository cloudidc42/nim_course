# Part 88 - Network Programming Internals

## บทนำ

Network programming ระดับต่ำด้วย Nim:
- Raw socket programming
- Custom protocol implementation
- DNS resolution
- TCP connection pooling
- HTTP/2 framing

---

## 1. Raw Socket Programming

```nim
# raw_socket.nim
# Low-level socket programming

import std/[net, nativesockets, strformat, times, endians]

# TCP Echo Server (low level)
proc startEchoServer*(port: int) =
  let serverSock = createNativeSocket(AF_INET, SOCK_STREAM, IPPROTO_TCP)
  if serverSock == osInvalidSocket:
    raise newException(OSError, "Failed to create socket")
  
  # Enable SO_REUSEADDR
  var optVal: cint = 1
  discard setsockopt(serverSock, SOL_SOCKET, SO_REUSEADDR,
    cast[cstring](addr optVal), sizeof(optVal).SockLen)
  
  var addr4: Sockaddr_in
  addr4.sin_family = AF_INET.uint16
  addr4.sin_port = htons(port.uint16)
  addr4.sin_addr.s_addr = INADDR_ANY
  
  if bindAddr(serverSock, cast[ptr SockAddr](addr addr4),
      sizeof(addr4).SockLen) < 0:
    raise newException(OSError, "Bind failed")
  
  if listen(serverSock, SOMAXCONN) < 0:
    raise newException(OSError, "Listen failed")
  
  echo &"Echo server listening on :{port}"
  
  while true:
    var clientAddr: Sockaddr_in
    var clientLen: SockLen = sizeof(clientAddr).SockLen
    
    let clientSock = accept(serverSock,
      cast[ptr SockAddr](addr clientAddr), addr clientLen)
    
    if clientSock == osInvalidSocket: continue
    
    # Get client IP
    let clientIP = $inet_ntoa(clientAddr.sin_addr)
    let clientPort = ntohs(clientAddr.sin_port)
    echo &"Connection from {clientIP}:{clientPort}"
    
    # Echo loop
    var buf: array[4096, byte]
    while true:
      let n = recv(clientSock, cast[cstring](addr buf[0]), buf.len, 0)
      if n <= 0: break
      discard send(clientSock, cast[cstring](addr buf[0]), n, 0)
    
    discard closesocket(clientSock)

# UDP socket
proc udpSend*(host: string, port: int, data: string) =
  let sock = createNativeSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
  defer: discard closesocket(sock)
  
  var serverAddr: Sockaddr_in
  serverAddr.sin_family = AF_INET.uint16
  serverAddr.sin_port = htons(port.uint16)
  serverAddr.sin_addr.s_addr = inet_addr(host.cstring)
  
  discard sendto(sock, data.cstring, data.len, 0,
    cast[ptr SockAddr](addr serverAddr), sizeof(serverAddr).SockLen)

proc udpReceive*(port: int, bufSize = 65536): (string, string, int) =
  ## Returns (data, senderIP, senderPort)
  let sock = createNativeSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
  defer: discard closesocket(sock)
  
  var addr4: Sockaddr_in
  addr4.sin_family = AF_INET.uint16
  addr4.sin_port = htons(port.uint16)
  addr4.sin_addr.s_addr = INADDR_ANY
  
  discard bindAddr(sock, cast[ptr SockAddr](addr addr4), sizeof(addr4).SockLen)
  
  var buf = newString(bufSize)
  var senderAddr: Sockaddr_in
  var senderLen: SockLen = sizeof(senderAddr).SockLen
  
  let n = recvfrom(sock, buf.cstring, bufSize, 0,
    cast[ptr SockAddr](addr senderAddr), addr senderLen)
  
  buf.setLen(max(0, n))
  let senderIP = $inet_ntoa(senderAddr.sin_addr)
  let senderPort = ntohs(senderAddr.sin_port).int
  
  (buf, senderIP, senderPort)
```

---

## 2. Custom Protocol: Simple Frame Protocol

```nim
# frame_protocol.nim
# Simple framed protocol: [4-byte length][1-byte type][payload]

import std/[asyncdispatch, asyncnet, strformat, endians]

type
  FrameType* = enum
    ftData = 1
    ftControl = 2
    ftHeartbeat = 3
    ftError = 4

  Frame* = object
    frameType*: FrameType
    payload*: string

  FramedConnection* = ref object
    socket*: AsyncSocket
    onFrame*: proc(conn: FramedConnection, frame: Frame): Future[void]

proc encodeFrame*(frameType: FrameType, payload: string): string =
  let totalLen = 1 + payload.len  # type byte + payload
  var header: array[4, byte]
  let n = totalLen.uint32
  bigEndian32(addr header[0], unsafeAddr n)
  
  result = newString(5 + payload.len)
  copyMem(addr result[0], addr header[0], 4)
  result[4] = chr(frameType.ord)
  if payload.len > 0:
    copyMem(addr result[5], unsafeAddr payload[0], payload.len)

proc sendFrame*(conn: FramedConnection, frameType: FrameType, payload: string) {.async.} =
  let encoded = encodeFrame(frameType, payload)
  await conn.socket.send(encoded)

proc readFrame*(conn: FramedConnection): Future[Frame] {.async.} =
  # Read 4-byte length header
  let headerData = await conn.socket.recv(4)
  if headerData.len < 4:
    raise newException(EOFError, "Connection closed")
  
  var length: uint32
  bigEndian32(addr length, unsafeAddr headerData[0])
  
  if length == 0:
    raise newException(ValueError, "Invalid frame: zero length")
  if length > 10 * 1024 * 1024:  # 10MB max
    raise newException(ValueError, &"Frame too large: {length}")
  
  # Read payload (type byte + data)
  let body = await conn.socket.recv(length.int)
  if body.len < 1:
    raise newException(EOFError, "Incomplete frame")
  
  let frameType = FrameType(body[0].ord)
  let payload = if body.len > 1: body[1..^1] else: ""
  
  Frame(frameType: frameType, payload: payload)

proc startFrameServer*(port: int,
    handler: proc(conn: FramedConnection, frame: Frame): Future[void]) {.async.} =
  let server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(port))
  server.listen()
  
  echo &"Frame server on :{port}"
  
  while true:
    let clientSock = await server.accept()
    let conn = FramedConnection(socket: clientSock, onFrame: handler)
    
    asyncCheck (proc() {.async.} =
      try:
        while true:
          let frame = await conn.readFrame()
          await handler(conn, frame)
      except:
        clientSock.close()
    )()

# Heartbeat manager
proc startHeartbeat*(conn: FramedConnection, intervalMs = 5000) {.async.} =
  while not conn.socket.isClosed:
    await sleepAsync(intervalMs)
    try:
      await conn.sendFrame(ftHeartbeat, "ping")
    except:
      break

when isMainModule:
  proc handleFrame(conn: FramedConnection, frame: Frame) {.async.} =
    case frame.frameType
    of ftData:
      echo &"Received data: {frame.payload}"
      await conn.sendFrame(ftData, "Echo: " & frame.payload)
    of ftHeartbeat:
      await conn.sendFrame(ftHeartbeat, "pong")
    of ftControl:
      echo &"Control: {frame.payload}"
    of ftError:
      echo &"Error from client: {frame.payload}"
  
  waitFor startFrameServer(9000, handleFrame)
```

---

## 3. DNS Client

```nim
# dns_client.nim
# Simple DNS query implementation

import std/[asyncdispatch, asyncnet, endians, strformat, tables, strutils]

type
  RecordType* = enum
    rtA = 1
    rtNS = 2
    rtCNAME = 5
    rtSOA = 6
    rtMX = 15
    rtAAAA = 28
    rtTXT = 16

  DnsAnswer* = object
    name*: string
    recordType*: RecordType
    ttl*: uint32
    data*: string

  DnsResponse* = object
    id*: uint16
    answers*: seq[DnsAnswer]
    authorities*: seq[DnsAnswer]

proc encodeName*(name: string): seq[byte] =
  for part in name.split('.'):
    result.add(byte(part.len))
    for c in part: result.add(byte(c))
  result.add(0)  # Null terminator

proc buildQuery*(domain: string, recordType: RecordType, id: uint16): seq[byte] =
  # Header: ID, Flags, QDCOUNT, ANCOUNT, NSCOUNT, ARCOUNT
  var header: array[12, byte]
  bigEndian16(addr header[0], unsafeAddr id)
  header[2] = 0x01  # RD (recursion desired)
  header[3] = 0x00
  header[5] = 1     # QDCOUNT = 1
  
  result = @header
  result.add(encodeName(domain))
  
  # QTYPE and QCLASS
  result.add(0); result.add(byte(recordType))
  result.add(0); result.add(1)  # IN class

proc parseName*(data: seq[byte], offset: var int): string =
  var parts: seq[string]
  var jumped = false
  var origOffset = offset
  
  while offset < data.len:
    let length = data[offset].int
    
    if length == 0:
      inc offset
      break
    
    # Pointer compression
    if (length and 0xC0) == 0xC0:
      if offset + 1 >= data.len: break
      let ptr = ((length and 0x3F) shl 8) or data[offset+1].int
      if not jumped:
        origOffset = offset + 2
        jumped = true
      offset = ptr
      continue
    
    inc offset
    if offset + length > data.len: break
    
    parts.add(cast[string](data[offset..<offset+length]))
    offset += length
  
  if jumped: offset = origOffset
  parts.join(".")

proc parseResponse*(data: seq[byte]): DnsResponse =
  if data.len < 12: return
  
  var id: uint16
  bigEndian16(addr id, unsafeAddr data[0])
  result.id = id
  
  let ancount = (data[6].int shl 8) or data[7].int
  
  var pos = 12
  
  # Skip question section
  discard parseName(data, pos)
  pos += 4  # QTYPE + QCLASS
  
  # Parse answers
  for _ in 0..<ancount:
    if pos >= data.len: break
    
    let name = parseName(data, pos)
    if pos + 10 > data.len: break
    
    let rtype = (data[pos].int shl 8) or data[pos+1].int
    pos += 4  # type + class
    
    var ttl: uint32
    bigEndian32(addr ttl, unsafeAddr data[pos])
    pos += 4
    
    let rdlen = (data[pos].int shl 8) or data[pos+1].int
    pos += 2
    
    if pos + rdlen > data.len: break
    
    var rdata = ""
    case rtype
    of 1:  # A record
      if rdlen == 4:
        rdata = &"{data[pos]}.{data[pos+1]}.{data[pos+2]}.{data[pos+3]}"
    of 28:  # AAAA
      if rdlen == 16:
        var parts: seq[string]
        for i in countup(0, 14, 2):
          parts.add(toHex((data[pos+i].int shl 8) or data[pos+i+1].int, 4))
        rdata = parts.join(":")
    of 5, 2:  # CNAME, NS
      var namePos = pos
      rdata = parseName(data, namePos)
    else:
      rdata = cast[string](data[pos..<pos+rdlen])
    
    pos += rdlen
    
    result.answers.add(DnsAnswer(
      name: name,
      recordType: try: RecordType(rtype) except: rtA,
      ttl: ttl,
      data: rdata
    ))

proc resolve*(domain: string, recordType = rtA,
    server = "8.8.8.8", port = 53): Future[seq[string]] {.async.} =
  let sock = newAsyncSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
  defer: sock.close()
  
  let id = 0x1234u16
  let query = buildQuery(domain, recordType, id)
  
  await sock.sendTo(server, Port(port), cast[string](query))
  
  let response = await sock.recvFrom(512)
  let parsed = parseResponse(cast[seq[byte]](response[0]))
  
  result = parsed.answers.mapIt(it.data)

when isMainModule:
  proc main() {.async.} =
    echo "Resolving nim-lang.org..."
    let ips = await resolve("nim-lang.org")
    for ip in ips:
      echo &"  A: {ip}"
    
    let mx = await resolve("gmail.com", rtMX)
    for record in mx:
      echo &"  MX: {record}"
  
  waitFor main()
```

---

## 4. Connection Pool

```nim
# conn_pool.nim
# TCP connection pooling

import std/[asyncdispatch, asyncnet, tables, times, locks, strformat]

type
  PooledConnection* = ref object
    socket*: AsyncSocket
    host*: string
    port*: int
    inUse*: bool
    lastUsed*: float
    id*: int

  ConnectionPool* = ref object
    connections*: seq[PooledConnection]
    maxSize*: int
    host*: string
    port*: int
    idleTimeout*: float  # seconds
    lock*: Lock
    nextId*: int

proc newConnectionPool*(host: string, port, maxSize: int,
    idleTimeout = 60.0): ConnectionPool =
  result = ConnectionPool(
    host: host,
    port: port,
    maxSize: maxSize,
    idleTimeout: idleTimeout,
    connections: @[]
  )
  initLock(result.lock)

proc createConnection*(pool: ConnectionPool): Future[PooledConnection] {.async.} =
  let sock = newAsyncSocket()
  await sock.connect(pool.host, Port(pool.port))
  
  acquire(pool.lock)
  let id = pool.nextId
  inc pool.nextId
  release(pool.lock)
  
  PooledConnection(
    socket: sock,
    host: pool.host,
    port: pool.port,
    inUse: true,
    lastUsed: cpuTime(),
    id: id
  )

proc acquire*(pool: ConnectionPool): Future[PooledConnection] {.async.} =
  acquire(pool.lock)
  
  # Find available connection
  for conn in pool.connections:
    if not conn.inUse and not conn.socket.isClosed:
      conn.inUse = true
      conn.lastUsed = cpuTime()
      release(pool.lock)
      return conn
  
  # Create new if under limit
  if pool.connections.len < pool.maxSize:
    release(pool.lock)
    let conn = await pool.createConnection()
    acquire(pool.lock)
    pool.connections.add(conn)
    release(pool.lock)
    return conn
  
  release(pool.lock)
  
  # Wait for available connection (simplified: poll)
  while true:
    await sleepAsync(10)
    acquire(pool.lock)
    for conn in pool.connections:
      if not conn.inUse:
        conn.inUse = true
        conn.lastUsed = cpuTime()
        release(pool.lock)
        return conn
    release(pool.lock)

proc release*(pool: ConnectionPool, conn: PooledConnection) =
  acquire(pool.lock)
  conn.inUse = false
  conn.lastUsed = cpuTime()
  release(pool.lock)

proc evictIdle*(pool: ConnectionPool) =
  acquire(pool.lock)
  let now = cpuTime()
  pool.connections = pool.connections.filterIt(
    it.inUse or (now - it.lastUsed < pool.idleTimeout)
  )
  release(pool.lock)

template withConnection*(pool: ConnectionPool, conn, body: untyped): untyped =
  let conn = await pool.acquire()
  defer: pool.release(conn)
  body
```

---

## สรุป

| Topic | Implements |
|-------|-----------|
| Raw sockets | TCP/UDP server foundation |
| Frame protocol | Length-prefixed messaging |
| DNS client | Name resolution |
| Connection pool | Reuse, idle eviction |

---

**Next**: [Part 89 - Database Internals](../advanced/part89_db_internals.md)
