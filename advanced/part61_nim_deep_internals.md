# Part 61 - Nim Compiler Internals Deep Dive

## บทนำ

การเข้าใจการทำงานภายใน Nim compiler ช่วยให้เขียนโค้ดที่มีประสิทธิภาพสูงและเข้าใจวิธีที่ macros ทำงาน

---

## 1. Compiler Architecture Overview

```
Nim Compiler Pipeline:

Source Code (.nim)
       ↓
   Lexer (scanner.nim)
       ↓ Token stream
   Parser (parser.nim)
       ↓ AST (PNode)
   Semantic Analysis (sem.nim, semexprs.nim, semstmts.nim)
       ↓ Typed AST
   Backend Selection
   ├── C Backend (cgen.nim)    → .c files → gcc/clang
   ├── C++ Backend            → .cpp files
   ├── JS Backend (jsgen.nim) → .js files
   ├── LLVM Backend           → .bc/.ll files
   └── NimVM (vm.nim)         → Interpreted (compile-time)
```

---

## 2. AST Node Types

```nim
# Understanding Nim's AST
# The AST is represented by PNode (pointer to TNode)

import macros

# AST Node kinds (from compiler/ast.nim)
# nkNone, nkEmpty, nkIdent, nkSym
# nkIntLit, nkFloatLit, nkStrLit
# nkCallKinds: nkCall, nkCommand, nkPrefix, nkInfix
# nkStmtList, nkIfStmt, nkForStmt, nkWhileStmt
# nkProcDef, nkFuncDef, nkMethodDef
# nkTypeDef, nkTypeSection
# nkDotExpr, nkBracketExpr
# nkAsgn, nkVarSection, nkLetSection, nkConstSection

# --- Exploring AST with dumpTree ---

macro exploreAST*(x: untyped): untyped =
  ## Print the AST of any expression/statement
  echo "AST of:", x.repr
  echo x.treeRepr
  result = x

# Examples
exploreAST:
  let x = 42
  echo x * 2

exploreAST:
  proc add(a, b: int): int = a + b

# Expected output:
# nkStmtList
#   nkLetSection
#     nkIdentDefs
#       nkIdent "x"
#       nkEmpty
#       nkIntLit 42
#   nkCommand
#     nkIdent "echo"
#     nkInfix
#       nkIdent "*"
#       nkIdent "x"
#       nkIntLit 2

# --- Building AST Manually ---

macro buildIfExpr*(cond, thenExpr, elseExpr: untyped): untyped =
  ## Build if expression from parts
  result = newNimNode(nnkIfExpr)
  
  let branch = newNimNode(nnkElifExpr)
  branch.add(cond)
  branch.add(thenExpr)
  result.add(branch)
  
  let elseBranch = newNimNode(nnkElseExpr)
  elseBranch.add(elseExpr)
  result.add(elseBranch)

let val = buildIfExpr(true, 42, 0)
assert val == 42

# --- Macro Hygiene ---

macro hygieneDemo*(x: typed): untyped =
  ## Demonstrates hygienic macros - generated symbols are unique
  # genSym creates a unique symbol to avoid name collisions
  let tempVar = genSym(nskVar, "temp")  # Creates unique _temp_XXX
  
  result = quote do:
    var `tempVar` = `x` * 2  # Won't conflict with user's 'temp' var
    echo `tempVar`

var temp = 100  # User's variable
hygieneDemo(21)  # Generated 'temp' doesn't conflict

# --- The quote do: macro in depth ---

macro timing*(body: untyped): untyped =
  ## Add timing to any code block
  let startVar = genSym(nskVar, "startTime")
  let endVar = genSym(nskVar, "endTime")
  
  result = quote do:
    let `startVar` = cpuTime()
    `body`
    let `endVar` = cpuTime()
    echo "Elapsed: ", (`endVar` - `startVar`) * 1000, "ms"

timing:
  var sum = 0
  for i in 0..1000000: sum += i
  echo "Sum: ", sum
```

---

## 3. Compile-Time Computation

```nim
# compile_time.nim
# Leverage Nim's compile-time execution (NimVM)

import macros, strformat, tables

# --- Static: runs at compile time ---

static:
  echo "This runs at COMPILE TIME"
  let nums = [1, 2, 3, 4, 5]
  var sum = 0
  for n in nums: sum += n
  echo "Compile-time sum: ", sum

# --- const: computed at compile time ---

proc fibonacci(n: int): int {.compileTime.} =
  if n <= 1: return n
  result = fibonacci(n-1) + fibonacci(n-2)

const
  FIB10 = fibonacci(10)  # Computed at compile time!
  FIB20 = fibonacci(20)

echo &"Fib(10) = {FIB10}, Fib(20) = {FIB20}"  # Runtime, but values pre-computed

# --- Compile-time type generation ---

macro generateEnum*(name: untyped, values: varargs[untyped]): untyped =
  ## Generate an enum type at compile time
  result = newNimNode(nnkTypeSection)
  
  let typeDef = newNimNode(nnkTypeDef)
  typeDef.add(name)
  typeDef.add(newNimNode(nnkEmpty))
  
  let enumTy = newNimNode(nnkEnumTy)
  enumTy.add(newNimNode(nnkEmpty))  # No base type
  
  for v in values:
    enumTy.add(v)
  
  typeDef.add(enumTy)
  result.add(typeDef)

generateEnum(Color, Red, Green, Blue, Alpha)
generateEnum(Direction, North, South, East, West)

let c: Color = Red
let d: Direction = North
echo &"Color: {c}, Direction: {d}"

# --- Static dispatch tables at compile time ---

type
  HandlerFn* = proc(x: int): int

macro buildDispatchTable*(pairs: varargs[untyped]): untyped =
  ## Build a lookup table at compile time
  result = newNimNode(nnkTableConstr)
  
  for pair in pairs:
    if pair.kind != nnkTupleConstr or pair.len != 2:
      error "Expected (key, fn) pairs"
    result.add(newColonExpr(pair[0], pair[1]))

proc double(x: int): int = x * 2
proc triple(x: int): int = x * 3
proc square(x: int): int = x * x

const ops = buildDispatchTable(
  ("double", double),
  ("triple", triple),
  ("square", square)
)

# --- compileTime pragmas and when ---

when defined(debug):
  proc debugLog*(msg: string) = echo "[DEBUG] ", msg
else:
  proc debugLog*(msg: string) {.inline.} = discard  # No-op in release

# Compile with: nim c -d:debug program.nim
debugLog("This only prints in debug builds")

# --- isMainModule for libraries ---

when isMainModule:
  echo "Running tests..."
  assert fibonacci(10) == 55
  assert fibonacci(0) == 0
  assert fibonacci(1) == 1
  echo "All compile-time tests passed!"
```

---

## 4. Custom Pragmas

```nim
# custom_pragmas.nim
# Create and use custom pragmas in Nim

import macros, tables, strformat

# --- Define custom pragma ---

pragma serializable
pragma deprecated(msg: string)
pragma route(path: string, methods: seq[string] = @["GET"])
pragma validate(min, max: int)

# --- Pragma processing with hasCustomPragma ---

type
  UserDTO* = object
    name*: string
    email*: string
    age* {.validate(0, 150).}: int
    role*: string

proc validateObject*[T](obj: T): seq[string] =
  ## Validate object fields using custom pragmas
  result = @[]
  
  for field, value in obj.fieldPairs:
    when value.hasCustomPragma(validate):
      let pragma = value.getCustomPragmaVal(validate)
      when value is int:
        if value < pragma.min:
          result.add(&"Field '{field}' value {value} < min {pragma.min}")
        elif value > pragma.max:
          result.add(&"Field '{field}' value {value} > max {pragma.max}")

let user = UserDTO(name: "Alice", email: "alice@example.com", age: 25, role: "admin")
let errors = validateObject(user)
if errors.len == 0:
  echo "Validation passed!"
else:
  for e in errors: echo "Error: ", e

# --- Route annotation system ---

type
  RouteInfo* = object
    path*: string
    methods*: seq[string]
    handler*: string

var registeredRoutes: seq[RouteInfo] = @[]

macro registerRoutes*(body: typed): untyped =
  ## Scan for route-annotated procedures
  result = body
  
  for stmt in body:
    if stmt.kind in {nnkProcDef, nnkFuncDef}:
      let procName = $stmt[0]
      # In real code, would inspect pragmas here
      # This is simplified
      discard

# --- Effect system pragmas ---

proc pureFunction(x: int): int {.noSideEffect.} =
  ## This proc promises no side effects
  x * x

proc withGC(): string {.raises: [].} =
  ## This proc promises not to raise exceptions
  result = "hello"

proc dangerous() {.raises: [IOError, ValueError].} =
  ## Document what exceptions can be raised
  raise newException(IOError, "test")

# --- locks pragma for thread safety ---

import locks

var globalLock: Lock
initLock(globalLock)

var sharedCounter = 0

proc safeIncrement() {.locks: [globalLock].} =
  ## Acquire lock before accessing shared state
  acquire(globalLock)
  defer: release(globalLock)
  inc sharedCounter

# --- Tags for effect tracking ---

type
  DbEffect* = object of RootEffect
  NetworkEffect* = object of RootEffect

proc queryDatabase(sql: string): string {.tags: [DbEffect].} =
  result = "result"

proc fetchUrl(url: string): string {.tags: [NetworkEffect].} =
  result = "content"

# Compile error: mixing effects without explicit allow
# proc mixedEffects() {.tags: [DbEffect].} =
#   discard fetchUrl("http://example.com")  # ERROR: NetworkEffect not allowed

proc allEffects() {.tags: [DbEffect, NetworkEffect].} =
  discard queryDatabase("SELECT 1")
  discard fetchUrl("http://example.com")

when isMainModule:
  safeIncrement()
  safeIncrement()
  echo &"Counter: {sharedCounter}"  # 2
```

---

## 5. Nim's Type System Internals

```nim
# type_system.nim
# Understanding and leveraging Nim's type system

import macros, typetraits

# --- typetraits for compile-time type info ---

proc printTypeInfo*[T](val: T) =
  echo &"Type: {T.name}"
  echo &"  Size: {sizeof(T)} bytes"
  echo &"  Align: {alignof(T)} bytes"
  
  when T is SomeInteger:
    echo "  Category: Integer"
  when T is SomeFloat:
    echo "  Category: Float"
  when T is string:
    echo "  Category: String"
  when T is object:
    echo "  Category: Object"
  when T is seq:
    echo "  Category: Sequence"

printTypeInfo(42)
printTypeInfo(3.14)
printTypeInfo("hello")

# --- Distinct types ---

type
  Meters* = distinct float64
  Kilograms* = distinct float64
  Seconds* = distinct float64

# Type-safe operations
proc `+`*(a, b: Meters): Meters = Meters(float64(a) + float64(b))
proc `*`*(a: Kilograms, b: Meters): distinct float64 = float64(a) * float64(b)

let distance = 5.0.Meters + 3.0.Meters  # 8.0 Meters
let mass = 70.0.Kilograms
# let error = distance + mass  # COMPILE ERROR: type mismatch

# --- phantom types for state machines ---

type
  Locked* = object
  Unlocked* = object
  Door*[State] = object
    id*: int

proc openDoor*(d: Door[Unlocked]): Door[Unlocked] =
  echo &"Opening door {d.id}"
  result = d

proc lockDoor*(d: Door[Unlocked]): Door[Locked] =
  echo &"Locking door {d.id}"
  result = Door[Locked](id: d.id)

proc unlockDoor*(d: Door[Locked]): Door[Unlocked] =
  echo &"Unlocking door {d.id}"
  result = Door[Unlocked](id: d.id)

var unlockedDoor = Door[Unlocked](id: 1)
let lockedDoor = unlockedDoor.lockDoor()
# openDoor(lockedDoor)  # COMPILE ERROR: cannot open a locked door
let reopened = lockedDoor.unlockDoor()
discard reopened.openDoor()

# --- Concepts (type constraints) ---

type
  Printable* = concept x
    $x is string
  
  Container* = concept c, type T
    c.len is int
    c[0] is T
    for item in c:
      item is T

proc printAll*[T: Container](container: T) =
  for item in container:
    echo item

printAll(@[1, 2, 3])
printAll(["a", "b", "c"])

# --- Variance annotations ---

type
  Covariant*[+T] = object  # Can upcast
    value*: T
  
  Contravariant*[-T] = object  # Can downcast  
    handler*: proc(x: T)

# --- Static types for zero-cost abstractions ---

type
  Matrix*[M, N: static int, T] = object
    data*: array[M * N, T]

proc `[]`*[M, N: static int, T](m: Matrix[M, N, T], row, col: int): T =
  m.data[row * N + col]

proc `[]=`*[M, N: static int, T](m: var Matrix[M, N, T], row, col: int, val: T) =
  m.data[row * N + col] = val

proc multiply*[M, N, K: static int, T: SomeFloat](
    a: Matrix[M, K, T], b: Matrix[K, N, T]): Matrix[M, N, T] =
  ## Type-safe matrix multiplication - dimensions checked at compile time!
  for i in 0..<M:
    for j in 0..<N:
      var sum: T = 0
      for k in 0..<K:
        sum += a[i, k] * b[k, j]
      result[i, j] = sum

var a: Matrix[2, 3, float]
var b: Matrix[3, 2, float]
let c = a.multiply(b)  # Result is Matrix[2, 2, float]
# a.multiply(a)  # COMPILE ERROR: dimension mismatch (2x3 * 2x3 is invalid)

when isMainModule:
  echo "Type system demo complete!"
  echo &"Distance: {float64(distance)} meters"
```

---

## 6. Nim's Memory Model

```nim
# memory_model.nim
# Understanding Nim's memory management strategies

import std/[strformat, times]

# --- 1. Garbage Collector (default: ORC) ---
# ORC (Ownership, Ref Counting, Cycle Collector)
# Combines reference counting with cycle detection

type
  Node* = ref object
    value*: int
    next*: Node  # Can create cycles

proc createCycle() =
  var a = Node(value: 1)
  var b = Node(value: 2)
  a.next = b
  b.next = a  # Cycle!
  # ORC handles this via cycle detection - no memory leak

# --- 2. Manual memory management with --mm:none ---

# In --mm:none mode, you manage memory manually:
# alloc/dealloc or allocShared/deallocShared for threads

proc manualAlloc*() =
  let size = 1024
  let p = alloc(size)       # Allocate memory
  defer: dealloc(p)         # Free when done
  
  # Use the memory
  let arr = cast[ptr array[256, int32]](p)
  arr[0] = 42
  echo &"Value: {arr[0]}"

# --- 3. Object pools for performance ---

type
  PooledObject* = object
    active*: bool
    data*: array[64, byte]

  ObjectPool*[T] = object
    storage*: seq[T]
    freeList*: seq[int]

proc newObjectPool*[T](capacity: int): ObjectPool[T] =
  result.storage = newSeq[T](capacity)
  result.freeList = newSeq[int](capacity)
  for i in 0..<capacity:
    result.freeList[i] = capacity - 1 - i  # Fill in reverse

proc acquire*[T](pool: var ObjectPool[T]): ptr T =
  if pool.freeList.len == 0:
    raise newException(ResourceExhaustedError, "Pool exhausted")
  let idx = pool.freeList.pop()
  result = addr pool.storage[idx]

proc release*[T](pool: var ObjectPool[T], obj: ptr T) =
  # Find the index
  let idx = int((cast[uint](obj) - cast[uint](addr pool.storage[0])) div sizeof(T).uint)
  pool.freeList.add(idx)

# --- 4. Stack vs heap allocation ---

proc stackVsHeap() =
  # Stack allocated (fast, automatic cleanup)
  var stackArr: array[1000, int64]  # 8KB on stack
  stackArr[0] = 42
  
  # Heap allocated (slower, manual/GC managed)
  var heapArr = newSeq[int64](1_000_000)  # 8MB on heap
  heapArr[0] = 42
  
  echo &"Stack value: {stackArr[0]}"
  echo &"Heap value: {heapArr[0]}"

# --- 5. Unsafe memory operations ---

proc unsafeOps() =
  var x: int32 = 0x12345678
  
  # View int32 as array of bytes (type punning)
  let bytes = cast[ptr array[4, byte]](addr x)
  echo &"Bytes: {bytes[0]:02X} {bytes[1]:02X} {bytes[2]:02X} {bytes[3]:02X}"
  
  # Pointer arithmetic
  var arr = @[1, 2, 3, 4, 5]
  let p = addr arr[0]
  let p2 = cast[ptr int](cast[uint](p) + sizeof(int).uint)
  echo &"p2 points to: {p2[]}"  # Should be 2

# --- 6. Shared memory between threads ---

import std/atomics

var counter: Atomic[int]
counter.store(0)
counter.fetchAdd(1)
counter.fetchAdd(1)
echo &"Atomic counter: {counter.load()}"

# --- 7. --mm:arc vs --mm:orc ---
# ARC: Reference counting only (no cycle detection)
# ORC: ARC + cycle detection (default)
# Use ARC when you know there are no cycles (faster)
# Use ORC for general-purpose code

# Compile with:
# nim c --mm:arc myprogram.nim   # ARC mode
# nim c --mm:orc myprogram.nim   # ORC mode (default)
# nim c --mm:none myprogram.nim  # Manual memory management

when isMainModule:
  manualAlloc()
  stackVsHeap()
  unsafeOps()
  createCycle()  # No leak with ORC!
```

---

## สรุป Part 61

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|------------|
| AST Types | PNode structure, tree representation |
| Compile-time execution | `static`, `const`, `compileTime` procs |
| Custom pragmas | `hasCustomPragma`, `getCustomPragmaVal` |
| Type system | Distinct types, concepts, phantom types, static generics |
| Memory model | ORC/ARC, pools, atomics |

### Performance Tips

1. ใช้ `--mm:arc` เมื่อไม่มี cycles — เร็วกว่า ORC 10-20%
2. `const` แทน `let` สำหรับค่าคงที่รู้ตอนคอมไพล์
3. `{.inline.}` สำหรับฟังก์ชันเล็กที่เรียกบ่อย
4. Object pools สำหรับ high-frequency allocations

**Next**: [Part 62 - Debugging and Profiling Nim Programs](part62_debugging.md)
