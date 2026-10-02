# Part 72 - Nim 2.0 Features

## บทนำ

Nim 2.0 เปิดตัวใน 2023 พร้อม features ใหม่สำคัญ:
- ORC เป็น default GC
- `--strictFuncs` enforcement
- Improved error messages
- `raises` tracking improvements
- `openArray` for `var` params
- Better `using` statement
- `sink` parameters และ move semantics ที่ดีขึ้น
- View types (`lent`, `openArray`)
- Nim scripting improvements

---

## 1. ORC เป็น Default GC

```nim
# orc_features.nim
# ORC (Ownership Reference Counting) — default ใน Nim 2.0

# Compile:
# nim c myapp.nim          # ORC (default ใน 2.0)
# nim c --mm:refc myapp.nim  # Old RC (Nim 1.x default)
# nim c --mm:arc myapp.nim   # ARC (no cycle detection)
# nim c --mm:boehm myapp.nim # Boehm GC

import std/[strformat, times]

type
  Node* = ref object
    value*: int
    next*: Node     # Potential cycle: a -> b -> a
    prev*: Node     # ORC handles this automatically!

# ใน Nim 1.x ด้วย refc: cycle a <-> b = memory leak
# ใน Nim 2.0 ด้วย ORC: automatic cycle detection & collection
proc createCycle() =
  let a = Node(value: 1)
  let b = Node(value: 2)
  a.next = b
  b.prev = a  # Cycle! ORC จัดการได้

# ORC move semantics
proc expensive(s: sink string): string =
  ## sink parameter — caller's string is moved (not copied)
  s & " processed"

proc callExpensive() =
  var data = "hello world"
  let result = expensive(data)  # data is moved, not copied
  # data is now invalid after move (don't use it)
  echo result

# lent — borrow without copying
proc readonlyRef(s: lent string): int =
  ## lent = read-only reference (no copy, no ownership)
  s.len

proc callLent() =
  let s = "hello"
  let n = readonlyRef(s)  # s is borrowed, not copied
  echo &"Length: {n}"

# ORC tracking control
proc forceCollect() =
  ## Force ORC cycle collection
  GC_fullCollect()

proc getMemStats() =
  ## Get memory usage info
  let stats = GC_getStatistics()
  echo stats
```

---

## 2. Strict Functions

```nim
# strict_funcs.nim
# --strictFuncs: enforce functional purity
# Compile: nim c --strictFuncs myapp.nim

{.experimental: "strictFuncs".}

# ✅ Pure function — ไม่ modify state
func add(a, b: int): int = a + b

# ✅ proc ทำได้ทุกอย่าง (no restriction)
proc sideEffect(x: var int) = inc x

# ❌ Error: func tries to call proc with side effects
# func bad(x: var int): int =
#   sideEffect(x)  # Error: func cannot call proc
#   x

# ✅ func ใช้ immutable params ได้
func transform(s: string): string =
  s.toUpperAscii()

# func ไม่สามารถ:
# 1. เรียก proc ที่มี side effects
# 2. เข้าถึง global mutable state
# 3. Modify reference type parameters

var globalCounter = 0  # Mutable global

# ❌ func ไม่สามารถเข้าถึง mutable global ได้
# func badFunc(): int =
#   globalCounter  # Error: cannot access mutable global in func

# ✅ แต่ proc ทำได้
proc getCounter(): int = globalCounter

# Template และ macro ยังใช้ได้ปกติ
template doubleIt(x: untyped): untyped = x * 2

func compute(n: int): int =
  doubleIt(n) + add(n, 1)
```

---

## 3. Improved Error Messages

```nim
# nim2_errors.nim
# Nim 2.0 มี error messages ที่ดีขึ้นมาก

# ตัวอย่าง error ใน Nim 1.x vs 2.0:

# Nim 1.x: "Error: type mismatch: got <int> but expected <string>"
# Nim 2.0: Detailed message with suggestion and context

# Error location now shows exact column
# Type mismatch shows full type hierarchy
# "Did you mean...?" suggestions

# New: --hint:all แสดง hints ทั้งหมด
# New: --warning:all แสดง warnings ทั้งหมด

# Improved: underline exactly the problematic code

# ตัวอย่าง: Nim 2.0 บอก undefined variable ได้ดีขึ้น
proc example() =
  let x = 42
  let y = x + 1  # ✅
  # let z = w + 1  # Error: undeclared identifier 'w' (with suggestions)

# New in 2.0: "hint: 'x' is unused"
proc unusedVarExample() =
  let x = 42  # hint: 'x' declared but not used
  discard 1
```

---

## 4. View Types และ openArray Improvements

```nim
# view_types.nim
# View types: safe references without ownership

import std/[strformat]

type
  # View into existing data (no copy, no ownership)
  DataView*[T] = openArray[T]  # Nim 2.0 openArray improvements

# openArray ใน 2.0 รองรับ var parameters ดีขึ้น
proc modifyInPlace*(data: var openArray[int], multiplier: int) =
  for i in 0..<data.len:
    data[i] *= multiplier

proc callModify() =
  var arr = [1, 2, 3, 4, 5]
  modifyInPlace(arr, 2)  # Works with array
  
  var s = @[1, 2, 3]
  modifyInPlace(s, 3)    # Works with seq

# lent (borrowed reference) — ใหม่ใน Nim 2.0
type
  LargeObject* = object
    data*: seq[int]

proc processLent*(obj: lent LargeObject): int =
  ## lent = no copy, read-only borrow
  obj.data.len

proc callLent() =
  let big = LargeObject(data: @[1, 2, 3, 4, 5])
  echo processLent(big)  # No copy of big

# Nim 2.0: improved var T borrow
proc swap*[T](a, b: var T) =
  let temp = move(a)
  a = move(b)
  b = move(temp)  # Explicit moves for efficiency

# openArray slicing (Nim 2.0)
proc sliceExample() =
  let arr = [1, 2, 3, 4, 5, 6, 7, 8]
  let view = arr.toOpenArray(2, 5)  # [3, 4, 5, 6]
  echo view
```

---

## 5. Improved `using` Statement

```nim
# using_stmt.nim
# 'using' ช่วยลด repetition ใน similar procs

type
  Connection* = ref object
    host*: string
    port*: int
    connected*: bool
  
  Request* = object
    path*: string
    method*: string

# ใช้ 'using' เพื่อ declare common parameters
using
  conn: Connection
  req: Request

# ไม่ต้องพิมพ์ types ซ้ำ
proc connect*(conn) =
  conn.connected = true

proc disconnect*(conn) =
  conn.connected = false

proc sendRequest*(conn; req): string =
  if not conn.connected:
    raise newException(IOError, "Not connected")
  &"{req.method} {req.path} -> {conn.host}:{conn.port}"

proc handleGet*(conn; req) =
  echo sendRequest(conn, req)

# Nim 2.0: using ใน object procs
type
  Stack*[T] = object
    items*: seq[T]

using
  stack: Stack[int]

proc push*(stack: var Stack[int]; item: int) =
  stack.items.add(item)

proc pop*(stack: var Stack[int]): int =
  stack.items.pop()

proc peek*(stack): int =
  stack.items[^1]
```

---

## 6. Move Semantics (sink)

```nim
# move_semantics.nim
# sink parameters สำหรับ efficient value passing

import std/[strformat, algorithm]

type
  Buffer* = object
    data*: seq[byte]
    name*: string

proc newBuffer*(name: string, size: int): Buffer =
  Buffer(name: name, data: newSeq[byte](size))

# sink = caller gives up ownership, no copy
proc consumeBuffer*(buf: sink Buffer): string =
  let result = &"Processed {buf.data.len} bytes from '{buf.name}'"
  # buf.data is moved here, not copied
  result

# ใช้ move() explicitly
proc transferData*(source: var Buffer, dest: var Buffer) =
  dest.data = move(source.data)  # Zero-cost transfer
  dest.name = move(source.name)
  # source is now empty

# sink in constructors
type
  Processor* = ref object
    buffer*: Buffer

proc newProcessor*(buf: sink Buffer): Processor =
  Processor(buffer: move(buf))  # Move into processor, no copy

# Return value optimization (RVO) — automatic in Nim 2.0
proc createLargeBuffer*(size: int): Buffer =
  # Nim 2.0 uses NRVO: no copy on return
  result = Buffer(name: "created", data: newSeq[byte](size))
  for i in 0..<size:
    result.data[i] = byte(i mod 256)

proc demonstrateMoves() =
  var buf1 = newBuffer("buf1", 1000)
  var buf2 = Buffer(name: "buf2")
  
  # Move buf1's data to buf2
  transferData(buf1, buf2)
  echo &"buf1 size: {buf1.data.len}"   # 0 — moved
  echo &"buf2 size: {buf2.data.len}"   # 1000
  
  # sink: consume and move
  let msg = consumeBuffer(move(buf2))
  echo msg
  echo &"buf2 size after consume: {buf2.data.len}"  # 0 — consumed
```

---

## 7. Concepts ที่ดีขึ้น

```nim
# concepts_v2.nim
# Nim 2.0 concepts — ชัดเจนและ composable มากขึ้น

type
  # Basic concept
  Printable* = concept x
    $x is string

  # Arithmetic concept  
  Numeric* = concept x
    x + x is typeof(x)
    x - x is typeof(x)
    x * x is typeof(x)
    x / x is typeof(x)

  # Container concept
  Container*[T] = concept c
    c.len is int
    c.add(T)
    c[int] is T
    
  # Ordered + comparable
  Ordered* = concept x
    x < x is bool
    x <= x is bool
    x > x is bool
    x >= x is bool

  # Composable concepts
  SortableContainer*[T: Ordered] = Container[T]

proc printAll*[T: Printable](items: openArray[T]) =
  for item in items:
    echo $item

proc sum*[T: Numeric](items: openArray[T]): T =
  result = default(T)
  for item in items:
    result = result + item

proc sortedVersion*[T: Ordered](items: seq[T]): seq[T] =
  result = items
  result.sort()

# Concept with required methods
type
  Serializer* = concept s
    s.serialize(string) is string
    s.deserialize(string) is string

type
  JsonSerializer* = object

proc serialize*(s: JsonSerializer, data: string): string =
  "{\"data\": \"" & data & "\"}"

proc deserialize*(s: JsonSerializer, json: string): string =
  # Simplified
  json[10..^3]

proc roundTrip*[S: Serializer](serializer: S, data: string): bool =
  let encoded = serializer.serialize(data)
  let decoded = serializer.deserialize(encoded)
  decoded == data

# Nim 2.0: concept inheritance
type
  ReadableStream* = concept s
    s.read(int) is string
    s.atEnd() is bool

  WritableStream* = concept s
    s.write(string)
    s.flush()

  ReadWriteStream* = ReadableStream and WritableStream

proc copyStream*[R: ReadableStream, W: WritableStream](
    src: R, dst: W, bufSize = 4096) =
  while not src.atEnd():
    let chunk = src.read(bufSize)
    dst.write(chunk)
  dst.flush()
```

---

## 8. Error Handling สำหรับ Nim 2.0

```nim
# error_handling_v2.nim
# Nim 2.0 มี raises tracking ที่ดีขึ้น

import std/[options, strformat]

# raises pragma — declare exactly what can be raised
proc parsePort*(s: string): int {.raises: [ValueError].} =
  let port = parseInt(s)
  if port < 1 or port > 65535:
    raise newException(ValueError, &"Invalid port: {port}")
  port

# noinit — skip zero-initialization for performance
proc createRawBuffer*(size: int): seq[byte] =
  result = newSeqUninitialized[byte](size)
  # result is NOT zero-initialized — faster but undefined content

# Custom exception hierarchy
type
  AppError* = object of CatchableError
  DatabaseError* = object of AppError
    query*: string
  NetworkError* = object of AppError
    url*: string
    statusCode*: int

proc connect*(url: string) {.raises: [NetworkError, IOError].} =
  if not url.startsWith("http"):
    raise NetworkError(msg: "Invalid URL", url: url, statusCode: 0)

# try/except with multiple exceptions
proc safeConnect*(url: string): bool =
  try:
    connect(url)
    true
  except NetworkError as e:
    echo &"Network error: {e.msg} (url={e.url})"
    false
  except IOError as e:
    echo &"IO error: {e.msg}"
    false

# Effect system (raises tracking)
proc withErrorTracking() {.raises: [].} =
  ## This proc guarantees it never raises
  try:
    discard safeConnect("http://example.com")
  except CatchableError:
    discard  # Catch everything

# Nim 2.0: improved tryExcept macro pattern
template tryGet*[T](expr: T, default: T): T =
  try:
    expr
  except CatchableError:
    default

proc safeParsePort*(s: string): int =
  tryGet(parsePort(s), 8080)
```

---

## 9. Nim 2.0 Standard Library Updates

```nim
# stdlib_updates.nim
# อัพเดท std library ใน Nim 2.0

import std/[
  syncio,       # Synchronous I/O (replaces some system.nim)
  objectdollar, # $() สำหรับ object types
  appdirs,      # Application directories
  paths,        # Path handling (replaces os path utils)
  cmdparse,     # Command-line parsing
  sysrand,      # Secure random numbers
  isolation     # Thread-safe value passing
]

# New: std/paths — type-safe path handling
proc pathExample() =
  let home = Path("/home/user")
  let config = home / "config"  # Type-safe join
  let file = config / "app.toml"
  echo file  # /home/user/config/app.toml
  
  let (dir, name, ext) = file.splitFile()
  echo &"dir={dir} name={name} ext={ext}"

# New: std/sysrand — cryptographically secure random
proc secureTokenExample() =
  var bytes: array[32, byte]
  sysrand.urandom(bytes)  # CSPRNG
  # Use for security tokens, not math.rand

# New: std/isolation — thread-safe value passing
# Isolation[T] = value that can be safely moved between threads
proc isolationExample() =
  var data: seq[int] = @[1, 2, 3, 4, 5]
  let isolated = isolate(data)  # Create isolated copy
  
  # Can now send to another thread safely
  # spawn processIsolated(isolated)

# Updated: std/asyncdispatch improvements
import std/asyncdispatch

proc asyncExample() {.async.} =
  # Nim 2.0: better async/await type inference
  let result = await sleepAsync(100)
  
  # Better error handling in async
  proc mightFail(): Future[int] {.async.} =
    await sleepAsync(10)
    return 42
  
  try:
    let val = await mightFail()
    echo &"Got: {val}"
  except CatchableError as e:
    echo &"Error: {e.msg}"

# Nim 2.0: strict mode
{.experimental: "strictDefs".}
# All variables must be initialized before use
proc strictExample() =
  var x: int  # ✅ Nim knows x = 0 by default
  x = 5       # ✅ 
  echo x
```

---

## สรุป Nim 2.0 Changes

| Feature | Description | Impact |
|---------|-------------|--------|
| ORC default | Automatic cycle detection | Less manual GC tuning |
| strictFuncs | Purity enforcement | Better code quality |
| sink/move | Explicit ownership transfer | Zero-copy performance |
| lent | Read-only borrow | No-copy references |
| openArray var | Mutable slice parameters | API flexibility |
| std/paths | Type-safe paths | Fewer path bugs |
| std/sysrand | CSPRNG | Security |
| Improved errors | Better messages | Faster debugging |

---

**Next**: [Part 73 - Language Server Protocol & IDE Integration](../advanced/part73_lsp.md)
