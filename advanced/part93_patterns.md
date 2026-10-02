# Part 93 - Advanced Nim Design Patterns

## บทนำ

Design patterns ที่ใช้งานได้จริงใน Nim:
- Functional patterns
- Builder / Fluent interface
- Observer + Event system
- Dependency injection
- Command pattern

---

## 1. Functional Patterns

```nim
# functional.nim
# Functional programming patterns ใน Nim

import std/[sequtils, options, strformat, tables]

# Maybe/Option chaining
proc maybeUser*(id: int): Option[string] =
  if id > 0: some(&"User_{id}") else: none(string)

proc maybeEmail*(user: string): Option[string] =
  some(&"{user.toLowerAscii()}@example.com")

proc maybeSendEmail*(email: string): Option[string] =
  some(&"Sent to {email}")

# Chain with andThen
proc processUser*(id: int): Option[string] =
  maybeUser(id)
    .andThen(maybeEmail)
    .andThen(maybeSendEmail)

# Functor / Monad interface for Result
type
  Result*[T, E] = object
    case ok*: bool
    of true: val*: T
    of false: err*: E

proc pure*[T, E](v: T): Result[T, E] = Result[T, E](ok: true, val: v)
proc fail*[T, E](e: E): Result[T, E] = Result[T, E](ok: false, err: e)

proc fmap*[T, U, E](r: Result[T, E], f: proc(v: T): U): Result[U, E] =
  if r.ok: pure[U, E](f(r.val)) else: fail[U, E](r.err)

proc bind*[T, U, E](r: Result[T, E], f: proc(v: T): Result[U, E]): Result[U, E] =
  if r.ok: f(r.val) else: fail[U, E](r.err)

# Pipe operator
template `|>`*(x: untyped, f: untyped): untyped = f(x)

# Partial application
proc partial*[A, B, C](f: proc(a: A, b: B): C, a: A): proc(b: B): C =
  proc(b: B): C = f(a, b)

proc partial2*[A, B, C, D](f: proc(a: A, b: B, c: C): D, a: A, b: B): proc(c: C): D =
  proc(c: C): D = f(a, b, c)

# Memoization
proc memoize*[A, B](f: proc(a: A): B): proc(a: A): B =
  var cache: Table[A, B]
  proc(a: A): B =
    if a in cache: return cache[a]
    let result = f(a)
    cache[a] = result
    result

when isMainModule:
  # Fibonacci with memoization
  var fib: proc(n: int): int
  fib = memoize(proc(n: int): int =
    if n <= 1: n
    else: fib(n-1) + fib(n-2)
  )
  
  for i in 0..10:
    stdout.write($fib(i) & " ")
  echo ""
  
  # Pipe example
  let result = 42 |> (proc(x: int): int = x * 2) |> (proc(x: int): string = $x)
  echo result  # "84"
  
  # Partial application
  let add = proc(a, b: int): int = a + b
  let add5 = partial(add, 5)
  echo add5(3)  # 8
```

---

## 2. Builder / Fluent Interface

```nim
# builder.nim
# Fluent builder pattern ด้วย method chaining

import std/[strformat, options, times]

type
  EmailBuilder* = ref object
    fromAddr*: string
    toAddrs*: seq[string]
    ccAddrs*: seq[string]
    bccAddrs*: seq[string]
    subject*: string
    body*: string
    isHTML*: bool
    attachments*: seq[string]
    replyTo*: string
    priority*: int

  Email* = object
    fromAddr*, subject*, body*: string
    toAddrs*, ccAddrs*, bccAddrs*: seq[string]
    isHTML*: bool
    attachments*: seq[string]
    priority*: int

proc newEmail*(): EmailBuilder = EmailBuilder(priority: 3)

proc `from`*(b: EmailBuilder, address: string): EmailBuilder =
  b.fromAddr = address; b

proc to*(b: EmailBuilder, addresses: varargs[string]): EmailBuilder =
  b.toAddrs.add(addresses); b

proc cc*(b: EmailBuilder, addresses: varargs[string]): EmailBuilder =
  b.ccAddrs.add(addresses); b

proc bcc*(b: EmailBuilder, addresses: varargs[string]): EmailBuilder =
  b.bccAddrs.add(addresses); b

proc subject*(b: EmailBuilder, s: string): EmailBuilder =
  b.subject = s; b

proc body*(b: EmailBuilder, text: string, html = false): EmailBuilder =
  b.body = text; b.isHTML = html; b

proc attach*(b: EmailBuilder, path: string): EmailBuilder =
  b.attachments.add(path); b

proc highPriority*(b: EmailBuilder): EmailBuilder =
  b.priority = 1; b

proc build*(b: EmailBuilder): Email =
  if b.fromAddr.len == 0: raise newException(ValueError, "From address required")
  if b.toAddrs.len == 0: raise newException(ValueError, "At least one recipient required")
  if b.subject.len == 0: raise newException(ValueError, "Subject required")
  
  Email(
    fromAddr: b.fromAddr,
    toAddrs: b.toAddrs,
    ccAddrs: b.ccAddrs,
    bccAddrs: b.bccAddrs,
    subject: b.subject,
    body: b.body,
    isHTML: b.isHTML,
    attachments: b.attachments,
    priority: b.priority
  )

# HTTP Request builder
type
  HttpRequestBuilder* = ref object
    method*: string
    url*: string
    headers*: seq[(string, string)]
    body*: string
    timeout*: int
    followRedirects*: bool

proc get*(url: string): HttpRequestBuilder =
  HttpRequestBuilder(method: "GET", url: url, timeout: 30000, followRedirects: true)

proc post*(url: string): HttpRequestBuilder =
  HttpRequestBuilder(method: "POST", url: url, timeout: 30000)

proc header*(b: HttpRequestBuilder, key, value: string): HttpRequestBuilder =
  b.headers.add((key, value)); b

proc bearer*(b: HttpRequestBuilder, token: string): HttpRequestBuilder =
  b.header("Authorization", "Bearer " & token)

proc json*(b: HttpRequestBuilder, payload: string): HttpRequestBuilder =
  b.header("Content-Type", "application/json").body = payload; b

proc timeout*(b: HttpRequestBuilder, ms: int): HttpRequestBuilder =
  b.timeout = ms; b

when isMainModule:
  let email = newEmail()
    .`from`("sender@example.com")
    .to("alice@example.com", "bob@example.com")
    .cc("manager@example.com")
    .subject("Important Update")
    .body("<h1>Hello!</h1>", html = true)
    .highPriority()
    .build()
  
  echo &"Sending '{email.subject}' to {email.toAddrs}"
```

---

## 3. Event System

```nim
# event_system.nim
# Strongly-typed event system

import std/[tables, strformat, times, strutils]

type
  EventId* = string

  EventHandler*[T] = proc(event: T)

  EventBus* = ref object
    handlers*: Table[string, seq[pointer]]  # type -> handlers

  Subscription* = object
    eventType*: string
    bus*: EventBus
    index*: int

proc newEventBus*(): EventBus =
  EventBus(handlers: initTable[string, seq[pointer]]())

proc typeKey*[T](): string = $T

proc subscribe*[T](bus: EventBus, handler: EventHandler[T]): Subscription =
  let key = typeKey[T]()
  if key notin bus.handlers:
    bus.handlers[key] = @[]
  
  let fn = cast[pointer](handler)
  bus.handlers[key].add(fn)
  
  Subscription(eventType: key, bus: bus, index: bus.handlers[key].len - 1)

proc unsubscribe*(sub: Subscription) =
  if sub.eventType in sub.bus.handlers:
    if sub.index < sub.bus.handlers[sub.eventType].len:
      sub.bus.handlers[sub.eventType].delete(sub.index)

proc emit*[T](bus: EventBus, event: T) =
  let key = typeKey[T]()
  if key notin bus.handlers: return
  
  for handlerPtr in bus.handlers[key]:
    let handler = cast[EventHandler[T]](handlerPtr)
    handler(event)

# Domain events
type
  UserRegistered* = object
    userId*: int
    email*: string
    registeredAt*: DateTime

  OrderPlaced* = object
    orderId*: int
    userId*: int
    amount*: float

  PaymentCompleted* = object
    orderId*: int
    amount*: float
    method*: string

when isMainModule:
  let bus = newEventBus()
  
  # Subscribe to events
  let sub1 = bus.subscribe[UserRegistered](proc(e: UserRegistered) =
    echo &"[Email] Welcome email to {e.email}"
  )
  
  let sub2 = bus.subscribe[UserRegistered](proc(e: UserRegistered) =
    echo &"[Analytics] New user: {e.userId}"
  )
  
  let sub3 = bus.subscribe[OrderPlaced](proc(e: OrderPlaced) =
    echo &"[Inventory] Reserve items for order {e.orderId}"
  )
  
  let sub4 = bus.subscribe[PaymentCompleted](proc(e: PaymentCompleted) =
    echo &"[Fulfillment] Ship order {e.orderId}"
  )
  
  # Emit events
  bus.emit(UserRegistered(userId: 1, email: "alice@example.com", registeredAt: now()))
  bus.emit(OrderPlaced(orderId: 100, userId: 1, amount: 99.99))
  bus.emit(PaymentCompleted(orderId: 100, amount: 99.99, method: "credit_card"))
```

---

## 4. Command Pattern

```nim
# command.nim
# Command pattern สำหรับ undo/redo

import std/[strformat, sequtils]

type
  Command* = ref object of RootObj

method execute*(cmd: Command) {.base.} = discard
method undo*(cmd: Command) {.base.} = discard
method describe*(cmd: Command): string {.base.} = "Unknown command"

# Text editor commands
type
  TextBuffer* = ref object
    content*: string
    cursor*: int

  InsertCommand* = ref object of Command
    buffer*: TextBuffer
    pos*: int
    text*: string

  DeleteCommand* = ref object of Command
    buffer*: TextBuffer
    pos*: int
    deleted*: string

  MoveCommand* = ref object of Command
    buffer*: TextBuffer
    from*, to*: int

method execute*(cmd: InsertCommand) =
  cmd.buffer.content.insert(cmd.text, cmd.pos)
  cmd.buffer.cursor = cmd.pos + cmd.text.len

method undo*(cmd: InsertCommand) =
  cmd.buffer.content.delete(cmd.pos, cmd.pos + cmd.text.len - 1)
  cmd.buffer.cursor = cmd.pos

method describe*(cmd: InsertCommand): string =
  &"Insert '{cmd.text}' at {cmd.pos}"

method execute*(cmd: DeleteCommand) =
  cmd.deleted = cmd.buffer.content[cmd.pos..cmd.pos+cmd.deleted.len-1]
  cmd.buffer.content.delete(cmd.pos, cmd.pos + cmd.deleted.len - 1)
  cmd.buffer.cursor = cmd.pos

method undo*(cmd: DeleteCommand) =
  cmd.buffer.content.insert(cmd.deleted, cmd.pos)
  cmd.buffer.cursor = cmd.pos + cmd.deleted.len

type
  CommandHistory* = ref object
    undoStack*: seq[Command]
    redoStack*: seq[Command]
    maxHistory*: int

proc newCommandHistory*(maxHistory = 100): CommandHistory =
  CommandHistory(maxHistory: maxHistory)

proc execute*(history: CommandHistory, cmd: Command) =
  cmd.execute()
  history.undoStack.add(cmd)
  history.redoStack.setLen(0)  # Clear redo after new command
  
  if history.undoStack.len > history.maxHistory:
    history.undoStack.delete(0)

proc undo*(history: CommandHistory): bool =
  if history.undoStack.len == 0: return false
  let cmd = history.undoStack.pop()
  cmd.undo()
  history.redoStack.add(cmd)
  true

proc redo*(history: CommandHistory): bool =
  if history.redoStack.len == 0: return false
  let cmd = history.redoStack.pop()
  cmd.execute()
  history.undoStack.add(cmd)
  true

proc canUndo*(history: CommandHistory): bool = history.undoStack.len > 0
proc canRedo*(history: CommandHistory): bool = history.redoStack.len > 0

when isMainModule:
  let buffer = TextBuffer(content: "Hello")
  let history = newCommandHistory()
  
  echo &"Initial: '{buffer.content}'"
  
  history.execute(InsertCommand(buffer: buffer, pos: 5, text: " World"))
  echo &"After insert: '{buffer.content}'"
  
  history.execute(InsertCommand(buffer: buffer, pos: 0, text: ">>> "))
  echo &"After insert: '{buffer.content}'"
  
  discard history.undo()
  echo &"After undo: '{buffer.content}'"
  
  discard history.undo()
  echo &"After undo: '{buffer.content}'"
  
  discard history.redo()
  echo &"After redo: '{buffer.content}'"
```

---

## สรุป

| Pattern | ใช้เมื่อ |
|---------|---------|
| Functor/Monad | Error propagation, optional values |
| Builder | Complex object construction |
| Event Bus | Decoupled components |
| Command + History | Undo/redo functionality |

---

**Next**: [Part 94 - Advanced Metaprogramming](../advanced/part94_meta.md)
