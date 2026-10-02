# Part 57 - gRPC in Nim

## บทนำ gRPC

gRPC เป็น high-performance RPC framework จาก Google ที่ใช้ Protocol Buffers (protobuf) สำหรับ serialization และ HTTP/2 เป็น transport

### ทำไมต้องใช้ gRPC?
- **Strong typing** — protobuf schema เป็น source of truth
- **High performance** — binary serialization + HTTP/2 multiplexing
- **Code generation** — generate client/server stubs อัตโนมัติ
- **Streaming** — unary, server-stream, client-stream, bidirectional

---

## 1. Protocol Buffers Basics

```protobuf
// user_service.proto
syntax = "proto3";

package userservice;

option go_package = "./pb";

// Message definitions
message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
  UserRole role = 5;
}

message UserRole {
  string name = 1;
  repeated string permissions = 2;
}

message GetUserRequest {
  int32 user_id = 1;
}

message GetUserResponse {
  User user = 1;
  bool found = 2;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  string password = 3;
}

message CreateUserResponse {
  User user = 1;
  string error = 2;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
  string filter = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
  int32 page = 3;
}

message StreamUpdateRequest {
  int32 user_id = 1;
}

message UserUpdate {
  User user = 1;
  string update_type = 2;
  int64 timestamp = 3;
}

// Service definition
service UserService {
  // Unary RPC
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  
  // Server-streaming RPC
  rpc ListUsers(ListUsersRequest) returns (stream ListUsersResponse);
  
  // Server-streaming for real-time updates
  rpc WatchUser(StreamUpdateRequest) returns (stream UserUpdate);
  
  // Client-streaming RPC
  rpc BatchCreateUsers(stream CreateUserRequest) returns (CreateUserResponse);
}
```

---

## 2. Nim gRPC Implementation (Manual HTTP/2)

Nim ไม่มี official gRPC library แต่เราสามารถ:
1. ใช้ **nim-grpc** (community library)
2. สร้าง gRPC client ผ่าน HTTP/2 manually
3. ใช้ **gRPC-gateway** สำหรับ REST-to-gRPC

```nim
# grpc_client.nim
# Manual gRPC over HTTP/2 implementation
# ใช้สำหรับเรียก gRPC services จาก Nim

import std/[httpclient, strformat, tables, endians]

# gRPC framing format:
# 1 byte: compression flag (0 = not compressed)
# 4 bytes: message length (big-endian uint32)
# N bytes: message body

proc encodeGrpcFrame*(data: string): string =
  ## Encode message into gRPC wire format
  result = ""
  result.add(chr(0))  # No compression
  
  # 4-byte big-endian length
  let length = data.len.uint32
  var be: uint32
  bigEndian32(addr be, addr length)
  result.add(chr(int(be shr 24) and 0xFF))
  result.add(chr(int(be shr 16) and 0xFF))
  result.add(chr(int(be shr 8) and 0xFF))
  result.add(chr(int(be) and 0xFF))
  
  result.add(data)

proc decodeGrpcFrame*(data: string): tuple[compressed: bool, body: string] =
  ## Decode gRPC wire format
  if data.len < 5:
    raise newException(ValueError, "Frame too short")
  
  result.compressed = data[0] == chr(1)
  
  var length: uint32
  let be = (data[1].ord shl 24) or (data[2].ord shl 16) or
           (data[3].ord shl 8) or data[4].ord
  length = be.uint32
  
  if data.len < 5 + length.int:
    raise newException(ValueError, "Frame body truncated")
  
  result.body = data[5..4+length.int]

type
  GrpcClient* = object
    baseUrl*: string
    headers*: seq[tuple[key, val: string]]
    client: HttpClient

proc newGrpcClient*(host: string, port: int = 443, useTls = true): GrpcClient =
  let scheme = if useTls: "https" else: "http"
  result.baseUrl = &"{scheme}://{host}:{port}"
  result.headers = @[
    ("Content-Type", "application/grpc"),
    ("TE", "trailers"),
  ]
  result.client = newHttpClient()

proc call*(client: GrpcClient, service, method: string, requestBody: string): string =
  ## Make a unary gRPC call
  let url = &"{client.baseUrl}/{service}/{method}"
  let frame = encodeGrpcFrame(requestBody)
  
  var headers = newHttpHeaders()
  for (k, v) in client.headers:
    headers[k] = v
  
  let response = client.client.request(url, httpMethod = HttpPost, body = frame, headers = headers)
  
  if response.status != "200 OK":
    raise newException(IOError, &"gRPC call failed: {response.status}")
  
  let (_, body) = decodeGrpcFrame(response.body)
  result = body

# --- Protobuf Manual Encoding (without codegen) ---

type
  ProtoWriter* = object
    buf*: string

proc newProtoWriter*(): ProtoWriter =
  result.buf = ""

proc writeVarint*(w: var ProtoWriter, value: uint64) =
  ## Write variable-length integer
  var v = value
  while v > 0x7F:
    w.buf.add(chr(int(v and 0x7F) or 0x80))
    v = v shr 7
  w.buf.add(chr(int(v)))

proc writeField*(w: var ProtoWriter, fieldNum: int, wireType: int, value: uint64) =
  ## Write field tag + value
  let tag = (fieldNum.uint64 shl 3) or wireType.uint64
  w.writeVarint(tag)
  w.writeVarint(value)

proc writeString*(w: var ProtoWriter, fieldNum: int, s: string) =
  ## Write string field (wire type 2 = length-delimited)
  let tag = (fieldNum.uint64 shl 3) or 2
  w.writeVarint(tag)
  w.writeVarint(s.len.uint64)
  w.buf.add(s)

proc writeInt32*(w: var ProtoWriter, fieldNum: int, value: int32) =
  ## Write int32 field
  w.writeField(fieldNum, 0, value.uint64 and 0xFFFFFFFF)

# --- Protobuf Manual Decoding ---

type
  ProtoReader* = object
    data*: string
    pos*: int

proc newProtoReader*(data: string): ProtoReader =
  result.data = data
  result.pos = 0

proc readVarint*(r: var ProtoReader): uint64 =
  result = 0
  var shift = 0
  while r.pos < r.data.len:
    let b = r.data[r.pos].ord
    inc r.pos
    result = result or ((b and 0x7F).uint64 shl shift)
    if (b and 0x80) == 0: break
    shift += 7

proc readTag*(r: var ProtoReader): tuple[fieldNum: int, wireType: int] =
  let tag = r.readVarint()
  result.fieldNum = int(tag shr 3)
  result.wireType = int(tag and 0x07)

proc readString*(r: var ProtoReader): string =
  let length = r.readVarint()
  result = r.data[r.pos..r.pos + length.int - 1]
  r.pos += length.int

proc readInt32*(r: var ProtoReader): int32 =
  r.readVarint().int32

# --- Example: Encode/Decode GetUserRequest ---

proc encodeGetUserRequest*(userId: int32): string =
  var w = newProtoWriter()
  w.writeInt32(1, userId)  # field 1: user_id
  result = w.buf

proc decodeUser*(data: string): tuple[id: int32, name, email: string] =
  var r = newProtoReader(data)
  while r.pos < r.data.len:
    let (fieldNum, wireType) = r.readTag()
    case fieldNum
    of 1:  # id
      result.id = r.readInt32()
    of 2:  # name
      result.name = r.readString()
    of 3:  # email
      result.email = r.readString()
    else:
      # Skip unknown fields
      case wireType
      of 0: discard r.readVarint()
      of 2: discard r.readString()
      else: break

# --- Demo ---

when isMainModule:
  echo "gRPC Client Demo"
  echo "================"
  
  # Encode request
  let requestBody = encodeGetUserRequest(42)
  echo &"Encoded GetUserRequest (hex): {requestBody.mapIt(it.ord.toHex(2)).join(\" \")}"
  
  let frame = encodeGrpcFrame(requestBody)
  echo &"gRPC frame size: {frame.len} bytes"
  
  # Decode frame back
  let (compressed, body) = decodeGrpcFrame(frame)
  echo &"Decoded: compressed={compressed}, body_len={body.len}"
  
  echo "\nTo use with a real gRPC server:"
  echo "  var client = newGrpcClient(\"localhost\", 50051, useTls = false)"
  echo "  let resp = client.call(\"userservice.UserService\", \"GetUser\", requestBody)"
```

---

## 3. gRPC Server (Using Jester + Custom Handler)

```nim
# grpc_server.nim
# Simple gRPC-compatible server using HTTP/2 framing
# Note: Full HTTP/2 requires a proper library; this shows the concept

import std/[asynchttpserver, asyncdispatch, strformat, tables, json]
import std/[strutils, times]

type
  GrpcHandler* = proc(body: string): Future[string] {.async.}
  GrpcService* = Table[string, GrpcHandler]

  GrpcServer* = object
    services*: Table[string, GrpcService]
    server*: AsyncHttpServer
    port*: int

proc newGrpcServer*(port: int = 50051): GrpcServer =
  result.port = port
  result.services = initTable[string, GrpcService]()
  result.server = newAsyncHttpServer()

proc register*(srv: var GrpcServer, serviceName, methodName: string,
               handler: GrpcHandler) =
  if serviceName notin srv.services:
    srv.services[serviceName] = initTable[string, GrpcHandler]()
  srv.services[serviceName][methodName] = handler

proc decodeFrame(data: string): tuple[compressed: bool, body: string] =
  if data.len < 5: return (false, "")
  result.compressed = data[0] == chr(1)
  let length = (data[1].ord shl 24) or (data[2].ord shl 16) or
               (data[3].ord shl 8) or data[4].ord
  result.body = data[5..4+length]

proc encodeFrame(data: string): string =
  result = chr(0) & chr(0) & chr(0) & chr(0) & chr(data.len)
  result[4] = chr(data.len and 0xFF)
  result[3] = chr((data.len shr 8) and 0xFF)
  result[2] = chr((data.len shr 16) and 0xFF)
  result[1] = chr((data.len shr 24) and 0xFF)
  result.add(data)

proc handle(srv: GrpcServer, req: Request): Future[void] {.async.} =
  # Parse path: /package.Service/Method
  let path = req.url.path
  let parts = path.split("/")
  
  if parts.len < 3:
    await req.respond(Http404, "Not found")
    return
  
  let serviceName = parts[1]
  let methodName = parts[2]
  
  if serviceName notin srv.services or
     methodName notin srv.services[serviceName]:
    await req.respond(Http404, &"Service or method not found: {serviceName}/{methodName}")
    return
  
  let handler = srv.services[serviceName][methodName]
  
  let (_, requestBody) = decodeFrame(req.body)
  
  try:
    let responseBody = await handler(requestBody)
    let frame = encodeFrame(responseBody)
    
    let headers = newHttpHeaders([
      ("content-type", "application/grpc"),
      ("grpc-status", "0"),
    ])
    await req.respond(Http200, frame, headers)
  except Exception as e:
    let headers = newHttpHeaders([
      ("content-type", "application/grpc"),
      ("grpc-status", "2"),  # UNKNOWN error
      ("grpc-message", e.msg),
    ])
    await req.respond(Http200, "", headers)

proc serve*(srv: GrpcServer) {.async.} =
  echo &"gRPC server listening on port {srv.port}"
  proc callback(req: Request): Future[void] {.async.} =
    await srv.handle(req)
  await srv.server.serve(Port(srv.port), callback)

# --- Example Service Implementation ---

type
  UserDB* = object
    users*: Table[int32, tuple[name, email: string]]
    nextId*: int32

var db = UserDB(
  users: {
    1'i32: ("Alice", "alice@example.com"),
    2'i32: ("Bob", "bob@example.com"),
  }.toTable,
  nextId: 3
)

proc getUserHandler(body: string): Future[string] {.async.} =
  # Decode request
  # field 1: user_id (varint)
  var userId: int32 = 0
  if body.len > 1:
    userId = body[1].ord.int32
  
  if userId in db.users:
    let (name, email) = db.users[userId]
    # Encode response (simplified)
    var resp = ""
    resp.add(chr(1 shl 3 or 0))  # field 1, varint: found=true
    resp.add(chr(1))
    # User sub-message (field 1, wire type 2)
    var user = ""
    user.add(chr(1 shl 3 or 0))  # id
    user.add(chr(userId))
    user.add(chr(2 shl 3 or 2))  # name
    user.add(chr(name.len))
    user.add(name)
    user.add(chr(3 shl 3 or 2))  # email
    user.add(chr(email.len))
    user.add(email)
    
    resp.add(chr(2 shl 3 or 2))  # user field in response
    resp.add(chr(user.len))
    resp.add(user)
    result = resp
  else:
    # found = false
    result = chr(1 shl 3 or 0) & chr(0)

when isMainModule:
  var server = newGrpcServer(50051)
  server.register("userservice.UserService", "GetUser", getUserHandler)
  
  echo "Starting gRPC server..."
  waitFor server.serve()
```

---

## 4. gRPC with nim-protobuf

```nim
# protobuf_usage.nim
# Using nim-protobuf library for proper protobuf support
# nimble install protobuf

import protobuf

# Define messages using Nim macros
type
  UserRole* = object of RootObj
    name*: string
    permissions*: seq[string]

  User* = object of RootObj
    id*: int32
    name*: string
    email*: string
    createdAt*: int64
    role*: UserRole

# Manual protobuf serialization using the library
proc encodeUser*(u: User): string =
  ## Encode User to protobuf bytes
  var encoder = newEncoder()
  encoder.writeInt32(1, u.id)
  encoder.writeString(2, u.name)
  encoder.writeString(3, u.email)
  encoder.writeInt64(4, u.createdAt)
  
  if u.role.name.len > 0:
    var roleEncoder = newEncoder()
    roleEncoder.writeString(1, u.role.name)
    for perm in u.role.permissions:
      roleEncoder.writeString(2, perm)
    encoder.writeBytes(5, roleEncoder.finish())
  
  result = encoder.finish()

proc decodeUser*(data: string): User =
  ## Decode User from protobuf bytes
  var decoder = newDecoder(data)
  while decoder.hasMore():
    let (tag, wireType) = decoder.readTag()
    case tag
    of 1: result.id = decoder.readInt32()
    of 2: result.name = decoder.readString()
    of 3: result.email = decoder.readString()
    of 4: result.createdAt = decoder.readInt64()
    of 5:
      let roleBytes = decoder.readBytes()
      var roleDecoder = newDecoder(roleBytes)
      while roleDecoder.hasMore():
        let (rt, _) = roleDecoder.readTag()
        case rt
        of 1: result.role.name = roleDecoder.readString()
        of 2: result.role.permissions.add(roleDecoder.readString())
        else: roleDecoder.skipField(wireType)
    else: decoder.skipField(wireType)

# Test
when isMainModule:
  let user = User(
    id: 42,
    name: "Charlie",
    email: "charlie@example.com",
    createdAt: 1706745600,
    role: UserRole(
      name: "admin",
      permissions: @["read", "write", "delete"]
    )
  )
  
  let encoded = encodeUser(user)
  echo &"Encoded size: {encoded.len} bytes"
  
  let decoded = decodeUser(encoded)
  echo &"Decoded: id={decoded.id}, name={decoded.name}, email={decoded.email}"
  echo &"Role: {decoded.role.name} with permissions: {decoded.role.permissions}"
```

---

## 5. REST/gRPC Gateway Pattern

```nim
# grpc_gateway.nim
# REST-to-gRPC gateway pattern in Nim
# Translate HTTP REST calls to gRPC internally

import std/[asynchttpserver, asyncdispatch, json, strformat, tables, uri]

type
  Route* = object
    httpMethod*: string
    path*: string
    grpcService*: string
    grpcMethod*: string
    requestMapper*: proc(body: JsonNode, pathParams: Table[string, string]): string
    responseMapper*: proc(grpcResp: string): JsonNode

  Gateway* = object
    routes*: seq[Route]
    grpcHost*: string
    grpcPort*: int
    server*: AsyncHttpServer

proc newGateway*(grpcHost: string = "localhost", grpcPort: int = 50051): Gateway =
  result.grpcHost = grpcHost
  result.grpcPort = grpcPort
  result.server = newAsyncHttpServer()

proc addRoute*(gw: var Gateway, route: Route) =
  gw.routes.add(route)

proc matchRoute*(gw: Gateway, httpMethod, path: string):
    tuple[found: bool, route: Route, params: Table[string, string]] =
  result.params = initTable[string, string]()
  for r in gw.routes:
    if r.httpMethod != httpMethod: continue
    
    # Simple path matching with :param support
    let routeParts = r.path.split("/")
    let reqParts = path.split("/")
    
    if routeParts.len != reqParts.len: continue
    
    var match = true
    for i, rp in routeParts:
      if rp.startsWith(":"):
        result.params[rp[1..^1]] = reqParts[i]
      elif rp != reqParts[i]:
        match = false
        break
    
    if match:
      return (true, r, result.params)
  
  result.found = false

proc callGrpc*(gw: Gateway, service, meth, body: string): Future[string] {.async.} =
  ## Call internal gRPC service
  let client = newAsyncHttpClient()
  defer: client.close()
  
  let url = &"http://{gw.grpcHost}:{gw.grpcPort}/{service}/{meth}"
  
  # Encode frame
  let frameBody = chr(0) & chr(0) & chr(0) & chr(0) & chr(body.len) & body
  
  var headers = newHttpHeaders()
  headers["Content-Type"] = "application/grpc"
  headers["TE"] = "trailers"
  
  let response = await client.request(url, httpMethod = HttpPost, body = frameBody)
  result = await response.body

proc handleRequest*(gw: Gateway, req: Request): Future[void] {.async.} =
  let (found, route, params) = gw.matchRoute($req.reqMethod, req.url.path)
  
  if not found:
    await req.respond(Http404, "{\"error\": \"Not found\"}")
    return
  
  # Parse JSON body
  var jsonBody: JsonNode
  try:
    jsonBody = if req.body.len > 0: parseJson(req.body) else: newJObject()
  except:
    await req.respond(Http400, "{\"error\": \"Invalid JSON\"}")
    return
  
  # Map REST request to gRPC
  let grpcBody = route.requestMapper(jsonBody, params)
  
  # Call gRPC
  let grpcResp = await gw.callGrpc(route.grpcService, route.grpcMethod, grpcBody)
  
  # Map gRPC response to REST
  let jsonResp = route.responseMapper(grpcResp)
  
  let headers = newHttpHeaders()  
  headers["Content-Type"] = "application/json"
  await req.respond(Http200, $jsonResp, headers)

proc serve*(gw: Gateway, httpPort: int = 8080) {.async.} =
  echo &"REST/gRPC Gateway listening on port {httpPort}"
  echo &"Proxying to gRPC at {gw.grpcHost}:{gw.grpcPort}"
  
  proc callback(req: Request): Future[void] {.async.} =
    await gw.handleRequest(req)
  
  await gw.server.serve(Port(httpPort), callback)

# --- Example Usage ---

when isMainModule:
  var gw = newGateway("localhost", 50051)
  
  # GET /users/:id -> UserService.GetUser
  gw.addRoute(Route(
    httpMethod: "GET",
    path: "/api/users/:id",
    grpcService: "userservice.UserService",
    grpcMethod: "GetUser",
    requestMapper: proc(body: JsonNode, params: Table[string, string]): string =
      let userId = parseInt(params.getOrDefault("id", "0")).int32
      # Encode as protobuf field 1 (user_id)
      result = chr(0x08) & chr(userId)  # field 1, varint
    ,
    responseMapper: proc(grpcResp: string): JsonNode =
      # Parse gRPC response and convert to JSON
      result = %*{"user": {"id": 1, "name": "Alice"}}
  ))
  
  # POST /users -> UserService.CreateUser
  gw.addRoute(Route(
    httpMethod: "POST",
    path: "/api/users",
    grpcService: "userservice.UserService",
    grpcMethod: "CreateUser",
    requestMapper: proc(body: JsonNode, params: Table[string, string]): string =
      let name = body["name"].getStr("")
      let email = body["email"].getStr("")
      # Encode as protobuf
      result = chr(0x0A) & chr(name.len) & name &
               chr(0x12) & chr(email.len) & email
    ,
    responseMapper: proc(grpcResp: string): JsonNode =
      result = %*{"created": true}
  ))
  
  waitFor gw.serve(8080)
```

---

## สรุป Part 57

| หัวข้อ | เนื้อหา |
|--------|----------|
| Protocol Buffers | Schema definition, wire format |
| gRPC framing | 5-byte header encoding/decoding |
| Nim gRPC client | HTTP/2 + protobuf calls |
| gRPC server | Route-based request dispatching |
| REST/gRPC gateway | Translate HTTP REST to gRPC |

### กรณีการใช้งาน

1. **Microservices** — กำหนด API contract ด้วย proto files
2. **High-performance APIs** — ใช้ binary protocol แทน JSON
3. **Streaming** — Real-time data ผ่าน server-streaming
4. **Polyglot systems** — Nim client เรียก Java/Go gRPC server

**Next**: [Part 58 - Advanced WebSocket Patterns](part58_websocket_advanced.md)
