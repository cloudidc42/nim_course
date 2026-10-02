# Part 62 - Debugging and Profiling Nim Programs

## บทนำ

การ debug และ profile Nim programs อย่างมีประสิทธิภาพช่วยให้ค้นหาปัญหาและปรับปรุงประสิทธิภาพได้อย่างตรงจุด

---

## 1. Built-in Debugging Tools

```nim
# debugging_basics.nim
# เครื่องมือ debug พื้นฐานใน Nim

import std/[strformat, times, os, logging]

# --- 1. dump macro ---

import std/sugar  # for dump

let x = 42
let s = "hello"
let arr = @[1, 2, 3]

dump x      # x = 42
dump s      # s = "hello"
dump arr    # arr = @[1, 2, 3]

# --- 2. debugEcho ---

when defined(debug):
  template debugEcho*(args: varargs[untyped]) =
    echo "[DEBUG] ", args
else:
  template debugEcho*(args: varargs[untyped]) = discard

debugEcho "Only in debug builds: ", x * 2

# --- 3. assert and doAssert ---

proc divide(a, b: int): int =
  doAssert b != 0, &"Division by zero! a={a}, b={b}"
  result = a div b

try:
  echo divide(10, 2)   # OK
  echo divide(10, 0)   # AssertionDefect
except AssertionDefect as e:
  echo "Assertion failed: ", e.msg

# assert is only in debug builds (disabled with --assertions:off)
# doAssert always runs

# --- 4. Backtrace on exceptions ---

proc level3() =
  raise newException(ValueError, "Deep error")

proc level2() = level3()
proc level1() = level2()

try:
  level1()
except ValueError as e:
  echo "Exception: ", e.msg
  # In debug mode, also shows: getStackTrace(e)
  when defined(debug):
    echo getStackTrace(e)

# --- 5. Logging framework ---

let logger = newConsoleLogger(
  levelThreshold = lvlDebug,
  fmtStr = "[$datetime] [$levelid] "
)
addHandler(logger)

debug "Debug message - only in development"
info "Application started"
warn "Low memory warning"
error "Database connection failed"

# File logger for production
let fileLog = newFileLogger(
  "app.log",
  levelThreshold = lvlInfo,
  fmtStr = "[$datetime] [$levelid] "
)
addHandler(fileLog)

info "This goes to both console and file"

# --- 6. repr for complex objects ---

type
  Config* = object
    host*: string
    port*: int
    debug*: bool
    tags*: seq[string]

let cfg = Config(host: "localhost", port: 8080, debug: true, tags: @["web", "api"])
echo repr(cfg)  # Detailed representation
```

---

## 2. GDB/LLDB Debugging

```nim
# gdb_debugging.nim
# Compile with debug symbols for GDB/LLDB

import std/strformat

# Compile command:
# nim c -g --debugger:native myprogram.nim
# Then: gdb ./myprogram
# Or: lldb ./myprogram

# GDB common commands:
# (gdb) run              - Start program
# (gdb) break main       - Breakpoint at main
# (gdb) break file.nim:42  - Breakpoint at line
# (gdb) next             - Step over
# (gdb) step             - Step into
# (gdb) continue         - Continue execution
# (gdb) print x          - Print variable
# (gdb) info locals      - Show local variables
# (gdb) backtrace        - Show call stack
# (gdb) watch x          - Watchpoint on variable

# Nim-specific GDB helpers:
# Use nimgdb (nim's GDB extension) for better symbol resolution
# pip install nimgdb

type
  TreeNode* = ref object
    value*: int
    left*, right*: TreeNode

proc insert*(tree: var TreeNode, val: int) =
  if tree == nil:
    tree = TreeNode(value: val)
    return
  if val < tree.value:
    insert(tree.left, val)
  else:
    insert(tree.right, val)

proc inorder*(tree: TreeNode): seq[int] =
  if tree == nil: return @[]
  result = inorder(tree.left) & @[tree.value] & inorder(tree.right)

var root: TreeNode
for val in [5, 3, 7, 1, 4, 6, 8]:
  root.insert(val)

let sorted = root.inorder()
assert sorted == @[1, 3, 4, 5, 6, 7, 8]
echo &"Sorted: {sorted}"
```

---

## 3. Profiling with nimprof

```nim
# profiling_demo.nim
# Profile Nim programs to find bottlenecks

# Compile with profiling:
# nim c --profiler:on --stackTrace:on myprogram.nim
# Or use perf on Linux:
# nim c -d:release --passC:-pg --passL:-pg myprogram.nim

import std/[times, strformat, sequtils, algorithm, random]

# --- CPU Profiling ---

proc bubbleSort*(arr: var seq[int]) =
  for i in 0..<arr.len:
    for j in 0..<arr.len - i - 1:
      if arr[j] > arr[j+1]:
        swap(arr[j], arr[j+1])

proc mergeSort*(arr: seq[int]): seq[int] =
  if arr.len <= 1: return arr
  let mid = arr.len div 2
  let left = mergeSort(arr[0..<mid])
  let right = mergeSort(arr[mid..^1])
  
  result = @[]
  var i, j = 0
  while i < left.len and j < right.len:
    if left[i] <= right[j]:
      result.add(left[i]); inc i
    else:
      result.add(right[j]); inc j
  result.add(left[i..^1])
  result.add(right[j..^1])

proc benchmark*(name: string, iterations: int, fn: proc()) =
  let start = cpuTime()
  for _ in 0..<iterations:
    fn()
  let elapsed = cpuTime() - start
  let perOp = elapsed / iterations.float * 1000
  echo &"[{name}] {elapsed * 1000:.2f}ms total, {perOp:.4f}ms/op"

when isMainModule:
  randomize()
  
  # Generate test data
  var testData = newSeq[int](1000)
  for i in 0..<1000: testData[i] = rand(10000)
  
  echo "Sorting Benchmark:"
  
  benchmark("BubbleSort", 10) do:
    var copy = testData
    bubbleSort(copy)
  
  benchmark("MergeSort", 100) do:
    let _ = mergeSort(testData)
  
  benchmark("stdlib sort", 1000) do:
    var copy = testData
    copy.sort()
```

---

## 4. Custom Profiler

```nim
# custom_profiler.nim
# Build a simple profiler in Nim

import std/[tables, times, strformat, algorithm, sequtils]

type
  ProfileEntry* = object
    name*: string
    totalTime*: float
    callCount*: int
    minTime*: float
    maxTime*: float
    avgTime*: float

  Profiler* = ref object
    entries*: Table[string, ProfileEntry]
    stack*: seq[tuple[name: string, startTime: float]]
    enabled*: bool

var globalProfiler* = Profiler(
  entries: initTable[string, ProfileEntry](),
  stack: @[],
  enabled: true
)

proc startProfile*(profiler: Profiler, name: string) =
  if not profiler.enabled: return
  profiler.stack.add((name, cpuTime()))

proc endProfile*(profiler: Profiler, name: string) =
  if not profiler.enabled: return
  
  let elapsed = cpuTime()
  
  # Find and remove from stack
  for i in countdown(profiler.stack.len - 1, 0):
    if profiler.stack[i].name == name:
      let duration = elapsed - profiler.stack[i].startTime
      profiler.stack.delete(i)
      
      # Record
      if name notin profiler.entries:
        profiler.entries[name] = ProfileEntry(
          name: name,
          minTime: duration,
          maxTime: duration
        )
      
      var entry = profiler.entries[name]
      entry.totalTime += duration
      inc entry.callCount
      entry.minTime = min(entry.minTime, duration)
      entry.maxTime = max(entry.maxTime, duration)
      entry.avgTime = entry.totalTime / entry.callCount.float
      profiler.entries[name] = entry
      break

template profile*(name: string, body: untyped): untyped =
  ## Profile a code block
  globalProfiler.startProfile(name)
  body
  globalProfiler.endProfile(name)

proc printReport*(profiler: Profiler, topN = 10) =
  echo "\n=== Profiler Report ==="
  echo &"{\"Function\":<40} {\"Calls\":>8} {\"Total(ms)\":>12} {\"Avg(ms)\":>10} {\"Max(ms)\":>10}"
  echo "-".repeat(82)
  
  var entries = toSeq(profiler.entries.values)
  entries.sort do (a, b: ProfileEntry) -> int:
    cmp(b.totalTime, a.totalTime)  # Sort by total time descending
  
  for entry in entries[0..min(topN-1, entries.len-1)]:
    let total = entry.totalTime * 1000
    let avg = entry.avgTime * 1000
    let maxT = entry.maxTime * 1000
    echo &"{entry.name:<40} {entry.callCount:>8} {total:>12.3f} {avg:>10.4f} {maxT:>10.4f}"
  
  echo ""
  var totalTime = 0.0
  for e in profiler.entries.values:
    totalTime += e.totalTime
  echo &"Total profiled time: {totalTime * 1000:.2f}ms"

# --- Usage Example ---

proc processData*(n: int): int =
  profile("processData"):
    var sum = 0
    for i in 0..<n:
      profile("innerLoop"):
        sum += i * i
    result = sum

proc fetchData*(key: string): string =
  profile("fetchData"):
    # Simulate work
    var dummy = 0
    for i in 0..10000:
      dummy += i
    result = &"data_{key}_{dummy}"

when isMainModule:
  echo "Running profiled code..."
  
  for i in 0..99:
    discard processData(1000)
    discard fetchData(&"key{i}")
  
  globalProfiler.printReport()
```

---

## 5. Memory Profiler

```nim
# memory_profiler.nim
# Track memory allocations and detect leaks

import std/[tables, strformat, times, typetraits]

type
  AllocRecord* = object
    size*: int
    file*: string
    line*: int
    timestamp*: float
    typeName*: string

  MemProfiler* = ref object
    allocations*: Table[pointer, AllocRecord]
    totalAllocated*: int
    totalFreed*: int
    peakUsage*: int
    currentUsage*: int

var memProfiler* = MemProfiler(
  allocations: initTable[pointer, AllocRecord]()
)

when defined(memProfile):
  proc trackAlloc*[T](size: int, file: string = "", line: int = 0): ptr T =
    result = cast[ptr T](alloc(size))
    let record = AllocRecord(
      size: size,
      file: file,
      line: line,
      timestamp: cpuTime(),
      typeName: T.name
    )
    memProfiler.allocations[cast[pointer](result)] = record
    memProfiler.totalAllocated += size
    memProfiler.currentUsage += size
    memProfiler.peakUsage = max(memProfiler.peakUsage, memProfiler.currentUsage)
  
  proc trackFree*[T](p: ptr T) =
    let pp = cast[pointer](p)
    if pp in memProfiler.allocations:
      let record = memProfiler.allocations[pp]
      memProfiler.totalFreed += record.size
      memProfiler.currentUsage -= record.size
      memProfiler.allocations.del(pp)
    dealloc(p)
else:
  proc trackAlloc*[T](size: int, file: string = "", line: int = 0): ptr T =
    cast[ptr T](alloc(size))
  
  proc trackFree*[T](p: ptr T) = dealloc(p)

proc printMemReport*(profiler: MemProfiler) =
  echo "\n=== Memory Profiler Report ==="
  echo &"Total allocated: {profiler.totalAllocated div 1024} KB"
  echo &"Total freed: {profiler.totalFreed div 1024} KB"
  echo &"Peak usage: {profiler.peakUsage div 1024} KB"
  echo &"Current usage: {profiler.currentUsage div 1024} KB"
  
  if profiler.allocations.len > 0:
    echo &"\n[!] LEAKS DETECTED: {profiler.allocations.len} allocations not freed:"
    var leaked = 0
    for p, record in profiler.allocations:
      echo &"  {record.typeName}: {record.size} bytes @ {record.file}:{record.line}"
      leaked += record.size
    echo &"Total leaked: {leaked} bytes"
  else:
    echo "[+] No memory leaks detected."

# --- Heap fragmentation checker ---

proc checkHeapFragmentation*(): float =
  ## Estimate heap fragmentation
  ## Returns fragmentation ratio (0 = no fragmentation, 1 = fully fragmented)
  let totalSize = getTotalMem()
  let occupiedSize = getOccupiedMem()
  let freeSize = getFreeMem()
  
  echo &"Total memory: {totalSize div 1024} KB"
  echo &"Occupied: {occupiedSize div 1024} KB"
  echo &"Free: {freeSize div 1024} KB"
  
  if totalSize > 0:
    result = 1.0 - (occupiedSize.float / totalSize.float)
  else:
    result = 0.0

when isMainModule:
  echo "Memory Profiling Demo"
  echo "====================="
  
  let fragmentation = checkHeapFragmentation()
  echo &"Heap fragmentation: {fragmentation * 100:.1f}%"
  
  # Test allocations
  when defined(memProfile):
    var ptrs: seq[ptr int]
    for i in 0..<100:
      ptrs.add(trackAlloc[int](sizeof(int)))
    
    # Free half
    for i in 0..<50:
      trackFree(ptrs[i])
    
    printMemReport(memProfiler)
```

---

## 6. Sanitizers and Static Analysis

```nim
# sanitizer_usage.nim
# Using address sanitizer, thread sanitizer, etc.

# --- Compile with sanitizers ---

# Address Sanitizer (detects buffer overflows, use-after-free):
# nim c --passC:"-fsanitize=address" --passL:"-fsanitize=address" prog.nim

# Thread Sanitizer (detects data races):
# nim c --passC:"-fsanitize=thread" --passL:"-fsanitize=thread" prog.nim

# UBSan (undefined behavior sanitizer):
# nim c --passC:"-fsanitize=undefined" --passL:"-fsanitize=undefined" prog.nim

# --- Valgrind ---
# Compile: nim c -g myprogram.nim
# Run: valgrind --leak-check=full --show-leak-kinds=all ./myprogram

# --- nimcheck (built-in linter) ---
# nim check myprogram.nim

# --- Static analysis hints ---

proc potentialNilDeref(x: ref int) =
  # Nim warns about potential nil dereference
  if x != nil:
    echo x[]
  # else: nothing - safe path

proc unsafeCast() =
  var x: int32 = -1
  # This would be a bug in other languages, but Nim handles it
  let unsigned = cast[uint32](x)  # 4294967295
  echo unsigned

# --- Exception safety ---

proc safeOpen*(path: string): auto =
  ## Exception-safe file handling
  result = open(path)
  # If this raises, no resource leak

proc processFile*(path: string) =
  let f = safeOpen(path)
  defer: f.close()  # Always called, even on exception
  
  for line in f.lines:
    echo line

# --- Undefined behavior detection ---

when defined(danger):
  # In danger mode, bounds checks are off
  proc fastAccess*(arr: seq[int], i: int): int =
    arr[i]  # No bounds check - UB if out of range!
else:
  proc fastAccess*(arr: seq[int], i: int): int =
    if i >= 0 and i < arr.len:
      arr[i]
    else:
      raise newException(IndexDefect, &"Index {i} out of bounds [0, {arr.len})")

when isMainModule:
  let arr = @[1, 2, 3, 4, 5]
  echo fastAccess(arr, 2)  # 3
  try:
    echo fastAccess(arr, 10)  # Error
  except IndexDefect as e:
    echo "Error: ", e.msg
```

---

## สรุป Part 62

| เครื่องมือ | การใช้งาน |
|-----------|----------|
| `dump` macro | พิมพ์เค่าตัวแปรพร้อมชื่อ |
| GDB/LLDB | `-g --debugger:native` |
| nimprof | `--profiler:on --stackTrace:on` |
| Custom profiler | Function-level timing |
| Memory profiler | Allocation tracking + leak detection |
| Sanitizers | ASan, TSan, UBSan, Valgrind |

### Debugging Workflow

1. **Reproduce** — หา minimal test case
2. **Isolate** — binary search โค้ดที่สงสัย
3. **Inspect** — `dump`, breakpoints, print
4. **Fix** — แก้ต้นเหตุที่แท้
5. **Verify** — test + sanitizers
6. **Profile** — measure ก่อน optimize

**Next**: [Part 63 - Full Stack Application in Nim](../web/part63_fullstack.md)
