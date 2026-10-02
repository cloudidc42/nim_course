# Part 15: Modules และ Packages - โมดูลและแพ็คเกจ

## การสร้าง Module

```nim
# mymath.nim - custom module

proc add*(a, b: int): int =
  ## Adds two integers
  a + b

proc subtract*(a, b: int): int =
  ## Subtracts b from a
  a - b

proc multiply*(a, b: int): int = a * b

# proc ที่ไม่ export (ไม่มี *)
proc internalHelper(x: int): int = x * 2

# Type export
type
  Point* = object
    x*, y*: float   # field export ด้วย *
  
  Color* = tuple[r, g, b: uint8]

proc distance*(a, b: Point): float =
  import math
  sqrt((a.x - b.x)^2 + (a.y - b.y)^2)

# Constant export
const
  PI* = 3.14159265358979
  E* = 2.71828182845905

# Variable export
var globalCounter* = 0

when isMainModule:
  # Code ที่รันเฉพาะเมื่อ run โดยตรง
  echo "Testing mymath..."
  echo add(3, 4)
  echo multiply(5, 6)
```

## การ import Module

```nim
# main.nim - using the module

# Import ทั้ง module
import mymath

echo add(5, 3)        # 8
echo PI               # 3.14159...

# Import เฉพาะบางอย่าง
from mymath import add, Point, PI
echo add(1, 2)
echo PI

# Import กับ alias
import mymath as mm
echo mm.add(1, 2)
echo mm.PI

# Import ทั้งหมดเข้า namespace
from mymath import nil  # ต้อง qualify ทุกอย่าง
echo mymath.add(1, 2)

# Exclude specific names
import mymath except subtract
# subtract ไม่สามารถใช้โดยตรงได้

# Import standard library modules
import std/[strutils, sequtils, algorithm]
import std/os
import std/tables
import std/json

# Multiple imports
import strutils, sequtils
```

## Standard Library Modules สำคัญ

```nim
# strutils - string operations
import strutils
echo "hello".toUpper()
echo "Hello World".split()
echo "42".parseInt()
echo 42.intToStr()

# sequtils - sequence operations
import sequtils, sugar
let nums = @[1, 2, 3, 4, 5]
echo nums.map(x => x * 2)
echo nums.filter(x => x mod 2 == 0)
echo nums.foldl(a + b)

# algorithm - sorting
import algorithm
var arr = @[3, 1, 4, 1, 5, 9, 2, 6]
arr.sort()
echo arr

# os - OS operations
import os
echo getCurrentDir()
echo getEnv("HOME")
echo fileExists("file.txt")

# math - math functions
import math
echo sqrt(16.0)    # 4.0
echo PI            # 3.14159...
echo ln(E)         # 1.0
echo sin(PI/2)     # 1.0

# times - time operations
import times
let t = now()
echo t.format("yyyy-MM-dd HH:mm:ss")
echo epochTime()

# json - JSON processing
import json
let j = parseJson("""{"key": "value"}""")
echo j["key"].getStr()

# tables - hash maps
import tables
var t2 = initTable[string, int]()
t2["one"] = 1
t2["two"] = 2

# httpclient - HTTP requests
# import httpclient
# let client = newHttpClient()
# let response = client.get("http://example.com")
# echo response.status
```

## Namespacing

```nim
# ปัญหา: name collision
# import strutils  # มี `find`
# import sequtils  # ก็มี `find`

# แก้ด้วย fully qualified name
import strutils
let s = "hello world"
echo strutils.find(s, "world")  # 6

from sequtils import find
let sq = @[1, 2, 3, 4, 5]
echo find(sq, 3)  # 2

# หรือ import เฉพาะที่ต้องการ
from strutils import `%`
let s2 = "Hello $1" % "World"  # Hello World
echo s2
```

## Package Structure

```
mypackage/
├── mypackage.nimble
├── src/
│   ├── mypackage.nim       # main module
│   └── mypackage/
│       ├── types.nim       # type definitions
│       ├── utils.nim       # utility functions
│       ├── config.nim      # configuration
│       └── internal/       # private modules
│           └── helpers.nim
├── tests/
│   ├── test_types.nim
│   └── test_utils.nim
└── docs/
    └── index.md
```

### mypackage.nimble
```nimble
version     = "1.0.0"
author      = "Your Name"
description = "My awesome package"
license     = "MIT"
srcDir      = "src"
bin         = @["mypackage"]

requires "nim >= 2.0.0"
requires "jester >= 0.5.0"

task test, "Run tests":
  exec "testament all"

task docs, "Build docs":
  exec "nim doc --project --index:on src/mypackage.nim"
```

### src/mypackage.nim
```nim
## My Package - Main module
## 
## Example:
## ```nim
## import mypackage
## echo greet("World")
## ```

import mypackage/[types, utils, config]

export types, utils

proc greet*(name: string): string =
  ## Greet someone by name
  "Hello, " & name & "!"
```

## Nimble Tasks

```bash
# สร้างโปรเจคต์
nimble init

# Build
nimble build

# Build release
nimble build -d:release

# Run
nimble run

# Test
nimble test

# Install to ~/.nimble/pkgs
nimble install
```

## Practical: Modular Application

```nim
# src/types.nim
type
  User* = object
    id*: int
    name*: string
    email*: string
    role*: string

  Product* = object
    id*: int
    name*: string
    price*: float
    stock*: int

  Order* = object
    id*: int
    userId*: int
    items*: seq[(int, int)]  # (productId, quantity)
    total*: float
```

```nim
# src/database.nim
import types, tables, options, sequtils

type Database* = object
  users*: Table[int, User]
  products*: Table[int, Product]
  orders*: Table[int, Order]
  nextId: int

proc newDatabase*(): Database =
  Database(
    users: initTable[int, User](),
    products: initTable[int, Product](),
    orders: initTable[int, Order](),
    nextId: 1
  )

proc addUser*(db: var Database, name, email, role: string): User =
  let user = User(id: db.nextId, name: name, email: email, role: role)
  db.users[db.nextId] = user
  inc db.nextId
  user

proc findUser*(db: Database, id: int): Option[User] =
  if id in db.users: some(db.users[id])
  else: none(User)

proc listUsers*(db: Database): seq[User] =
  toSeq(db.users.values)
```

```nim
# src/api.nim - API handlers
import types, json, strformat

proc formatUser(u: User): JsonNode =
  %* {"id": u.id, "name": u.name, "email": u.email, "role": u.role}

# src/main.nim - Entry point
var db = newDatabase()

discard db.addUser("Alice", "alice@mail.com", "admin")
discard db.addUser("Bob", "bob@mail.com", "user")
discard db.addUser("Carol", "carol@mail.com", "user")

let user = db.findUser(1)
if user.isSome:
  echo "Found: ", user.get().name

echo "Total users: ", db.listUsers().len
```

## แบบฝึกหัด Part 15

### แบบฝึกหัดที่ 1: Create Math Module
```nim
# mathutils.nim
import math

proc clamp*[T: SomeNumber](x, lo, hi: T): T =
  if x < lo: lo elif x > hi: hi else: x

proc lerp*(a, b, t: float): float =
  a + (b - a) * t

proc isPrime*(n: int): bool =
  if n < 2: return false
  if n == 2: return true
  if n mod 2 == 0: return false
  for i in 3..int(sqrt(float(n))):
    if n mod i == 0: return false
  true

proc primes*(n: int): seq[int] =
  result = @[]
  for i in 2..n:
    if isPrime(i): result.add(i)

# Test
echo "Primes up to 50: ", primes(50)
echo "lerp(0, 100, 0.5) = ", lerp(0.0, 100.0, 0.5)
echo "clamp(150, 0, 100) = ", clamp(150, 0, 100)
```

## สรุป Part 15

ในบทนี้เราได้เรียนรู้:
- ✅ สร้าง module ด้วย `*` export
- ✅ import รูปแบบต่างๆ
- ✅ Standard library modules สำคัญ
- ✅ Namespace management
- ✅ Package structure
- ✅ nimble tasks
- ✅ Modular application design

---

**Previous**: [Part 14 - Error Handling](part14_error_handling.md)
**Next**: [Part 16 - OOP: Objects และ Methods](../intermediate/part16_oop.md)
