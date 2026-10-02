# Part 85 - Advanced Error Handling Patterns

## บทนำ

Error handling ที่ดีทำให้ application:
- Predictable และ debuggable
- ไม่ crash โดยไม่คาดคิด
- มี error context ที่มีประโยชน์

---

## 1. Result Type

```nim
# result_type.nim
# Functional error handling ด้วย Result[T, E]

type
  ResultKind* = enum
    rkOk, rkErr

  Result*[T, E] = object
    case kind*: ResultKind
    of rkOk: value*: T
    of rkErr: error*: E

proc ok*[T, E](value: T): Result[T, E] =
  Result[T, E](kind: rkOk, value: value)

proc err*[T, E](error: E): Result[T, E] =
  Result[T, E](kind: rkErr, error: error)

proc isOk*[T, E](r: Result[T, E]): bool = r.kind == rkOk
proc isErr*[T, E](r: Result[T, E]): bool = r.kind == rkErr

proc get*[T, E](r: Result[T, E]): T =
  assert r.kind == rkOk, "Called get() on Err result"
  r.value

proc getError*[T, E](r: Result[T, E]): E =
  assert r.kind == rkErr, "Called getError() on Ok result"
  r.error

proc getOrDefault*[T, E](r: Result[T, E], default: T): T =
  if r.isOk: r.value else: default

proc getOrElse*[T, E](r: Result[T, E], fn: proc(e: E): T): T =
  if r.isOk: r.value else: fn(r.error)

proc map*[T, U, E](r: Result[T, E], fn: proc(v: T): U): Result[U, E] =
  if r.isOk: ok[U, E](fn(r.value)) else: err[U, E](r.error)

proc mapErr*[T, E, F](r: Result[T, E], fn: proc(e: E): F): Result[T, F] =
  if r.isOk: ok[T, F](r.value) else: err[T, F](fn(r.error))

proc flatMap*[T, U, E](r: Result[T, E], fn: proc(v: T): Result[U, E]): Result[U, E] =
  if r.isOk: fn(r.value) else: err[U, E](r.error)

proc andThen*[T, U, E](r: Result[T, E], fn: proc(v: T): Result[U, E]): Result[U, E] =
  r.flatMap(fn)

# ? operator equivalent
template `?`*[T, E](r: Result[T, E]): T =
  let res = r
  if res.isErr:
    return err(res.error)
  res.value

# Example domain errors
type
  AppErrorKind* = enum
    aeNotFound
    aeValidation
    aeDatabase
    aeNetwork
    aePermission

  AppError* = object
    kind*: AppErrorKind
    message*: string
    cause*: string

proc notFound*(msg: string): AppError =
  AppError(kind: aeNotFound, message: msg)

proc validation*(msg: string): AppError =
  AppError(kind: aeValidation, message: msg)

proc dbError*(msg: string, cause = ""): AppError =
  AppError(kind: aeDatabase, message: msg, cause: cause)

type
  AppResult*[T] = Result[T, AppError]

# Usage example
proc findUser*(id: int): AppResult[string] =
  if id <= 0:
    return err(validation("User ID must be positive"))
  if id > 1000:
    return err(notFound(&"User {id} not found"))
  ok(&"User_{id}")

proc getUserEmail*(userId: int): AppResult[string] =
  let user = findUser(userId)?
  ok(&"{user.toLowerAscii()}@example.com")

proc processUser*(id: int): AppResult[string] =
  let email = getUserEmail(id)?
  ok(&"Processed email: {email}")

when isMainModule:
  for id in [-1, 5, 1500]:
    let result = processUser(id)
    if result.isOk:
      echo &"Success: {result.get()}"
    else:
      let e = result.getError()
      echo &"Error [{e.kind}]: {e.message}"
```

---

## 2. Error Context Chaining

```nim
# error_context.nim
# Rich error messages ด้วย context chaining

import std/[strformat, sequtils, strutils]

type
  ErrorFrame* = object
    message*: string
    file*: string
    line*: int
    function*: string

  RichError* = ref object of CatchableError
    frames*: seq[ErrorFrame]
    original*: ref Exception

proc newRichError*(msg: string, file = "", line = 0, fn = ""): RichError =
  result = RichError(msg: msg)
  result.frames = @[ErrorFrame(message: msg, file: file, line: line, function: fn)]

proc wrap*(err: ref Exception, context: string, file = "", line = 0, fn = ""): RichError =
  if err of RichError:
    let rich = RichError(err)
    rich.frames.add(ErrorFrame(message: context, file: file, line: line, function: fn))
    return rich
  
  result = RichError(msg: context)
  result.original = err
  result.frames = @[
    ErrorFrame(message: err.msg, file: "", line: 0, function: ""),
    ErrorFrame(message: context, file: file, line: line, function: fn)
  ]

proc formatError*(err: RichError): string =
  var lines = @[&"Error: {err.frames[^1].message}"]
  lines.add("Stack:")
  for i in countdown(err.frames.len - 1, 0):
    let f = err.frames[i]
    var loc = ""
    if f.file.len > 0: loc = &" at {f.file}:{f.line}"
    if f.function.len > 0: loc &= &" in {f.function}"
    lines.add(&"  {i}: {f.message}{loc}")
  lines.join("\n")

# Contextual error macro
template withContext*(context: string, body: untyped): untyped =
  try:
    body
  except CatchableError as e:
    raise wrap(e, context, instantiationInfo().filename,
      instantiationInfo().line, instantiationInfo().column.`$`)

# Example: Deep call chain with context
proc readFile*(path: string): string =
  if not fileExists(path):
    raise newRichError(&"File not found: {path}", instantiationInfo().filename,
      instantiationInfo().line, "readFile")
  readFile(path)

proc parseConfig*(path: string): seq[(string, string)] =
  withContext(&"parsing config from {path}"):
    let content = readFile(path)
    for line in content.splitLines():
      let parts = line.split('=', 1)
      if parts.len == 2:
        result.add((parts[0].strip(), parts[1].strip()))

proc initApp*(configPath: string) =
  withContext("initializing application"):
    let config = parseConfig(configPath)
    echo &"Loaded {config.len} config entries"
```

---

## 3. Panic Recovery

```nim
# panic_recovery.nim
# Graceful panic handling

import std/[strformat, times, os, locks]

type
  PanicHandler* = proc(msg: string, trace: string)

var globalPanicHandler*: PanicHandler
var panicHandlerLock*: Lock
initLock(panicHandlerLock)

proc setPanicHandler*(handler: PanicHandler) =
  acquire(panicHandlerLock)
  globalPanicHandler = handler
  release(panicHandlerLock)

proc defaultPanicHandler*(msg, trace: string) =
  let timestamp = now().format("yyyy-MM-dd HH:mm:ss")
  let logPath = &"/tmp/panic_{timestamp.replace(\" \", \"_\").replace(\":\", \"-\")}.log"
  
  let logContent = &"""
PANIC at {timestamp}
Message: {msg}

Stack trace:
{trace}
"""
  
  try:
    writeFile(logPath, logContent)
    echo &"Panic log written to: {logPath}"
  except: discard
  
  echo logContent

# Safe execution wrapper
proc safely*[T](fn: proc(): T, default: T, onError: proc(e: ref Exception) = nil): T =
  try:
    fn()
  except CatchableError as e:
    if not onError.isNil: onError(e)
    default
  except Defect as e:
    if not onError.isNil: onError(e)
    if not globalPanicHandler.isNil:
      globalPanicHandler(e.msg, getStackTrace(e))
    default

template safeCall*(body: untyped): bool =
  try:
    body
    true
  except CatchableError:
    false

# Retry with exponential backoff
proc withRetry*[T](fn: proc(): T, maxRetries = 3, 
    baseDelayMs = 100, onRetry: proc(attempt: int, e: ref Exception) = nil): T =
  var lastError: ref Exception
  
  for attempt in 0..maxRetries:
    try:
      return fn()
    except CatchableError as e:
      lastError = e
      if attempt < maxRetries:
        if not onRetry.isNil:
          onRetry(attempt + 1, e)
        let delay = baseDelayMs * (1 shl attempt)  # Exponential backoff
        sleep(delay)
  
  raise lastError

# Circuit breaker error types
type
  CircuitOpenError* = object of CatchableError
  TimeoutError* = object of CatchableError

proc withTimeout*[T](fn: proc(): Future[T], timeoutMs: int): Future[T] {.async.} =
  let resultFut = fn()
  let timeoutFut = sleepAsync(timeoutMs)
  
  let winner = await race(resultFut, timeoutFut)
  if winner == 1:  # timeout won
    raise newException(TimeoutError, &"Operation timed out after {timeoutMs}ms")
  
  await resultFut
```

---

## 4. Validation Framework

```nim
# validation.nim
# Declarative validation

import std/[strformat, strutils, sequtils, re]

type
  ValidationError* = object
    field*: string
    message*: string
    code*: string

  ValidationResult* = object
    errors*: seq[ValidationError]

  Validator*[T] = proc(value: T): Option[string]

proc valid*(): ValidationResult = ValidationResult(errors: @[])

proc isValid*(r: ValidationResult): bool = r.errors.len == 0

proc addError*(r: var ValidationResult, field, msg, code = "") =
  r.errors.add(ValidationError(field: field, message: msg, code: code))

proc merge*(a, b: ValidationResult): ValidationResult =
  ValidationResult(errors: a.errors & b.errors)

# String validators
proc minLen*(n: int): Validator[string] =
  proc(s: string): Option[string] =
    if s.len < n: some(&"Must be at least {n} characters")
    else: none(string)

proc maxLen*(n: int): Validator[string] =
  proc(s: string): Option[string] =
    if s.len > n: some(&"Must be at most {n} characters")
    else: none(string)

proc notEmpty*(): Validator[string] =
  proc(s: string): Option[string] =
    if s.strip().len == 0: some("Must not be empty")
    else: none(string)

proc matches*(pattern: string): Validator[string] =
  let rx = re(pattern)
  proc(s: string): Option[string] =
    if not s.match(rx): some(&"Must match pattern: {pattern}")
    else: none(string)

proc isEmail*(): Validator[string] =
  matches(r"^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$")

# Number validators
proc minVal*(n: float): Validator[float] =
  proc(v: float): Option[string] =
    if v < n: some(&"Must be at least {n}")
    else: none(string)

proc maxVal*(n: float): Validator[float] =
  proc(v: float): Option[string] =
    if v > n: some(&"Must be at most {n}")
    else: none(string)

proc inRange*(low, high: float): Validator[float] =
  proc(v: float): Option[string] =
    if v < low or v > high: some(&"Must be between {low} and {high}")
    else: none(string)

# Field validation DSL
type
  Schema*[T] = ref object
    fieldValidators*: seq[(string, proc(obj: T): Option[string])]

proc newSchema*[T](): Schema[T] = Schema[T](fieldValidators: @[])

proc field*[T, F](schema: Schema[T], name: string, getter: proc(obj: T): F,
    validators: varargs[Validator[F]]): Schema[T] =
  let name = name
  let getter = getter
  let validators = @validators
  
  schema.fieldValidators.add((name, proc(obj: T): Option[string] =
    let value = getter(obj)
    for v in validators:
      let err = v(value)
      if err.isSome: return err
    none(string)
  ))
  schema

proc validate*[T](schema: Schema[T], obj: T): ValidationResult =
  result = valid()
  for (name, validator) in schema.fieldValidators:
    let err = validator(obj)
    if err.isSome:
      result.addError(name, err.get())

# Example
type
  CreateUserRequest* = object
    name*: string
    email*: string
    age*: float
    password*: string

let userSchema = newSchema[CreateUserRequest]()
  .field("name", proc(r: CreateUserRequest): string = r.name,
    notEmpty(), minLen(2), maxLen(50))
  .field("email", proc(r: CreateUserRequest): string = r.email,
    notEmpty(), isEmail())
  .field("age", proc(r: CreateUserRequest): float = r.age,
    inRange(18, 120))
  .field("password", proc(r: CreateUserRequest): string = r.password,
    minLen(8), maxLen(128))

when isMainModule:
  let req = CreateUserRequest(
    name: "A",
    email: "not-an-email",
    age: 15.0,
    password: "short"
  )
  
  let result = userSchema.validate(req)
  if result.isValid:
    echo "Valid!"
  else:
    echo "Validation errors:"
    for err in result.errors:
      echo &"  {err.field}: {err.message}"
```

---

## สรุป

| Pattern | ใช้เมื่อ |
|---------|---------|
| Result[T,E] | Recoverable errors, functional style |
| Error context | Long call chains, debugging |
| Panic recovery | Production safety net |
| Retry + backoff | Network/transient failures |
| Validation schema | User input, API requests |

---

**Next**: [Part 86 - Static Analysis & Code Generation](../advanced/part86_codegen.md)
