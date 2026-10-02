# Part 14: Error Handling - การจัดการข้อผิดพลาด

## Exceptions

```nim
# try/except/finally
try:
  let n = parseInt("not a number")
  echo "Parsed: ", n
except ValueError as e:
  echo "ValueError: ", e.msg
except IOError as e:
  echo "IOError: ", e.msg
except:
  echo "Unknown error: ", getCurrentExceptionMsg()
finally:
  echo "Cleanup (always runs)"

# raise exception
proc divide(a, b: int): int =
  if b == 0:
    raise newException(DivByZeroDefect, "Cannot divide by zero!")
  a div b

try:
  echo divide(10, 2)   # 5
  echo divide(10, 0)   # raises
except DivByZeroDefect as e:
  echo "Error: ", e.msg

# Exception hierarchy
# Exception (base)
# ├── Defect (programming errors, unrecoverable)
# │   ├── AssertionDefect
# │   ├── DivByZeroDefect
# │   ├── IndexDefect
# │   ├── OverflowDefect
# │   └── AccessViolationDefect
# └── CatchableError (recoverable errors)
#     ├── IOError
#     ├── ValueError
#     ├── KeyError
#     ├── OSError
#     └── ... custom errors
```

## Custom Exceptions

```nim
# Custom exception types
type
  AppError = object of CatchableError
  
  ValidationError = object of AppError
    field: string
  
  DatabaseError = object of AppError
    query: string
    code: int
  
  NetworkError = object of AppError
    url: string
    statusCode: int

# Create and raise custom exceptions
proc validateEmail(email: string) =
  if not email.contains('@'):
    var e = newException(ValidationError, "Invalid email format")
    e.field = "email"
    raise e
  if email.len < 5:
    var e = newException(ValidationError, "Email too short")
    e.field = "email"
    raise e

proc queryDatabase(sql: string) =
  # Simulate DB error
  if sql.contains("DROP"):
    var e = newException(DatabaseError, "Dangerous query blocked")
    e.query = sql
    e.code = 403
    raise e

# Catch specific exceptions
try:
  validateEmail("notvalid")
except ValidationError as e:
  echo "Validation failed: ", e.msg
  echo "Field: ", e.field

try:
  queryDatabase("DROP TABLE users")
except DatabaseError as e:
  echo "DB Error: ", e.msg
  echo "Query: ", e.query
  echo "Code: ", e.code
except AppError as e:
  echo "App Error: ", e.msg
```

## Result Type Pattern

```nim
# Result type - functional error handling
type
  Result[T, E] = object
    case ok: bool
    of true:  value: T
    of false: error: E

proc ok[T, E](value: T): Result[T, E] =
  Result[T, E](ok: true, value: value)

proc err[T, E](error: E): Result[T, E] =
  Result[T, E](ok: false, error: error)

# Usage
proc parseInt2(s: string): Result[int, string] =
  try:
    ok[int, string](parseInt(s))
  except ValueError as e:
    err[int, string](e.msg)

let r1 = parseInt2("42")
let r2 = parseInt2("not a number")

if r1.ok:
  echo "Parsed: ", r1.value
else:
  echo "Error: ", r1.error

if r2.ok:
  echo "Parsed: ", r2.value
else:
  echo "Error: ", r2.error

# Using std/options
import options

proc safeDivide(a, b: int): Option[int] =
  if b == 0: none(int)
  else: some(a div b)

let res = safeDivide(10, 2)
echo res.get()  # 5

let res2 = safeDivide(10, 0)
echo res2.isSome  # false
echo res2.get(0)  # 0 (default)
```

## Result Type แบบสมบูรณ์

```nim
# Implementing Result type properly
type
  ResultKind = enum rkOk, rkErr
  
  Result[T, E] = object
    case kind: ResultKind
    of rkOk:  val: T
    of rkErr: err: E

template ok[T, E](v: T): Result[T, E] =
  Result[T, E](kind: rkOk, val: v)

template err[T, E](e: E): Result[T, E] =
  Result[T, E](kind: rkErr, err: e)

proc isOk[T, E](r: Result[T, E]): bool = r.kind == rkOk
proc isErr[T, E](r: Result[T, E]): bool = r.kind == rkErr
proc get[T, E](r: Result[T, E]): T =
  assert r.isOk, "Result is error: " & $r.err
  r.val
proc getErr[T, E](r: Result[T, E]): E =
  assert r.isErr, "Result is ok"
  r.err

proc map[T, U, E](r: Result[T, E], f: proc(x: T): U): Result[U, E] =
  if r.isOk: ok[U, E](f(r.val))
  else: err[U, E](r.err)

proc flatMap[T, U, E](r: Result[T, E], f: proc(x: T): Result[U, E]): Result[U, E] =
  if r.isOk: f(r.val)
  else: err[U, E](r.err)

# Chain of operations
proc readNumber(s: string): Result[int, string] =
  try: ok[int, string](parseInt(s))
  except ValueError: err[int, string]("Not a number: " & s)

proc validatePositive(n: int): Result[int, string] =
  if n > 0: ok[int, string](n)
  else: err[int, string]("Must be positive, got: " & $n)

proc computeSqrt(n: int): Result[float, string] =
  import math
  ok[float, string](sqrt(float(n)))

# Chain
let chainResult = readNumber("25")
  .flatMap(validatePositive)
  .flatMap(computeSqrt)

if chainResult.isOk:
  echo "Result: ", chainResult.get()
else:
  echo "Error: ", chainResult.getErr()
```

## Assertions

```nim
# assert - runtime check
assert 2 + 2 == 4
assert "hello".len == 5, "String length should be 5"

proc divide(a, b: int): int =
  assert b != 0, "Divisor cannot be zero"
  a div b

# doAssert - always checked (even in release)
doAssert 2 + 2 == 4

# debug only assertion
# In release mode, assert is compiled away
when not defined(release):
  discard  # assert expensiveCheck(), "Debug check failed"

# require/ensure patterns
proc factorial(n: int): int =
  # Precondition
  assert n >= 0, "n must be non-negative"
  
  result = 1
  for i in 2..n:
    result *= i
  
  # Postcondition
  assert result >= 1, "Result should be at least 1"
```

## Practical: Safe Parser

```nim
# safe_parser.nim - Parser with comprehensive error handling

import strutils, options, strformat

type
  ParseError = object of CatchableError
    position: int
    expected: string

  Token = object
    kind: string
    value: string
    pos: int

  Parser = object
    input: string
    pos: int
    tokens: seq[Token]

proc newParser(input: string): Parser =
  Parser(input: input, pos: 0)

proc peek(p: Parser): char =
  if p.pos < p.input.len: p.input[p.pos]
  else: '\0'

proc consume(p: var Parser): char =
  result = p.peek()
  inc p.pos

proc skipWhitespace(p: var Parser) =
  while p.peek() in {' ', '\t', '\n', '\r'}:
    discard p.consume()

proc parseNumber(p: var Parser): Option[float] =
  p.skipWhitespace()
  var numStr = ""
  
  if p.peek() == '-':
    numStr &= p.consume()
  
  while p.peek().isDigit():
    numStr &= p.consume()
  
  if p.peek() == '.':
    numStr &= p.consume()
    while p.peek().isDigit():
      numStr &= p.consume()
  
  if numStr.len == 0 or numStr == "-":
    return none(float)
  
  try:
    some(parseFloat(numStr))
  except ValueError:
    none(float)

proc parseString(p: var Parser): Option[string] =
  p.skipWhitespace()
  if p.peek() != '"':
    return none(string)
  
  discard p.consume()  # consume "
  var s = ""
  
  while p.peek() != '"' and p.pos < p.input.len:
    if p.peek() == '\\':
      discard p.consume()
      case p.peek()
      of 'n':  s &= '\n'; discard p.consume()
      of 't':  s &= '\t'; discard p.consume()
      of '"':  s &= '"';  discard p.consume()
      of '\\': s &= '\\'; discard p.consume()
      else:
        var e = newException(ParseError, "Unknown escape sequence")
        e.position = p.pos
        e.expected = "valid escape"
        raise e
    else:
      s &= p.consume()
  
  if p.peek() != '"':
    var e = newException(ParseError, "Unterminated string")
    e.position = p.pos
    e.expected = "\""
    raise e
  
  discard p.consume()  # consume closing "
  some(s)

# Test the parser
var p = newParser("""{"name": "Alice", "age": 30}""")

try:
  p.skipWhitespace()
  echo "Peek: ", p.peek()
  
  let strResult = parseString(p)
  echo "String? ", strResult
  
except ParseError as e:
  echo &"Parse error at position {e.position}: {e.msg}"
  echo &"Expected: {e.expected}"
```

## Error Handling Best Practices

```nim
# 1. Use specific exception types
proc readConfig(path: string): string =
  try:
    readFile(path)
  except IOError as e:
    raise newException(IOError, "Cannot read config: " & e.msg)

# 2. Provide context when re-raising
# proc processUser(id: int) =
#   try:
#     let data = fetchUser(id)
#     validateUser(data)
#   except DatabaseError as e:
#     raise newException(AppError, 
#       "Failed to process user " & $id & ": " & e.msg)

# 3. Use defer for cleanup
proc safeOperation(path: string) =
  let f = open(path, fmRead)
  defer: f.close()
  # If any exception occurs, file will still be closed
  echo f.readAll()

# 4. Validate at boundaries
type User = object
  name: string
  age: int
  email: string

proc createUser(name: string, age: int, email: string): User =
  # Validate input
  if name.len == 0:
    raise newException(ValueError, "Name required")
  if age < 0 or age > 150:
    raise newException(ValueError, "Invalid age: " & $age)
  if "@" notin email:
    raise newException(ValueError, "Invalid email: " & email)
  
  User(name: name, age: age, email: email)

# 6. Use Option/Result for expected failures
proc findItem(items: seq[string], target: string): Option[int] =
  import options
  let idx = items.find(target)
  if idx >= 0: some(idx)
  else: none(int)
```

## แบบฝึกหัด Part 14

### แบบฝึกหัดที่ 1: Validated Input
```nim
type
  ValidationResult = object
    valid: bool
    errors: seq[string]

proc validate(name: string, age: int, email: string): ValidationResult =
  var errors: seq[string] = @[]
  
  if name.strip().len == 0:
    errors.add("Name cannot be empty")
  elif name.len > 100:
    errors.add("Name too long (max 100 chars)")
  
  if age < 0: errors.add("Age cannot be negative")
  elif age > 150: errors.add("Age seems unrealistic")
  
  if "@" notin email: errors.add("Email must contain @")
  elif "." notin email.split("@")[^1]: errors.add("Email domain invalid")
  
  ValidationResult(valid: errors.len == 0, errors: errors)

let r = validate("Alice", 25, "alice@example.com")
echo "Valid: ", r.valid
if not r.valid:
  for e in r.errors:
    echo "  Error: ", e
```

## สรุป Part 14

ในบทนี้เราได้เรียนรู้:
- ✅ try/except/finally
- ✅ Custom exceptions
- ✅ Exception hierarchy
- ✅ Result type pattern
- ✅ Option type
- ✅ Assertions
- ✅ Error handling best practices
- ✅ Practical: Safe parser

---

**Previous**: [Part 13 - File I/O](part13_file_io.md)
**Next**: [Part 15 - Modules](part15_modules.md)
