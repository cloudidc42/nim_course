# Part 90 - Capstone: Full-Stack Production Application

## บทนำ

รวมทุกสิ่งที่เรียนมาสร้าง production-ready application:
**TaskFlow** — Task management API with real-time updates

---

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│              Clients                        │
│   Web Browser    Mobile App    CLI Tool     │
└──────────────────┬──────────────────────────┘
                   │ HTTP / WebSocket
┌──────────────────▼──────────────────────────┐
│              API Gateway (:8080)            │
│    Rate limit │ Auth │ Circuit Breaker      │
└──────────────────┬──────────────────────────┘
        ┌──────────┴──────────┐
        ▼                     ▼
┌───────────────┐    ┌────────────────┐
│  Task Service │    │  User Service  │
│     (:8081)   │    │     (:8082)    │
└───────┬───────┘    └───────┬────────┘
        │                    │
        └──────────┬─────────┘
                   ▼
        ┌──────────────────┐
        │   PostgreSQL DB  │
        └──────────────────┘
```

---

## 1. Project Structure

```
taskflow/
├── taskflow.nimble
├── config/
│   └── config.nim
├── src/
│   ├── main.nim              # Entry point
│   ├── gateway/
│   │   ├── gateway.nim       # API Gateway
│   │   ├── middleware.nim    # Auth, rate limit
│   │   └── proxy.nim         # Reverse proxy
│   ├── services/
│   │   ├── task_service.nim  # Task CRUD
│   │   └── user_service.nim  # User management
│   ├── models/
│   │   ├── task.nim
│   │   └── user.nim
│   ├── db/
│   │   ├── pool.nim          # Connection pool
│   │   └── migrations.nim    # Schema migrations
│   └── realtime/
│       └── websocket.nim     # WebSocket hub
├── tests/
│   ├── test_tasks.nim
│   └── test_users.nim
└── docker-compose.yml
```

---

## 2. Core Models

```nim
# models/task.nim
import std/[times, json, strutils, options, strformat]

type
  TaskStatus* = enum
    tsTodo = "todo"
    tsInProgress = "in_progress"
    tsDone = "done"
    tsCancelled = "cancelled"

  TaskPriority* = enum
    tpLow = "low"
    tpMedium = "medium"
    tpHigh = "high"
    tpUrgent = "urgent"

  Task* = object
    id*: int64
    title*: string
    description*: string
    status*: TaskStatus
    priority*: TaskPriority
    assigneeId*: Option[int64]
    dueDate*: Option[DateTime]
    tags*: seq[string]
    createdBy*: int64
    createdAt*: DateTime
    updatedAt*: DateTime

  CreateTaskRequest* = object
    title*: string
    description*: string
    priority*: string
    assigneeId*: Option[int64]
    dueDate*: Option[string]
    tags*: seq[string]

  UpdateTaskRequest* = object
    title*: Option[string]
    description*: Option[string]
    status*: Option[string]
    priority*: Option[string]
    assigneeId*: Option[int64]

  TaskFilter* = object
    status*: Option[string]
    priority*: Option[string]
    assigneeId*: Option[int64]
    page*: int
    perPage*: int

  PagedResult*[T] = object
    items*: seq[T]
    total*: int
    page*: int
    perPage*: int
    totalPages*: int

proc toJson*(t: Task): JsonNode =
  result = %*{
    "id": t.id,
    "title": t.title,
    "description": t.description,
    "status": $t.status,
    "priority": $t.priority,
    "tags": t.tags,
    "createdBy": t.createdBy,
    "createdAt": t.createdAt.format("yyyy-MM-dd'T'HH:mm:ss'Z'"),
    "updatedAt": t.updatedAt.format("yyyy-MM-dd'T'HH:mm:ss'Z'")
  }
  if t.assigneeId.isSome:
    result["assigneeId"] = %t.assigneeId.get()
  if t.dueDate.isSome:
    result["dueDate"] = %t.dueDate.get().format("yyyy-MM-dd")
```

---

## 3. Task Service

```nim
# services/task_service.nim
import std/[asyncdispatch, json, options, strformat, times, tables]
import ../models/task
import ../db/pool

type
  TaskRepository* = ref object
    pool*: DBPool

  TaskService* = ref object
    repo*: TaskRepository
    cache*: Table[int64, Task]

proc newTaskRepository*(pool: DBPool): TaskRepository =
  TaskRepository(pool: pool)

proc newTaskService*(repo: TaskRepository): TaskService =
  TaskService(repo: repo, cache: initTable[int64, Task]())

# Simulated DB operations (replace with real postgres)
var taskStore: Table[int64, Task]
var nextId: int64 = 1

proc create*(svc: TaskService, req: CreateTaskRequest, userId: int64): Future[Task] {.async.} =
  let now = now()
  let task = Task(
    id: nextId,
    title: req.title,
    description: req.description,
    status: tsTodo,
    priority: try: parseEnum[TaskPriority](req.priority) except: tpMedium,
    assigneeId: req.assigneeId,
    tags: req.tags,
    createdBy: userId,
    createdAt: now,
    updatedAt: now
  )
  inc nextId
  taskStore[task.id] = task
  svc.cache[task.id] = task
  task

proc getById*(svc: TaskService, id: int64): Future[Option[Task]] {.async.} =
  # Check cache first
  if id in svc.cache:
    return some(svc.cache[id])
  
  if id in taskStore:
    let task = taskStore[id]
    svc.cache[id] = task
    return some(task)
  
  none(Task)

proc list*(svc: TaskService, filter: TaskFilter): Future[PagedResult[Task]] {.async.} =
  var tasks: seq[Task]
  
  for id, task in taskStore:
    var matches = true
    
    if filter.status.isSome and $task.status != filter.status.get():
      matches = false
    if filter.priority.isSome and $task.priority != filter.priority.get():
      matches = false
    if filter.assigneeId.isSome and task.assigneeId != filter.assigneeId:
      matches = false
    
    if matches: tasks.add(task)
  
  let total = tasks.len
  let start = (filter.page - 1) * filter.perPage
  let endIdx = min(start + filter.perPage, tasks.len)
  let items = if start < tasks.len: tasks[start..<endIdx] else: @[]
  
  PagedResult[Task](
    items: items,
    total: total,
    page: filter.page,
    perPage: filter.perPage,
    totalPages: (total + filter.perPage - 1) div filter.perPage
  )

proc update*(svc: TaskService, id: int64, req: UpdateTaskRequest): Future[Option[Task]] {.async.} =
  if id notin taskStore: return none(Task)
  
  var task = taskStore[id]
  
  if req.title.isSome: task.title = req.title.get()
  if req.description.isSome: task.description = req.description.get()
  if req.status.isSome:
    task.status = try: parseEnum[TaskStatus](req.status.get()) except: task.status
  if req.priority.isSome:
    task.priority = try: parseEnum[TaskPriority](req.priority.get()) except: task.priority
  if req.assigneeId.isSome: task.assigneeId = req.assigneeId
  
  task.updatedAt = now()
  
  taskStore[id] = task
  svc.cache[id] = task
  some(task)

proc delete*(svc: TaskService, id: int64): Future[bool] {.async.} =
  if id notin taskStore: return false
  taskStore.del(id)
  svc.cache.del(id)
  true
```

---

## 4. HTTP Handler + WebSocket

```nim
# main.nim
# Tie everything together

import std/[asyncdispatch, asynchttpserver, asyncnet, json, strformat, tables, options]
import std/[strutils, uri, times]
import services/task_service
import models/task

type
  AppContext* = ref object
    taskSvc*: TaskService
    wsClients*: Table[int, AsyncSocket]
    nextClientId*: int

var app = AppContext(wsClients: initTable[int, AsyncSocket]())

# Initialize services
let repo = newTaskRepository(nil)  # nil = in-memory for demo
app.taskSvc = newTaskService(repo)

proc parseBody*(req: Request): JsonNode =
  try: parseJson(req.body) except: newJObject()

proc userId*(req: Request): int64 =
  # In real app, decode JWT
  1i64

proc jsonResp*(req: Request, status: HttpCode, body: JsonNode) {.async.} =
  let headers = newHttpHeaders({"Content-Type": "application/json"})
  await req.respond(status, $body, headers)

proc handleTasks*(req: Request) {.async.} =
  let path = req.url.path
  let method = $req.reqMethod
  
  # POST /tasks
  if method == "POST" and path == "/tasks":
    let body = req.parseBody()
    let createReq = CreateTaskRequest(
      title: body{"title"}.getStr("Untitled"),
      description: body{"description"}.getStr(""),
      priority: body{"priority"}.getStr("medium"),
      tags: body{"tags"}.getElems().mapIt(it.getStr())
    )
    let task = await app.taskSvc.create(createReq, req.userId())
    
    # Notify WebSocket clients
    let event = %*{"type": "task_created", "task": task.toJson()}
    for id, sock in app.wsClients:
      try: asyncCheck sock.send($event)
      except: discard
    
    await req.jsonResp(Http201, task.toJson())
    return
  
  # GET /tasks
  if method == "GET" and path == "/tasks":
    let params = req.url.query.parseUri().query
    let filter = TaskFilter(
      page: 1,
      perPage: 20
    )
    let result = await app.taskSvc.list(filter)
    
    let arr = newJArray()
    for t in result.items: arr.add(t.toJson())
    
    await req.jsonResp(Http200, %*{
      "items": arr,
      "total": result.total,
      "page": result.page,
      "totalPages": result.totalPages
    })
    return
  
  # GET/PUT/DELETE /tasks/:id
  let parts = path.split('/')
  if parts.len == 3 and parts[1] == "tasks":
    let id = try: parseInt(parts[2]).int64 except: -1i64
    if id < 0:
      await req.jsonResp(Http400, %*{"error": "Invalid task ID"})
      return
    
    if method == "GET":
      let task = await app.taskSvc.getById(id)
      if task.isNone:
        await req.jsonResp(Http404, %*{"error": "Task not found"})
      else:
        await req.jsonResp(Http200, task.get().toJson())
    
    elif method == "PUT":
      let body = req.parseBody()
      let updateReq = UpdateTaskRequest(
        title: if body.hasKey("title"): some(body["title"].getStr()) else: none(string),
        status: if body.hasKey("status"): some(body["status"].getStr()) else: none(string),
        priority: if body.hasKey("priority"): some(body["priority"].getStr()) else: none(string)
      )
      let updated = await app.taskSvc.update(id, updateReq)
      if updated.isNone:
        await req.jsonResp(Http404, %*{"error": "Task not found"})
      else:
        let event = %*{"type": "task_updated", "task": updated.get().toJson()}
        for _, sock in app.wsClients:
          try: asyncCheck sock.send($event)
          except: discard
        await req.jsonResp(Http200, updated.get().toJson())
    
    elif method == "DELETE":
      let deleted = await app.taskSvc.delete(id)
      if deleted:
        await req.jsonResp(Http204, newJObject())
      else:
        await req.jsonResp(Http404, %*{"error": "Task not found"})
    
    return
  
  await req.jsonResp(Http404, %*{"error": "Not found"})

proc handleHealth*(req: Request) {.async.} =
  await req.jsonResp(Http200, %*{
    "status": "healthy",
    "service": "taskflow-api",
    "version": "1.0.0",
    "timestamp": now().format("yyyy-MM-dd'T'HH:mm:ss'Z'")
  })

proc router*(req: Request) {.async.} =
  let path = req.url.path
  
  if path == "/health": await handleHealth(req)
  elif path.startsWith("/tasks"): await handleTasks(req)
  else: await req.jsonResp(Http404, %*{"error": "Route not found"})

when isMainModule:
  let server = newAsyncHttpServer()
  echo "TaskFlow API running on :8081"
  waitFor server.serve(Port(8081), router)
```

---

## 5. Test Suite

```nim
# tests/test_tasks.nim
import std/[unittest, asyncdispatch, json, options]
import ../src/services/task_service
import ../src/models/task

suite "Task Service":
  
  var svc: TaskService
  
  setup:
    let repo = newTaskRepository(nil)
    svc = newTaskService(repo)
  
  test "create task":
    let req = CreateTaskRequest(
      title: "Test task",
      description: "Description",
      priority: "high",
      tags: @["nim", "test"]
    )
    let task = waitFor svc.create(req, 1)
    check task.title == "Test task"
    check task.status == tsTodo
    check task.priority == tpHigh
    check task.tags == @["nim", "test"]
  
  test "get by id":
    let req = CreateTaskRequest(title: "Find me", priority: "low")
    let created = waitFor svc.create(req, 1)
    let found = waitFor svc.getById(created.id)
    check found.isSome
    check found.get().title == "Find me"
  
  test "get nonexistent task":
    let found = waitFor svc.getById(99999)
    check found.isNone
  
  test "update task status":
    let req = CreateTaskRequest(title: "Update me", priority: "medium")
    let created = waitFor svc.create(req, 1)
    let updateReq = UpdateTaskRequest(status: some("in_progress"))
    let updated = waitFor svc.update(created.id, updateReq)
    check updated.isSome
    check updated.get().status == tsInProgress
  
  test "delete task":
    let req = CreateTaskRequest(title: "Delete me", priority: "low")
    let created = waitFor svc.create(req, 1)
    let deleted = waitFor svc.delete(created.id)
    check deleted == true
    let found = waitFor svc.getById(created.id)
    check found.isNone
  
  test "list with filter":
    discard waitFor svc.create(CreateTaskRequest(title: "Task 1", priority: "high"), 1)
    discard waitFor svc.create(CreateTaskRequest(title: "Task 2", priority: "low"), 1)
    
    let filter = TaskFilter(page: 1, perPage: 10)
    let result = waitFor svc.list(filter)
    check result.total >= 2
```

---

## สรุป — จบ Part 90

นี่คือ capstone project ที่รวมทุกอย่างจากหลักสูตร:

| Part | หัวข้อ |
|------|--------|
| 1-30 | Nim Basics |
| 31-64 | Web, Security, Advanced Nim |
| 65-76 | Distributed, AI/ML, Real-time |
| 77-89 | Concurrency, Deployment, Internals |
| **90** | **Capstone Application** |

---

**Next**: [Part 91 - Advanced Security: Offensive Tools (Educational)](../security/part91_offensive_tools.md)
