# Part 46: Microservices Architecture ใน Nim

## Microservices คืออะไร

```
Monolith:
  Single application
  ทุกอย่างอยู่ในสายเดียว
  
  [Users] [Orders] [Products] [Payments] <-- อยู่ในเดียวกัน

Microservices:
  แต่ละส่วนเป็น service แยก
  
  [User Service]    [Order Service]
       |                  |
  [Product Service] [Payment Service]
       |                  |
  สื่อสารผ่าน HTTP API, gRPC, message queue

ข้อดี:
- Scale แต่ละส่วนอิสระ
- แต่ละ service deploy ได้แยกกัน
- ผิดพลาดในส่วนเดียวไม่กระทบรวม
ข้อเสีย:
- ซับซ้อนกว่า
- Network latency
- Distributed tracing ยาก
```

## Service แรก: User Service

```nim
# user_service/main.nim
# User management microservice

import asynchttpserver, asyncdispatch, json, tables, strutils, strformat

type
  User = object
    id: int
    username: string
    email: string
    role: string

# In-memory DB (use real DB in production)
var userDb = @[
  User(id: 1, username: "alice", email: "alice@mail.com", role: "admin"),
  User(id: 2, username: "bob",   email: "bob@mail.com",   role: "user")
]
var nextId = 3

proc jsonResponse(code: HttpCode, body: JsonNode, req: Request) {.async.} =
  await req.respond(code, $body,
    newHttpHeaders({"Content-Type": "application/json",
                    "X-Service": "user-service"}))

proc handler(req: Request) {.async.} =
  let path = req.url.path
  let parts = path.split("/").filterIt(it.len > 0)
  
  case req.reqMethod:
  of HttpGet:
    if path == "/health":
      await jsonResponse(Http200, %*{"status": "ok", "service": "user"}, req)
    
    elif path == "/users":
      await jsonResponse(Http200,
        %userDb.mapIt(%*{"id": it.id, "username": it.username, "email": it.email}),
        req)
    
    elif parts.len == 2 and parts[0] == "users":
      let id = parseInt(parts[1])
      let found = userDb.filterIt(it.id == id)
      if found.len > 0:
        let u = found[0]
        await jsonResponse(Http200,
          %*{"id": u.id, "username": u.username, "email": u.email, "role": u.role},
          req)
      else:
        await jsonResponse(Http404, %*{"error": "User not found"}, req)
    else:
      await jsonResponse(Http404, %*{"error": "Not found"}, req)
  
  of HttpPost:
    if path == "/users":
      let data = parseJson(req.body)
      let user = User(
        id: nextId,
        username: data["username"].getStr(),
        email: data["email"].getStr(),
        role: data.getOrDefault("role", %"user").getStr()
      )
      inc nextId
      userDb.add(user)
      await jsonResponse(Http201, %*{"id": user.id, "username": user.username}, req)
  
  of HttpDelete:
    if parts.len == 2 and parts[0] == "users":
      let id = parseInt(parts[1])
      let before = userDb.len
      userDb.keepItIf(it.id != id)
      if userDb.len < before:
        await jsonResponse(Http200, %*{"message": "Deleted"}, req)
      else:
        await jsonResponse(Http404, %*{"error": "Not found"}, req)
  
  else:
    await jsonResponse(Http405, %*{"error": "Method not allowed"}, req)

let port = 8001
echo &"User Service on :{port}"
let server = newAsyncHttpServer()
waitFor server.serve(Port(port), handler)
```

## API Gateway

```nim
# api_gateway/main.nim
# Reverse proxy + API gateway

import asynchttpserver, asynchttpclient, asyncdispatch, json, strutils, strformat

type
  ServiceConfig = object
    name: string
    url: string
  
  RouteConfig = object
    prefix: string
    service: string

const services = [
  ServiceConfig(name: "users", url: "http://localhost:8001"),
  ServiceConfig(name: "orders", url: "http://localhost:8002"),
  ServiceConfig(name: "products", url: "http://localhost:8003"),
]

const routes = [
  RouteConfig(prefix: "/api/users", service: "users"),
  RouteConfig(prefix: "/api/orders", service: "orders"),
  RouteConfig(prefix: "/api/products", service: "products"),
]

proc findService(path: string): Option[ServiceConfig] =
  import options
  for route in routes:
    if path.startsWith(route.prefix):
      for svc in services:
        if svc.name == route.service:
          return some(svc)
  none(ServiceConfig)

proc handler(req: Request) {.async.} =
  # Health check
  if req.url.path == "/health":
    await req.respond(Http200, """{"status":"ok","service":"gateway"}""",
      newHttpHeaders({"Content-Type": "application/json"}))
    return
  
  # Find service
  let svc = findService(req.url.path)
  if svc.isNone:
    await req.respond(Http404, """{"error":"Service not found"}""",
      newHttpHeaders({"Content-Type": "application/json"}))
    return
  
  # Proxy request
  let client = newAsyncHttpClient()
  defer: client.close()
  
  # Forward headers
  var headers = newHttpHeaders()
  for (key, val) in req.headers:
    if key.toLowerAscii() notin ["host", "connection"]:
      headers[key] = val
  headers["X-Gateway-Version"] = "1.0"
  
  # Construct upstream URL
  let upstreamUrl = svc.get().url & req.url.path & "?" & req.url.query
  
  try:
    let resp = case req.reqMethod:
      of HttpGet: await client.get(upstreamUrl)
      of HttpPost: await client.post(upstreamUrl, req.body)
      of HttpPut: await client.request(upstreamUrl, HttpPut, req.body)
      of HttpDelete: await client.delete(upstreamUrl)
      else: await client.get(upstreamUrl)
    
    let body = await resp.body
    await req.respond(resp.code, body,
      newHttpHeaders({"Content-Type": "application/json",
                      "X-Service": svc.get().name}))
  except:
    let errMsg = getCurrentExceptionMsg()
    await req.respond(Http502,
      $(% *{"error": "Service unavailable", "detail": errMsg}),
      newHttpHeaders({"Content-Type": "application/json"}))

let port = 8080
echo &"API Gateway on :{port}"
let server = newAsyncHttpServer()
waitFor server.serve(Port(port), handler)
```

## Service Discovery และ Health Check

```nim
# service_discovery.nim
# Simple service registry

import asynchttpserver, asyncdispatch, json, tables, times, strutils

type
  ServiceInstance = object
    id: string
    name: string
    host: string
    port: int
    health: string
    lastHeartbeat: Time
    metadata: JsonNode

var registry = initTable[string, seq[ServiceInstance]]()

proc isHealthy(s: ServiceInstance): bool =
  s.health == "healthy" and
  (getTime() - s.lastHeartbeat).inSeconds() < 30  # 30s timeout

proc registryHandler(req: Request) {.async.} =
  let path = req.url.path
  
  if req.reqMethod == HttpPost and path == "/register":
    let data = parseJson(req.body)
    let svc = ServiceInstance(
      id: data["id"].getStr(),
      name: data["name"].getStr(),
      host: data["host"].getStr(),
      port: data["port"].getInt(),
      health: "healthy",
      lastHeartbeat: getTime(),
      metadata: data.getOrDefault("metadata", %*{})
    )
    
    if svc.name notin registry:
      registry[svc.name] = @[]
    registry[svc.name].add(svc)
    
    await req.respond(Http201, $(%*{"id": svc.id, "message": "Registered"}),
      newHttpHeaders({"Content-Type": "application/json"}))
  
  elif req.reqMethod == HttpPost and path.contains("/heartbeat"):
    let parts = path.split("/")
    let svcId = parts[^1]
    for name, instances in registry.mpairs:
      for i in 0..<instances.len:
        if instances[i].id == svcId:
          instances[i].lastHeartbeat = getTime()
          await req.respond(Http200, "{}",
            newHttpHeaders({"Content-Type": "application/json"}))
          return
    await req.respond(Http404, $(%*{"error": "Service not found"}),
      newHttpHeaders({"Content-Type": "application/json"}))
  
  elif req.reqMethod == HttpGet and path.startsWith("/services"):
    let parts = path.split("/").filterIt(it.len > 0)
    if parts.len == 1:
      # List all services
      var result = newJObject()
      for name, instances in registry:
        result[name] = %instances.filterIt(it.isHealthy()).mapIt(
          %*{"id": it.id, "host": it.host, "port": it.port}
        )
      await req.respond(Http200, $result,
        newHttpHeaders({"Content-Type": "application/json"}))
    elif parts.len == 2:
      # Get specific service
      let name = parts[1]
      let healthy = if name in registry:
        registry[name].filterIt(it.isHealthy())
      else: @[]
      
      await req.respond(Http200,
        $(%healthy.mapIt(%*{"id": it.id, "host": it.host, "port": it.port})),
        newHttpHeaders({"Content-Type": "application/json"}))
  else:
    await req.respond(Http404, """{"error":"Not found"}""",
      newHttpHeaders({"Content-Type": "application/json"}))

let server = newAsyncHttpServer()
waitFor server.serve(Port(8500), registryHandler)
```

## สรุป Part 46

ในบทนี้เราได้เรียนรู้:
- ␅ Microservices architecture overview
- ␅ User Service microservice
- ␅ API Gateway / Reverse Proxy
- ␅ Service Discovery และ Health Check
- ␅ Load balancing concepts

---

**Previous**: [Part 45 - WebSocket](part45_websocket.md)
**Next**: [Part 47 - Docker and CI/CD](part47_docker.md)
