# Part 18: Templates และ Macros - Metaprogramming

## Templates

```nim
# Template - compile-time code substitution
template square(x: untyped): untyped =
  x * x

echo square(5)       # 25
echo square(3.14)    # 9.8596
echo square(2 + 3)   # 25 (2+3=5, then 5*5)

# Template กับ side effects
template swap(a, b: untyped): untyped =
  let temp = a
  a = b
  b = temp

var x = 10
var y = 20
swap(x, y)
echo x, " ", y  # 20 10

# Template สำหรับ DSL
template assertEq(a, b: untyped, msg: string = "") =
  if a != b:
    when msg.len > 0:
      echo "FAIL: Expected ", a, " == ", b, ". ", msg
    else:
      echo "FAIL: Expected ", a, " == ", b
  else:
    echo "PASS"

assertEq(2 + 2, 4)
assertEq(2 + 2, 5, "Math is broken!")

# Template กับ multiple statements
template withFile(filename: string, body: untyped): untyped =
  let f = open(filename, fmWrite)
  try:
    body
  finally:
    f.close()

withFile("output.txt"):
  f.writeLine("Hello")
  f.writeLine("World")

# Template สำหรับ timing
import times
template bench(name: string, body: untyped): untyped =
  let start = cpuTime()
  body
  let elapsed = (cpuTime() - start) * 1000
  echo name, ": ", elapsed, "ms"

bench("Fibonacci"):
  proc fib(n: int): int =
    if n <= 1: n else: fib(n-1) + fib(n-2)
  echo fib(35)

# Template สำหรับ logging
template log(level: string, msg: untyped): untyped =
  when defined(logging):
    echo "[", level, "] ", msg

log("INFO", "Application started")
log("DEBUG", "Processing item " & $i)  # only compiled if -d:logging
```

## Macros เบื้องต้น

```nim
import macros

# Macro เบื้องต้น - inspect AST
macro dumpAST(x: untyped): untyped =
  echo x.treeRepr()
  x  # return unchanged

dumpAST:
  let x = 1 + 2

# Macro สร้าง code
macro repeat(n: static int, body: untyped): untyped =
  result = newStmtList()
  for i in 0..<n:
    result.add(body)

repeat(3):
  echo "Hello!"

# Output:
# Hello!
# Hello!
# Hello!

# Macro กับ AST manipulation
macro debugExpr(x: untyped): untyped =
  let exprStr = x.toStrLit()
  result = quote do:
    let value = `x`
    echo `exprStr`, " = ", value
    value

let result = debugExpr(2 + 3 * 4)
# Output: 2 + 3 * 4 = 14
```

## Advanced Templates

```nim
# Template สำหรับ OOP mixin
template makeAccessors(T: typedesc, field: untyped, FieldType: typedesc): untyped =
  proc `field`*(obj: T): FieldType = obj.field
  proc `field=`*(obj: var T, val: FieldType) = obj.field = val

type
  Person = object
    name: string
    age: int

makeAccessors(Person, name, string)
makeAccessors(Person, age, int)

var p = Person(name: "Alice", age: 30)
echo p.name   # Alice
p.age = 31
echo p.age    # 31

# Template สำหรับ Builder Pattern
template builder(T: typedesc, body: untyped): untyped =
  block:
    var obj: T
    body
    obj

type Config = object
  host: string
  port: int
  debug: bool

let cfg = builder(Config):
  obj.host = "localhost"
  obj.port = 8080
  obj.debug = true

echo cfg.host   # localhost
echo cfg.port   # 8080
```

## Practical Macros

```nim
import macros

# Macro สร้าง getter/setter
macro property(T: typedesc, name: untyped, PropType: typedesc): untyped =
  let getterName = name
  let setterName = ident($name & "=")
  let fieldName = ident("_" & $name)
  
  result = quote do:
    proc `getterName`*(obj: `T`): `PropType` = obj.`fieldName`
    proc `setterName`*(obj: var `T`, val: `PropType`) = obj.`fieldName` = val

# Macro สำหรับ enum iteration
macro enumValues(E: typedesc): untyped =
  let typ = E.getTypeImpl()
  result = newTree(nnkBracket)
  for i, field in typ[1..^1]:
    result.add(newDotExpr(E, field))

# Macro สร้าง test framework
macro test(name: string, body: untyped): untyped =
  quote do:
    block:
      echo "TEST: ", `name`
      try:
        `body`
        echo "  PASS"
      except AssertionDefect as e:
        echo "  FAIL: ", e.msg
      except:
        echo "  ERROR: ", getCurrentExceptionMsg()

test("Addition"):
  assert 1 + 1 == 2
  assert 2 + 2 == 4
  assert 3 + 3 == 6

test("String operations"):
  assert "hello".len == 5
  assert "HELLO" == "hello".toUpper()

test("Intentional fail"):
  assert 1 + 1 == 3  # This will fail
```

## DSL Creation

```nim
# HTML DSL
import strutils

type HtmlNode = object
  tag: string
  attrs: seq[(string, string)]
  children: seq[string]
  content: string

proc `$`(n: HtmlNode): string =
  var attrStr = ""
  for (k, v) in n.attrs:
    attrStr &= " " & k & "=\"" & v & "\""
  
  if n.children.len > 0 or n.content.len > 0:
    let inner = n.content & n.children.join("")
    "<" & n.tag & attrStr & ">" & inner & "</" & n.tag & ">"
  else:
    "<" & n.tag & attrStr & " />"

template html(body: untyped): string =
  var result = "<!DOCTYPE html>\n<html>\n"
  body
  result & "</html>"

# SQL DSL
type Query = object
  table: string
  conditions: seq[string]
  columns: seq[string]
  limitVal: int

proc select(cols: varargs[string]): Query =
  Query(columns: @cols)

proc `from`(q: Query, table: string): Query =
  Query(table: table, columns: q.columns, conditions: q.conditions, limitVal: q.limitVal)

proc where(q: Query, cond: string): Query =
  Query(table: q.table, columns: q.columns, 
        conditions: q.conditions & @[cond], limitVal: q.limitVal)

proc limit(q: Query, n: int): Query =
  Query(table: q.table, columns: q.columns, 
        conditions: q.conditions, limitVal: n)

proc build(q: Query): string =
  let cols = if q.columns.len > 0: q.columns.join(", ") else: "*"
  result = "SELECT " & cols & " FROM " & q.table
  if q.conditions.len > 0:
    result &= " WHERE " & q.conditions.join(" AND ")
  if q.limitVal > 0:
    result &= " LIMIT " & $q.limitVal

# Usage
let query = select("name", "age", "email")
  .from("users")
  .where("age > 18")
  .where("active = true")
  .limit(10)
  .build()

echo query
# SELECT name, age, email FROM users WHERE age > 18 AND active = true LIMIT 10
```

## Practical: Test Framework

```nim
# nimtest.nim - Mini test framework using macros

import macros, strutils, times

type
  TestResult = object
    name: string
    passed: bool
    message: string
    duration: float

var allTests: seq[TestResult] = @[]

template suite(suiteName: string, body: untyped) =
  echo "\n=== Suite: ", suiteName, " ==="
  body

template test(testName: string, body: untyped) =
  let start = cpuTime()
  var passed = true
  var msg = ""
  
  try:
    body
  except AssertionDefect as e:
    passed = false
    msg = e.msg
  except:
    passed = false
    msg = getCurrentExceptionMsg()
  
  let duration = (cpuTime() - start) * 1000
  
  allTests.add(TestResult(
    name: testName,
    passed: passed,
    message: msg,
    duration: duration
  ))
  
  if passed:
    echo "  ✓ ", testName, " (", duration.formatFloat(ffDecimal, 2), "ms)"
  else:
    echo "  ✗ ", testName
    if msg.len > 0:
      echo "    ", msg

template check(condition: bool, msg: string = "") =
  if not condition:
    let failMsg = if msg.len > 0: msg
                  else: "Check failed"
    raise newException(AssertionDefect, failMsg)

template checkEq(a, b: untyped) =
  if a != b:
    raise newException(AssertionDefect, 
      "Expected " & $a & " == " & $b)

template checkNe(a, b: untyped) =
  if a == b:
    raise newException(AssertionDefect,
      "Expected " & $a & " != " & $b)

proc printSummary() =
  let total = allTests.len
  let passed = allTests.filterIt(it.passed).len
  let failed = total - passed
  
  echo "\n=== Test Summary ==="
  echo "Total:  ", total
  echo "Passed: ", passed
  echo "Failed: ", failed
  
  if failed > 0:
    echo "\nFailed tests:"
    for t in allTests:
      if not t.passed:
        echo "  ✗ ", t.name, ": ", t.message

# Use the framework
suite("Math Tests"):
  test("Addition"):
    checkEq(1 + 1, 2)
    checkEq(2 + 2, 4)
  
  test("Multiplication"):
    checkEq(3 * 4, 12)
    check(5 * 5 == 25)
  
  test("Division"):
    checkEq(10 div 2, 5)
    checkNe(10 div 3, 3)

suite("String Tests"):
  test("Concatenation"):
    checkEq("Hello" & " " & "World", "Hello World")
  
  test("Length"):
    checkEq("Hello".len, 5)
  
  test("Uppercase"):
    checkEq("hello".toUpper(), "HELLO")

printSummary()
```

## แบบฝึกหัด Part 18

### แบบฝึกหัดที่ 1: Memoize Template
```nim
template memoize(procName: untyped, returnType: typedesc, body: untyped): untyped =
  var cache = initTable[int, returnType]()
  proc procName(n: int): returnType =
    if n in cache: return cache[n]
    result = body
    cache[n] = result

memoize(fib, int):
  if n <= 1: n
  else: fib(n-1) + fib(n-2)

echo fib(40)  # Fast!
```

## สรุป Part 18

ในบทนี้เราได้เรียนรู้:
- ✅ Templates: compile-time code substitution
- ✅ Templates สำหรับ DSL
- ✅ Templates สำหรับ timing, logging
- ✅ Macros เบื้องต้น: AST inspection
- ✅ Advanced macros: code generation
- ✅ DSL creation: SQL query builder
- ✅ Practical: Mini test framework

---

**Previous**: [Part 17 - Generics](part17_generics.md)
**Next**: [Part 19 - Async/Await](part19_async.md)
