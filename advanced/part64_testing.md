# Part 64 - Testing Strategies in Nim

## บทนำ

การเขียนเทสที่ดีใน Nim สำหรับ code ที่เชื่อถือได้และบำรุงง่าย

---

## 1. Built-in Test Framework

```nim
# test_basics.nim
# ใช้ unittest module ใน Nim standard library

import std/[unittest, strformat, times]

# --- Basic test structure ---

suite "String Operations":
  
  test "concatenation":
    let result = "hello" & " " & "world"
    check result == "hello world"
  
  test "uppercase":
    check "hello".toUpperAscii() == "HELLO"
  
  test "split and join":
    let parts = "a,b,c".split(",")
    check parts == @["a", "b", "c"]
    check parts.join("|") == "a|b|c"
  
  test "string contains":
    check "hello world".contains("world")
    check not "hello world".contains("xyz")

suite "Math Operations":
  
  setup:
    # Runs before each test in this suite
    let tolerance = 1e-6
  
  teardown:
    # Runs after each test in this suite
    discard
  
  test "integer arithmetic":
    check 2 + 2 == 4
    check 10 div 3 == 3
    check 10 mod 3 == 1
  
  test "float arithmetic":
    let result = 1.0 / 3.0
    check abs(result - 0.333333) < 0.001
  
  test "overflow detection":
    expect OverflowDefect:
      discard high(int8) + 1.int8

# --- Property-based testing concepts ---

import std/random

proc generateRandomString*(len: int): string =
  randomize()
  result = ""
  for _ in 0..<len:
    result.add(chr(rand(25) + ord('a')))

suite "Property Tests":
  
  test "reverse of reverse is identity":
    for _ in 0..<100:
      let s = generateRandomString(rand(1..20))
      check s.reversed() == s.reversed().reversed().reversed()  # Wait...
      check s == s.reversed().reversed()  # Correct
  
  test "sort is stable and produces sorted output":
    for _ in 0..<50:
      var arr: seq[int] = @[]
      for _ in 0..<rand(2..20):
        arr.add(rand(100))
      
      arr.sort()
      
      for i in 0..<arr.len - 1:
        check arr[i] <= arr[i + 1]
```

---

## 2. Test Fixtures and Helpers

```nim
# test_helpers.nim
# Reusable test utilities

import std/[unittest, json, times, options, strformat]

# --- Custom check helpers ---

template checkClose*(a, b: float, delta = 1e-6) =
  ## Check floating point equality within tolerance
  check abs(a - b) < delta, &"Expected {a} ≈ {b} (delta={delta})"

template checkLen*(s: typed, n: int) =
  ## Check sequence/string length
  check s.len == n, &"Expected length {n}, got {s.len}"

template checkJson*(actual: JsonNode, expected: JsonNode) =
  ## Check JSON equality with better error messages
  if actual != expected:
    echo "Expected: ", $expected
    echo "Actual:   ", $actual
    check actual == expected

template checkNone*[T](val: Option[T]) =
  check val.isNone, &"Expected None, got Some({val.get()})"

template checkSome*[T](val: Option[T]) =
  check val.isSome, "Expected Some, got None"

template checkSome*[T](val: Option[T], expected: T) =
  check val.isSome, "Expected Some, got None"
  check val.get() == expected

# --- Test data builders ---

type
  TestUserBuilder* = object
    id*: int
    email*: string
    username*: string
    role*: string

proc testUser*(id = 1, email = "test@example.com",
               username = "testuser", role = "user"): TestUserBuilder =
  TestUserBuilder(id: id, email: email, username: username, role: role)

proc withEmail*(b: TestUserBuilder, email: string): TestUserBuilder =
  result = b
  result.email = email

proc withRole*(b: TestUserBuilder, role: string): TestUserBuilder =
  result = b
  result.role = role

# --- Timing assertions ---

template checkFaster*(maxMs: float, body: untyped) =
  let start = cpuTime()
  body
  let elapsed = (cpuTime() - start) * 1000
  check elapsed < maxMs, &"Expected under {maxMs}ms, took {elapsed:.2f}ms"

# --- Database test utilities ---

type
  TestDB* = object
    # In-memory SQLite for testing
    tableName*: string
    data*: seq[seq[string]]

proc newTestDB*(): TestDB =
  TestDB(data: @[])

proc seedData*(db: var TestDB, rows: seq[seq[string]]) =
  db.data.add(rows)

proc reset*(db: var TestDB) =
  db.data = @[]

# Usage example
suite "Test Helpers Demo":
  
  test "close floats":
    checkClose(3.14159, 3.14159265, 0.001)
  
  test "option checks":
    let x: Option[int] = some(42)
    checkSome(x, 42)
    
    let y: Option[string] = none(string)
    checkNone(y)
  
  test "performance check":
    checkFaster(100.0):  # Must complete in 100ms
      var sum = 0
      for i in 0..100000:
        sum += i
      check sum > 0
  
  test "builder pattern":
    let adminUser = testUser().withRole("admin").withEmail("admin@example.com")
    check adminUser.role == "admin"
    check adminUser.email == "admin@example.com"
```

---

## 3. Mocking and Stubbing

```nim
# test_mocking.nim
# Mock dependencies for unit tests

import std/[unittest, tables, json, options]

# --- Interface-based design for testability ---

type
  # Define interfaces as concepts or object types
  UserRepository* = concept r
    r.findById(0) is Option[JsonNode]
    r.save(newJObject()) is JsonNode
  
  EmailService* = concept e
    e.sendEmail("", "", "") is bool

# --- Mock implementations ---

type
  MockUserRepository* = object
    users*: Table[int, JsonNode]
    saveCallCount*: int
    findCallLog*: seq[int]

proc findById*(mock: var MockUserRepository, id: int): Option[JsonNode] =
  mock.findCallLog.add(id)
  if id in mock.users:
    some(mock.users[id])
  else:
    none(JsonNode)

proc save*(mock: var MockUserRepository, user: JsonNode): JsonNode =
  inc mock.saveCallCount
  let id = user["id"].getInt()
  mock.users[id] = user
  user

type
  MockEmailService* = object
    sentEmails*: seq[tuple[to, subject, body: string]]
    shouldFail*: bool

proc sendEmail*(mock: var MockEmailService, to, subject, body: string): bool =
  if mock.shouldFail:
    return false
  mock.sentEmails.add((to, subject, body))
  true

# --- Service being tested ---

type
  UserServiceT*[R, E] = object
    repo*: R
    email*: E

proc newUserService*[R, E](repo: R, email: E): UserServiceT[R, E] =
  UserServiceT[R, E](repo: repo, email: email)

proc getUser*[R, E](svc: var UserServiceT[R, E], id: int): Option[JsonNode] =
  svc.repo.findById(id)

proc registerUser*[R, E](svc: var UserServiceT[R, E], email, name: string): JsonNode =
  let user = %*{"id": 1, "email": email, "name": name}
  let saved = svc.repo.save(user)
  discard svc.email.sendEmail(email, "Welcome!", &"Hi {name}, welcome!")
  saved

# --- Tests with mocks ---

suite "UserService Tests (Mocked)":
  
  var repo: MockUserRepository
  var emailSvc: MockEmailService
  
  setup:
    repo = MockUserRepository(
      users: initTable[int, JsonNode]()
    )
    emailSvc = MockEmailService()
  
  test "getUser returns existing user":
    repo.users[42] = %*{"id": 42, "name": "Alice"}
    var svc = newUserService(repo, emailSvc)
    
    let user = svc.getUser(42)
    
    checkSome(user)
    check user.get()["name"].getStr() == "Alice"
    check repo.findCallLog == @[42]
  
  test "getUser returns none for missing user":
    var svc = newUserService(repo, emailSvc)
    
    let user = svc.getUser(999)
    
    checkNone(user)
  
  test "registerUser saves and sends welcome email":
    var svc = newUserService(repo, emailSvc)
    
    let user = svc.registerUser("alice@test.com", "Alice")
    
    check user["email"].getStr() == "alice@test.com"
    check repo.saveCallCount == 1
    check emailSvc.sentEmails.len == 1
    check emailSvc.sentEmails[0].to == "alice@test.com"
    check "Welcome!" in emailSvc.sentEmails[0].subject
  
  test "registerUser continues even if email fails":
    emailSvc.shouldFail = true
    var svc = newUserService(repo, emailSvc)
    
    let user = svc.registerUser("bob@test.com", "Bob")
    
    check user["email"].getStr() == "bob@test.com"
    check repo.saveCallCount == 1
    check emailSvc.sentEmails.len == 0  # Email failed but user was saved
```

---

## 4. Integration Tests

```nim
# test_integration.nim
# Integration tests with real DB (SQLite for testing)

import std/[unittest, db_sqlite, options, os, strformat]

type
  TestFixture* = object
    db*: DbConn
    dbPath*: string

proc newTestFixture*(): TestFixture =
  let dbPath = getTempDir() / &"test_{getCurrentTime()}.db"
  result = TestFixture(
    db: open(dbPath, "", "", ""),
    dbPath: dbPath
  )
  
  # Setup schema
  result.db.exec sql"""
    CREATE TABLE users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      email TEXT UNIQUE NOT NULL,
      username TEXT NOT NULL,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  """
  
  result.db.exec sql"""
    CREATE TABLE todos (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id INTEGER REFERENCES users(id),
      title TEXT NOT NULL,
      status TEXT DEFAULT 'pending',
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  """

proc cleanup*(fixture: TestFixture) =
  fixture.db.close()
  removeFile(fixture.dbPath)

proc createUser*(db: DbConn, email, username: string): int =
  db.exec(sql"INSERT INTO users (email, username) VALUES (?, ?)", email, username)
  parseInt(db.getValue(sql"SELECT last_insert_rowid()"))

proc createTodo*(db: DbConn, userId: int, title: string): int =
  db.exec(sql"INSERT INTO todos (user_id, title) VALUES (?, ?)", userId, title)
  parseInt(db.getValue(sql"SELECT last_insert_rowid()"))

proc getTodoCount*(db: DbConn, userId: int): int =
  parseInt(db.getValue(sql"SELECT COUNT(*) FROM todos WHERE user_id = ?", userId))

# --- Integration tests ---

suite "Todo Integration Tests":
  var fixture: TestFixture
  
  setup:
    fixture = newTestFixture()
  
  teardown:
    fixture.cleanup()
  
  test "create and retrieve todo":
    let userId = createUser(fixture.db, "alice@test.com", "alice")
    let todoId = createTodo(fixture.db, userId, "Buy groceries")
    
    let row = fixture.db.getRow(sql"SELECT * FROM todos WHERE id = ?", todoId)
    check row[2] == "Buy groceries"  # title
    check row[3] == "pending"         # status
  
  test "cascade delete removes todos":
    let userId = createUser(fixture.db, "bob@test.com", "bob")
    discard createTodo(fixture.db, userId, "Todo 1")
    discard createTodo(fixture.db, userId, "Todo 2")
    
    check getTodoCount(fixture.db, userId) == 2
    
    fixture.db.exec(sql"DELETE FROM users WHERE id = ?", userId)
    
    check getTodoCount(fixture.db, userId) == 0
  
  test "unique email constraint":
    discard createUser(fixture.db, "charlie@test.com", "charlie")
    
    expect DbError:
      discard createUser(fixture.db, "charlie@test.com", "charlie2")
```

---

## 5. Continuous Integration

```yaml
# .github/workflows/test.yml
name: Nim Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: nimtest
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Nim
        uses: jiro4989/setup-nim-action@v1
        with:
          nim-version: stable
      
      - name: Install dependencies
        run: nimble install -y
      
      - name: Run unit tests
        run: nim c -r tests/test_unit.nim
      
      - name: Run integration tests
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_NAME: nimtest
          DB_USER: postgres
          DB_PASS: postgres
        run: nim c -r tests/test_integration.nim
      
      - name: Run with coverage
        run: |
          nim c --coverage:on -r tests/all_tests.nim
          # Generate coverage report
```

---

## สรุป Part 64

| ประเภทเทส | เมื่อใช้ |
|------------|----------|
| Unit tests | สำหรับ logic functions |
| Integration tests | สำหรับ database/service interactions |
| Mock tests | แยก dependencies ออก |
| Property tests | ทดสอบกับ random inputs |
| CI/CD | Automated testing on every push |

### Testing Best Practices

1. **FIRST** — Fast, Isolated, Repeatable, Self-validating, Timely
2. **AAA Pattern** — Arrange, Act, Assert
3. **Test behavior** — ไม่ใช่การทำงานภายใน
4. **One assertion per test** — ผล test อ่านง่าย
5. **Fast feedback loop** — unit tests < 1s, integration < 30s

**Next**: [Part 65 - Distributed Systems Concepts](../advanced/part65_distributed.md)
