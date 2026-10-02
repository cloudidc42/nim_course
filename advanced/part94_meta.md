# Part 94 - Advanced Metaprogramming

## บทนำ

Metaprogramming ระดับสูงใน Nim:
- Template metaprogramming
- Compile-time reflection
- Code transformation macros
- Concept-based generic programming

---

## 1. Template Metaprogramming

```nim
# template_meta.nim
# Compile-time computation ด้วย templates

import std/[macros, strformat, typetraits]

# Compile-time integer operations
template staticAdd*(a, b: static[int]): int = a + b
template staticMax*(a, b: static[int]): int = (if a > b: a else: b)
template staticMin*(a, b: static[int]): int = (if a < b: a else: b)

# Static array operations
template staticFill*[T; N: static[int]](val: T): array[N, T] =
  var arr: array[N, T]
  for i in 0..<N: arr[i] = val
  arr

# Type-level computations
type
  TypeList*[T] = object  # Phantom type for type-level list
  
  Nil* = object
  Cons*[H, T] = object

# Type-safe heterogeneous list (HList)
type
  HNil* = object
  HCons*[H, T] = object
    head*: H
    tail*: T

proc hNil*(): HNil = HNil()

proc hCons*[H, T](head: H, tail: T): HCons[H, T] =
  HCons[H, T](head: head, tail: tail)

# Get length at compile time
proc hlen*(_: HNil): int = 0
proc hlen*[H, T](l: HCons[H, T]): int = 1 + hlen(l.tail)

# Strongly typed tuple from HList
when isMainModule:
  let list = hCons(1, hCons("hello", hCons(3.14, hNil())))
  echo list.head          # 1
  echo list.tail.head     # "hello"
  echo list.tail.tail.head # 3.14
  echo hlen(list)         # 3

# Compile-time string operations
macro staticJoin*(parts: varargs[string]): string =
  var joined = ""
  for part in parts:
    joined &= part.strVal
  newLit(joined)

const greeting = staticJoin("Hello", ", ", "World", "!")
echo greeting  # "Hello, World!"

# Type-safe printf
macro printf*(fmt: static[string], args: varargs[typed]): untyped =
  # Validate format string at compile time
  var argIdx = 0
  var i = 0
  while i < fmt.len:
    if fmt[i] == '%' and i + 1 < fmt.len:
      inc i
      case fmt[i]
      of 'd', 'i':
        if argIdx >= args.len:
          error("Not enough arguments for format string")
        # Check arg is integer
        inc argIdx
      of 's':
        if argIdx >= args.len:
          error("Not enough arguments for format string")
        inc argIdx
      of 'f':
        if argIdx >= args.len:
          error("Not enough arguments for format string")
        inc argIdx
      of '%': discard
      else: error(&"Unknown format specifier: %{fmt[i]}")
    inc i
  
  if argIdx != args.len:
    error(&"Too many arguments for format string (expected {argIdx}, got {args.len})")
  
  # Generate format string code
  result = newCall(bindSym"strformat.fmt", newLit(fmt))
  for arg in args:
    result.add(arg)
```

---

## 2. Compile-Time Reflection

```nim
# reflection.nim
# Inspect types at compile time

import std/[macros, strformat, typetraits, tables]

# Get field names of a type
macro fieldNames*(T: typedesc): seq[string] =
  let typeImpl = T.getTypeImpl()
  var names: seq[NimNode]
  
  for field in typeImpl[2]:
    if field.kind == nnkIdentDefs:
      names.add(newLit(($field[0]).strip(chars = {'*'})))
  
  result = newNimNode(nnkBracket)
  for n in names: result.add(n)

# Get field types
macro fieldTypes*(T: typedesc): seq[string] =
  let typeImpl = T.getTypeImpl()
  var types: seq[NimNode]
  
  for field in typeImpl[2]:
    if field.kind == nnkIdentDefs:
      types.add(newLit(field[1].repr))
  
  result = newNimNode(nnkBracket)
  for t in types: result.add(t)

# Iterate over object fields at runtime using compile-time info
macro forEachField*(obj: typed, body: untyped): untyped =
  result = newNimNode(nnkStmtList)
  let typeImpl = obj.getTypeImpl()
  
  for field in typeImpl[2]:
    if field.kind != nnkIdentDefs: continue
    let fieldName = $field[0]
    let fieldIdent = newIdentNode(fieldName.strip(chars = {'*'}))
    
    result.add(newBlockStmt(
      newStmtList(
        newLetStmt(newIdentNode("fieldName"), newLit(fieldName.strip(chars = {'*'}))),
        newLetStmt(newIdentNode("fieldValue"), newDotExpr(obj, fieldIdent)),
        body
      )
    ))

# Auto-generate equality
macro deriveEq*(T: typedesc): untyped =
  let typeImpl = T.getTypeImpl()
  var checks = newNimNode(nnkStmtList)
  
  for field in typeImpl[2]:
    if field.kind != nnkIdentDefs: continue
    let fieldName = newIdentNode(($field[0]).strip(chars = {'*'}))
    checks.add(quote do:
      if a.`fieldName` != b.`fieldName`: return false
    )
  
  checks.add(quote do: return true)
  
  quote do:
    proc `==`*(a, b: `T`): bool =
      `checks`

# Auto-generate hash
macro deriveHash*(T: typedesc): untyped =
  let typeImpl = T.getTypeImpl()
  var hashCode = newNimNode(nnkStmtList)
  hashCode.add(quote do: result = 17)
  
  for field in typeImpl[2]:
    if field.kind != nnkIdentDefs: continue
    let fieldName = newIdentNode(($field[0]).strip(chars = {'*'}))
    hashCode.add(quote do:
      result = result * 31 + hash(obj.`fieldName`)
    )
  
  quote do:
    proc hash*(obj: `T`): Hash =
      `hashCode`

# Usage
type
  Point* = object
    x*, y*: float

deriveEq(Point)
deriveHash(Point)

when isMainModule:
  echo "Fields of Point: ", fieldNames(Point)
  echo "Types of Point: ", fieldTypes(Point)
  
  let p = Point(x: 1.0, y: 2.0)
  forEachField(p):
    echo &"  {fieldName} = {fieldValue}"
  
  let p1 = Point(x: 1.0, y: 2.0)
  let p2 = Point(x: 1.0, y: 2.0)
  echo p1 == p2  # true (generated)
```

---

## 3. Domain-Specific Languages

```nim
# dsl_advanced.nim
# Advanced DSL implementation

import std/[macros, asyncdispatch, json, tables, strformat]

# HTML generation DSL
type
  HtmlNode* = ref object
    tag*: string
    attrs*: seq[(string, string)]
    children*: seq[HtmlNode]
    text*: string
    isText*: bool

proc render*(node: HtmlNode): string =
  if node.isText: return node.text
  
  var attrStr = ""
  for (k, v) in node.attrs:
    attrStr &= &" {k}=\"{v}\""
  
  if node.children.len == 0 and node.text.len == 0:
    return &"<{node.tag}{attrStr}/>"
  
  var inner = node.text
  for child in node.children:
    inner &= render(child)
  
  &"<{node.tag}{attrStr}>{inner}</{node.tag}>"

macro html*(body: untyped): HtmlNode =
  ## HTML DSL:
  ## html:
  ##   div(class="container"):
  ##     h1: "Title"
  ##     p: "Content"
  proc transform(node: NimNode): NimNode =
    if node.kind == nnkStrLit:
      return quote do:
        HtmlNode(isText: true, text: `node`)
    
    if node.kind == nnkCall:
      let tag = $node[0]
      var attrs: seq[(string, string)]
      var children: seq[NimNode]
      
      for i in 1..<node.len:
        let arg = node[i]
        if arg.kind == nnkExprEqExpr:
          let k = $arg[0]
          let v = arg[1]
          # will use runtime attrs
          children.add(arg)
        elif arg.kind == nnkStmtList:
          for child in arg:
            children.add(transform(child))
        else:
          children.add(transform(arg))
      
      var childNodes = newNimNode(nnkBracket)
      for c in children: childNodes.add(c)
      
      return quote do:
        HtmlNode(tag: `tag`, children: @`childNodes`)
    
    node
  
  if body.kind == nnkStmtList and body.len > 0:
    return transform(body[0])
  transform(body)

# Configuration DSL
type
  Config* = object
    values*: Table[string, string]
    sections*: Table[string, Config]

macro config*(body: untyped): Config =
  ## config:
  ##   host = "localhost"
  ##   port = "8080"
  ##   database:
  ##     host = "db"
  ##     name = "myapp"
  
  var assignments: seq[(string, NimNode)]
  var sections: seq[(string, NimNode)]
  
  for node in body:
    if node.kind == nnkAsgn:
      assignments.add(($node[0], node[1]))
    elif node.kind == nnkCall and node.len == 2:
      sections.add(($node[0], node[1]))
  
  var valuesNode = newNimNode(nnkTableConstr)
  for (k, v) in assignments:
    valuesNode.add(newNimNode(nnkExprColonExpr).add(newLit(k), v))
  
  quote do:
    Config(values: `valuesNode`.toTable)
```

---

## 4. Generic Algorithms with Concepts

```nim
# generic_algorithms.nim
# Type-safe generic algorithms using concepts

import std/[algorithm, strformat, math]

# Concepts
type
  Ordered* = concept x, y
    (x < y) is bool
    (x == y) is bool

  Numeric* = concept x, y
    (x + y) is typeof(x)
    (x - y) is typeof(x)
    (x * y) is typeof(x)
    (x / y) is float

  Container*[T] = concept c
    c.len is int
    c[0] is T

# Generic sort with custom comparator
proc sortBy*[T](items: var seq[T], key: proc(x: T): auto) =
  items.sort(proc(a, b: T): int =
    let ka = key(a)
    let kb = key(b)
    if ka < kb: -1
    elif ka > kb: 1
    else: 0
  )

# Binary search on any ordered sequence
proc binarySearch*[T: Ordered](arr: openArray[T], target: T): int =
  var lo = 0
  var hi = arr.len - 1
  
  while lo <= hi:
    let mid = (lo + hi) div 2
    if arr[mid] == target: return mid
    elif arr[mid] < target: lo = mid + 1
    else: hi = mid - 1
  
  -1

# Generic statistics
proc mean*[T: Numeric](items: openArray[T]): float =
  if items.len == 0: return 0.0
  var sum: T
  for x in items: sum = sum + x
  sum.float / items.len.float

proc variance*[T: Numeric](items: openArray[T]): float =
  if items.len < 2: return 0.0
  let m = mean(items)
  var sumSq = 0.0
  for x in items:
    let diff = x.float - m
    sumSq += diff * diff
  sumSq / (items.len - 1).float

proc stddev*[T: Numeric](items: openArray[T]): float =
  sqrt(variance(items))

# Generic pipeline
type
  Pipeline*[T] = ref object
    steps*: seq[proc(x: T): T]

proc newPipeline*[T](): Pipeline[T] = Pipeline[T]()

proc pipe*[T](p: Pipeline[T], fn: proc(x: T): T): Pipeline[T] =
  p.steps.add(fn); p

proc run*[T](p: Pipeline[T], input: T): T =
  result = input
  for step in p.steps:
    result = step(result)

when isMainModule:
  # Generic sort
  var people = @[("Alice", 30), ("Bob", 25), ("Charlie", 35)]
  people.sortBy(proc(p: (string, int)): int = p[1])
  echo "Sorted by age: ", people
  
  # Binary search
  let arr = @[1, 3, 5, 7, 9, 11, 13]
  echo "Found 7 at index: ", binarySearch(arr, 7)
  
  # Statistics
  let data = @[2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0]
  echo &"Mean: {mean(data):.2f}"
  echo &"StdDev: {stddev(data):.2f}"
  
  # Pipeline
  let pipeline = newPipeline[string]()
    .pipe(proc(s: string): string = s.strip())
    .pipe(proc(s: string): string = s.toLowerAscii())
    .pipe(proc(s: string): string = s.replace(" ", "_"))
  
  echo pipeline.run("  Hello World  ")  # "hello_world"
```

---

## สรุป

| Technique | Power |
|-----------|-------|
| Template metaprog | Compile-time computation |
| Reflection macros | Auto-generate boilerplate |
| DSL | Domain-specific syntax |
| Concepts | Type-safe generics |

---

**Next**: [Part 95 - Systems Programming](../advanced/part95_systems.md)
