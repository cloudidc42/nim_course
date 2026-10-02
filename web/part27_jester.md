# Part 27: Jester Framework - Web Framework สำหรับ Nim

## การติดตั้ง Jester

```bash
# ติดตั้ง nimble package manager
nimble install jester

# หรือเพิ่มใน .nimble file
# requires "jester >= 0.5.0"
```

## Jester เบื้องต้น

```nim
# hello.nim
import jester, asyncdispatch

routes:
  get "/":
    resp "Hello from Jester!"
  
  get "/hello/@name":
    resp "Hello, " & @"name" & "!"
  
  get "/add/@a/@b":
    let sum = parseInt(@"a") + parseInt(@"b")
    resp $sum

runForever()
```

## Full Application

```nim
# app.nim
import jester, asyncdispatch, json, tables, strutils

# In-memory database
type
  Task = object
    id: int
    title: string
    done: bool
    createdAt: string

var tasks: seq[Task] = @[]
var nextId = 1

proc taskToJson(t: Task): JsonNode =
  %*{
    "id": t.id,
    "title": t.title,
    "done": t.done,
    "createdAt": t.createdAt
  }

settings:
  port = Port(8080)
  bindAddr = "127.0.0.1"

routes:
  # Serve HTML
  get "/":
    let html = """
<!DOCTYPE html>
<html>
<head>
  <title>Todo App</title>
  <meta charset="utf-8">
  <style>
    body { font-family: Arial; max-width: 600px; margin: 50px auto; padding: 0 20px; }
    .task { padding: 10px; margin: 5px 0; background: #f0f0f0; border-radius: 4px; }
    .done { text-decoration: line-through; color: #888; }
    input[type=text] { width: 70%; padding: 8px; }
    button { padding: 8px 16px; background: #007bff; color: white; border: none; cursor: pointer; }
  </style>
</head>
<body>
  <h1>Todo App</h1>
  <form onsubmit="addTask(event)">
    <input type="text" id="title" placeholder="New task..." />
    <button type="submit">Add</button>
  </form>
  <div id="tasks"></div>
  <script>
    async function loadTasks() {
      const resp = await fetch('/api/tasks');
      const tasks = await resp.json();
      const div = document.getElementById('tasks');
      div.innerHTML = tasks.map(t => `
        <div class="task ${t.done ? 'done' : ''}">
          <input type="checkbox" ${t.done ? 'checked' : ''} 
                 onchange="toggleTask(${t.id})">
          ${t.title}
          <button onclick="deleteTask(${t.id})" style="float:right">Delete</button>
        </div>
      `).join('');
    }
    
    async function addTask(e) {
      e.preventDefault();
      const title = document.getElementById('title').value;
      await fetch('/api/tasks', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({title})
      });
      document.getElementById('title').value = '';
      loadTasks();
    }
    
    async function toggleTask(id) {
      await fetch('/api/tasks/' + id + '/toggle', {method: 'PUT'});
      loadTasks();
    }
    
    async function deleteTask(id) {
      await fetch('/api/tasks/' + id, {method: 'DELETE'});
      loadTasks();
    }
    
    loadTasks();
  </script>
</body>
</html>"""
    resp html

  # API endpoints
  get "/api/tasks":
    var arr = newJArray()
    for t in tasks:
      arr.add(taskToJson(t))
    resp $arr, "application/json"

  post "/api/tasks":
    let data = parseJson(request.body)
    let task = Task(
      id: nextId,
      title: data["title"].getStr(),
      done: false,
      createdAt: $now()
    )
    tasks.add(task)
    inc nextId
    resp(Http201, $taskToJson(task), "application/json")

  put "/api/tasks/@id/toggle":
    let id = parseInt(@"id")
    for t in tasks.mitems:
      if t.id == id:
        t.done = not t.done
        resp $taskToJson(t), "application/json"
        break
    resp(Http404, $(%*{"error": "Not found"}), "application/json")

  delete "/api/tasks/@id":
    let id = parseInt(@"id")
    let idx = tasks.findIt(it.id == id)
    if idx >= 0:
      let deleted = tasks[idx]
      tasks.del(idx)
      resp $taskToJson(deleted), "application/json"
    else:
      resp(Http404, $(%*{"error": "Not found"}), "application/json")

runForever()
```

## Middleware

```nim
import jester, asyncdispatch, times, strutils, json, tables

# Request logging middleware
proc logRequest(request: Request, resp: ResponseData): Future[void] {.async.} =
  let timestamp = now().format("HH:mm:ss")
  echo &"[{timestamp}] {request.reqMethod} {request.url.path} -> {resp.code}"

# Rate limiting
var rateLimitMap = initTable[string, seq[Time]]()

proc rateLimit(ip: string, maxReqs: int, windowSecs: int): bool =
  let now = getTime()
  let cutoff = now - initDuration(seconds = windowSecs)
  
  if ip notin rateLimitMap:
    rateLimitMap[ip] = @[]
  
  # Remove old requests
  rateLimitMap[ip] = rateLimitMap[ip].filterIt(it > cutoff)
  
  if rateLimitMap[ip].len >= maxReqs:
    return false
  
  rateLimitMap[ip].add(now)
  return true

# Auth middleware
type AuthError = object of CatchableError

proc requireAuth(request: Request): string =
  let authHeader = request.headers.getOrDefault("Authorization", "")
  if not authHeader.startsWith("Bearer "):
    raise newException(AuthError, "Missing token")
  
  let token = authHeader[7..^1]
  if token != "valid-token":
    raise newException(AuthError, "Invalid token")
  
  "user123"  # return user ID

routes:
  before:
    # Run before every request
    let ip = request.ip
    if not rateLimit(ip, 100, 60):
      resp(Http429, $(%*{"error": "Too many requests"}), "application/json")
    
    echo "Request from: ", ip

  get "/public":
    resp "This is public"

  get "/protected":
    try:
      let userId = requireAuth(request)
      resp "Hello, user " & userId
    except AuthError as e:
      resp(Http401, e.msg)

  error Http404:
    resp "Custom 404 page"
  
  error Http500:
    resp "Internal Server Error"

runForever()
```

## File Upload

```nim
import jester, asyncdispatch, os, strutils, json

const uploadDir = "uploads"
createDir(uploadDir)

routes:
  get "/upload":
    resp """
<html>
<body>
  <form method="POST" enctype="multipart/form-data" action="/upload">
    <input type="file" name="file" multiple>
    <input type="submit" value="Upload">
  </form>
</body>
</html>"""

  post "/upload":
    var uploaded: seq[string] = @[]
    
    for name, part in request.formData:
      if part.filename.len > 0:
        let filename = uploadDir / part.filename
        writeFile(filename, part.body)
        uploaded.add(part.filename)
        echo "Uploaded: ", part.filename, " (", part.body.len, " bytes)"
    
    resp $(%*{"uploaded": uploaded})

  get "/files":
    var files: seq[string] = @[]
    for f in walkFiles(uploadDir & "/*"):
      files.add(extractFilename(f))
    resp $(%*{"files": files})

  get "/files/@name":
    let filename = uploadDir / @"name"
    if fileExists(filename):
      sendFile filename
    else:
      resp(Http404, "File not found")

runForever()
```

## WebSockets

```nim
import jester, asyncdispatch, asyncnet, json, tables

var wsClients: Table[int, AsyncSocket] = initTable[int, AsyncSocket]()
var wsClientId = 0

proc broadcastMessage(msg: string, excludeId: int = -1) {.async.} =
  var toRemove: seq[int] = @[]
  
  for id, socket in wsClients:
    if id != excludeId:
      try:
        await socket.send(msg)
      except:
        toRemove.add(id)
  
  for id in toRemove:
    wsClients.del(id)

routes:
  get "/ws":
    # Upgrade to WebSocket
    if request.isWebSocket():
      let socket = request.webSocket()
      let clientId = wsClientId
      inc wsClientId
      
      wsClients[clientId] = socket
      echo "WS client ", clientId, " connected"
      
      # Broadcast join message
      asyncCheck broadcastMessage($(%*{
        "type": "join",
        "id": clientId,
        "message": "Client " & $clientId & " joined"
      }))
      
      try:
        while true:
          let msg = await socket.receiveStrPacket()
          if msg.len == 0: break
          
          # Broadcast to all
          let broadcast = $(%*{
            "type": "message",
            "from": clientId,
            "message": msg
          })
          asyncCheck broadcastMessage(broadcast, clientId)
      except:
        discard
      finally:
        wsClients.del(clientId)
        echo "WS client ", clientId, " disconnected"
        asyncCheck broadcastMessage($(%*{
          "type": "leave",
          "id": clientId
        }))
    else:
      resp "Please use WebSocket"

runForever()
```

## Database Integration

```nim
import jester, asyncdispatch, db_sqlite, json, strutils

let db = open("app.db", "", "", "")

db.exec(sql"""
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
  )
""")

proc rowToJson(row: Row): JsonNode =
  %*{
    "id": parseInt(row[0]),
    "name": row[1],
    "email": row[2],
    "created_at": row[3]
  }

routes:
  get "/users":
    let rows = db.getAllRows(sql"SELECT * FROM users ORDER BY created_at DESC")
    var arr = newJArray()
    for row in rows:
      arr.add(rowToJson(row))
    resp $arr, "application/json"

  get "/users/@id":
    let id = @"id"
    let rows = db.getAllRows(sql"SELECT * FROM users WHERE id = ?", id)
    if rows.len > 0:
      resp $rowToJson(rows[0]), "application/json"
    else:
      resp(Http404, $(%*{"error": "User not found"}), "application/json")

  post "/users":
    let data = parseJson(request.body)
    let name = data["name"].getStr()
    let email = data["email"].getStr()
    
    try:
      db.exec(sql"INSERT INTO users (name, email) VALUES (?, ?)", name, email)
      let id = db.getLastId()
      let row = db.getRow(sql"SELECT * FROM users WHERE id = ?", $id)
      resp(Http201, $rowToJson(row), "application/json")
    except DbError as e:
      resp(Http400, $(%*{"error": e.msg}), "application/json")

  delete "/users/@id":
    let id = @"id"
    let affected = db.execAffectedRows(sql"DELETE FROM users WHERE id = ?", id)
    if affected > 0:
      resp $(%*{"deleted": true, "id": parseInt(id)}), "application/json"
    else:
      resp(Http404, $(%*{"error": "Not found"}), "application/json")

runForever()
```

## สรุป Part 27

ในบทนี้เราได้เรียนรู้:
- ✅ Jester framework basics
- ✅ Full CRUD API
- ✅ Middleware (logging, rate limiting, auth)
- ✅ File upload
- ✅ WebSockets
- ✅ SQLite database integration

---

**Previous**: [Part 26 - Web Intro](part26_web_intro.md)
**Next**: [Part 28 - Database](part28_database.md)
