# Part 58 - Advanced WebSocket Patterns in Nim

## บทนำ Advanced WebSockets

ใน Part 45 เราเรียนรู้ WebSocket พื้นฐานแล้ว ในส่วนนี้จะดู advanced patterns:
- Room-based pub/sub architecture
- Presence tracking (online users)
- Rate limiting per connection
- Reconnection with exponential backoff
- Binary message protocol
- Horizontal scaling ด้วย Redis pub/sub

---

## 1. Room Manager with Presence

```nim
# ws_room_manager.nim
# Advanced room-based WebSocket with presence tracking

import std/[asyncdispatch, asynchttpserver, tables, sets, json, times]
import std/[strformat, sequtils, random, options]
import ws

# --- Types ---

type
  UserId* = string
  RoomId* = string

  UserPresence* = object
    userId*: UserId
    username*: string
    joinedAt*: DateTime
    lastSeen*: DateTime
    metadata*: JsonNode

  RoomMessage* = object
    id*: string
    roomId*: RoomId
    userId*: UserId
    username*: string
    content*: string
    msgType*: string  # "chat", "system", "action"
    timestamp*: DateTime

  ClientConn* = object
    ws*: WebSocket
    userId*: UserId
    username*: string
    rooms*: HashSet[RoomId]
    connectedAt*: DateTime
    lastActivity*: DateTime
    msgCount*: int  # For rate limiting
    windowStart*: DateTime

  Room* = object
    id*: RoomId
    name*: string
    members*: HashSet[UserId]
    maxMembers*: int
    isPrivate*: bool
    createdAt*: DateTime
    messageHistory*: seq[RoomMessage]
    historyLimit*: int

  RoomManager* = ref object
    rooms*: Table[RoomId, Room]
    connections*: Table[UserId, ClientConn]
    userToConn*: Table[UserId, WebSocket]
    rateLimitWindow*: int  # seconds
    rateLimitMax*: int     # messages per window

# --- Room Manager Implementation ---

proc newRoomManager*(rateLimitWindow = 10, rateLimitMax = 30): RoomManager =
  RoomManager(
    rooms: initTable[RoomId, Room](),
    connections: initTable[UserId, ClientConn](),
    userToConn: initTable[UserId, WebSocket](),
    rateLimitWindow: rateLimitWindow,
    rateLimitMax: rateLimitMax
  )

proc generateId*(): string =
  randomize()
  result = ""
  for i in 0..7:
    result.add(chr(rand(25) + ord('a')))

proc createRoom*(mgr: RoomManager, name: string,
                 maxMembers = 100, isPrivate = false): Room =
  result = Room(
    id: generateId(),
    name: name,
    members: initHashSet[UserId](),
    maxMembers: maxMembers,
    isPrivate: isPrivate,
    createdAt: now(),
    messageHistory: @[],
    historyLimit: 100
  )
  mgr.rooms[result.id] = result

proc joinRoom*(mgr: RoomManager, userId: UserId, roomId: RoomId): bool =
  if roomId notin mgr.rooms: return false
  var room = mgr.rooms[roomId]
  if room.members.len >= room.maxMembers: return false
  
  room.members.incl(userId)
  mgr.rooms[roomId] = room
  
  if userId in mgr.connections:
    var conn = mgr.connections[userId]
    conn.rooms.incl(roomId)
    mgr.connections[userId] = conn
  
  result = true

proc leaveRoom*(mgr: RoomManager, userId: UserId, roomId: RoomId) =
  if roomId in mgr.rooms:
    var room = mgr.rooms[roomId]
    room.members.excl(userId)
    mgr.rooms[roomId] = room
  
  if userId in mgr.connections:
    var conn = mgr.connections[userId]
    conn.rooms.excl(roomId)
    mgr.connections[userId] = conn

proc isRateLimited*(mgr: RoomManager, userId: UserId): bool =
  if userId notin mgr.connections: return false
  var conn = mgr.connections[userId]
  
  let now = now()
  let windowDiff = (now - conn.windowStart).inSeconds
  
  if windowDiff >= mgr.rateLimitWindow:
    conn.msgCount = 0
    conn.windowStart = now
    mgr.connections[userId] = conn
    return false
  
  result = conn.msgCount >= mgr.rateLimitMax

proc recordMessage*(mgr: RoomManager, userId: UserId) =
  if userId in mgr.connections:
    var conn = mgr.connections[userId]
    inc conn.msgCount
    conn.lastActivity = now()
    mgr.connections[userId] = conn

proc broadcastToRoom*(mgr: RoomManager, roomId: RoomId,
                      msg: JsonNode, exceptUser: UserId = ""): Future[void] {.async.} =
  if roomId notin mgr.rooms: return
  let room = mgr.rooms[roomId]
  
  for memberId in room.members:
    if memberId == exceptUser: continue
    if memberId in mgr.userToConn:
      let ws = mgr.userToConn[memberId]
      try:
        await ws.send($msg)
      except:
        discard  # Connection may have closed

proc addToHistory*(mgr: RoomManager, msg: RoomMessage) =
  if msg.roomId in mgr.rooms:
    var room = mgr.rooms[msg.roomId]
    room.messageHistory.add(msg)
    
    # Trim history if needed
    if room.messageHistory.len > room.historyLimit:
      room.messageHistory = room.messageHistory[^room.historyLimit..^1]
    
    mgr.rooms[msg.roomId] = room

proc getPresence*(mgr: RoomManager, roomId: RoomId): seq[JsonNode] =
  result = @[]
  if roomId notin mgr.rooms: return
  
  let room = mgr.rooms[roomId]
  for userId in room.members:
    if userId in mgr.connections:
      let conn = mgr.connections[userId]
      result.add(%*{
        "userId": userId,
        "username": conn.username,
        "lastSeen": conn.lastActivity.format("HH:mm:ss")
      })

# --- Message Protocol ---

type
  WsMessageType* = enum
    wmtJoin = "join"
    wmtLeave = "leave"
    wmtChat = "chat"
    wmtPresence = "presence"
    wmtHistory = "history"
    wmtError = "error"
    wmtPing = "ping"
    wmtPong = "pong"
    wmtTyping = "typing"

proc buildMessage*(msgType: WsMessageType, data: JsonNode): string =
  $(%*{
    "type": $msgType,
    "data": data,
    "timestamp": now().format("yyyy-MM-dd'T'HH:mm:sszzz")
  })

proc handleWsMessage*(mgr: RoomManager, userId: UserId, ws: WebSocket,
                      rawMsg: string): Future[void] {.async.} =
  var parsed: JsonNode
  try:
    parsed = parseJson(rawMsg)
  except:
    let err = buildMessage(wmtError, %*{"message": "Invalid JSON"})
    await ws.send(err)
    return
  
  let msgType = parsed["type"].getStr("")
  let data = parsed["data"]
  
  case msgType
  of "join":
    let roomId = data["roomId"].getStr("")
    if mgr.joinRoom(userId, roomId):
      # Send history
      if roomId in mgr.rooms:
        let room = mgr.rooms[roomId]
        let history = room.messageHistory.mapIt(%*{
          "userId": it.userId,
          "username": it.username,
          "content": it.content,
          "timestamp": it.timestamp.format("HH:mm:ss")
        })
        let histMsg = buildMessage(wmtHistory, %*{"messages": %history})
        await ws.send(histMsg)
      
      # Notify others
      let joinNotif = buildMessage(wmtPresence, %*{
        "event": "join",
        "userId": userId,
        "username": mgr.connections[userId].username if userId in mgr.connections else userId
      })
      await mgr.broadcastToRoom(roomId, parseJson(joinNotif), userId)
      
      # Send presence list to joiner
      let presence = mgr.getPresence(roomId)
      let presMsg = buildMessage(wmtPresence, %*{
        "event": "list",
        "members": %presence
      })
      await ws.send(presMsg)
    else:
      await ws.send(buildMessage(wmtError, %*{"message": "Cannot join room"}))
  
  of "leave":
    let roomId = data["roomId"].getStr("")
    mgr.leaveRoom(userId, roomId)
    
    let leaveNotif = buildMessage(wmtPresence, %*{
      "event": "leave",
      "userId": userId
    })
    await mgr.broadcastToRoom(roomId, parseJson(leaveNotif))
  
  of "chat":
    if mgr.isRateLimited(userId):
      await ws.send(buildMessage(wmtError, %*{
        "message": "Rate limit exceeded. Slow down."
      }))
      return
    
    mgr.recordMessage(userId)
    
    let roomId = data["roomId"].getStr("")
    let content = data["content"].getStr("")
    let username = if userId in mgr.connections: mgr.connections[userId].username else: userId
    
    let msg = RoomMessage(
      id: generateId(),
      roomId: roomId,
      userId: userId,
      username: username,
      content: content,
      msgType: "chat",
      timestamp: now()
    )
    
    mgr.addToHistory(msg)
    
    let chatMsg = buildMessage(wmtChat, %*{
      "id": msg.id,
      "userId": userId,
      "username": username,
      "content": content,
      "roomId": roomId
    })
    await mgr.broadcastToRoom(roomId, parseJson(chatMsg))
  
  of "ping":
    await ws.send(buildMessage(wmtPong, %*{"ts": $now()}))
  
  of "typing":
    let roomId = data["roomId"].getStr("")
    let typingMsg = buildMessage(wmtTyping, %*{
      "userId": userId,
      "roomId": roomId
    })
    await mgr.broadcastToRoom(roomId, parseJson(typingMsg), userId)
  
  else:
    await ws.send(buildMessage(wmtError, %*{"message": &"Unknown type: {msgType}"}))
```

---

## 2. WebSocket with Redis Pub/Sub (Horizontal Scaling)

```nim
# ws_redis_scale.nim
# Scale WebSocket server horizontally using Redis pub/sub
# Multiple server instances share messages via Redis

import std/[asyncdispatch, asyncnet, json, strformat, tables, hashes, options]

# Simplified Redis client for pub/sub
type
  RedisClient* = ref object
    sock*: AsyncSocket
    host*: string
    port*: int
    subscriptions*: Table[string, seq[proc(msg: string): Future[void] {.async.}]]

proc newRedisClient*(host = "localhost", port = 6379): RedisClient =
  RedisClient(
    sock: newAsyncSocket(),
    host: host,
    port: port,
    subscriptions: initTable[string, seq[proc(msg: string): Future[void] {.async.}]]()
  )

proc connect*(client: RedisClient): Future[void] {.async.} =
  await client.sock.connect(client.host, Port(client.port))

proc sendCommand*(client: RedisClient, cmd: varargs[string]): Future[void] {.async.} =
  var data = &"*{cmd.len}\r\n"
  for arg in cmd:
    data.add(&"${arg.len}\r\n{arg}\r\n")
  await client.sock.send(data)

proc publish*(client: RedisClient, channel, message: string): Future[void] {.async.} =
  await client.sendCommand("PUBLISH", channel, message)

proc subscribe*(client: RedisClient, channel: string,
                handler: proc(msg: string): Future[void] {.async.}): Future[void] {.async.} =
  if channel notin client.subscriptions:
    client.subscriptions[channel] = @[]
    await client.sendCommand("SUBSCRIBE", channel)
  client.subscriptions[channel].add(handler)

proc readLoop*(client: RedisClient): Future[void] {.async.} =
  ## Process incoming Redis pub/sub messages
  while true:
    let line = await client.sock.recvLine()
    if line.len == 0: break
    
    if line == "*3":  # Array of 3 elements (subscribe/message)
      discard await client.sock.recvLine()  # $9
      let msgType = await client.sock.recvLine()  # "message"
      discard await client.sock.recvLine()  # $<channel_len>
      let channel = await client.sock.recvLine()
      discard await client.sock.recvLine()  # $<data_len>
      let data = await client.sock.recvLine()
      
      if msgType == "message" and channel in client.subscriptions:
        for handler in client.subscriptions[channel]:
          await handler(data)

# --- Distributed WebSocket Hub ---

type
  LocalConnection* = object
    ws*: pointer  # WebSocket (type-erased for demo)
    userId*: string

  DistributedHub* = ref object
    serverId*: string
    pubClient*: RedisClient  # For publishing
    subClient*: RedisClient  # For subscribing
    localConns*: Table[string, LocalConnection]
    channelPrefix*: string

proc newDistributedHub*(serverId: string, redisHost = "localhost", redisPort = 6379): DistributedHub =
  DistributedHub(
    serverId: serverId,
    pubClient: newRedisClient(redisHost, redisPort),
    subClient: newRedisClient(redisHost, redisPort),
    localConns: initTable[string, LocalConnection](),
    channelPrefix: "ws:"
  )

proc start*(hub: DistributedHub): Future[void] {.async.} =
  await hub.pubClient.connect()
  await hub.subClient.connect()
  
  # Subscribe to global broadcast channel
  await hub.subClient.subscribe(&"{hub.channelPrefix}broadcast") do (msg: string):
    let data = parseJson(msg)
    # Forward to all local connections
    let fromServer = data["serverId"].getStr("")
    if fromServer != hub.serverId:  # Don't echo own messages
      echo &"[{hub.serverId}] Got broadcast from {fromServer}: {data[\"content\"].getStr(\"\")}"
  
  # Start listening loop
  asyncCheck hub.subClient.readLoop()

proc publishToRoom*(hub: DistributedHub, roomId, message: string): Future[void] {.async.} =
  ## Publish message to all instances handling this room
  let channel = &"{hub.channelPrefix}room:{roomId}"
  let envelope = $(%*{
    "serverId": hub.serverId,
    "roomId": roomId,
    "content": message,
    "ts": $now()
  })
  await hub.pubClient.publish(channel, envelope)

proc subscribeRoom*(hub: DistributedHub, roomId: string,
                    handler: proc(msg: string): Future[void] {.async.}): Future[void] {.async.} =
  let channel = &"{hub.channelPrefix}room:{roomId}"
  await hub.subClient.subscribe(channel, handler)
```

---

## 3. Binary WebSocket Protocol

```nim
# ws_binary_protocol.nim
# Custom binary protocol over WebSockets for performance
# ใช้ทดแทน JSON เพื่อลด overhead

import std/[endians, strformat]

# Message format:
# 2 bytes: message type (uint16)
# 2 bytes: payload length (uint16)
# N bytes: payload

type
  BinMsgType* = enum
    bmtPing = 0x0001
    bmtPong = 0x0002
    bmtChat = 0x0010
    bmtJoin = 0x0020
    bmtLeave = 0x0021
    bmtPresence = 0x0030
    bmtError = 0x00FF

  BinaryMessage* = object
    msgType*: BinMsgType
    payload*: seq[byte]

proc encodeBinMsg*(msgType: BinMsgType, payload: seq[byte]): seq[byte] =
  result = newSeq[byte](4 + payload.len)
  
  # Type (big-endian uint16)
  result[0] = (msgType.uint16 shr 8).byte
  result[1] = (msgType.uint16 and 0xFF).byte
  
  # Length (big-endian uint16)
  let length = payload.len.uint16
  result[2] = (length shr 8).byte
  result[3] = (length and 0xFF).byte
  
  # Payload
  for i, b in payload:
    result[4 + i] = b

proc decodeBinMsg*(data: seq[byte]): BinaryMessage =
  if data.len < 4:
    raise newException(ValueError, "Message too short")
  
  let typeVal = (data[0].uint16 shl 8) or data[1].uint16
  let length = (data[2].uint16 shl 8) or data[3].uint16
  
  result.msgType = BinMsgType(typeVal)
  result.payload = data[4..3+length.int]

# --- Chat message binary encoding ---

type
  ChatPayload* = object
    userId*: uint32
    timestamp*: uint32  # Unix timestamp
    contentLen*: uint16
    content*: string

proc encodeChatPayload*(userId: uint32, content: string): seq[byte] =
  let ts = uint32(epochTime())
  result = newSeq[byte](4 + 4 + 2 + content.len)
  
  var i = 0
  # userId
  result[i] = (userId shr 24).byte; inc i
  result[i] = (userId shr 16).byte; inc i
  result[i] = (userId shr 8).byte; inc i
  result[i] = (userId and 0xFF).byte; inc i
  # timestamp
  result[i] = (ts shr 24).byte; inc i
  result[i] = (ts shr 16).byte; inc i
  result[i] = (ts shr 8).byte; inc i
  result[i] = (ts and 0xFF).byte; inc i
  # content length
  let clen = content.len.uint16
  result[i] = (clen shr 8).byte; inc i
  result[i] = (clen and 0xFF).byte; inc i
  # content
  for c in content:
    result[i] = c.byte; inc i

proc decodeChatPayload*(data: seq[byte]): ChatPayload =
  if data.len < 10: raise newException(ValueError, "Invalid chat payload")
  
  result.userId = (data[0].uint32 shl 24) or (data[1].uint32 shl 16) or
                  (data[2].uint32 shl 8) or data[3].uint32
  result.timestamp = (data[4].uint32 shl 24) or (data[5].uint32 shl 16) or
                     (data[6].uint32 shl 8) or data[7].uint32
  result.contentLen = (data[8].uint16 shl 8) or data[9].uint16
  result.content = cast[string](data[10..9+result.contentLen.int])

when isMainModule:
  echo "Binary WebSocket Protocol Demo"
  echo "================================"
  
  # Encode a chat message
  let userId = 42'u32
  let content = "Hello, world!"
  
  let chatPayload = encodeChatPayload(userId, content)
  echo &"Chat payload size: {chatPayload.len} bytes (vs JSON: ~60 bytes)"
  
  let msg = encodeBinMsg(bmtChat, chatPayload)
  echo &"Full message size: {msg.len} bytes"
  
  # Decode it back
  let decoded = decodeBinMsg(msg)
  let chat = decodeChatPayload(decoded.payload)
  echo &"Decoded: userId={chat.userId}, content='{chat.content}'"
  
  # Comparison
  let jsonSize = $(%*{"type": "chat", "userId": userId, "content": content, "ts": 1706745600}).len
  echo &"JSON equivalent size: {jsonSize} bytes"
  echo &"Binary is {(jsonSize - msg.len) * 100 div jsonSize}% smaller"
```

---

## 4. WebSocket Reconnection Client

```nim
# ws_reconnect_client.nim
# WebSocket client with automatic reconnection and backoff

import std/[asyncdispatch, asyncnet, json, strformat, times, math]
import ws

type
  ReconnectConfig* = object
    initialDelay*: int  # milliseconds
    maxDelay*: int
    maxAttempts*: int
    jitterFactor*: float

  ReconnectClient* = ref object
    url*: string
    config*: ReconnectConfig
    ws*: WebSocket
    connected*: bool
    attemptCount*: int
    onMessage*: proc(msg: string): Future[void] {.async.}
    onConnect*: proc(): Future[void] {.async.}
    onDisconnect*: proc(reason: string): Future[void] {.async.}
    messageQueue*: seq[string]  # Queue messages when offline

proc defaultConfig*(): ReconnectConfig =
  ReconnectConfig(
    initialDelay: 1000,
    maxDelay: 30000,
    maxAttempts: -1,  # Unlimited
    jitterFactor: 0.1
  )

proc newReconnectClient*(url: string, config = defaultConfig()): ReconnectClient =
  ReconnectClient(
    url: url,
    config: config,
    connected: false,
    attemptCount: 0,
    messageQueue: @[]
  )

proc calcBackoff*(client: ReconnectClient): int =
  ## Exponential backoff with jitter
  let delay = min(
    client.config.initialDelay * (2 ^ client.attemptCount),
    client.config.maxDelay
  )
  # Add jitter (±10% by default)
  let jitter = int(delay.float * client.config.jitterFactor * (rand(1.0) * 2 - 1))
  result = max(0, delay + jitter)

proc connect*(client: ReconnectClient): Future[void] {.async.} =
  while true:
    if client.config.maxAttempts > 0 and
       client.attemptCount >= client.config.maxAttempts:
      echo "Max reconnection attempts reached."
      break
    
    try:
      echo &"Connecting to {client.url} (attempt {client.attemptCount + 1})"
      client.ws = await newWebSocket(client.url)
      client.connected = true
      client.attemptCount = 0
      
      if client.onConnect != nil:
        await client.onConnect()
      
      # Drain queued messages
      while client.messageQueue.len > 0:
        let msg = client.messageQueue[0]
        client.messageQueue.delete(0)
        await client.ws.send(msg)
      
      # Read loop
      while client.ws.readyState == Open:
        let msg = await client.ws.receiveStrPacket()
        if client.onMessage != nil:
          await client.onMessage(msg)
    
    except Exception as e:
      client.connected = false
      inc client.attemptCount
      
      if client.onDisconnect != nil:
        await client.onDisconnect(e.msg)
      
      let backoff = client.calcBackoff()
      echo &"Connection failed: {e.msg}. Retrying in {backoff}ms..."
      await sleepAsync(backoff)

proc send*(client: ReconnectClient, msg: string): Future[void] {.async.} =
  if client.connected:
    try:
      await client.ws.send(msg)
    except:
      client.messageQueue.add(msg)
  else:
    client.messageQueue.add(msg)
    echo &"Queued message (offline). Queue size: {client.messageQueue.len}"

# --- Heartbeat/Keep-Alive ---

proc startHeartbeat*(client: ReconnectClient, intervalMs = 30000): Future[void] {.async.} =
  ## Send periodic pings to keep connection alive
  while true:
    await sleepAsync(intervalMs)
    if client.connected:
      try:
        await client.send($(%*{"type": "ping", "ts": epochTime().int}))
      except:
        discard

# --- Demo ---

when isMainModule:
  let client = newReconnectClient("ws://localhost:8080/ws")
  
  client.onConnect = proc(): Future[void] {.async.} =
    echo "[+] Connected!"
    await client.send($(%*{"type": "join", "data": {"roomId": "general"}}))
  
  client.onDisconnect = proc(reason: string): Future[void] {.async.} =
    echo &"[-] Disconnected: {reason}"
  
  client.onMessage = proc(msg: string): Future[void] {.async.} =
    try:
      let data = parseJson(msg)
      echo &"[<] {data[\"type\"].getStr}: {msg[0..min(80, msg.len-1)]}"
    except:
      echo &"[<] {msg[0..min(80, msg.len-1)]}"
  
  # Start heartbeat
  asyncCheck client.startHeartbeat(30000)
  
  # Connect with auto-reconnect
  waitFor client.connect()
```

---

## สรุป Part 58

| Pattern | ประโยชน์ |
|---------|---------|
| Room Manager | จัดการ chat rooms + presence tracking |
| Redis Pub/Sub | Scale across multiple server instances |
| Binary Protocol | ลด bandwidth 40-60% เมื่อเทียบ JSON |
| Reconnect Client | Client-side reliability + message queuing |
| Rate Limiting | ป้องกัน spam/DoS |

### Best Practices

1. **Heartbeats** — Detect dead connections ก่อน TCP timeout
2. **Message queuing** — Buffer messages during disconnection
3. **Binary for performance** — ใช้ binary protocol สำหรับ high-frequency data
4. **Redis for scale** — Pub/sub เพื่อ share state ระหว่าง instances
5. **Presence tracking** — Update last_seen timestamp เสมอ

**Next**: [Part 59 - Offensive Security Tooling (Educational)](../security/part59_offensive_tooling.md)
