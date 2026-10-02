# Part 99 - Nim Ecosystem

## บทนำ

Nim Ecosystem — ไลบรารีและเครื่องมือสำคัญ:
- Nimble package manager
- Popular libraries
- Testing tools
- Development workflow

---

## 1. Nimble Package Manager

```bash
# nimble_guide.sh
# Nimble workflow

# Install package
nimble install prologue
nimble install norm
nimble install jsony

# Create new project
nimble init myproject
cd myproject

# Run tests
nimble test

# Build release
nimble build -d:release

# Publish package
nimble publish

# nimble.lock file for reproducible builds
nimble lock

# Update dependencies
nimble update
```

```nim
# myproject.nimble
# Package definition

version       = "1.0.0"
author        = "Your Name"
description   = "My Nim Project"
license       = "MIT"
srcDir        = "src"
bin           = @["myproject"]

# Dependencies
requires "nim >= 2.0.0"
requires "prologue >= 0.6.4"
requires "norm >= 2.8.0"
requires "jsony >= 1.1.5"
requires "dotenv >= 2.0.0"

# Tasks
task test, "Run tests":
  exec "nim c -r tests/test_main.nim"

task docs, "Generate docs":
  exec "nim doc --project src/myproject.nim"

task lint, "Run linter":
  exec "nim check src/myproject.nim"

task fmt, "Format code":
  exec "nimpretty --indent:2 src/"
```

---

## 2. Key Libraries

```nim
# jsony_example.nim
# Fast JSON serialization

import jsony

type
  User = object
    id: int
    name: string
    email: string
    active: bool
    tags: seq[string]

let user = User(
  id: 1,
  name: "Alice",
  email: "alice@example.com",
  active: true,
  tags: @["admin", "user"]
)

# Serialize
let json = user.toJson()
echo json  # {"id":1,"name":"Alice",...}

# Deserialize
let parsed = json.fromJson(User)
echo parsed.name  # "Alice"

# Hooks for custom types
type Money = object
  cents: int

proc toJson*(m: Money): string =
  result = $(m.cents.float / 100.0)

proc fromJson*(s: string, m: var Money) =
  m.cents = int(s.parseFloat() * 100)
```

```nim
# norm_example.nim
# ORM for SQLite/PostgreSQL

import norm/[model, sqlite]

type
  Person = ref object of Model
    name*: string
    age*: int
    email*: string

# Create database
let db = open("myapp.db", "", "", "")
db.createTables(Person())

# Insert
var person = Person(name: "Alice", age: 30, email: "alice@example.com")
db.insert(person)
echo "Inserted ID: ", person.id

# Query
var people = @[Person()]
db.select(people, "age > ?", 25)
for p in people:
  echo p.name, " (", p.age, ")"

# Update
person.name = "Alice Smith"
db.update(person)

# Delete
db.delete(person)

# Close
db.close()
```

```nim
# prologue_example.nim
# Web framework

import prologue

let app = newApp()

# Middleware
app.use(staticFileMiddleware("public"))

# Routes
app.get("/") do(ctx: Context) {.async.}:
  resp htmlResponse("<h1>Hello Nim!</h1>")

app.get("/api/users") do(ctx: Context) {.async.}:
  resp jsonResponse(%*[{"id": 1, "name": "Alice"}])

app.post("/api/users") do(ctx: Context) {.async.}:
  let body = ctx.request.body
  resp jsonResponse(%*{"created": true}, Http201)

app.get("/users/{id}") do(ctx: Context) {.async.}:
  let id = ctx.getPathParams("id")
  resp htmlResponse(&"<h1>User {id}</h1>")

app.run()
```

```nim
# results_example.nim
# Result type from results library

import results

type
  ParseError = object of CatchableError

proc parseInt(s: string): Result[int, string] =
  try:
    ok(s.parseInt())
  except:
    err("Cannot parse: " & s)

proc double(n: int): Result[int, string] =
  if n > 1000: err("Too large")
  else: ok(n * 2)

# Chaining with ?
proc processInput(s: string): Result[string, string] =
  let n = ? parseInt(s)
  let doubled = ? double(n)
  ok($doubled)

let r1 = processInput("21")
echo r1  # Ok(42)

let r2 = processInput("abc")
echo r2  # Err("Cannot parse: abc")

# Pattern matching
case parseInt("42")
of ok(v): echo "Got: ", v
of err(e): echo "Error: ", e
```

---

## 3. Testing with unittest2

```nim
# test_suite.nim
# Comprehensive testing

import unittest2
import std/[strformat, times]

suite "Number operations":
  setup:
    let x = 42
    let y = 8
  
  test "addition":
    check x + y == 50
  
  test "subtraction":
    check x - y == 34
  
  test "division":
    check x div y == 5
    check x mod y == 2

suite "String operations":
  test "concatenation":
    let s = "Hello" & " " & "World"
    check s == "Hello World"
    check s.len == 11
  
  test "case insensitive compare":
    check "Nim".toLowerAscii() == "nim"
  
  test "split and join":
    let parts = "a,b,c".split(',')
    check parts == @["a", "b", "c"]
    check parts.join("-") == "a-b-c"

suite "Async operations":
  test "future completes":
    proc myAsync(): Future[int] {.async.} =
      await sleepAsync(1)
      return 42
    
    let result = waitFor myAsync()
    check result == 42
  
  test "multiple futures":
    proc delayed(n: int): Future[int] {.async.} =
      await sleepAsync(1)
      return n
    
    let results = waitFor all(@[delayed(1), delayed(2), delayed(3)])
    check results == @[1, 2, 3]

# Benchmark test
test "performance":
  let start = cpuTime()
  var sum = 0
  for i in 0..<1_000_000:
    sum += i
  let elapsed = cpuTime() - start
  
  check sum == 499_999_500_000
  check elapsed < 1.0  # Must finish in 1 second
```

---

## 4. Tooling & Development Workflow

```nim
# nims_script.nims
# NimScript for build automation

import strformat, os

# Define tasks
task build, "Build release binary":
  exec "nim c -d:release -d:strip --opt:speed src/main.nim"

task debug, "Build debug binary":
  exec "nim c -d:debug --debuginfo src/main.nim"

task test, "Run all tests":
  for f in listFiles("tests"):
    if f.endsWith(".nim"):
      exec &"nim c -r {f}"

task clean, "Remove build artifacts":
  removeDir("nimcache")
  for f in listFiles("."):
    if f.endsWith(".exe") or (not f.contains(".")):
      if fileExists(f): removeFile(f)

task docker, "Build Docker image":
  exec "docker build -t myapp:latest ."

# Helper procs
proc checkDeps() =
  for dep in ["gcc", "git"]:
    if findExe(dep) == "":
      echo &"Missing dependency: {dep}"
      quit(1)

before build:
  checkDeps()
```

```nim
# config.nims
# Project-wide Nim configuration

# Compiler settings
switch("opt", "speed")
switch("d", "release")
switch("threads", "on")

# Warning settings
warning("Deprecated", off)
warning("UnusedImport", on)

# Hints
hint("CC", off)
hint("Link", off)

# Custom defines
when defined(release):
  switch("d", "strip")
  switch("passL", "-flto")

# Platform-specific
when defined(linux):
  switch("passL", "-static-libgcc")

when defined(windows):
  switch("passL", "-mwindows")  # Hide console window
```

---

## 5. Profiling & Benchmarking

```nim
# profiling.nim
# Profile and optimize Nim code

import std/[times, strformat, algorithm, stats]

# Micro benchmark framework
type
  BenchResult* = object
    name*: string
    iterations*: int
    totalNs*: float
    meanNs*: float
    stddevNs*: float
    minNs*: float
    maxNs*: float

proc bench*(name: string, iterations = 1000, warmup = 100,
            fn: proc()): BenchResult =
  # Warmup
  for _ in 0..<warmup: fn()
  
  var times = newSeq[float](iterations)
  
  for i in 0..<iterations:
    let start = getMonoTime()
    fn()
    let elapsed = getMonoTime() - start
    times[i] = elapsed.inNanoseconds.float
  
  times.sort()
  
  var rs: RunningStat
  for t in times: rs.push(t)
  
  BenchResult(
    name: name,
    iterations: iterations,
    totalNs: rs.sum,
    meanNs: rs.mean,
    stddevNs: rs.standardDeviation,
    minNs: times[0],
    maxNs: times[^1]
  )

proc printBench*(r: BenchResult) =
  echo &"\n=== {r.name} ==="
  echo &"  Iterations: {r.iterations}"
  echo &"  Mean:   {r.meanNs:.1f} ns"
  echo &"  StdDev: {r.stddevNs:.1f} ns"
  echo &"  Min:    {r.minNs:.1f} ns"
  echo &"  Max:    {r.maxNs:.1f} ns"
  echo &"  Throughput: {1e9 / r.meanNs:.0f} ops/s"

proc compare*(a, b: BenchResult) =
  let ratio = a.meanNs / b.meanNs
  if ratio < 1.0:
    echo &"{a.name} is {1/ratio:.2f}x FASTER than {b.name}"
  else:
    echo &"{a.name} is {ratio:.2f}x SLOWER than {b.name}"

when isMainModule:
  # Compare different implementations
  var hashTable = initTable[string, int]()
  var linearSeq: seq[(string, int)]
  
  for i in 0..999:
    let k = $i
    hashTable[k] = i
    linearSeq.add((k, i))
  
  let hashBench = bench("Table lookup", 10000) do:
    discard hashTable.getOrDefault("500")
  
  let linearBench = bench("Linear search", 10000) do:
    for (k, v) in linearSeq:
      if k == "500": discard v; break
  
  hashBench.printBench()
  linearBench.printBench()
  compare(hashBench, linearBench)
```

---

## สรุป

| Tool/Library | Purpose |
|-------------|---------|
| Nimble | Package manager |
| jsony | Fast JSON |
| norm | ORM |
| prologue | Web framework |
| results | Error handling |
| unittest2 | Testing |

---

**Next**: [Part 100 - Course Finale](../advanced/part100_finale.md)
