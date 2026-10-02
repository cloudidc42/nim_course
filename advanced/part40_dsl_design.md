# Part 40: DSL Design และ Language-Oriented Programming

## DSL คืออะไร

```
DSL = Domain-Specific Language
ภาษาที่ออกแบบมาสำหรับงานเฉพาะด้าน

DSL ใน Nim ทำได้ด้วย:
- Macros: แปลง syntax ใหม่ตอน compile time
- Templates: เอา code block ไปแทนที่
- UFCS: เสมือน method chaining
- Operator overloading: สร้าง operator ใหม่
ตัวอย่าง: jester routes, unittest suite/test, niml HTML DSL
```

## HTML DSL

```nim
# html_dsl.nim
# สร้าง HTML แบบ type-safe ใน Nim

import macros, strutils, sequtils

type
  HtmlNode = ref object
    tag: string
    attrs: seq[(string, string)]
    children: seq[HtmlNode]
    text: string

proc newHtml(tag: string): HtmlNode =
  HtmlNode(tag: tag, attrs: @[], children: @[])

proc attr*(n: HtmlNode, key, val: string): HtmlNode =
  n.attrs.add((key, val))
  n

proc add*(n: HtmlNode, child: HtmlNode): HtmlNode =
  n.children.add(child)
  n

proc text*(n: HtmlNode, t: string): HtmlNode =
  n.text = t
  n

proc render*(n: HtmlNode, indent: int = 0): string =
  let pad = "  ".repeat(indent)
  var attStr = ""
  for (k, v) in n.attrs:
    attStr &= " " & k & "=\"" & v & "\""
  
  if n.text.len > 0:
    result = pad & "<" & n.tag & attStr & ">" & n.text & "</" & n.tag & ">"
  elif n.children.len == 0:
    result = pad & "<" & n.tag & attStr & " />"
  else:
    result = pad & "<" & n.tag & attStr & ">\n"
    for child in n.children:
      result &= child.render(indent + 1) & "\n"
    result &= pad & "</" & n.tag & ">"

# Fluent API
let page = newHtml("html").add(
  newHtml("head").add(
    newHtml("title").text("My Page")
  )
).add(
  newHtml("body").attr("class", "container").add(
    newHtml("h1").text("Hello, World!")
  ).add(
    newHtml("p").attr("class", "intro").text("Welcome to Nim HTML DSL")
  ).add(
    newHtml("ul").add(
      newHtml("li").text("Item 1")
    ).add(
      newHtml("li").text("Item 2")
    )
  )
)

echo render(page)

# Macro-based DSL (more ergonomic)
macro buildHtml(body: untyped): untyped =
  ## Generate HTML building code from DSL syntax
  proc processElement(n: NimNode): NimNode =
    if n.kind == nnkCall:
      let tag = newLit($n[0])
      var children: seq[NimNode] = @[]
      var attrs: seq[NimNode] = @[]
      var textNode: NimNode = nil
      
      for i in 1..<n.len:
        let child = n[i]
        if child.kind == nnkStrLit:   # Text content
          textNode = child
        elif child.kind == nnkExprEqExpr:  # attribute
          attrs.add(newLit($child[0]))
          attrs.add(child[1])
        elif child.kind == nnkStmtList:  # Nested
          for sub in child:
            children.add(processElement(sub))
      
      result = quote do:
        newHtml(`tag`)
    else:
      newNilLit()
  
  processElement(body)
```

## Query Builder DSL

```nim
# query_builder.nim
# Type-safe SQL query builder

import strutils, sequtils

type
  QueryBuilder = object
    table: string
    conditions: seq[string]
    orderByField: string
    limitVal: int
    offsetVal: int
    selectFields: seq[string]
    joinClauses: seq[string]

proc select*(fields: varargs[string]): QueryBuilder =
  QueryBuilder(selectFields: @fields, limitVal: -1)

proc fromTable*(q: QueryBuilder, table: string): QueryBuilder =
  result = q
  result.table = table

proc where*(q: QueryBuilder, condition: string): QueryBuilder =
  result = q
  result.conditions.add(condition)

proc andWhere*(q: QueryBuilder, condition: string): QueryBuilder =
  q.where(condition)

proc orderBy*(q: QueryBuilder, field: string): QueryBuilder =
  result = q
  result.orderByField = field

proc limit*(q: QueryBuilder, n: int): QueryBuilder =
  result = q
  result.limitVal = n

proc offset*(q: QueryBuilder, n: int): QueryBuilder =
  result = q
  result.offsetVal = n

proc join*(q: QueryBuilder, table, condition: string): QueryBuilder =
  result = q
  result.joinClauses.add("JOIN " & table & " ON " & condition)

proc leftJoin*(q: QueryBuilder, table, condition: string): QueryBuilder =
  result = q
  result.joinClauses.add("LEFT JOIN " & table & " ON " & condition)

proc build*(q: QueryBuilder): string =
  let fields = if q.selectFields.len == 0: "*"
               else: q.selectFields.join(", ")
  
  result = "SELECT " & fields & " FROM " & q.table
  
  for joinClause in q.joinClauses:
    result &= "\n" & joinClause
  
  if q.conditions.len > 0:
    result &= "\nWHERE " & q.conditions.join(" AND ")
  
  if q.orderByField.len > 0:
    result &= "\nORDER BY " & q.orderByField
  
  if q.limitVal > 0:
    result &= "\nLIMIT " & $q.limitVal
  
  if q.offsetVal > 0:
    result &= "\nOFFSET " & $q.offsetVal

# Usage - reads like SQL!
let query = select("u.id", "u.name", "p.title")
  .fromTable("users u")
  .leftJoin("posts p", "p.user_id = u.id")
  .where("u.active = 1")
  .andWhere("u.age > 18")
  .orderBy("u.name")
  .limit(20)
  .offset(0)
  .build()

echo query
```

## Routing DSL

```nim
# routing_dsl.nim
# HTTP router DSL without external dependencies

import asynchttpserver, asyncdispatch, strutils, tables, re

type
  HttpHandler = proc(req: Request): Future[void] {.gcsafe.}
  Route = object
    pattern: Regex
    handler: HttpHandler
    paramNames: seq[string]
  
  Router = object
    getRoutes: seq[Route]
    postRoutes: seq[Route]
    putRoutes: seq[Route]
    deleteRoutes: seq[Route]

proc pathToRegex(path: string): tuple[r: Regex, params: seq[string]] =
  var pattern = path
  var params: seq[string] = @[]
  
  # Extract :param patterns
  for m in findAll(path, re":(\w+)"):
    params.add(m[1..^1])
    pattern = pattern.replace(m, "([^/]+)")
  
  (re("^" & pattern & "$"), params)

var router = Router()

template get*(path: string, handler: untyped) =
  let (pattern, params) = pathToRegex(path)
  router.getRoutes.add(Route(
    pattern: pattern,
    handler: handler,
    paramNames: params
  ))

template post*(path: string, handler: untyped) =
  let (pattern, params) = pathToRegex(path)
  router.postRoutes.add(Route(
    pattern: pattern,
    handler: handler,
    paramNames: params
  ))

proc dispatch*(router: Router, req: Request): Future[void] {.async.} =
  let routes = case req.reqMethod
    of HttpGet: router.getRoutes
    of HttpPost: router.postRoutes
    of HttpPut: router.putRoutes
    of HttpDelete: router.deleteRoutes
    else: @[]
  
  for route in routes:
    var m: RegexMatch
    if match(req.url.path, route.pattern, m):
      await route.handler(req)
      return
  
  await req.respond(Http404, "Not Found")

# Usage:
get "/" do (req: Request):
  await req.respond(Http200, "<h1>Home</h1>")

get "/users/:id" do (req: Request):
  await req.respond(Http200, "User page")

post "/users" do (req: Request):
  await req.respond(Http201, "Created")
```

## Config DSL

```nim
# config_dsl.nim
# สร้าง config system แบบ type-safe

import macros, tables, strutils, os

# DSL สำหรับ กำหนด config schema
macro config(body: untyped): untyped =
  ## Create type-safe config from DSL
  var fields: seq[(string, string, NimNode)] = @[]  # (name, type, default)
  
  for stmt in body:
    if stmt.kind == nnkCall:
      let fieldName = $stmt[0]
      let fieldType = $stmt[1][0]  # first arg = type
      let defaultVal = if stmt[1].len > 1: stmt[1][1]
                       else: newEmptyNode()
      fields.add((fieldName, fieldType, defaultVal))
  
  # Generate AppConfig type
  var recList = nnkRecList.newTree()
  for (name, typ, _) in fields:
    recList.add nnkIdentDefs.newTree(
      ident(name), ident(typ), newEmptyNode()
    )
  
  result = nnkTypeSection.newTree(
    nnkTypeDef.newTree(
      ident("AppConfig"),
      newEmptyNode(),
      nnkObjectTy.newTree(
        newEmptyNode(), newEmptyNode(), recList
      )
    )
  )

# Simple config loader
proc loadConfig(path: string): Table[string, string] =
  result = initTable[string, string]()
  if not fileExists(path): return
  
  for line in lines(path):
    let stripped = line.strip()
    if stripped.startsWith("#") or stripped.len == 0: continue
    let parts = stripped.split("=", maxsplit = 1)
    if parts.len == 2:
      result[parts[0].strip()] = parts[1].strip()

# Type-safe config with env override
type ServerConfig = object
  host: string
  port: int
  debug: bool
  maxConns: int
  dbUrl: string

proc defaultConfig(): ServerConfig =
  ServerConfig(
    host: "0.0.0.0",
    port: 8080,
    debug: false,
    maxConns: 100,
    dbUrl: "sqlite:///app.db"
  )

proc withEnv(c: ServerConfig): ServerConfig =
  result = c
  if (let h = getEnv("SERVER_HOST"); h.len > 0): result.host = h
  if (let p = getEnv("SERVER_PORT"); p.len > 0): result.port = parseInt(p)
  if (let d = getEnv("DEBUG"); d.len > 0): result.debug = d == "true"
  if (let db = getEnv("DATABASE_URL"); db.len > 0): result.dbUrl = db

let cfg = defaultConfig().withEnv()
echo "Server: ", cfg.host, ":", cfg.port
echo "Debug: ", cfg.debug
```

## สรุป Part 40

ในบทนี้เราได้เรียนรู้:
- ✅ HTML DSL ด้วย fluent API
- ✅ SQL Query Builder DSL
- ✅ HTTP Routing DSL
- ␅ Config DSL
- ✅ Macro-based DSL patterns

---

**Previous**: [Part 39 - Compiler Internals](part39_compiler_internals.md)
**Next**: [Part 41 - Documentation](part41_documentation.md)
