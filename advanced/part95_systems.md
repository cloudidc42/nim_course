# Part 95 - Systems Programming

## บทนำ

Systems programming ระดับต่ำใน Nim:
- Memory management โดยตรง
- OS primitives
- Hardware interaction
- Performance optimization

---

## 1. Manual Memory Management

```nim
# memory_management.nim
# ควบคุม memory โดยตรง

import std/[strformat, os, osproc]
import std/atomics

# Custom allocator interface
type
  AllocFn* = proc(size: int): pointer {.noconv.}
  FreeFn* = proc(p: pointer) {.noconv.}
  ReallocFn* = proc(p: pointer, newSize: int): pointer {.noconv.}

  Allocator* = object
    alloc*: AllocFn
    free*: FreeFn
    realloc*: ReallocFn
    name*: string

# Arena allocator — fast bump allocator, bulk free only
type
  ArenaBlock = object
    data: UncheckedArray[byte]
  
  Arena* = object
    blocks: seq[ptr ArenaBlock]
    blockSize: int
    currentBlock: int
    offset: int
    totalAllocated*: int

proc newArena*(blockSize = 64 * 1024): Arena =
  Arena(blockSize: blockSize, blocks: @[], currentBlock: -1)

proc addBlock(arena: var Arena) =
  let blk = cast[ptr ArenaBlock](alloc(arena.blockSize))
  arena.blocks.add(blk)
  inc arena.currentBlock
  arena.offset = 0

proc arenaAlloc*(arena: var Arena, size: int): pointer =
  # Align to 8 bytes
  let alignedSize = (size + 7) and (not 7)
  
  if arena.currentBlock == -1 or arena.offset + alignedSize > arena.blockSize:
    arena.addBlock()
  
  result = addr arena.blocks[arena.currentBlock].data[arena.offset]
  inc arena.offset, alignedSize
  inc arena.totalAllocated, alignedSize

proc reset*(arena: var Arena) =
  arena.currentBlock = if arena.blocks.len > 0: 0 else: -1
  arena.offset = 0
  arena.totalAllocated = 0

proc destroy*(arena: var Arena) =
  for blk in arena.blocks: dealloc(blk)
  arena.blocks = @[]
  arena.currentBlock = -1

# Pool allocator — fixed-size objects
type
  PoolBlock[T] = object
    next: ptr PoolBlock[T]
    value: T
  
  Pool*[T] = object
    freeList: ptr PoolBlock[T]
    blocks: seq[pointer]
    blockCapacity: int
    allocated*: int

proc newPool*[T](blockCapacity = 64): Pool[T] =
  Pool[T](blockCapacity: blockCapacity)

proc allocChunk[T](pool: var Pool[T]) =
  let chunk = cast[ptr UncheckedArray[PoolBlock[T]]](
    alloc(pool.blockCapacity * sizeof(PoolBlock[T]))
  )
  pool.blocks.add(chunk)
  
  # Link free list
  for i in 0..<pool.blockCapacity - 1:
    chunk[i].next = addr chunk[i+1]
  chunk[pool.blockCapacity - 1].next = nil
  pool.freeList = addr chunk[0]

proc acquire*[T](pool: var Pool[T]): ptr T =
  if pool.freeList == nil: pool.allocChunk()
  
  let node = pool.freeList
  pool.freeList = node.next
  inc pool.allocated
  result = addr node.value

proc release*[T](pool: var Pool[T], p: ptr T) =
  let node = cast[ptr PoolBlock[T]](
    cast[int](p) - offsetOf(PoolBlock[T], value)
  )
  node.next = pool.freeList
  pool.freeList = node
  dec pool.allocated

proc destroy*[T](pool: var Pool[T]) =
  for blk in pool.blocks: dealloc(blk)
  pool.blocks = @[]
  pool.freeList = nil

when isMainModule:
  var arena = newArena(64 * 1024)
  defer: arena.destroy()
  
  # Allocate from arena — no individual frees needed
  let p1 = cast[ptr int](arena.arenaAlloc(sizeof(int)))
  p1[] = 42
  
  let p2 = cast[ptr float](arena.arenaAlloc(sizeof(float)))
  p2[] = 3.14
  
  echo &"Arena allocated: {arena.totalAllocated} bytes"
  echo &"Values: {p1[]}, {p2[]}"
  
  # Pool for fixed-size objects
  type Node = object
    value: int
    next: ptr Node
  
  var pool = newPool[Node](128)
  defer: pool.destroy()
  
  let n1 = pool.acquire()
  n1.value = 10
  let n2 = pool.acquire()
  n2.value = 20
  n2.next = n1
  
  echo &"Pool allocated: {pool.allocated}"
  pool.release(n1)
  echo &"After release: {pool.allocated}"
```

---

## 2. OS Primitives

```nim
# os_primitives.nim
# Interact with OS directly

import std/[posix, os, strformat, tables]

# File system operations with low-level API
type
  FileDescriptor* = distinct cint

proc openFile*(path: string, flags: cint, mode: Mode = 0o644.Mode): FileDescriptor =
  let fd = posix.open(path.cstring, flags, mode)
  if fd == -1: raiseOSError(osLastError(), path)
  FileDescriptor(fd)

proc closeFile*(fd: FileDescriptor) =
  if posix.close(cint(fd)) == -1:
    raiseOSError(osLastError())

proc readBytes*(fd: FileDescriptor, buf: var openArray[byte]): int =
  result = posix.read(cint(fd), addr buf[0], buf.len)
  if result == -1: raiseOSError(osLastError())

proc writeBytes*(fd: FileDescriptor, buf: openArray[byte]): int =
  result = posix.write(cint(fd), unsafeAddr buf[0], buf.len)
  if result == -1: raiseOSError(osLastError())

# Memory mapping
type
  MappedMemory* = object
    `addr`*: pointer
    size*: int

proc mapFile*(path: string): MappedMemory =
  let fd = openFile(path, O_RDONLY)
  defer: fd.closeFile()
  
  var st: Stat
  if fstat(cint(fd), st) == -1:
    raiseOSError(osLastError())
  
  let size = st.st_size.int
  let p = mmap(nil, size, PROT_READ, MAP_PRIVATE, cint(fd), 0)
  if p == MAP_FAILED:
    raiseOSError(osLastError())
  
  MappedMemory(`addr`: p, size: size)

proc unmap*(m: MappedMemory) =
  if munmap(m.`addr`, m.size) == -1:
    raiseOSError(osLastError())

# Process management
type
  Process* = object
    pid*: Pid
    stdin*, stdout*, stderr*: FileDescriptor

proc spawnProcess*(cmd: string, args: seq[string]): Process =
  var stdinPipe, stdoutPipe, stderrPipe: array[2, cint]
  discard pipe(stdinPipe)
  discard pipe(stdoutPipe)
  discard pipe(stderrPipe)
  
  let pid = fork()
  if pid == 0:  # Child
    discard dup2(stdinPipe[0], STDIN_FILENO)
    discard dup2(stdoutPipe[1], STDOUT_FILENO)
    discard dup2(stderrPipe[1], STDERR_FILENO)
    discard close(stdinPipe[1])
    discard close(stdoutPipe[0])
    discard close(stderrPipe[0])
    
    var cArgs = allocCStringArray(@[cmd] & args)
    discard execvp(cmd.cstring, cArgs)
    deallocCStringArray(cArgs)
    quit(1)
  
  # Parent
  discard close(stdinPipe[0])
  discard close(stdoutPipe[1])
  discard close(stderrPipe[1])
  
  Process(
    pid: pid,
    stdin: FileDescriptor(stdinPipe[1]),
    stdout: FileDescriptor(stdoutPipe[0]),
    stderr: FileDescriptor(stderrPipe[0])
  )

proc wait*(p: Process): int =
  var status: cint
  discard waitpid(p.pid, status, 0)
  WEXITSTATUS(status)

# Signal handling
type SignalHandler* = proc(sig: cint) {.noconv.}

proc setSignalHandler*(sig: int, handler: SignalHandler) =
  var sa: Sigaction
  sa.sa_handler = handler
  discard sigaction(cint(sig), sa, nil)

var shutdownFlag* = false

proc handleShutdown(sig: cint) {.noconv.} =
  shutdownFlag = true

# Named semaphore / mutex via POSIX
type
  Mutex* = object
    m: PthreadMutex

proc initMutex*(mtx: var Mutex) =
  if pthreadMutexInit(mtx.m, nil) != 0:
    raise newException(OSError, "Failed to init mutex")

proc lock*(mtx: var Mutex) =
  discard pthreadMutexLock(mtx.m)

proc unlock*(mtx: var Mutex) =
  discard pthreadMutexUnlock(mtx.m)

proc destroy*(mtx: var Mutex) =
  discard pthreadMutexDestroy(mtx.m)

when isMainModule:
  setSignalHandler(SIGTERM, handleShutdown)
  setSignalHandler(SIGINT, handleShutdown)
  
  echo "Process PID: ", getCurrentProcessId()
  
  # Map a file
  let tmpFile = "/tmp/test_mmap.txt"
  writeFile(tmpFile, "Hello, mmap!")
  
  let mapped = mapFile(tmpFile)
  defer: mapped.unmap()
  
  let content = cast[cstring](mapped.`addr`)
  echo "Mapped content: ", $content
  
  echo "Running until SIGTERM/SIGINT..."
```

---

## 3. Low-Level Optimization

```nim
# low_level_opt.nim
# SIMD, cache optimization, branch prediction

import std/[times, strformat, math]

# SIMD via C intrinsics (SSE2/AVX2)
{.passC: "-msse2 -mavx2".}

type
  Vec4f* {.importc: "__m128", header: "<immintrin.h>".} = object
  Vec8f* {.importc: "__m256", header: "<immintrin.h>".} = object

proc mm_set_ps*(d, c, b, a: float32): Vec4f
  {.importc: "_mm_set_ps", header: "<immintrin.h>".}

proc mm_add_ps*(a, b: Vec4f): Vec4f
  {.importc: "_mm_add_ps", header: "<immintrin.h>".}

proc mm_mul_ps*(a, b: Vec4f): Vec4f
  {.importc: "_mm_mul_ps", header: "<immintrin.h>".}

proc mm_storeu_ps*(p: ptr float32, a: Vec4f)
  {.importc: "_mm_storeu_ps", header: "<immintrin.h>".}

proc mm_loadu_ps*(p: ptr float32): Vec4f
  {.importc: "_mm_loadu_ps", header: "<immintrin.h>".}

# Cache-friendly data structures
type
  # AoS (Array of Structures) — bad for SIMD
  ParticleAoS* = object
    x*, y*, z*: float32
    vx*, vy*, vz*: float32
    mass*: float32
  
  # SoA (Structure of Arrays) — good for SIMD
  ParticlesSoA* = object
    x*, y*, z*: seq[float32]
    vx*, vy*, vz*: seq[float32]
    mass*: seq[float32]
    count*: int

proc newParticlesSoA*(n: int): ParticlesSoA =
  ParticlesSoA(
    x: newSeq[float32](n), y: newSeq[float32](n), z: newSeq[float32](n),
    vx: newSeq[float32](n), vy: newSeq[float32](n), vz: newSeq[float32](n),
    mass: newSeq[float32](n), count: n
  )

proc updateSoA*(p: var ParticlesSoA, dt: float32) =
  ## SIMD-friendly particle update
  for i in 0..<p.count:
    p.x[i] += p.vx[i] * dt
    p.y[i] += p.vy[i] * dt
    p.z[i] += p.vz[i] * dt

# Prefetch hints
proc prefetchRead*(p: pointer) {.importc: "__builtin_prefetch", varargs.}

# Branch prediction hints
template likely*(cond: bool): bool =
  cast[bool](
    (proc (x: bool): cint {.importc: "__builtin_expect", noDecl.})(cond, 1)
  )

template unlikely*(cond: bool): bool =
  cast[bool](
    (proc (x: bool): cint {.importc: "__builtin_expect", noDecl.})(cond, 0)
  )

# Benchmarking
template benchmark*(name: string, iterations: int, body: untyped) =
  let start = cpuTime()
  for _ in 0..<iterations:
    body
  let elapsed = cpuTime() - start
  echo &"{name}: {elapsed * 1000.0:.3f}ms for {iterations} iterations"
  echo &"  {elapsed * 1e9 / iterations.float:.1f}ns per iteration"

when isMainModule:
  const N = 1_000_000
  
  var particles = newParticlesSoA(N)
  
  # Initialize
  for i in 0..<N:
    particles.x[i] = i.float32 * 0.001
    particles.vx[i] = 1.0
  
  benchmark("SoA update", 100):
    particles.updateSoA(0.016)
  
  echo &"Final x[0]: {particles.x[0]:.3f}"

# String interning for performance
type
  InternPool* = object
    table: seq[string]
    index: seq[tuple[hash: int, idx: int]]

proc intern*(pool: var InternPool, s: string): int =
  let h = hash(s)
  for entry in pool.index:
    if entry.hash == h and pool.table[entry.idx] == s:
      return entry.idx
  pool.table.add(s)
  pool.index.add((h, pool.table.len - 1))
  pool.table.len - 1

proc get*(pool: InternPool, idx: int): string = pool.table[idx]
```

---

## 4. Lock-Free Data Structures

```nim
# lockfree_advanced.nim
# Advanced lock-free programming

import std/atomics

type
  # Hazard Pointer for safe memory reclamation
  HazardPointer* = object
    ptrs: array[8, Atomic[pointer]]
    retiring: seq[pointer]

var globalHazard*: HazardPointer

proc protect*(hp: var HazardPointer, idx: int, p: pointer) =
  hp.ptrs[idx].store(p, moRelease)

proc clear*(hp: var HazardPointer, idx: int) =
  hp.ptrs[idx].store(nil, moRelease)

proc isHazardous*(hp: var HazardPointer, p: pointer): bool =
  for i in 0..<8:
    if hp.ptrs[i].load(moAcquire) == p: return true
  false

proc retire*(hp: var HazardPointer, p: pointer, freeFn: proc(x: pointer)) =
  hp.retiring.add(p)
  if hp.retiring.len > 16:  # Scan and collect
    var remaining: seq[pointer]
    for r in hp.retiring:
      if not hp.isHazardous(r):
        freeFn(r)
      else:
        remaining.add(r)
    hp.retiring = remaining

# Wait-free read counter
type
  WaitFreeCounter* = object
    counters: array[64, Atomic[int64]]  # per-CPU slot approximation

proc inc*(c: var WaitFreeCounter, amount: int64 = 1) =
  let slot = getThreadId() mod 64
  c.counters[slot].atomicInc(amount)

proc read*(c: var WaitFreeCounter): int64 =
  for i in 0..<64:
    result += c.counters[i].load(moRelaxed)

# Epoch-based reclamation
type
  Epoch* = object
    global: Atomic[int]
    local: int
    pending: array[3, seq[pointer]]

var globalEpoch*: Epoch

proc enterCritical*(ep: var Epoch) =
  ep.local = ep.global.load(moAcquire)

proc exitCritical*(ep: var Epoch) =
  ep.local = -1
  # Try to advance epoch
  let cur = ep.global.load(moAcquire)
  discard ep.global.compareExchange(cur, cur + 1, moRelease, moRelaxed)
  
  # Reclaim objects from epoch - 2
  let safe = cur - 2
  if safe >= 0:
    let idx = safe mod 3
    for p in ep.pending[idx]: dealloc(p)
    ep.pending[idx] = @[]

proc deferFree*(ep: var Epoch, p: pointer) =
  let cur = ep.global.load(moAcquire)
  ep.pending[cur mod 3].add(p)

when isMainModule:
  # Demonstrate wait-free counter
  var counter: WaitFreeCounter
  counter.inc(10)
  counter.inc(5)
  echo "Counter: ", counter.read()  # 15
```

---

## สรุป

| Topic | Key Concept |
|-------|-------------|
| Arena allocator | Bump pointer, bulk free |
| Pool allocator | Fixed-size freelist |
| SoA layout | Cache & SIMD friendly |
| Hazard pointers | Safe lock-free reclamation |
| Epoch reclamation | Batch deferred free |

---

**Next**: [Part 96 - Advanced FFI](../advanced/part96_ffi.md)
