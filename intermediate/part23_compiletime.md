# Part 23: Compile-time Programming - การโปรแกรมตอน Compile

## static และ when

```nim
# static - evaluate at compile time
const maxItems = 100
static:
  echo "Compiling..."
  let x = 2 + 2
  assert x == 4, "Math is broken"

# when - compile-time if
when defined(windows):
  echo "Running on Windows"
elif defined(linux):
  echo "Running on Linux"
elif defined(macosx):
  echo "Running on macOS"
else:
  echo "Unknown OS"

# Size-specific code
when sizeof(int) == 8:
  echo "64-bit system"
elif sizeof(int) == 4:
  echo "32-bit system"

# Debug vs Release
when defined(release):
  proc log(msg: string) = discard  # no-op in release
else:
  proc log(msg: string) = echo "[LOG] ", msg

log("This is a debug message")

# Feature flags
when defined(featureA):
  proc doFeatureA() = echo "Feature A enabled"
else:
  proc doFeatureA() = echo "Feature A disabled"

# Compile with: nim c -d:featureA myfile.nim
```

## Compile-time Functions

```nim
# proc กับ static parameters
proc makeArray(size: static int): array[size, int] =
  for i in 0..<size:
    result[i] = i * i

let squares = makeArray(5)
echo squares  # [0, 1, 4, 9, 16]

# Computed at compile time
proc factorial(n: static int): int =
  when n <= 1: 1
  else: n * factorial(n - 1)

const fact5 = factorial(5)
echo fact5  # 120 (computed at compile time!)

# Type-level programming
proc isPowerOfTwo(n: static int): bool =
  n > 0 and (n and (n - 1)) == 0

when isPowerOfTwo(16):
  echo "16 is power of 2"  # yes

# Static type introspection
proc typeName[T](): string =
  when T is int:    "integer"
  elif T is float:  "float"
  elif T is string: "string"
  elif T is bool:   "boolean"
  else:             "unknown"

echo typeName[int]()    # integer
echo typeName[float]()  # float
echo typeName[string]() # string
```

## Templates กับ Compile-time

```nim
import macros

# Template ที่ evaluate เงื่อนไขตอน compile
template debugOnly(body: untyped): untyped =
  when not defined(release):
    body

debugOnly:
  echo "Debug info here"

# Template สร้าง type-safe code
template expectType(x: untyped, T: typedesc): untyped =
  when x isnot T:
    {.error: "Expected " & $T & " but got " & $type(x).}
  x

let n = 42
let m = expectType(n, int)  # OK
# let s = expectType(n, string)  # Compile error!

# Template สำหรับ enum
template enumToString(E: typedesc[enum], val: E): string =
  # Generate switch at compile time
  var result = ""
  for e in E:
    if e == val:
      result = $e
      break
  result
```

## Macros สำหรับ Code Generation

```nim
import macros

# Generate getter/setter procs
macro genAccessors(T: typedesc, fields: varargs[untyped]): untyped =
  result = newStmtList()
  for field in fields:
    let getter = field
    let setter = ident($field & "=")
    result.add quote do:
      proc `getter`*(obj: `T`): auto = obj.`field`
      proc `setter`*(obj: var `T`, val: auto) = obj.`field` = val

type Person = object
  name: string
  age: int
  email: string

genAccessors(Person, name, age, email)

var p = Person(name: "Alice", age: 30, email: "alice@example.com")
echo p.name   # Alice
p.age = 31
echo p.age    # 31

# Generate serialization code
macro makeSerializable(T: typedesc): untyped =
  let typ = T.getTypeImpl()
  var fields: seq[NimNode] = @[]
  
  for child in typ[2]:
    if child.kind == nnkIdentDefs:
      fields.add(child[0])
  
  # Generate toJson proc
  var jsonCode = newStmtList()
  jsonCode.add quote do:
    import json
  
  result = quote do:
    proc toJson*(obj: `T`): JsonNode =
      result = newJObject()
      # fields would be iterated here

# Compile-time type checking
macro checkFields(T: typedesc, required: varargs[string]): untyped =
  let typ = T.getTypeImpl()
  var existingFields: seq[string] = @[]
  
  for child in typ[2]:
    if child.kind == nnkIdentDefs:
      existingFields.add($child[0])
  
  for req in required:
    let reqStr = $req
    if reqStr notin existingFields:
      error("Type " & $T & " missing required field: " & reqStr)
  
  result = newEmptyNode()

# This would fail at compile time if 'name' or 'id' are missing
# checkFields(Person, "name", "age")
```

## Compile-time Reflection

```nim
import typetraits, macros

# Type traits
type Animal = object
  name: string
  legs: int

echo Animal.name       # "Animal"
echo Person.fieldNames # @["name", "age", "email"]

# isNil, isRef, etc.
type
  ValueType = object
  RefType = ref object

echo isRef(ValueType)  # false
echo isRef(RefType)    # true

# Generic type checking
proc processNumber[T](x: T) =
  when T is SomeInteger:
    echo "Integer: ", x
  elif T is SomeFloat:
    echo "Float: ", x.formatFloat(ffDecimal, 2)
  else:
    {.error: "Not a number type!".}

processNumber(42)
processNumber(3.14)
# processNumber("hello")  # Compile error

# hasCustomPragma
type
  Validated = object
    name {.required.}: string
    age {.range: 0..150.}: int

proc validate[T](obj: T) =
  for field, val in obj.fieldPairs:
    when val.hasCustomPragma(required):
      if val.len == 0:
        echo field, " is required!"
```

## Static Data Structures

```nim
# Compile-time hash table
import tables

const lookup = {
  "one":   1,
  "two":   2,
  "three": 3,
  "four":  4,
  "five":  5
}.toTable()

# Fast const lookup (in program data segment)
echo lookup["three"]  # 3

# Compile-time string to enum
type Day = enum
  Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday

const dayMap = block:
  var t = initTable[string, Day]()
  for d in Day:
    t[$d] = d
  t

proc parseDay(s: string): Day =
  if s in dayMap: dayMap[s]
  else: raise newException(ValueError, "Invalid day: " & s)

echo parseDay("Monday")   # Monday
echo parseDay("Friday")   # Friday

# Compile-time lookup table (LUT)
const sinTable = block:
  import math
  var t: array[360, float]
  for i in 0..<360:
    t[i] = sin(float(i) * PI / 180.0)
  t

proc fastSin(degrees: int): float =
  sinTable[degrees mod 360]

echo fastSin(90)   # ~1.0
echo fastSin(180)  # ~0.0
```

## Practical: JSON Schema Validator

```nim
# Compile-time JSON schema with macros

import macros, json

macro defineSchema(name: untyped, body: untyped): untyped =
  # Parse the schema body and generate validation code
  result = newStmtList()
  
  var fields: seq[(string, string)] = @[]
  
  for stmt in body:
    if stmt.kind == nnkCall:
      let fieldName = $stmt[0]
      let fieldType = $stmt[1]
      fields.add((fieldName, fieldType))
  
  # Generate validator proc
  let procName = ident("validate" & $name)
  let errors = ident("errors")
  
  var validatorBody = newStmtList()
  validatorBody.add quote do:
    var `errors`: seq[string] = @[]
  
  for (fname, ftype) in fields:
    let fn = fname
    let ft = ftype
    validatorBody.add quote do:
      if `fn` notin node:
        `errors`.add("Missing field: " & `fn`)
      else:
        when `ft` == "string":
          if node[`fn`].kind != JString:
            `errors`.add(`fn` & " must be string")
        elif `ft` == "int":
          if node[`fn`].kind != JInt:
            `errors`.add(`fn` & " must be integer")
  
  validatorBody.add quote do:
    return `errors`
  
  result.add newProc(procName, 
    [ident("seq[string]"), newIdentDefs(ident("node"), ident("JsonNode"))],
    validatorBody)

# Define schema at compile time
defineSchema UserSchema:
  name("string")
  age("int")
  email("string")

# Use at runtime
let userJson = parseJson("""{"name": "Alice", "age": 30, "email": "a@b.com"}""")
# let errors = validateUserSchema(userJson)
```

## สรุป Part 23

ในบทนี้เราได้เรียนรู้:
- ✅ static/when สำหรับ compile-time branching
- ✅ Compile-time functions และ static parameters
- ✅ Templates กับ compile-time evaluation
- ✅ Macros สำหรับ code generation
- ✅ Compile-time reflection ด้วย typetraits
- ✅ Static data structures (const lookup tables)
- ✅ Practical: Compile-time schema generation

---

**Previous**: [Part 22 - Memory Management](part22_memory.md)
**Next**: [Part 24 - Testing](part24_testing.md)
