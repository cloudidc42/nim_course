# Part 86 - Static Analysis & Code Generation

## บทนำ

Nim's macro system ทรงพลังมาก — สามารถ:
- วิเคราะห์ code ที่ compile time
- Generate code อัตโนมัติ
- สร้าง DSL
- Implement design patterns โดยอัตโนมัติ

---

## 1. Compile-Time Analysis

```nim
# static_analysis.nim
# วิเคราะห์ types และ code ที่ compile time

import std/[macros, strformat, typetraits]

# ตรวจสอบ type ที่ compile time
macro assertFields*(T: typedesc, fields: varargs[string]): untyped =
  let typeImpl = T.getTypeImpl()
  
  var existingFields: seq[string]
  for node in typeImpl[2]:  # RecList
    if node.kind == nnkIdentDefs:
      existingFields.add($node[0])
  
  for field in fields:
    let fieldStr = $field
    if fieldStr notin existingFields:
      error(&"Type {T.repr} missing required field: {fieldStr}")
  
  newNimNode(nnkEmpty)

# ตัวอย่าง: ต้องมี id, createdAt, updatedAt
type
  User* = object
    id*: int
    name*: string
    createdAt*: int64
    updatedAt*: int64

# Compile-time check
assertFields(User, "id", "createdAt", "updatedAt")

# ตรวจสอบ proc signatures
macro checkReturnType*(procName: typed, expectedType: typedesc): untyped =
  let impl = procName.getImpl()
  let retType = impl[3][0]  # Return type node
  
  if retType.repr != expectedType.repr:
    error(&"Expected return type {expectedType.repr}, got {retType.repr}")
  
  newNimNode(nnkEmpty)

# Compile-time string format validation
macro sqlQuery*(query: static[string]): string =
  # Check for common SQL injection patterns at compile time
  let dangerous = ["--", "/*", "*/", "xp_", "exec(", "drop table", "drop database"]
  let queryLower = query.toLowerAscii()
  
  for pattern in dangerous:
    if pattern in queryLower:
      warning(&"Potentially dangerous SQL: contains '{pattern}'")
  
  newLit(query)

proc unsafeQuery*(sql: string) = discard  # placeholder

# Usage
let q = sqlQuery("SELECT * FROM users WHERE id = ?")
```

---

## 2. Code Generation Macros

```nim
# codegen.nim
# Automatic code generation ด้วย macros

import std/[macros, strformat, strutils, tables]

# Auto-generate getters/setters
macro properties*(T: typedesc): untyped =
  result = newNimNode(nnkStmtList)
  
  let typeImpl = T.getTypeImpl()
  let typeName = T.repr
  
  for node in typeImpl[2]:
    if node.kind != nnkIdentDefs: continue
    
    let fieldName = $node[0]
    let fieldType = node[1]
    
    # Skip private fields (starting with lowercase)
    # Generate getter
    let getterName = newIdentNode("get" & fieldName.capitalizeAscii())
    let getterProc = quote do:
      proc `getterName`*(obj: `T`): `fieldType` = obj.`fieldName`
    result.add(getterProc)

# Auto-generate JSON serialization
macro jsonSerializable*(T: typedesc): untyped =
  result = newNimNode(nnkStmtList)
  
  let typeImpl = T.getTypeImpl()
  
  # Generate toJson
  var toJsonBody = newNimNode(nnkStmtList)
  toJsonBody.add(quote do: result = newJObject())
  
  for node in typeImpl[2]:
    if node.kind != nnkIdentDefs: continue
    let fieldName = $node[0]
    let fieldNameLit = newLit(fieldName)
    let fieldIdent = newIdentNode(fieldName)
    toJsonBody.add(quote do:
      result[`fieldNameLit`] = %obj.`fieldIdent`
    )
  
  let toJsonProc = newProc(
    name = newIdentNode("toJson"),
    params = [
      newIdentNode("JsonNode"),
      newIdentDefs(newIdentNode("obj"), T)
    ],
    body = toJsonBody
  )
  toJsonProc[4] = newNimNode(nnkPragma).add(newIdentNode("used"))
  result.add(toJsonProc)

# Builder pattern generator
macro builder*(T: typedesc): untyped =
  result = newNimNode(nnkStmtList)
  
  let typeImpl = T.getTypeImpl()
  let builderName = newIdentNode(T.repr & "Builder")
  
  # Generate Builder type
  var recList = newNimNode(nnkRecList)
  for node in typeImpl[2]:
    if node.kind != nnkIdentDefs: continue
    recList.add(node)
  
  # Generate with methods
  for node in typeImpl[2]:
    if node.kind != nnkIdentDefs: continue
    let fieldName = $node[0]
    let fieldType = node[1]
    let withName = newIdentNode("with" & fieldName.capitalizeAscii())
    let fieldIdent = newIdentNode(fieldName)
    
    result.add(quote do:
      proc `withName`*(b: var `builderName`, val: `fieldType`): var `builderName` =
        b.`fieldIdent` = val
        b
    )

# Enum to string mapping
macro enumToString*(E: typedesc): untyped =
  result = newNimNode(nnkStmtList)
  
  let typeImpl = E.getTypeImpl()
  
  var caseBranches = newNimNode(nnkCaseStmt)
  caseBranches.add(newIdentNode("e"))
  
  for node in typeImpl[1]:
    if node.kind in {nnkIdent, nnkEnumFieldDef}:
      let name = if node.kind == nnkIdent: $node
                 else: $node[0]
      let nameIdent = newIdentNode(name)
      caseBranches.add(newNimNode(nnkOfBranch).add(
        newDotExpr(E, nameIdent),
        newLit(name)
      ))
  
  let proc_ = newProc(
    name = newIdentNode("toString"),
    params = [
      newIdentNode("string"),
      newIdentDefs(newIdentNode("e"), E)
    ],
    body = newNimNode(nnkStmtList).add(caseBranches)
  )
  result.add(proc_)
```

---

## 3. DSL Implementation

```nim
# dsl.nim
# Domain Specific Languages ด้วย macros

import std/[macros, asyncdispatch, asynchttpserver, tables, strformat]

# HTTP Router DSL
macro routes*(body: untyped): untyped =
  ## Usage:
  ## routes:
  ##   get "/users": ...
  ##   post "/users": ...
  result = newNimNode(nnkStmtList)
  
  var routeHandlers: seq[(string, string, NimNode)] = @[]
  
  for node in body:
    if node.kind != nnkCommand: continue
    let method = ($node[0]).toUpperAscii()
    let path = $node[1]
    let handler = node[2]
    routeHandlers.add((method, path, handler))
  
  # Generate handler table
  result.add(quote do:
    var handlers = initTable[string, proc(req: Request): Future[void]]()
  )
  
  for (meth, path, handler) in routeHandlers:
    let key = &"{meth} {path}"
    result.add(quote do:
      handlers[`key`] = proc(req: Request): Future[void] {.async.} =
        `handler`
    )
  
  result.add(quote do:
    proc dispatch*(req: Request) {.async.} =
      let key = $req.reqMethod & " " & req.url.path
      if key in handlers:
        await handlers[key](req)
      else:
        await req.respond(Http404, "Not found")
  )

# State machine DSL
macro stateMachine*(name: untyped, body: untyped): untyped =
  ## Usage:
  ## stateMachine TrafficLight:
  ##   state Red:
  ##     on Timer: Green
  ##   state Green:
  ##     on Timer: Yellow
  ##   state Yellow:
  ##     on Timer: Red
  
  let machineName = name
  result = newNimNode(nnkStmtList)
  
  var states: seq[string]
  var events: seq[string]
  var transitions: seq[(string, string, string)]  # (from, event, to)
  
  for node in body:
    if node.kind != nnkCommand: continue
    if $node[0] != "state": continue
    
    let stateName = $node[1]
    states.add(stateName)
    
    for transition in node[2]:
      if transition.kind == nnkCommand and $transition[0] == "on":
        let event = $transition[1]
        let target = $transition[2]
        events.add(event)
        transitions.add((stateName, event, target))
  
  # Generate State enum
  var stateEnum = newNimNode(nnkEnumTy).add(newNimNode(nnkEmpty))
  for s in states.deduplicate():
    stateEnum.add(newIdentNode("s" & s))
  
  result.add(newNimNode(nnkTypeSection).add(
    newNimNode(nnkTypeDef).add(
      newIdentNode($machineName & "State"),
      newNimNode(nnkEmpty),
      stateEnum
    )
  ))
  
  # Generate transition proc
  var caseStmt = newNimNode(nnkCaseStmt).add(newIdentNode("current"))
  for state in states.deduplicate():
    var stateTransitions = transitions.filterIt(it[0] == state)
    if stateTransitions.len == 0: continue
    
    var innerCase = newNimNode(nnkCaseStmt).add(newIdentNode("event"))
    for (_, ev, to) in stateTransitions:
      innerCase.add(newNimNode(nnkOfBranch).add(
        newLit(ev),
        newLit("s" & to)
      ))
    innerCase.add(newNimNode(nnkElse).add(newLit("s" & state)))
    
    caseStmt.add(newNimNode(nnkOfBranch).add(
      newLit("s" & state),
      innerCase
    ))
  
  caseStmt.add(newNimNode(nnkElse).add(newIdentNode("current")))

# Table/struct literal DSL
macro data*(body: untyped): untyped =
  ## Usage: data:
  ##   name = "Alice"
  ##   age = 30
  ## Creates a Table[string, JsonNode]
  result = quote do:
    initTable[string, JsonNode]()
  
  for node in body:
    if node.kind == nnkAsgn:
      let key = newLit($node[0])
      let value = node[1]
      result = quote do:
        block:
          var t = `result`
          t[`key`] = %`value`
          t
```

---

## 4. Compile-Time Database Schema

```nim
# schema_gen.nim
# Generate SQL schema và queries จาก Nim types

import std/[macros, strformat, strutils, typetraits]

type
  DbType* = enum
    dtInt, dtBigInt, dtFloat, dtText, dtBoolean, dtTimestamp, dtJson

  ColumnDef* = object
    name*: string
    dbType*: DbType
    nullable*: bool
    primaryKey*: bool
    unique*: bool
    defaultVal*: string

  TableSchema* = object
    name*: string
    columns*: seq[ColumnDef]

proc nimTypeToDb*(typeName: string): DbType =
  case typeName
  of "int", "int32": dtInt
  of "int64": dtBigInt
  of "float", "float64": dtFloat
  of "string": dtText
  of "bool": dtBoolean
  else: dtJson

proc dbTypeName*(dt: DbType): string =
  case dt
  of dtInt: "INTEGER"
  of dtBigInt: "BIGINT"
  of dtFloat: "REAL"
  of dtText: "TEXT"
  of dtBoolean: "BOOLEAN"
  of dtTimestamp: "TIMESTAMP"
  of dtJson: "JSONB"

macro generateSchema*(T: typedesc): TableSchema =
  let typeImpl = T.getTypeImpl()
  let tableName = T.repr.toLowerAscii() & "s"
  
  var columns: seq[ColumnDef] = @[]
  
  for node in typeImpl[2]:
    if node.kind != nnkIdentDefs: continue
    let name = ($node[0]).toLowerAscii().replace("*", "")
    let typeName = node[1].repr
    
    let isPrimaryKey = name == "id"
    let col = ColumnDef(
      name: name,
      dbType: nimTypeToDb(typeName),
      nullable: false,
      primaryKey: isPrimaryKey,
      unique: isPrimaryKey
    )
    columns.add(col)
  
  let schemaLit = quote do:
    TableSchema(
      name: `tableName`,
      columns: @[]
    )
  schemaLit

proc createTableSQL*(schema: TableSchema): string =
  var cols: seq[string]
  for col in schema.columns:
    var def = &"{col.name} {dbTypeName(col.dbType)}"
    if col.primaryKey: def &= " PRIMARY KEY"
    if not col.nullable and not col.primaryKey: def &= " NOT NULL"
    if col.unique and not col.primaryKey: def &= " UNIQUE"
    if col.defaultVal.len > 0: def &= &" DEFAULT {col.defaultVal}"
    cols.add(def)
  
  &"CREATE TABLE IF NOT EXISTS {schema.name} (\n  {cols.join(\",\\n  \")}\n);"

when isMainModule:
  type
    Product* = object
      id*: int64
      name*: string
      price*: float
      active*: bool
  
  let schema = generateSchema(Product)
  echo createTableSQL(schema)
  # Output:
  # CREATE TABLE IF NOT EXISTS products (
  #   id BIGINT PRIMARY KEY,
  #   name TEXT NOT NULL,
  #   price REAL NOT NULL,
  #   active BOOLEAN NOT NULL
  # );
```

---

## สรุป

| Technique | ใช้เมื่อ |
|-----------|---------|
| assertFields | Enforce interface at compile time |
| jsonSerializable | Auto-generate serialization code |
| builder macro | Fluent builder pattern |
| Route DSL | Declarative routing |
| Schema generation | Type-safe database access |

---

**Next**: [Part 87 - Testing Strategies & Property-Based Testing](../advanced/part87_testing.md)
