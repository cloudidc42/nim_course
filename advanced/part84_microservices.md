# Part 84 - Microservices & Service Mesh

## บทนำ

Nim เหมาะสำหรับ microservices เพราะ binary เล็ก startup เร็ว และ memory ต่ำ

---

## 1. Service Template

```nim
# service_base.nim
# Base template สำหรับทุก microservice

import std/[asyncdispatch, asynchttpserver, json, strformat, times, os]

type
  ServiceConfig* = object
    name*: string
    port*: int
    version*: string
    healthPath*: string

  ServiceHealth* = object
    status*: string
    service*: string
    version*: string
    uptime*: float
    timestamp*: string

  Route* = object
    method*: string
    path*: string
    handler*: proc(req: Request): Future[void]

  Service* = ref object
    config*: ServiceConfig
    routes*: seq[Route]
    startTime*: float
    server*: AsyncHttpServer

var serviceStartTime = cpuTime()

proc newService*(name: string, port: int, version = "1.0.0"): Service =
  Service(
    config: ServiceConfig(
      name: name,
      port: port,
      version: version,
      healthPath: "/health"
    ),
    server: newAsyncHttpServer()
  )

proc route*(svc: Service, `method`, path: string,
    handler: proc(req: Request): Future[void]): Service =
  svc.routes.add(Route(`method`: method, path: path, handler: handler))
  svc

proc health*(svc: Service): ServiceHealth =
  ServiceHealth(
    status: "healthy",
    service: svc.config.name,
    version: svc.config.version,
    uptime: cpuTime() - serviceStartTime,
    timestamp: now().format("yyyy-MM-dd'T'HH:mm:ss'Z'")
  )

proc jsonResponse*(req: Request, status: HttpCode, body: JsonNode) {.async.} =
  let headers = newHttpHeaders({"Content-Type": "application/json"})
  await req.respond(status, $body, headers)

proc matchRoute*(svc: Service, req: Request): Option[Route] =
  for route in svc.routes:
    if route.method == $req.reqMethod and route.path == req.url.path:
      return some(route)
    # Prefix match for parameterized routes
    if route.path.endsWith("*") and req.url.path.startsWith(route.path[0..^2]):
      return some(route)
  none(Route)

proc serve*(svc: Service) {.async.} =
  proc handler(req: Request) {.async.} =
    # Health check
    if req.url.path == svc.config.healthPath:
      await req.jsonResponse(Http200, %svc.health())
      return
    
    # Find matching route
    let route = svc.matchRoute(req)
    if route.isSome:
      await route.get().handler(req)
    else:
      await req.jsonResponse(Http404, %*{"error": "Not found"})
  
  echo &"[{svc.config.name}] listening on :{svc.config.port}"
  await svc.server.serve(Port(svc.config.port), handler)

# Example: User service
type
  User* = object
    id*: int
    name*: string
    email*: string

when isMainModule:
  var users = @[
    User(id: 1, name: "Alice", email: "alice@example.com"),
    User(id: 2, name: "Bob", email: "bob@example.com")
  ]
  
  let svc = newService("user-service", 8081)
  
  svc.route("GET", "/users") do(req: Request) {.async.}:
    let arr = newJArray()
    for u in users: arr.add(%*{"id": u.id, "name": u.name, "email": u.email})
    await req.jsonResponse(Http200, arr)
  
  svc.route("GET", "/users/*") do(req: Request) {.async.}:
    let parts = req.url.path.split('/')
    let id = try: parseInt(parts[^1]) except: -1
    let found = users.filterIt(it.id == id)
    if found.len > 0:
      let u = found[0]
      await req.jsonResponse(Http200, %*{"id": u.id, "name": u.name, "email": u.email})
    else:
      await req.jsonResponse(Http404, %*{"error": "User not found"})
  
  waitFor svc.serve()
```

---

## 2. Service Discovery & Registry

```nim
# service_registry.nim
# Consul-style service registry

import std/[asyncdispatch, asynchttpserver, json, tables, times, strformat, locks]

type
  ServiceInstance* = object
    id*: string
    name*: string
    host*: string
    port*: int
    tags*: seq[string]
    health*: string    # "passing" | "warning" | "critical"
    lastSeen*: float

  ServiceRegistry* = ref object
    services*: Table[string, seq[ServiceInstance]]  # name -> instances
    lock*: Lock
    ttl*: float  # Seconds before instance expires

proc newServiceRegistry*(ttl = 30.0): ServiceRegistry =
  result = ServiceRegistry(ttl: ttl, services: initTable[string, seq[ServiceInstance]]())
  initLock(result.lock)

proc register*(reg: ServiceRegistry, instance: ServiceInstance) =
  acquire(reg.lock)
  defer: release(reg.lock)
  
  if instance.name notin reg.services:
    reg.services[instance.name] = @[]
  
  # Update or add
  var found = false
  for i, inst in reg.services[instance.name]:
    if inst.id == instance.id:
      reg.services[instance.name][i] = instance
      reg.services[instance.name][i].lastSeen = cpuTime()
      found = true
      break
  
  if not found:
    var inst = instance
    inst.lastSeen = cpuTime()
    reg.services[instance.name].add(inst)

proc deregister*(reg: ServiceRegistry, id: string) =
  acquire(reg.lock)
  defer: release(reg.lock)
  
  for name in reg.services.keys.toSeq():
    reg.services[name] = reg.services[name].filterIt(it.id != id)

proc heartbeat*(reg: ServiceRegistry, id: string) =
  acquire(reg.lock)
  defer: release(reg.lock)
  
  for name in reg.services.keys:
    for i, inst in reg.services[name]:
      if inst.id == id:
        reg.services[name][i].lastSeen = cpuTime()
        return

proc discover*(reg: ServiceRegistry, name: string, healthyOnly = true): seq[ServiceInstance] =
  acquire(reg.lock)
  defer: release(reg.lock)
  
  let now = cpuTime()
  if name notin reg.services: return @[]
  
  result = reg.services[name].filterIt(
    now - it.lastSeen < reg.ttl and
    (not healthyOnly or it.health == "passing")
  )

proc evictExpired*(reg: ServiceRegistry) =
  acquire(reg.lock)
  defer: release(reg.lock)
  
  let now = cpuTime()
  for name in reg.services.keys.toSeq():
    reg.services[name] = reg.services[name].filterIt(now - it.lastSeen < reg.ttl)

# Load balancing strategies
proc roundRobin*(instances: seq[ServiceInstance]): ServiceInstance =
  {.global.}: var counter: int
  if instances.len == 0: raise newException(ValueError, "No instances available")
  result = instances[counter mod instances.len]
  inc counter

proc leastConnections*(instances: seq[ServiceInstance],
    connCounts: Table[string, int]): ServiceInstance =
  if instances.len == 0: raise newException(ValueError, "No instances available")
  result = instances[0]
  var minConns = connCounts.getOrDefault(result.id, 0)
  for inst in instances[1..^1]:
    let conns = connCounts.getOrDefault(inst.id, 0)
    if conns < minConns:
      minConns = conns
      result = inst

# Registry HTTP API server
proc startRegistryServer*(reg: ServiceRegistry, port = 8500) {.async.} =
  let server = newAsyncHttpServer()
  
  proc handler(req: Request) {.async.} =
    let path = req.url.path
    
    if req.reqMethod == HttpGet and path.startsWith("/v1/catalog/service/"):
      let name = path[20..^1]
      let instances = reg.discover(name)
      let arr = newJArray()
      for inst in instances:
        arr.add(%*{
          "ID": inst.id,
          "ServiceName": inst.name,
          "ServiceAddress": inst.host,
          "ServicePort": inst.port,
          "ServiceTags": inst.tags,
          "Checks": [{"Status": inst.health}]
        })
      let headers = newHttpHeaders({"Content-Type": "application/json"})
      await req.respond(Http200, $arr, headers)
    
    elif req.reqMethod == HttpPut and path.startsWith("/v1/agent/service/register"):
      try:
        let body = parseJson(req.body)
        let inst = ServiceInstance(
          id: body["ID"].getStr(),
          name: body["Name"].getStr(),
          host: body["Address"].getStr("localhost"),
          port: body["Port"].getInt(),
          health: "passing"
        )
        reg.register(inst)
        await req.respond(Http200, "{}")
      except:
        await req.respond(Http400, "{\"error\": \"Invalid registration\"}")
    
    elif req.reqMethod == HttpPut and path.startsWith("/v1/agent/check/pass/"):
      let id = path[21..^1]
      reg.heartbeat(id)
      await req.respond(Http200, "{}")
    
    else:
      await req.respond(Http404, "{\"error\": \"Not found\"}")
  
  echo &"Service registry listening on :{port}"
  await server.serve(Port(port), handler)
```

---

## 3. API Gateway

```nim
# api_gateway.nim
# Simple API gateway ด้วย reverse proxy

import std/[asyncdispatch, asynchttpserver, asyncnet, json, tables, strformat, uri]
import service_registry

type
  RouteConfig* = object
    prefix*: string
    serviceName*: string
    stripPrefix*: bool
    timeout*: int  # milliseconds
    retries*: int

  Gateway* = ref object
    registry*: ServiceRegistry
    routes*: seq[RouteConfig]
    port*: int

proc newGateway*(registry: ServiceRegistry, port = 8080): Gateway =
  Gateway(registry: registry, port: port, routes: @[])

proc addRoute*(gw: Gateway, prefix, serviceName: string,
    stripPrefix = true, timeout = 30000, retries = 2): Gateway =
  gw.routes.add(RouteConfig(
    prefix: prefix,
    serviceName: serviceName,
    stripPrefix: stripPrefix,
    timeout: timeout,
    retries: retries
  ))
  gw

proc proxyRequest*(targetHost: string, targetPort: int,
    req: Request): Future[string] {.async.} =
  ## Simple HTTP proxy
  let sock = newAsyncSocket()
  await sock.connect(targetHost, Port(targetPort))
  
  var reqStr = &"{req.reqMethod} {req.url.path} HTTP/1.1\r\n"
  reqStr &= &"Host: {targetHost}:{targetPort}\r\n"
  for k, v in req.headers:
    reqStr &= &"{k}: {v}\r\n"
  reqStr &= "\r\n"
  if req.body.len > 0:
    reqStr &= req.body
  
  await sock.send(reqStr)
  
  var response = ""
  while true:
    let chunk = await sock.recv(4096)
    if chunk.len == 0: break
    response &= chunk
  
  sock.close()
  response

proc serve*(gw: Gateway) {.async.} =
  let server = newAsyncHttpServer()
  
  proc handler(req: Request) {.async.} =
    # Match route
    var matched: Option[RouteConfig]
    for route in gw.routes:
      if req.url.path.startsWith(route.prefix):
        matched = some(route)
        break
    
    if matched.isNone:
      await req.respond(Http404, "{\"error\": \"No route matched\"}")
      return
    
    let route = matched.get()
    let instances = gw.registry.discover(route.serviceName)
    
    if instances.len == 0:
      await req.respond(Http503,
        &"{{\"error\": \"Service {route.serviceName} unavailable\"}}")
      return
    
    let target = instances.roundRobin()
    
    var path = req.url.path
    if route.stripPrefix:
      path = path[route.prefix.len..^1]
      if path.len == 0: path = "/"
    
    for attempt in 0..route.retries:
      try:
        let response = await proxyRequest(target.host, target.port, req)
        
        # Parse response status line
        let lines = response.split("\r\n")
        if lines.len > 0:
          let statusParts = lines[0].split(' ', 2)
          if statusParts.len >= 2:
            let statusCode = try: parseInt(statusParts[1]) except: 502
            
            # Find body after headers
            let bodyStart = response.find("\r\n\r\n")
            let body = if bodyStart >= 0: response[bodyStart+4..^1] else: ""
            
            await req.respond(HttpCode(statusCode), body)
            return
        
        await req.respond(Http502, "{\"error\": \"Bad gateway response\"}")
        return
      except:
        if attempt == route.retries:
          await req.respond(Http502,
            &"{{\"error\": \"Service {route.serviceName} failed after {route.retries+1} attempts\"}}")
  
  echo &"API Gateway listening on :{gw.port}"
  await server.serve(Port(gw.port), handler)

when isMainModule:
  let registry = newServiceRegistry()
  
  # Pre-register services (in real world, services register themselves)
  registry.register(ServiceInstance(
    id: "user-svc-1",
    name: "user-service",
    host: "localhost",
    port: 8081,
    health: "passing"
  ))
  registry.register(ServiceInstance(
    id: "order-svc-1",
    name: "order-service",
    host: "localhost",
    port: 8082,
    health: "passing"
  ))
  
  let gw = newGateway(registry, 8080)
    .addRoute("/api/users", "user-service")
    .addRoute("/api/orders", "order-service")
  
  waitFor gw.serve()
```

---

## สรุป

| Component | Pattern |
|-----------|---------|
| Service | Thin HTTP wrapper + health endpoint |
| Registry | TTL-based instance tracking |
| Gateway | Prefix routing + load balancing |
| Discovery | Round-robin / least connections |

---

**Next**: [Part 85 - Advanced Error Handling Patterns](../advanced/part85_errors.md)
