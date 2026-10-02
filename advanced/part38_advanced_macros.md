# Part 38: Advanced Macros และ DSL Creation

## Macro คืออะไร

```
Macro = เครื่องมือแปลง AST (Abstract Syntax Tree) ตอน compile time

ประโยชน์:
1. Code generation - ลด boilerplate
2. DSL creation - สร้างภาษาพิเศษในภายใน Nim
3. Compile-time validation
4. Domain-specific optimizations
5. Metaprogramming
```

## AST เบื้องต้น

```nim
import macros

# ดู AST ของ expression
macro showAst(x: untyped): untyped =
  echo x.treeRepr
  x

showAst:
  let x = 1 + 2 * 3

# Output:
# StmtList
#   LetSection
#     IdentDefs
#       Ident "x"
#       Empty
#       Infix
#         Ident "+"
#         IntLit 1
#         Infix
#           Ident "*"
#           IntLit 2
#           IntLit 3

# Node kinds
proc dumpNode(n: NimNode, indent: int = 0) =
  echo " ".repeat(indent), n.kind, ": ",
       case n.kind
       of nnkIdent: $n.ident
       of nnkStrLit: n.strVal
       of nnkIntLit: $n.intVal
       of nnkFloatLit: $n.floatVal
       else: ""
  for child in n:
    dumpNode(child, indent + 2)

macro inspect(x: untyped): untyped =
  dumpNode(x)
  x
```

## Code Generation

```nim
import macros

# Generate multiple procs from a list
macro genArithmetic(ops: static seq[tuple[name: string, op: string]]): untyped =
  result = newStmtList()
  for (name, op) in ops:
    let procName = ident(name)
    let opIdent = ident(op)
    result.add quote do:
      proc `procName`*(a, b: int): int = `opIdent`(a, b)

# Generate add, sub, mul, div procs
genArithmetic(@[
  ("add", "+"),
  ("sub", "-"),
  ("mul", "*"),
])

echo add(3, 4)   # 7
echo sub(10, 3)  # 7
echo mul(4, 5)   # 20

# Generate serialization
macro genSerializer(T: typedesc): untyped =
  let typImpl = T.getTypeImpl()
  var fields: seq[(string, NimNode)] = @[]
  
  for child in typImpl[2]:
    if child.kind == nnkIdentDefs:
      fields.add(($child[0], child[1]))
  
  let objParam = ident("obj")
  var stmts = newStmtList()
  
  stmts.add quote do:
    import json
    var result = newJObject()
  
  for (fname, ftype) in fields:
    let fn = newLit(fname)
    let fident = ident(fname)
    stmts.add quote do:
      result[`fn`] = %obj.`fident`
  
  let procName = ident("toJson")
  result = quote do:
    proc `procName`*(`objParam`: `T`): JsonNode =
      `stmts`
      result

type User = object
  name: string
  age: int
  email: string

genSerializer(User)

let u = User(name: "Alice", age: 30, email: "alice@example.com")
# echo toJson(u)
```

## DSL Creation

```nim
import macros, tables

# HTML DSL
macro html(body: untyped): string =
  # Convert Nim-like syntax to HTML
  proc processNode(n: NimNode): string =
    case n.kind
    of nnkCall:
      let tag = $n[0]
      var attrs = ""
      var content = ""
      
      for i in 1..<n.len:
        if n[i].kind == nnkExprEqExpr:
          # Attribute: class="foo"
          attrs &= " " & $n[i][0] & "=\"" & $n[i][1] & "\""
        elif n[i].kind == nnkStmtList:
          # Nested elements
          for child in n[i]:
            content &= processNode(child)
        elif n[i].kind == nnkStrLit:
          content = n[i].strVal
      
      "<" & tag & attrs & ">" & content & "</" & tag & ">"
    
    of nnkStrLit:
      n.strVal
    
    of nnkStmtList:
      var result = ""
      for child in n:
        result &= processNode(child)
      result
    
    else:
      ""
  
  newLit(processNode(body))

# Usage (conceptual DSL):
# let page = html:
#   div(class="container"):
#     h1: "Hello, World!"
#     p: "This is generated HTML"

# State machine DSL
macro stateMachine(name: untyped, body: untyped): untyped =
  ## Define state machine with transitions
  ## Usage:
  ## stateMachine TrafficLight:
  ##   state Red:
  ##     on Timer: -> Green
  ##   state Green:
  ##     on Timer: -> Yellow
  ##   state Yellow:
  ##     on Timer: -> Red
  
  result = newStmtList()
  let smName = name
  
  # Generate state enum
  var states: seq[NimNode] = @[]
  var transitions: seq[(string, string, string)] = @[]  # (from, event, to)
  
  for stmt in body:
    if stmt.kind == nnkCall and $stmt[0] == "state":
      states.add(ident($stmt[1]))
  
  # Generate enum type
  let enumName = ident($name & "State")
  let enumDef = nnkTypeSection.newTree(
    nnkTypeDef.newTree(
      enumName,
      newEmptyNode(),
      nnkEnumTy.newTree(@[newEmptyNode()] & states)
    )
  )
  result.add(enumDef)
  echo result.repr
```

## Macro Debugging

```nim
import macros

# Debug macro output
macro debugMacro(x: untyped): untyped =
  result = x
  echo "Macro output:"
  echo result.repr  # Show generated code

# expandMacros in nim compiler:
# nim c --expandMacro:myMacro myfile.nim

# Type-safe printf using macros
macro fmt(formatStr: static string, args: varargs[untyped]): string =
  ## Type-checked format string (simplified)
  var parts: seq[NimNode] = @[]
  var i = 0
  var argIdx = 0
  var current = ""
  
  while i < formatStr.len:
    if formatStr[i] == '%' and i + 1 < formatStr.len:
      if current.len > 0:
        parts.add(newLit(current))
        current = ""
      
      case formatStr[i + 1]
      of 'd', 'i':
        if argIdx < args.len:
          let arg = args[argIdx]
          parts.add quote do:
            $`arg`
          inc argIdx
      of 's':
        if argIdx < args.len:
          let arg = args[argIdx]
          parts.add(arg)
          inc argIdx
      of '%':
        current &= "%"
      else:
        current &= formatStr[i]
      
      i += 2
    else:
      current &= formatStr[i]
      inc i
  
  if current.len > 0:
    parts.add(newLit(current))
  
  # Join all parts
  result = parts[0]
  for i in 1..<parts.len:
    result = infix(result, "&", parts[i])

# Usage
echo fmt("%s is %d years old", "Alice", 30)
```

## Practical: Compile-time ORM

```nim
import macros, strutils

# Compile-time ORM DSL
macro model(name: untyped, body: untyped): untyped =
  ## Generate SQLite model with CRUD procs
  result = newStmtList()
  
  let typeName = name
  let tableName = newLit(($name).toLowerAscii() & "s")
  
  # Parse fields
  var fields: seq[(string, NimNode)] = @[]
  for stmt in body:
    if stmt.kind == nnkCall and stmt.len == 2:
      fields.add(($stmt[0], stmt[1]))
  
  # Generate type definition
  var typeFields = nnkRecList.newTree()
  typeFields.add nnkIdentDefs.newTree(
    ident("id"), ident("int"), newEmptyNode()
  )
  for (fname, ftype) in fields:
    typeFields.add nnkIdentDefs.newTree(
      ident(fname), ftype, newEmptyNode()
    )
  
  result.add nnkTypeSection.newTree(
    nnkTypeDef.newTree(
      typeName, newEmptyNode(),
      nnkObjectTy.newTree(
        newEmptyNode(), newEmptyNode(), typeFields
      )
    )
  )
  
  # Generate createTable proc
  var columnDefs = "id INTEGER PRIMARY KEY AUTOINCREMENT"
  for (fname, ftype) in fields:
    let sqlType = case $ftype
      of "string": "TEXT"
      of "int": "INTEGER"
      of "float": "REAL"
      of "bool": "INTEGER"
      else: "TEXT"
    columnDefs &= ", " & fname & " " & sqlType
  
  let createSql = newLit("CREATE TABLE IF NOT EXISTS " & $tableName &
                          " (" & columnDefs & ")")
  let createProc = ident("create" & $typeName & "Table")
  
  result.add quote do:
    proc `createProc`*(db: auto) =
      db.exec(sql`createSql`)
  
  # echo result.repr  # Debug

# Usage:
# model User:
#   name(string)
#   age(int)
#   email(string)

# Generated:
# type User = object
#   id: int
#   name: string
#   age: int
#   email: string
# proc createUserTable(db: auto) = ...
```

## สรุป Part 38

ในบทนี้เราได้เรียนรู้:
- ✅ AST node types และ treeRepr
- ✅ Code generation ด้วย quote do:
- ✅ DSL creation (HTML, state machine)
- ✅ Macro debugging
- ✅ Type-safe format string
- ✅ Compile-time ORM

---

**Previous**: [Part 37 - Cross Compilation](part37_cross_compilation.md)
**Next**: [Part 39 - Nim Compiler Internals](part39_compiler_internals.md)
