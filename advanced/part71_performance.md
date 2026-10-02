# Part 71 - Performance Optimization Deep Dive

## บทนำ

Nim มี performance ที่ใกล้เคียง C/C++ แต่ต้องรู้วิธี unlock มัน:
- Compiler flags
- Data-oriented design
- SIMD intrinsics
- Cache-friendly layouts
- Zero-copy techniques

---

## 1. Compiler Flags สำคัญ

```bash
# Maximum speed
nim c \
  -d:release \          # Enable optimizations, disable assertions
  -d:danger \           # Extra optimizations (disable all checks)
  --opt:speed \         # Optimize for speed (not size)
  --passC:-march=native \  # Use CPU-specific instructions
  --passC:-O3 \         # GCC/Clang level 3 optimization
  --passC:-flto \       # Link-time optimization
  --passL:-flto \
  --mm:arc \            # ARC: deterministic, no cycle detection overhead
  myapp.nim

# Profile
nim c \
  -d:release \
  --debugger:native \
  --passC:-pg \         # gprof profiling
  --passL:-pg \
  myapp.nim
  
# With Valgrind/perf
nim c -d:release --passC:-g myapp.nim
perf stat ./myapp
perf record ./myapp && perf report
```

---

## 2. Data-Oriented Design (DOD)

```nim
# dod.nim
# AoS (Array of Structs) vs SoA (Struct of Arrays)
# SoA ดีกว่าสำหรับ SIMD และ cache efficiency

import std/[times, strformat, math, random]

const N = 1_000_000

# === Bad: Array of Structs (AoS) ===
type
  ParticleAoS* = object
    x*, y*, z*: float32
    vx*, vy*, vz*: float32
    mass*: float32
    alive*: bool
    pad: array[3, uint8]  # Padding makes struct larger

var particlesAoS: array[1000, ParticleAoS]

proc updateAoS(particles: var openArray[ParticleAoS], dt: float32) =
  # Cache miss likely: each particle = 32 bytes, reads x,y,z,vx,vy,vz
  # But also loads mass, alive, pad which we don't need
  for p in particles.mitems:
    p.x += p.vx * dt
    p.y += p.vy * dt
    p.z += p.vz * dt

# === Good: Struct of Arrays (SoA) ===
type
  ParticlesSoA* = object
    x*, y*, z*: seq[float32]      # Contiguous arrays — SIMD friendly
    vx*, vy*, vz*: seq[float32]
    mass*: seq[float32]
    alive*: seq[bool]
    count*: int

proc newParticlesSoA*(n: int): ParticlesSoA =
  ParticlesSoA(
    x: newSeq[float32](n),
    y: newSeq[float32](n),
    z: newSeq[float32](n),
    vx: newSeq[float32](n),
    vy: newSeq[float32](n),
    vz: newSeq[float32](n),
    mass: newSeq[float32](n),
    alive: newSeq[bool](n),
    count: n
  )

proc updateSoA*(p: var ParticlesSoA, dt: float32) =
  # All x values are contiguous — CPU can prefetch efficiently
  # Compiler can auto-vectorize this loop
  {.push boundChecks: off, overflowChecks: off.}
  for i in 0..<p.count:
    p.x[i] += p.vx[i] * dt
    p.y[i] += p.vy[i] * dt
    p.z[i] += p.vz[i] * dt
  {.pop.}

# AoSoA (Array of Small Structs of Arrays) — best for SIMD width 4/8
const SIMD_WIDTH = 4

type
  ParticleBlock* = object  # Process 4 particles at once (SSE)
    x*, y*, z*: array[SIMD_WIDTH, float32]
    vx*, vy*, vz*: array[SIMD_WIDTH, float32]

proc updateAoSoA*(blocks: var seq[ParticleBlock], dt: float32) =
  for blk in blocks.mitems:
    for i in 0..<SIMD_WIDTH:
      blk.x[i] += blk.vx[i] * dt
      blk.y[i] += blk.vy[i] * dt
      blk.z[i] += blk.vz[i] * dt

# Benchmark comparison
proc benchDOD() =
  var soa = newParticlesSoA(100_000)
  randomize()
  
  for i in 0..<soa.count:
    soa.x[i] = rand(100.0).float32
    soa.vx[i] = rand(1.0).float32
  
  let t0 = cpuTime()
  for _ in 0..99:
    soa.updateSoA(0.016)
  let t1 = cpuTime()
  
  echo &"SoA update 100k particles x100: {(t1-t0)*1000:.1f}ms"
```

---

## 3. Memory Pool Allocator

```nim
# pool_allocator.nim
# Pool allocation ป้องกัน heap fragmentation และลด allocation overhead

import std/[math, strformat]

type
  PoolBlock*[T] = object
    data*: T
    next*: ptr PoolBlock[T]  # Free list

  ObjectPool*[T] = ref object
    memory*: seq[PoolBlock[T]]
    freeList*: ptr PoolBlock[T]
    capacity*: int
    used*: int

proc newObjectPool*[T](capacity: int): ObjectPool[T] =
  result = ObjectPool[T](
    memory: newSeq[PoolBlock[T]](capacity),
    capacity: capacity,
    used: 0
  )
  # Build free list
  for i in 0..<capacity - 1:
    result.memory[i].next = addr result.memory[i + 1]
  result.memory[capacity - 1].next = nil
  result.freeList = addr result.memory[0]

proc alloc*[T](pool: ObjectPool[T]): ptr T =
  if pool.freeList == nil:
    raise newException(OutOfMemDefect, "Pool exhausted")
  
  let blk = pool.freeList
  pool.freeList = blk.next
  blk.next = nil
  inc pool.used
  addr blk.data

proc free*[T](pool: ObjectPool[T], p: ptr T) =
  let blk = cast[ptr PoolBlock[T]](p)
  blk.next = pool.freeList
  pool.freeList = blk
  dec pool.used

template withPool*[T](pool: ObjectPool[T], body: untyped): untyped =
  let p = pool.alloc()
  try:
    let it {.inject.} = p
    body
  finally:
    pool.free(p)

# Slab allocator for same-size objects
type
  Slab*[T] = ref object
    pages*: seq[ptr UncheckedArray[T]]
    pageSize*: int
    freeSlots*: seq[int]
    nextSlot*: int

proc newSlab*[T](pageSize = 64): Slab[T] =
  Slab[T](pageSize: pageSize)

proc allocSlab*[T](slab: Slab[T]): ptr T =
  if slab.freeSlots.len > 0:
    let idx = slab.freeSlots.pop()
    let pageIdx = idx div slab.pageSize
    let slotIdx = idx mod slab.pageSize
    return addr slab.pages[pageIdx][slotIdx]
  
  if slab.nextSlot mod slab.pageSize == 0:
    # Allocate new page
    let page = cast[ptr UncheckedArray[T]](alloc(sizeof(T) * slab.pageSize))
    slab.pages.add(page)
  
  let pageIdx = slab.pages.len - 1
  let slotIdx = slab.nextSlot mod slab.pageSize
  inc slab.nextSlot
  addr slab.pages[pageIdx][slotIdx]

proc freeSlab*[T](slab: Slab[T], p: ptr T) =
  # Find index and add to free list
  for pageIdx, page in slab.pages:
    let pageStart = cast[int](page)
    let pAddr = cast[int](p)
    let slotSize = sizeof(T)
    
    if pAddr >= pageStart and pAddr < pageStart + slotSize * slab.pageSize:
      let slotIdx = (pAddr - pageStart) div slotSize
      slab.freeSlots.add(pageIdx * slab.pageSize + slotIdx)
      return
```

---

## 4. SIMD Intrinsics

```nim
# simd.nim
# SIMD via C emit — สำหรับ maximum performance

# Note: ใช้ nim-simd package หรือ emit C SIMD intrinsics โดยตรง

{.push checks: off.}

proc addFloats_SSE*(a, b: ptr float32, result: ptr float32, n: int) {.inline.} =
  ## Add two float arrays using SSE intrinsics (4 floats at once)
  {.emit: """
  #include <immintrin.h>
  float *pa = (float*)a, *pb = (float*)b, *pr = (float*)result;
  int i = 0;
  int m = n - (n % 4);
  
  for (; i < m; i += 4) {
    __m128 va = _mm_loadu_ps(pa + i);
    __m128 vb = _mm_loadu_ps(pb + i);
    __m128 vr = _mm_add_ps(va, vb);
    _mm_storeu_ps(pr + i, vr);
  }
  for (; i < n; i++) {
    pr[i] = pa[i] + pb[i];
  }
  """.}

proc dotProduct_AVX*(a, b: ptr float32, n: int): float32 {.inline.} =
  ## Dot product using AVX (8 floats at once)
  var sum: float32 = 0.0
  {.emit: """
  #include <immintrin.h>
  float *pa = (float*)a, *pb = (float*)b;
  `sum` = 0.0f;
  
  #ifdef __AVX__
  __m256 vsum = _mm256_setzero_ps();
  int i = 0;
  int m = n - (n % 8);
  
  for (; i < m; i += 8) {
    __m256 va = _mm256_loadu_ps(pa + i);
    __m256 vb = _mm256_loadu_ps(pb + i);
    vsum = _mm256_fmadd_ps(va, vb, vsum);
  }
  
  // Horizontal sum of 8 floats
  __m128 hi = _mm256_extractf128_ps(vsum, 1);
  __m128 lo = _mm256_castps256_ps128(vsum);
  lo = _mm_add_ps(lo, hi);
  lo = _mm_hadd_ps(lo, lo);
  lo = _mm_hadd_ps(lo, lo);
  `sum` = _mm_cvtss_f32(lo);
  
  for (; i < n; i++) {
    `sum` += pa[i] * pb[i];
  }
  #else
  for (int i = 0; i < n; i++) {
    `sum` += pa[i] * pb[i];
  }
  #endif
  """.}
  sum

# Pure Nim version (compiler might auto-vectorize)
proc dotProductNim*(a, b: openArray[float32]): float32 =
  result = 0.0
  let n = min(a.len, b.len)
  for i in 0..<n:
    result += a[i] * b[i]

{.pop.}
```

---

## 5. Cache-Friendly Algorithms

```nim
# cache_friendly.nim
# Cache oblivious matrix multiply และ cache-friendly data structures

import std/[math, strformat, times]

type
  Matrix*[N: static int, T] = object
    data*: array[N * N, T]

proc `[]`*[N: static int, T](m: Matrix[N, T], r, c: int): T =
  m.data[r * N + c]

proc `[]=`*[N: static int, T](m: var Matrix[N, T], r, c: int, v: T) =
  m.data[r * N + c] = v

# Naive matrix multiply (cache unfriendly — column access)
proc matMulNaive*[N: static int](a, b: Matrix[N, float64],
    result: var Matrix[N, float64]) =
  for i in 0..<N:
    for j in 0..<N:
      var sum = 0.0
      for k in 0..<N:
        sum += a[i, k] * b[k, j]  # b[k,j] — column traversal = cache miss!
      result[i, j] = sum

# Cache-friendly: transpose B first, then row-row multiply
proc matMulTranspose*[N: static int](a, b: Matrix[N, float64],
    result: var Matrix[N, float64]) =
  var bT: Matrix[N, float64]
  
  # Transpose B
  for i in 0..<N:
    for j in 0..<N:
      bT[j, i] = b[i, j]
  
  # Now both a[i,k] and bT[j,k] are row accesses = cache friendly
  for i in 0..<N:
    for j in 0..<N:
      var sum = 0.0
      for k in 0..<N:
        sum += a[i, k] * bT[j, k]
      result[i, j] = sum

# Blocked/tiled matrix multiply (cache oblivious)
proc matMulBlocked*[N: static int](a, b: Matrix[N, float64],
    result: var Matrix[N, float64], blockSize = 64) =
  for ii in countup(0, N - 1, blockSize):
    for jj in countup(0, N - 1, blockSize):
      for kk in countup(0, N - 1, blockSize):
        # Process block
        for i in ii..<min(ii + blockSize, N):
          for j in jj..<min(jj + blockSize, N):
            var sum = result[i, j]
            for k in kk..<min(kk + blockSize, N):
              sum += a[i, k] * b[k, j]
            result[i, j] = sum

# Prefetch hints
template prefetch*(p: pointer, rw = 0, locality = 3) =
  {.emit: "__builtin_prefetch(`p`, `rw`, `locality`);".}

# Binary search with prefetch
proc binarySearchFast*[T](arr: openArray[T], target: T): int =
  var lo = 0
  var hi = arr.len - 1
  
  while lo <= hi:
    let mid = (lo + hi) shr 1
    
    # Prefetch next likely positions
    if mid + 32 < arr.len:
      prefetch(unsafeAddr arr[mid + 32])
    
    if arr[mid] == target: return mid
    elif arr[mid] < target: lo = mid + 1
    else: hi = mid - 1
  
  -1
```

---

## 6. Zero-Copy Techniques

```nim
# zero_copy.nim
# ลด copying ด้วย openArray, views, และ borrowing

import std/[strformat, strutils]

# openArray — ไม่ copy, ทำงานกับ seq, array, string
proc processData*(data: openArray[byte]): int =
  var sum = 0
  for b in data:
    sum += int(b)
  sum

# String view (avoid copy when parsing)
type
  StringView* = object
    data*: ptr char
    len*: int

proc toView*(s: string): StringView =
  StringView(data: cast[ptr char](s[0].unsafeAddr), len: s.len)

proc toView*(s: string, start, length: int): StringView =
  StringView(data: cast[ptr char](s[start].unsafeAddr), len: length)

proc `$`*(v: StringView): string =
  result = newString(v.len)
  copyMem(result[0].addr, v.data, v.len)

proc `[]`*(v: StringView, i: int): char =
  cast[ptr UncheckedArray[char]](v.data)[i]

proc slice*(v: StringView, start, stop: int): StringView =
  StringView(data: cast[ptr char](cast[int](v.data) + start), len: stop - start)

# Fast CSV parser using views (no allocation per field)
proc parseCSVLine*(line: string, handler: proc(field: StringView, col: int)) =
  var col = 0
  var fieldStart = 0
  
  for i in 0..line.len:
    let c = if i < line.len: line[i] else: ','
    if c == ',' or c == '\n':
      handler(line.toView(fieldStart, i - fieldStart), col)
      inc col
      fieldStart = i + 1

# Memory-mapped file (zero-copy file reading)
when defined(posix):
  import posix
  
  type
    MMapFile* = ref object
      fd*: cint
      data*: ptr UncheckedArray[byte]
      size*: int
  
  proc openMMap*(path: string): MMapFile =
    let fd = open(path, O_RDONLY)
    if fd < 0:
      raise newException(IOError, &"Cannot open: {path}")
    
    var st: Stat
    discard fstat(fd, st)
    let size = st.st_size
    
    let data = mmap(nil, size, PROT_READ, MAP_PRIVATE, fd, 0)
    if data == MAP_FAILED:
      discard close(fd)
      raise newException(IOError, "mmap failed")
    
    MMapFile(fd: fd, data: cast[ptr UncheckedArray[byte]](data), size: int(size))
  
  proc close*(f: MMapFile) =
    discard munmap(f.data, f.size)
    discard close(f.fd)
  
  proc readByte*(f: MMapFile, offset: int): byte =
    f.data[offset]
  
  proc toOpenArray*(f: MMapFile): openArray[byte] =
    f.data.toOpenArray(0, f.size - 1)

# Arena allocator for temporary allocations
type
  Arena* = ref object
    buffer*: seq[byte]
    offset*: int

proc newArena*(size: int): Arena =
  Arena(buffer: newSeq[byte](size), offset: 0)

proc alloc*(arena: Arena, size: int): pointer =
  let aligned = (arena.offset + 7) and not 7  # 8-byte align
  if aligned + size > arena.buffer.len:
    raise newException(OutOfMemDefect, "Arena full")
  result = addr arena.buffer[aligned]
  arena.offset = aligned + size

proc reset*(arena: Arena) =
  arena.offset = 0  # O(1) free all — just reset offset!

template withArena*(size: int, body: untyped) =
  let arena {.inject.} = newArena(size)
  body
  # arena freed when out of scope
```

---

## 7. Profiling

```nim
# profiler.nim
# CPU และ memory profiling

import std/[times, tables, strformat, sequtils, algorithm, locks]

type
  ProfileEntry* = object
    name*: string
    calls*: int
    totalTime*: float
    minTime*: float
    maxTime*: float
    selfTime*: float

  Profiler* = ref object
    entries*: Table[string, ProfileEntry]
    callStack*: seq[tuple[name: string, startTime: float]]
    lock*: Lock

var globalProfiler* = Profiler(entries: initTable[string, ProfileEntry]())
initLock(globalProfiler.lock)

template profile*(name: string, body: untyped) =
  let _t0 = cpuTime()
  globalProfiler.lock.acquire()
  globalProfiler.callStack.add((name: name, startTime: _t0))
  globalProfiler.lock.release()
  
  body
  
  let _t1 = cpuTime()
  let _elapsed = _t1 - _t0
  
  globalProfiler.lock.acquire()
  discard globalProfiler.callStack.pop()
  
  if not globalProfiler.entries.hasKey(name):
    globalProfiler.entries[name] = ProfileEntry(
      name: name,
      minTime: float.high,
      maxTime: float.low
    )
  
  var entry = globalProfiler.entries[name]
  inc entry.calls
  entry.totalTime += _elapsed
  entry.minTime = min(entry.minTime, _elapsed)
  entry.maxTime = max(entry.maxTime, _elapsed)
  globalProfiler.entries[name] = entry
  globalProfiler.lock.release()

proc printProfileReport*(profiler: Profiler = globalProfiler) =
  var entries = profiler.entries.values.toSeq()
  entries.sort(proc(a, b: ProfileEntry): int =
    cmp(b.totalTime, a.totalTime)  # Sort by total time descending
  )
  
  echo "\n=== PROFILE REPORT ==="
  echo &"{'Name':<30} {'Calls':>8} {'Total ms':>12} {'Avg μs':>10} {'Min μs':>10} {'Max μs':>10}"
  echo "-".repeat(85)
  
  for e in entries:
    echo &"{e.name:<30} {e.calls:>8} {e.totalTime*1000:>12.2f} {e.totalTime/float(e.calls)*1e6:>10.2f} {e.minTime*1e6:>10.2f} {e.maxTime*1e6:>10.2f}"
```

---

## สรุป Optimization Checklist

```
[ ] ใช้ -d:release สำหรับ production build
[ ] เลือก --mm:arc หรือ --mm:orc ให้เหมาะกับ use case
[ ] Profile ก่อน optimize (อย่า premature optimization)
[ ] ใช้ SoA แทน AoS สำหรับ hot data
[ ] ใช้ openArray แทน seq parameter (ลด allocation)
[ ] ใช้ pool allocator สำหรับ frequent small allocations
[ ] ใช้ arena allocator สำหรับ temporary allocations
[ ] ลด cache misses ด้วย data locality
[ ] ใช้ --passC:-march=native สำหรับ CPU-specific SIMD
[ ] Benchmark ก่อนและหลัง optimization
```

---

**Next**: [Part 72 - Nim 2.0 Features](../advanced/part72_nim2.md)
