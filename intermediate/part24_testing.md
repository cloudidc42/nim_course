# Part 24: Testing - การทดสอบโปรแกรม

## unittest Module

```nim
import unittest

# Test suite เบื้องต้น
suite "Math Tests":
  test "addition":
    check 1 + 1 == 2
    check 2 + 3 == 5

  test "subtraction":
    check 10 - 3 == 7

  test "multiplication":
    check 4 * 5 == 20

  test "division":
    check 10 div 2 == 5
    check 10 mod 3 == 1

  test "floating point":
    let result = 0.1 + 0.2
    check abs(result - 0.3) < 1e-10

# Setup/teardown
suite "File Tests":
  setup:
    writeFile("test.txt", "test content")

  teardown:
    removeFile("test.txt")

  test "file exists":
    check fileExists("test.txt")

  test "read file":
    let content = readFile("test.txt")
    check content == "test content"

# Expect exceptions
suite "Error Tests":
  test "division by zero":
    expect DivByZeroDefect:
      let _ = 1 div 0

  test "invalid parse":
    expect ValueError:
      let _ = parseInt("not a number")

  test "index out of bounds":
    expect IndexDefect:
      var arr = [1, 2, 3]
      let _ = arr[10]
```

## Check Macros

```nim
import unittest

suite "Check Examples":
  test "basic check":
    check 2 + 2 == 4
    check "hello".len == 5
    check @[1, 2, 3].len > 0

  test "check with message":
    let x = 5
    check(x > 0, "x should be positive")

  test "require":
    # require stops test immediately if false
    require @[1, 2, 3].len > 0
    check @[1, 2, 3][0] == 1

  test "checkFails - expect false":
    check not (1 == 2)
    check not "".len > 0

  test "check sequences":
    let s = @[1, 2, 3, 4, 5]
    check s.len == 5
    check s[0] == 1
    check s[^1] == 5
    check 3 in s
    check 6 notin s

  test "check strings":
    let msg = "Hello, World!"
    check msg.startsWith("Hello")
    check msg.endsWith("World!")
    check "World" in msg
    check msg.len == 13

# Custom assertion helper
template checkApprox(a, b: float, eps: float = 1e-6) =
  if abs(a - b) > eps:
    checkpoint("Values not approximately equal: " & $a & " vs " & $b)
    fail()

suite "Float Tests":
  test "approximation":
    checkApprox(0.1 + 0.2, 0.3, 1e-10)
    checkApprox(sin(PI), 0.0, 1e-10)
    checkApprox(cos(0.0), 1.0, 1e-10)
```

## Property-based Testing

```nim
import unittest, random

# Property-based testing
proc genRandomInts(count: int, lo, hi: int): seq[int] =
  result = newSeq[int](count)
  for i in 0..<count:
    result[i] = rand(lo..hi)

suite "Property Tests":
  test "sort is idempotent":
    for _ in 0..9:
      var arr = genRandomInts(20, -100, 100)
      arr.sort()
      let sorted1 = arr
      arr.sort()
      let sorted2 = arr
      check sorted1 == sorted2

  test "sort preserves elements":
    for _ in 0..9:
      let original = genRandomInts(20, -100, 100)
      var arr = original
      arr.sort()
      check arr.len == original.len
      for x in original:
        check x in arr

  test "reverse twice is identity":
    for _ in 0..9:
      let original = genRandomInts(10, 0, 100)
      var arr = original
      arr.reverse()
      arr.reverse()
      check arr == original

  test "max after sort is last":
    for _ in 0..9:
      var arr = genRandomInts(10, 1, 100)
      let maxVal = arr.max()
      arr.sort()
      check arr[^1] == maxVal

  test "sum is commutative":
    for _ in 0..9:
      let arr = genRandomInts(10, 1, 10)
      check arr.foldl(a + b) == arr.reversed().foldl(a + b)
```

## Mocking and Stubs

```nim
import unittest

# Interface for dependency injection
type
  Storage = concept s
    s.save(string, string) is void
    s.load(string) is string

# Real implementation
type FileStorage = object
  basePath: string

proc save(fs: FileStorage, key, value: string) =
  writeFile(fs.basePath & "/" & key, value)

proc load(fs: FileStorage, key: string): string =
  readFile(fs.basePath & "/" & key)

# Mock implementation for testing
type MockStorage = object
  data: Table[string, string]
  saveCallCount: int
  loadCallCount: int

proc save(ms: var MockStorage, key, value: string) =
  ms.data[key] = value
  inc ms.saveCallCount

proc load(ms: var MockStorage, key: string): string =
  inc ms.loadCallCount
  if key in ms.data: ms.data[key]
  else: raise newException(KeyError, "Key not found: " & key)

# Service using the storage
type UserService = object
  storage: MockStorage

proc saveUser(svc: var UserService, name, email: string) =
  svc.storage.save("user:" & name, email)

proc getUser(svc: var UserService, name: string): string =
  svc.storage.load("user:" & name)

suite "User Service Tests":
  var svc: UserService

  setup:
    svc = UserService(storage: MockStorage(data: initTable[string, string]()))

  test "save user":
    svc.saveUser("Alice", "alice@example.com")
    check svc.storage.saveCallCount == 1
    check svc.storage.data["user:Alice"] == "alice@example.com"

  test "get user":
    svc.saveUser("Bob", "bob@example.com")
    let email = svc.getUser("Bob")
    check email == "bob@example.com"
    check svc.storage.loadCallCount == 1

  test "user not found":
    expect KeyError:
      discard svc.getUser("NonExistent")
```

## Snapshot Testing

```nim
import unittest, os, strutils

proc approvalTest(testName, actual: string) =
  let approvedFile = "tests/approved/" & testName & ".approved.txt"
  let receivedFile = "tests/received/" & testName & ".received.txt"
  
  createDir("tests/approved")
  createDir("tests/received")
  
  writeFile(receivedFile, actual)
  
  if not fileExists(approvedFile):
    writeFile(approvedFile, actual)
    echo "New approval created: ", approvedFile
    return
  
  let approved = readFile(approvedFile)
  if actual != approved:
    echo "MISMATCH for ", testName
    echo "Expected:\n", approved
    echo "Got:\n", actual
    check actual == approved

# Example
proc generateReport(data: seq[(string, int)]): string =
  result = "=== Report ===\n"
  for (name, value) in data:
    result &= name & ": " & $value & "\n"
  result &= "Total: " & $data.foldl(a + b[1], 0) & "\n"

suite "Snapshot Tests":
  test "report format":
    let data = @[("Alpha", 10), ("Beta", 20), ("Gamma", 30)]
    let report = generateReport(data)
    approvalTest("basic_report", report)
```

## Integration Tests

```nim
import unittest, asyncdispatch, asynchttpclient, json

# Integration test with real HTTP (use httpbin.org for testing)
suite "HTTP Integration Tests":
  test "GET request":
    proc doTest() {.async.} =
      let client = newAsyncHttpClient()
      defer: client.close()
      
      let resp = await client.get("https://httpbin.org/get")
      check resp.status == "200 OK"
      
      let body = parseJson(await resp.body)
      check body.kind == JObject
      check "url" in body
    
    waitFor doTest()

  test "POST request":
    proc doPost() {.async.} =
      let client = newAsyncHttpClient()
      defer: client.close()
      
      client.headers = newHttpHeaders({"Content-Type": "application/json"})
      let body = $(%*{"key": "value"})
      
      let resp = await client.post("https://httpbin.org/post", body)
      check resp.status == "200 OK"
    
    waitFor doPost()
```

## Test Coverage Report

```bash
# Compile with coverage
nim c --debugger:native -d:coverage myapp.nim

# Run tests
./myapp

# Generate coverage report
lcov --capture --directory . --output-file coverage.info
genhtml coverage.info --output-directory coverage-report

# View coverage
firefox coverage-report/index.html
```

## Practical: TDD Example

```nim
# Test-Driven Development example
# Write tests FIRST, then implement

import unittest

# 1. Write tests first
suite "Calculator TDD":
  test "add two numbers":
    check add(3, 4) == 7
    check add(-1, 1) == 0
    check add(0, 0) == 0

  test "subtract":
    check subtract(10, 3) == 7
    check subtract(0, 5) == -5

  test "multiply":
    check multiply(3, 4) == 12
    check multiply(-2, 3) == -6
    check multiply(0, 100) == 0

  test "divide":
    check divide(10, 2) == 5.0
    check divide(7, 2) == 3.5

  test "divide by zero":
    expect DivByZeroDefect:
      discard divide(10, 0)

  test "power":
    check power(2, 10) == 1024
    check power(3, 3) == 27
    check power(5, 0) == 1

# 2. Implement to make tests pass
proc add(a, b: int): int = a + b
proc subtract(a, b: int): int = a - b
proc multiply(a, b: int): int = a * b
proc divide(a, b: int): float =
  if b == 0: raise newException(DivByZeroDefect, "Cannot divide by zero")
  float(a) / float(b)
proc power(base, exp: int): int =
  if exp == 0: return 1
  result = 1
  for i in 0..<exp:
    result *= base

# 3. Run tests -> all pass!
```

## สรุป Part 24

ในบทนี้เราได้เรียนรู้:
- ✅ unittest module
- ✅ check, require, expect macros
- ✅ suite, test, setup, teardown
- ✅ Property-based testing
- ✅ Mocking และ Dependency Injection
- ✅ Snapshot/approval testing
- ✅ Integration tests
- ✅ TDD workflow

---

**Previous**: [Part 23 - Compile-time](part23_compiletime.md)
**Next**: [Part 25 - Performance](part25_performance.md)
