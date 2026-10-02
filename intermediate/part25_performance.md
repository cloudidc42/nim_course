# Part 25: Performance และ Optimization

## Profiling

```nim
# Built-in profiler
# compile: nim c --profiler:on --stacktrace:on myapp.nim
# run: ./myapp
# output: profile_results.txt

# Manual timing
import times, strformat

template bench(name: string, iterations: int, body: untyped): untyped =
  let start = cpuTime()
  for _ in 0..<iterations:
    body
  let elapsed = (cpuTime() - start) * 1000
  echo &"{name}: {elapsed:.2f}ms ({iterations} iterations, {elapsed/float(iterations)*1000:.2f}μ/iter)"

# String concatenation comparison
bench("&=", 10000):
  var s = ""
  for i in 0..<100:
    s &= $i

bench("add/join", 10000):
  var parts: seq[string] = @[]
  for i in 0..<100:
    parts.add($i)
  let s = parts.join("")

bench("StringBuilder", 10000):
  import std/strutils
  var sb = ""
  sb.setLen(0)
  for i in 0..<100:
    sb.add($i)

# Memory profiling
when defined(memProfiler):
  import memProfiler
  # automatically tracks allocations
```

## Compiler Optimizations

```bash
# Optimization levels
nim c -O0 myapp.nim     # no optimization
nim c -O1 myapp.nim     # basic optimization
nim c -O2 myapp.nim     # standard optimization
nim c -O3 myapp.nim     # aggressive
nim c -d:release myapp.nim  # release mode (includes -O3)
nim c -d:release --opt:size myapp.nim  # optimize for size

# LTO (Link Time Optimization)
nim c -d:release --passC:"-flto" --passL:"-flto" myapp.nim

# PGO (Profile Guided Optimization)
# 1. Build with instrumentation
nim c -d:release --passC:"-fprofile-generate" myapp.nim
# 2. Run to collect profile
./myapp
# 3. Build with profile
nim c -d:release --passC:"-fprofile-use" myapp.nim
```

## Performance Tips

```nim
# 1. Avoid unnecessary allocations
proc badConcat(items: seq[string]): string =
  result = ""
  for item in items:
    result = result & ", " & item  # allocates each time!

proc goodConcat(items: seq[string]): string =
  result = ""
  for i, item in items:
    if i > 0: result.add(", ")
    result.add(item)  # in-place modification

# 2. Use openArray instead of seq for read-only
proc processSeq(data: seq[int]): int =  # copies on call
  data.foldl(a + b)

proc processOpenArray(data: openArray[int]): int =  # no copy
  data.foldl(a + b)

let arr = [1, 2, 3, 4, 5]
let s = @[1, 2, 3, 4, 5]
echo processOpenArray(arr)  # works with both
echo processOpenArray(s)

# 3. Pre-allocate sequences
proc badGrow(): seq[int] =
  result = @[]
  for i in 0..<1000:
    result.add(i)  # may reallocate many times

proc goodGrow(): seq[int] =
  result = newSeqOfCap[int](1000)  # pre-allocate
  for i in 0..<1000:
    result.add(i)  # never reallocates

# 4. Avoid boxing in closures
proc createAdder(n: int): proc(x: int): int =
  proc adder(x: int): int = x + n  # n is captured
  adder

let add5 = createAdder(5)
echo add5(10)  # 15

# 5. Use {.noSideEffect.} for optimization hints
proc pureCalc(x, y: int): int {.noSideEffect.} =
  x * x + y * y

# 6. Inline short functions
proc square(x: int): int {.inline.} = x * x

echo square(5)  # inlined by compiler
```

## Data Structures Performance

```nim
import tables, sets, algorithm, sequtils, times

# Hash vs Linear search
proc linearSearch(arr: seq[int], target: int): bool =
  target in arr

proc hashSearch(hs: HashSet[int], target: int): bool =
  target in hs

let n = 100000
let data = toSeq(1..n)
let dataSet = data.toHashSet()

bench("Linear search", 10000):
  discard linearSearch(data, n div 2)

bench("Hash search", 10000):
  discard hashSearch(dataSet, n div 2)

# Sorting
var unsorted = toSeq(1..10000).reversed()

bench("std sort", 100):
  var arr = unsorted
  arr.sort()

bench("std sort (sorted)", 100):
  var arr = unsorted
  arr.sort()
  arr.sort()  # already sorted - should be fast

# String vs char comparison
let str = "hello world " & "x".repeat(1000)

bench("contains (string)", 10000):
  discard str.contains("xyz")

bench("indexOf (char)", 10000):
  discard str.find('x')
```

## SIMD and Low-level Optimization

```nim
# Using built-in SIMD via C
{.emit: """
#include <immintrin.h>

void addArraysSIMD(float* a, float* b, float* result, int n) {
    int i = 0;
    for (; i <= n - 8; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);
        __m256 vb = _mm256_loadu_ps(b + i);
        __m256 vr = _mm256_add_ps(va, vb);
        _mm256_storeu_ps(result + i, vr);
    }
    // Handle remainder
    for (; i < n; i++) {
        result[i] = a[i] + b[i];
    }
}
""".}

proc addArraysSIMD(a, b, result: ptr float32, n: cint) 
  {.importc, cdecl.}

# Use it
let n = 1000000
var arrA = newSeq[float32](n)
var arrB = newSeq[float32](n)
var arrC = newSeq[float32](n)

for i in 0..<n:
  arrA[i] = float32(i)
  arrB[i] = float32(n - i)

addArraysSIMD(addr arrA[0], addr arrB[0], addr arrC[0], n.cint)
echo arrC[0], " ", arrC[n-1]  # n, n

# Bit manipulation tricks
proc isPowerOfTwo(n: int): bool = n > 0 and (n and (n - 1)) == 0
proc nextPowerOfTwo(n: int): int =
  var x = n - 1
  x = x or (x shr 1)
  x = x or (x shr 2)
  x = x or (x shr 4)
  x = x or (x shr 8)
  x = x or (x shr 16)
  x = x or (x shr 32)
  x + 1

proc countBits(n: int): int =
  var x = n
  while x != 0:
    inc result
    x = x and (x - 1)

echo isPowerOfTwo(64)    # true
echo nextPowerOfTwo(100) # 128
echo countBits(255)      # 8
```

## Cache Efficiency

```nim
# Row-major vs Column-major access
const N = 1000

# Row-major (cache-friendly)
proc sumRowMajor(mat: array[N, array[N, int]]): int =
  for i in 0..<N:
    for j in 0..<N:
      result += mat[i][j]  # sequential memory access

# Column-major (cache-unfriendly)
proc sumColMajor(mat: array[N, array[N, int]]): int =
  for j in 0..<N:
    for i in 0..<N:
      result += mat[i][j]  # strided memory access

# Structure of Arrays vs Array of Structures
# AoS (cache-unfriendly for partial access)
type Point3D_AoS = object
  x, y, z: float

# SoA (cache-friendly for x-only access)
type Points3D_SoA = object
  x: seq[float]
  y: seq[float]
  z: seq[float]

proc sumX_AoS(points: seq[Point3D_AoS]): float =
  for p in points:
    result += p.x  # accesses every 3rd float

proc sumX_SoA(points: Points3D_SoA): float =
  points.x.foldl(a + b)  # sequential access
```

## Practical: High-Performance Text Processing

```nim
import strutils, sequtils, algorithm

type
  WordFreq = object
    word: string
    count: int

proc countWords(text: string): seq[WordFreq] =
  # Efficient word counting
  var counts = initCountTable[string]()
  
  # Process without splitting into sequences
  var wordStart = -1
  for i, c in text:
    if c.isAlphaAscii():
      if wordStart < 0: wordStart = i
    else:
      if wordStart >= 0:
        let word = text[wordStart..<i].toLowerAscii()
        counts.inc(word)
        wordStart = -1
  
  if wordStart >= 0:
    let word = text[wordStart..^1].toLowerAscii()
    counts.inc(word)
  
  # Sort by count descending
  result = @[]
  for word, count in counts:
    result.add(WordFreq(word: word, count: count))
  
  result.sort(proc(a, b: WordFreq): int = b.count - a.count)

proc topWords(text: string, n: int): seq[WordFreq] =
  countWords(text)[0..<min(n, countWords(text).len)]

let text = "the quick brown fox jumps over the lazy dog the fox"
let top = topWords(text, 5)
for wf in top:
  echo wf.word, ": ", wf.count
```

## สรุป Part 25

ในบทนี้เราได้เรียนรู้:
- ✅ Profiling techniques
- ✅ Compiler optimization flags
- ✅ Performance tips (allocation, openArray)
- ✅ Data structure performance comparison
- ✅ SIMD optimization
- ✅ Cache efficiency (SoA vs AoS)
- ✅ Practical: High-performance text processing

---

**Previous**: [Part 24 - Testing](part24_testing.md)
**Next**: [Part 26 - Web Development](../web/part26_web_intro.md)
