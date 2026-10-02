# Part 87 - Testing Strategies & Property-Based Testing

## บทนำ

Testing ระดับ professional ใน Nim:
- Unit tests ด้วย unittest
- Property-based testing
- Fuzz testing
- Mutation testing concepts
- Snapshot testing

---

## 1. Advanced Unit Testing

```nim
# test_advanced.nim
# Advanced testing patterns

import std/[unittest, strformat, times, math, sequtils, algorithm, options]

# Custom matchers
template checkApprox*(a, b: float, eps = 1e-6): untyped =
  check abs(a - b) < eps

template checkThrows*(T: typedesc, body: untyped): untyped =
  var threw = false
  try:
    body
  except T:
    threw = true
  check threw

template checkNotThrows*(body: untyped): untyped =
  var threw = false
  var msg = ""
  try:
    body
  except CatchableError as e:
    threw = true
    msg = e.msg
  if threw:
    checkpoint(&"Expected no exception, but got: {msg}")
    check false

# Test fixtures
type
  TestFixture*[T] = ref object
    setup*: proc(): T
    teardown*: proc(t: T)

proc fixture*[T](setup: proc(): T, teardown: proc(t: T) = nil): TestFixture[T] =
  TestFixture[T](setup: setup, teardown: teardown)

template withFixture*[T](fix: TestFixture[T], name: untyped, body: untyped): untyped =
  let name = fix.setup()
  defer:
    if not fix.teardown.isNil:
      fix.teardown(name)
  body

# Parameterized tests
template parameterized*(name: string, cases: openArray[auto], body: untyped): untyped =
  for testCase in cases:
    let tc {.inject.} = testCase
    test &"{name} [{tc}]":
      body

# Benchmark helper
proc benchmark*(name: string, iterations: int, fn: proc()) =
  let start = cpuTime()
  for _ in 0..<iterations:
    fn()
  let elapsed = cpuTime() - start
  let perOp = elapsed / iterations.float * 1e9  # nanoseconds
  echo &"  {name}: {perOp:.1f} ns/op ({iterations} iterations)"

# Example tests
suite "String utilities":
  
  setup:
    let testStrings = @["hello", "world", "nim", "testing"]
  
  test "capitalize works":
    check "hello".capitalizeAscii() == "Hello"
    check "".capitalizeAscii() == ""
  
  test "join works with separator":
    check @["a", "b", "c"].join(", ") == "a, b, c"
  
  test "split and rejoin roundtrip":
    let original = "a,b,c,d"
    let parts = original.split(",")
    check parts.join(",") == original
  
  test "float comparison":
    checkApprox(0.1 + 0.2, 0.3)
  
  test "exception handling":
    checkThrows(ValueError):
      discard parseInt("not a number")

suite "Sorting":
  
  parameterized("sort order", @[
    @[3, 1, 4, 1, 5, 9],
    @[1],
    @[9, 8, 7],
    @[1, 2, 3, 4, 5]
  ]):
    test "result is sorted":
      var arr = tc
      arr.sort()
      for i in 0..<arr.len - 1:
        check arr[i] <= arr[i+1]
```

---

## 2. Property-Based Testing

```nim
# property_testing.nim
# Property-based testing (QuickCheck style)

import std/[random, sequtils, strutils, algorithm, math, times, options]

type
  Generator*[T] = proc(rng: var Rand, size: int): T

  ShrinkFn*[T] = proc(value: T): seq[T]

  Property*[T] = proc(value: T): bool

  TestResult* = object
    passed*: bool
    counterexample*: string
    iterations*: int
    shrinkSteps*: int

# Generators
proc genInt*(min, max: int): Generator[int] =
  proc(rng: var Rand, size: int): int =
    rng.rand(min..max)

proc genFloat*(min, max: float): Generator[float] =
  proc(rng: var Rand, size: int): float =
    min + rng.rand(1.0) * (max - min)

proc genString*(maxLen = 100, charset = "abcdefghijklmnopqrstuvwxyz"): Generator[string] =
  proc(rng: var Rand, size: int): string =
    let len = rng.rand(0..min(maxLen, size))
    for _ in 0..<len:
      result.add(charset[rng.rand(charset.len - 1)])

proc genBool*(): Generator[bool] =
  proc(rng: var Rand, size: int): bool = rng.rand(1) == 1

proc genSeq*[T](elem: Generator[T], maxLen = 20): Generator[seq[T]] =
  proc(rng: var Rand, size: int): seq[T] =
    let len = rng.rand(0..min(maxLen, size))
    for _ in 0..<len:
      result.add(elem(rng, size))

proc genOption*[T](elem: Generator[T]): Generator[Option[T]] =
  proc(rng: var Rand, size: int): Option[T] =
    if rng.rand(1) == 0: none(T)
    else: some(elem(rng, size))

proc map*[T, U](gen: Generator[T], fn: proc(v: T): U): Generator[U] =
  proc(rng: var Rand, size: int): U =
    fn(gen(rng, size))

proc filter*[T](gen: Generator[T], pred: proc(v: T): bool,
    maxAttempts = 100): Generator[T] =
  proc(rng: var Rand, size: int): T =
    for _ in 0..<maxAttempts:
      let v = gen(rng, size)
      if pred(v): return v
    raise newException(ValueError, "Could not generate value satisfying predicate")

# Shrinking
proc shrinkInt*(v: int): seq[int] =
  if v == 0: return @[]
  result = @[0, v div 2, v - (if v > 0: 1 else: -1)]
  result = result.filterIt(abs(it) < abs(v))
  result.deduplicate()

proc shrinkString*(s: string): seq[string] =
  if s.len == 0: return @[]
  result = @[""]
  # Drop each character
  for i in 0..<s.len:
    result.add(s[0..<i] & s[i+1..^1])
  # Drop first/last half
  result.add(s[0..<s.len div 2])
  result.add(s[s.len div 2..^1])
  result = result.filterIt(it.len < s.len)
  result.deduplicate()

proc shrinkSeq*[T](s: seq[T], elemShrink: ShrinkFn[T]): seq[seq[T]] =
  if s.len == 0: return @[]
  result.add(@[])
  # Drop each element
  for i in 0..<s.len:
    result.add(s[0..<i] & s[i+1..^1])
  # First/last half
  result.add(s[0..<s.len div 2])
  # Shrink elements
  for i, elem in s:
    for shrunk in elemShrink(elem):
      var copy = s
      copy[i] = shrunk
      result.add(copy)

# Property checker
proc check*[T](gen: Generator[T], prop: Property[T],
    shrink: ShrinkFn[T] = nil,
    iterations = 100, seed = 0u64): TestResult =
  
  var rng = if seed == 0: initRand() else: initRand(seed)
  
  for i in 0..<iterations:
    let size = (i / iterations.float * 100.0).int + 1
    let value = gen(rng, size)
    
    if not prop(value):
      # Found counterexample — try to shrink
      var counterexample = value
      var shrinkSteps = 0
      
      if not shrink.isNil:
        var candidates = shrink(value)
        while candidates.len > 0:
          var foundSmaller = false
          for candidate in candidates:
            if not prop(candidate):
              counterexample = candidate
              candidates = shrink(candidate)
              inc shrinkSteps
              foundSmaller = true
              break
          if not foundSmaller: break
      
      return TestResult(
        passed: false,
        counterexample: $counterexample,
        iterations: i + 1,
        shrinkSteps: shrinkSteps
      )
  
  TestResult(passed: true, iterations: iterations)

# Example properties
when isMainModule:
  # Property: sort is idempotent
  let sortIdempotent = check(
    genSeq(genInt(-100, 100)),
    proc(xs: seq[int]): bool =
      var a = xs; a.sort()
      var b = a; b.sort()
      a == b,
    proc(xs: seq[int]): seq[seq[int]] = shrinkSeq(xs, shrinkInt)
  )
  
  if sortIdempotent.passed:
    echo &"✓ sort is idempotent ({sortIdempotent.iterations} tests)"
  else:
    echo &"✗ counterexample: {sortIdempotent.counterexample}"
  
  # Property: reverse(reverse(xs)) == xs
  let reverseInvolution = check(
    genSeq(genInt(-100, 100)),
    proc(xs: seq[int]): bool =
      xs.reversed().reversed() == xs
  )
  echo if reverseInvolution.passed: "✓ reverse is an involution" else: "✗ failed"
  
  # Property: sorted array has elements in order
  let sortOrders = check(
    genSeq(genInt(-1000, 1000), maxLen = 50),
    proc(xs: seq[int]): bool =
      var sorted = xs; sorted.sort()
      for i in 0..<sorted.len - 1:
        if sorted[i] > sorted[i+1]: return false
      true
  )
  echo if sortOrders.passed: "✓ sort produces ordered output" else: "✗ failed"
  
  # Property: string roundtrip
  let stringRoundtrip = check(
    genString(50),
    proc(s: string): bool =
      let encoded = s.toHex()
      let decoded = parseHexStr(encoded)
      decoded == s,
    shrinkString
  )
  echo if stringRoundtrip.passed: "✓ hex encode/decode roundtrip" else: "✗ failed"
```

---

## 3. Snapshot Testing

```nim
# snapshot_testing.nim
# Snapshot testing สำหรับ output verification

import std/[os, strutils, strformat, json, algorithm]

type
  SnapshotStore* = ref object
    dir*: string
    update*: bool   # True = update snapshots instead of checking

proc newSnapshotStore*(dir = "tests/snapshots",
    update = existsEnv("UPDATE_SNAPSHOTS")): SnapshotStore =
  createDir(dir)
  SnapshotStore(dir: dir, update: update)

proc snapshotPath*(store: SnapshotStore, name: string): string =
  store.dir / name.replace("/", "_").replace(" ", "_") & ".snap"

proc matchSnapshot*(store: SnapshotStore, name: string, value: string): bool =
  let path = store.snapshotPath(name)
  
  if store.update or not fileExists(path):
    writeFile(path, value)
    echo &"[snapshot] {'Updated' if fileExists(path) else 'Created'}: {name}"
    return true
  
  let existing = readFile(path)
  if existing == value:
    return true
  
  # Show diff
  echo &"[snapshot] FAILED: {name}"
  echo "Expected (snapshot):"
  for line in existing.splitLines()[0..<min(10, existing.splitLines().len)]:
    echo &"  - {line}"
  echo "Got:"
  for line in value.splitLines()[0..<min(10, value.splitLines().len)]:
    echo &"  + {line}"
  
  false

proc matchJsonSnapshot*(store: SnapshotStore, name: string, value: JsonNode): bool =
  # Normalize JSON for stable comparison
  let normalized = value.pretty()
  store.matchSnapshot(name, normalized)

# Test helper
var snaps* = newSnapshotStore()

template snapshotTest*(name: string, value: untyped): untyped =
  test name:
    check snaps.matchSnapshot(name, $value)

when isMainModule:
  # Example: Test API response format
  let response = %*{
    "users": [
      {"id": 1, "name": "Alice", "role": "admin"},
      {"id": 2, "name": "Bob", "role": "user"}
    ],
    "total": 2,
    "page": 1
  }
  
  if snaps.matchJsonSnapshot("api_users_response", response):
    echo "✓ Response matches snapshot"
  else:
    echo "✗ Response differs from snapshot"
```

---

## 4. Fuzz Testing Helper

```nim
# fuzz_testing.nim
# Fuzz testing helpers

import std/[os, random, strformat, times, sequtils]

type
  FuzzConfig* = object
    maxIterations*: int
    maxInputSize*: int
    seedCorpus*: seq[seq[byte]]
    timeout*: float  # seconds

  FuzzResult* = object
    crashed*: bool
    input*: seq[byte]
    iterations*: int
    coverage*: int

proc defaultFuzzConfig*(): FuzzConfig =
  FuzzConfig(
    maxIterations: 10000,
    maxInputSize: 4096,
    seedCorpus: @[],
    timeout: 30.0
  )

proc mutate*(input: seq[byte], rng: var Rand): seq[byte] =
  result = input
  if result.len == 0: return @[rng.rand(255).byte]
  
  let mutations = rng.rand(1..3)
  for _ in 0..<mutations:
    case rng.rand(5)
    of 0:  # Flip bit
      let pos = rng.rand(result.len - 1)
      result[pos] = result[pos] xor (1.byte shl rng.rand(7))
    of 1:  # Insert byte
      let pos = rng.rand(result.len)
      result.insert(rng.rand(255).byte, pos)
    of 2:  # Delete byte
      if result.len > 0:
        let pos = rng.rand(result.len - 1)
        result.delete(pos)
    of 3:  # Replace byte
      let pos = rng.rand(result.len - 1)
      result[pos] = rng.rand(255).byte
    of 4:  # Duplicate range
      if result.len > 1:
        let start = rng.rand(result.len - 1)
        let len = rng.rand(min(16, result.len - start))
        let src = result[start..<start+len]
        result.insert(src, start)
    else: discard

proc fuzz*(target: proc(data: seq[byte]), config = defaultFuzzConfig()): FuzzResult =
  var rng = initRand()
  let startTime = cpuTime()
  
  var corpus = config.seedCorpus
  if corpus.len == 0:
    corpus.add(@[])
  
  for i in 0..<config.maxIterations:
    if cpuTime() - startTime > config.timeout: break
    
    # Pick from corpus and mutate
    let base = corpus[rng.rand(corpus.len - 1)]
    let input = mutate(base, rng)
    
    try:
      target(input)
      # If execution was interesting (coverage increase), add to corpus
      if i mod 100 == 0:  # Simplified: add every 100th
        corpus.add(input)
        if corpus.len > 1000:
          corpus.delete(rng.rand(corpus.len - 1))
    except CatchableError as e:
      return FuzzResult(crashed: true, input: input, iterations: i + 1)
    except Defect as e:
      return FuzzResult(crashed: true, input: input, iterations: i + 1)
    
    result.iterations = i + 1
  
  result.crashed = false

# Example: Fuzz a JSON parser
when isMainModule:
  import std/json
  
  proc fuzzJson(data: seq[byte]) =
    let s = cast[string](data)
    try:
      discard parseJson(s)
    except JsonParsingError:
      discard  # Expected for malformed input
  
  echo "Fuzzing JSON parser..."
  let result = fuzz(fuzzJson, FuzzConfig(
    maxIterations: 1000,
    maxInputSize: 256,
    timeout: 5.0,
    seedCorpus: @[
      cast[seq[byte]]("{\"key\": \"value\"}"),
      cast[seq[byte]]("[1, 2, 3]"),
      cast[seq[byte]]("null")
    ]
  ))
  
  if result.crashed:
    echo &"CRASH found after {result.iterations} iterations!"
    echo &"Input: {result.input}"
  else:
    echo &"No crashes in {result.iterations} iterations"
```

---

## สรุป

| Strategy | ใช้เมื่อ |
|----------|---------|
| Unit tests | Function correctness |
| Property-based | Invariant verification |
| Snapshot | Complex output stability |
| Fuzz | Parser, serializer safety |
| Benchmark | Performance regression |

---

**Next**: [Part 88 - Network Programming Internals](../advanced/part88_network.md)
