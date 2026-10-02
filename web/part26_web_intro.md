# Part 26: Web Development ใน Nim - เบื้องต้น

## HTTP Client

```nim
import httpclient, json, strutils

# GET request เบื้องต้น
let client = newHttpClient()
defer: client.close()

let response = client.get("https://httpbin.org/get")
echo "Status: ", response.status
echo "Body: ", response.body[0..200]

# GET with headers
client.headers = newHttpHeaders({
  "Accept": "application/json",
  "Authorization": "Bearer mytoken123",
  "User-Agent": "NimApp/1.0"
})

let jsonResp = client.get("https://httpbin.org/headers")
let data = parseJson(jsonResp.body)
echo "Headers sent: ", data["headers"]

# POST request
let postBody = $(%*{
  "username": "alice",
  "password": "secret",
  "rememberMe": true
})

let postResp = client.post(
  "https://httpbin.org/post",
  body = postBody,
  contentType = "application/json"
)
echo "POST status: ", postResp.status

# PUT request
let putResp = client.put(
  "https://httpbin.org/put",
  body = """{"id": 1, "name": "Updated"}""",
  contentType = "application/json"
)

# DELETE request
let delResp = client.delete("https://httpbin.org/delete")

# HEAD request (headers only, no body)
let headResp = client.head("https://httpbin.org/get")
echo "Content-Type: ", headResp.headers["Content-Type"]
```

## Async HTTP Client

```nim
import asyncdispatch, asynchttpclient, json, sequtils

proc fetchUrl(url: string): Future[string] {.async.} =
  let client = newAsyncHttpClient()
  defer: client.close()
  
  client.headers = newHttpHeaders({
    "User-Agent": "NimAsyncBot/1.0"
  })
  
  let resp = await client.get(url)
  return await resp.body

proc fetchMultiple(urls: seq[string]): Future[seq[string]] {.async.} =
  var futures = urls.mapIt(fetchUrl(it))
  return await all(futures)

proc main() {.async.} =
  let urls = @[
    "https://httpbin.org/uuid",
    "https://httpbin.org/user-agent",
    "https://httpbin.org/ip"
  ]
  
  echo "Fetching ", urls.len, " URLs concurrently..."
  let results = await fetchMultiple(urls)
  
  for i, r in results:
    let j = parseJson(r)
    echo "\nURL: ", urls[i]
    echo "Response: ", r[0..min(100, r.len-1)]

waitFor main()
```

## Simple HTTP Server

```nim
import asynchttpserver, asyncdispatch, strutils, json

proc handler(req: Request) {.async.} =
  let path = req.url.path
  let `method` = req.reqMethod
  
  echo `method`, " ", path
  
  case path
  of "/":
    await req.respond(Http200, "Hello from Nim!", 
      newHttpHeaders({"Content-Type": "text/html"}))
  
  of "/health":
    let body = $(%*{"status": "ok", "timestamp": $now()})
    await req.respond(Http200, body,
      newHttpHeaders({"Content-Type": "application/json"}))
  
  of "/echo":
    let body = req.body
    await req.respond(Http200, body)
  
  else:
    await req.respond(Http404, "Not Found")

proc main() {.async.} =
  var server = newAsyncHttpServer()
  echo "Server starting on http://localhost:8080"
  server.listen(Port(8080))
  
  while true:
    if server.shouldClose:
      break
    await server.acceptRequest(handler)

asyncCheck main()
runForever()
```

## Routing

```nim
import asynchttpserver, asyncdispatch, strutils, tables, json, re

type
  RouteHandler = proc(req: Request, params: Table[string, string]): Future[void] {.async.}
  
  Route = object
    pattern: Regex
    paramNames: seq[string]
    handler: RouteHandler
  
  Router = object
    routes: Table[string, seq[Route]]  # method -> routes

proc newRouter(): Router =
  Router(routes: initTable[string, seq[Route]]())

proc addRoute(router: var Router, `method`, path: string, handler: RouteHandler) =
  # Convert /users/:id to regex
  var paramNames: seq[string] = @[]
  var pattern = "^"
  
  for part in path.split("/"):
    if part.len == 0: continue
    if part.startsWith(":"):
      paramNames.add(part[1..^1])
      pattern &= "/([^/]+)"
    elif part.startsWith("*"):
      pattern &= "/(.*)"
    else:
      pattern &= "/" & part
  
  pattern &= "$"
  
  let route = Route(
    pattern: re(pattern),
    paramNames: paramNames,
    handler: handler
  )
  
  if `method` notin router.routes:
    router.routes[`method`] = @[]
  router.routes[`method`].add(route)

proc dispatch(router: Router, req: Request): Future[void] {.async.} =
  let `method` = $req.reqMethod
  let path = req.url.path
  
  if `method` in router.routes:
    for route in router.routes[`method`]:
      let matches = path.findAll(route.pattern)
      if matches.len > 0:
        var params = initTable[string, string]()
        # Extract params (simplified)
        await route.handler(req, params)
        return
  
  await req.respond(Http404, "Not found")

# Usage
var router = newRouter()

router.addRoute("GET", "/", proc(req: Request, params: Table[string, string]) {.async.} =
  await req.respond(Http200, "<h1>Home</h1>")
)

router.addRoute("GET", "/users/:id", proc(req: Request, params: Table[string, string]) {.async.} =
  let id = params.getOrDefault("id", "0")
  await req.respond(Http200, $(%*{"id": id, "name": "User " & id}))
)

router.addRoute("POST", "/users", proc(req: Request, params: Table[string, string]) {.async.} =
  let data = parseJson(req.body)
  await req.respond(Http201, $(%*{"created": true, "data": data}))
)
```

## JSON API Pattern

```nim
import asynchttpserver, asyncdispatch, json, strutils, options

type
  User = object
    id: int
    name: string
    email: string
    age: int

var users: seq[User] = @[
  User(id: 1, name: "Alice", email: "alice@example.com", age: 30),
  User(id: 2, name: "Bob", email: "bob@example.com", age: 25),
  User(id: 3, name: "Carol", email: "carol@example.com", age: 35)
]
var nextId = 4

proc userToJson(u: User): JsonNode =
  %*{"id": u.id, "name": u.name, "email": u.email, "age": u.age}

proc corsHeaders(): HttpHeaders =
  newHttpHeaders({
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type, Authorization",
    "Content-Type": "application/json"
  })

proc handleUsers(req: Request) {.async.} =
  let headers = corsHeaders()
  
  case req.reqMethod
  of HttpGet:
    var arr = newJArray()
    for u in users:
      arr.add(userToJson(u))
    await req.respond(Http200, $arr, headers)
  
  of HttpPost:
    try:
      let data = parseJson(req.body)
      let user = User(
        id: nextId,
        name: data["name"].getStr(),
        email: data["email"].getStr(),
        age: data["age"].getInt()
      )
      users.add(user)
      inc nextId
      await req.respond(Http201, $userToJson(user), headers)
    except:
      await req.respond(Http400, $(%*{"error": "Invalid request body"}), headers)
  
  of HttpOptions:
    await req.respond(Http200, "", headers)
  
  else:
    await req.respond(Http405, $(%*{"error": "Method not allowed"}), headers)

proc handleUser(req: Request, id: int) {.async.} =
  let headers = corsHeaders()
  let idx = users.findIt(it.id == id)
  
  if idx < 0:
    await req.respond(Http404, $(%*{"error": "User not found"}), headers)
    return
  
  case req.reqMethod
  of HttpGet:
    await req.respond(Http200, $userToJson(users[idx]), headers)
  
  of HttpPut:
    let data = parseJson(req.body)
    users[idx].name = data.getOrDefault("name", %users[idx].name).getStr()
    users[idx].email = data.getOrDefault("email", %users[idx].email).getStr()
    await req.respond(Http200, $userToJson(users[idx]), headers)
  
  of HttpDelete:
    let deleted = users[idx]
    users.del(idx)
    await req.respond(Http200, $userToJson(deleted), headers)
  
  else:
    await req.respond(Http405, $(%*{"error": "Method not allowed"}), headers)

proc apiHandler(req: Request) {.async.} =
  let path = req.url.path
  
  if path == "/api/users":
    await handleUsers(req)
  elif path.startsWith("/api/users/"):
    let idStr = path["/api/users/".len..^1]
    let id = parseInt(idStr)
    await handleUser(req, id)
  else:
    await req.respond(Http404, $(%*{"error": "Not found"}))

# Main
proc main() {.async.} =
  var server = newAsyncHttpServer()
  echo "API Server: http://localhost:8080/api/users"
  server.listen(Port(8080))
  while true:
    if server.shouldClose: break
    await server.acceptRequest(apiHandler)

# waitFor main()
```

## สรุป Part 26

ในบทนี้เราได้เรียนรู้:
- ✅ HTTP Client (sync และ async)
- ✅ Simple HTTP Server
- ✅ URL Routing
- ✅ JSON REST API
- ✅ CORS headers
- ✅ CRUD operations

---

**Previous**: [Part 25 - Performance](../intermediate/part25_performance.md)
**Next**: [Part 27 - Jester Framework](part27_jester.md)
