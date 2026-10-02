# Part 79 - Writing a Compiler/Interpreter in Nim

## บทนำ

สร้าง interpreter สำหรับ "MiniLang" — ภาษาเล็กที่ครอบคลุม:
- Lexer (tokenizer)
- Parser (AST)
- Evaluator (tree-walking interpreter)
- Bytecode compiler + VM

---

## 1. Lexer

```nim
# lexer.nim
# Tokenizer สำหรับ MiniLang

type
  TokenKind* = enum
    # Literals
    tkInt, tkFloat, tkString, tkBool, tkNil
    # Identifiers / Keywords
    tkIdent
    tkLet, tkFn, tkIf, tkElse, tkWhile, tkReturn
    tkAnd, tkOr, tkNot, tkTrue, tkFalse
    # Operators
    tkPlus, tkMinus, tkStar, tkSlash, tkPercent
    tkEq, tkNeq, tkLt, tkLe, tkGt, tkGe
    tkAssign
    # Delimiters
    tkLParen, tkRParen, tkLBrace, tkRBrace
    tkLBracket, tkRBracket
    tkComma, tkSemicolon, tkColon, tkDot
    # Special
    tkEOF, tkError

  Token* = object
    kind*: TokenKind
    lexeme*: string
    line*: int
    col*: int

  Lexer* = ref object
    source*: string
    pos*: int
    line*: int
    col*: int
    tokens*: seq[Token]

const Keywords* = {
  "let": tkLet, "fn": tkFn, "if": tkIf, "else": tkElse,
  "while": tkWhile, "return": tkReturn,
  "and": tkAnd, "or": tkOr, "not": tkNot,
  "true": tkTrue, "false": tkFalse, "nil": tkNil
}.toTable

proc newLexer*(source: string): Lexer =
  Lexer(source: source, pos: 0, line: 1, col: 1)

proc peek*(l: Lexer, offset = 0): char =
  let i = l.pos + offset
  if i < l.source.len: l.source[i] else: '\0'

proc advance*(l: Lexer): char =
  result = l.source[l.pos]
  inc l.pos
  if result == '\n':
    inc l.line
    l.col = 1
  else:
    inc l.col

proc match*(l: Lexer, expected: char): bool =
  if l.pos < l.source.len and l.source[l.pos] == expected:
    discard l.advance()
    true
  else:
    false

proc makeToken*(l: Lexer, kind: TokenKind, lexeme: string): Token =
  Token(kind: kind, lexeme: lexeme, line: l.line, col: l.col)

proc skipWhitespace*(l: Lexer) =
  while l.pos < l.source.len:
    case l.peek()
    of ' ', '\t', '\r', '\n':
      discard l.advance()
    of '#':  # Single-line comment
      while l.pos < l.source.len and l.peek() != '\n':
        discard l.advance()
    else:
      break

proc readString*(l: Lexer): Token =
  let startLine = l.line
  let startCol = l.col
  var s = ""
  
  while l.pos < l.source.len and l.peek() != '"':
    let c = l.advance()
    if c == '\\':
      case l.advance()
      of 'n': s.add('\n')
      of 't': s.add('\t')
      of '"': s.add('"')
      of '\\': s.add('\\')
      else: s.add('?')
    else:
      s.add(c)
  
  if l.pos >= l.source.len:
    return Token(kind: tkError, lexeme: "Unterminated string", line: startLine, col: startCol)
  
  discard l.advance()  # closing "
  Token(kind: tkString, lexeme: s, line: startLine, col: startCol)

proc readNumber*(l: Lexer, first: char): Token =
  var s = $first
  var isFloat = false
  
  while l.pos < l.source.len and l.peek().isDigit():
    s.add(l.advance())
  
  if l.peek() == '.' and l.peek(1).isDigit():
    isFloat = true
    s.add(l.advance())  # '.'
    while l.pos < l.source.len and l.peek().isDigit():
      s.add(l.advance())
  
  Token(kind: if isFloat: tkFloat else: tkInt, lexeme: s, line: l.line, col: l.col)

proc readIdent*(l: Lexer, first: char): Token =
  var s = $first
  while l.pos < l.source.len and (l.peek().isAlphaNumeric() or l.peek() == '_'):
    s.add(l.advance())
  
  let kind = Keywords.getOrDefault(s, tkIdent)
  Token(kind: kind, lexeme: s, line: l.line, col: l.col)

proc tokenize*(l: Lexer): seq[Token] =
  while true:
    l.skipWhitespace()
    if l.pos >= l.source.len:
      l.tokens.add(l.makeToken(tkEOF, ""))
      break
    
    let c = l.advance()
    let tok = case c
      of '(': l.makeToken(tkLParen, "(")
      of ')': l.makeToken(tkRParen, ")")
      of '{': l.makeToken(tkLBrace, "{")
      of '}': l.makeToken(tkRBrace, "}")
      of '[': l.makeToken(tkLBracket, "[")
      of ']': l.makeToken(tkRBracket, "]")
      of ',': l.makeToken(tkComma, ",")
      of ';': l.makeToken(tkSemicolon, ";")
      of ':': l.makeToken(tkColon, ":")
      of '.': l.makeToken(tkDot, ".")
      of '+': l.makeToken(tkPlus, "+")
      of '-': l.makeToken(tkMinus, "-")
      of '*': l.makeToken(tkStar, "*")
      of '/': l.makeToken(tkSlash, "/")
      of '%': l.makeToken(tkPercent, "%")
      of '=':
        if l.match('='): l.makeToken(tkEq, "==")
        else: l.makeToken(tkAssign, "=")
      of '!':
        if l.match('='): l.makeToken(tkNeq, "!=")
        else: l.makeToken(tkError, "unexpected '!'")
      of '<':
        if l.match('='): l.makeToken(tkLe, "<=")
        else: l.makeToken(tkLt, "<")
      of '>':
        if l.match('='): l.makeToken(tkGe, ">=")
        else: l.makeToken(tkGt, ">")
      of '"': l.readString()
      else:
        if c.isDigit(): l.readNumber(c)
        elif c.isAlpha() or c == '_': l.readIdent(c)
        else: l.makeToken(tkError, &"unexpected '{c}'")
    
    l.tokens.add(tok)
  
  l.tokens
```

---

## 2. AST + Parser

```nim
# parser.nim
# Recursive descent parser → AST

import lexer, std/[strformat, strutils]

type
  NodeKind* = enum
    nkInt, nkFloat, nkString, nkBool, nkNil
    nkIdent
    nkBinOp, nkUnOp
    nkAssign
    nkBlock
    nkIf
    nkWhile
    nkLet
    nkFn, nkCall
    nkReturn
    nkIndex
    nkArray
    nkProgram

  Node* = ref object
    line*: int
    case kind*: NodeKind
    of nkInt: intVal*: int64
    of nkFloat: floatVal*: float64
    of nkString: strVal*: string
    of nkBool: boolVal*: bool
    of nkNil: discard
    of nkIdent: name*: string
    of nkBinOp:
      op*: string
      left*, right*: Node
    of nkUnOp:
      unOp*: string
      operand*: Node
    of nkAssign:
      target*: Node
      value*: Node
    of nkBlock:
      stmts*: seq[Node]
    of nkIf:
      cond*: Node
      then*: Node
      els*: Node  # may be nil
    of nkWhile:
      whileCond*: Node
      whileBody*: Node
    of nkLet:
      letName*: string
      letValue*: Node
    of nkFn:
      fnName*: string
      params*: seq[string]
      body*: Node
    of nkCall:
      callee*: Node
      args*: seq[Node]
    of nkReturn:
      retVal*: Node
    of nkIndex:
      indexTarget*: Node
      index*: Node
    of nkArray:
      elements*: seq[Node]
    of nkProgram:
      program*: seq[Node]

  ParseError* = object of CatchableError

  Parser* = ref object
    tokens*: seq[Token]
    pos*: int

proc newParser*(tokens: seq[Token]): Parser =
  Parser(tokens: tokens, pos: 0)

proc peek*(p: Parser): Token = p.tokens[p.pos]
proc prev*(p: Parser): Token = p.tokens[p.pos - 1]
proc isAtEnd*(p: Parser): bool = p.peek().kind == tkEOF

proc advance*(p: Parser): Token =
  if not p.isAtEnd(): inc p.pos
  p.prev()

proc check*(p: Parser, kind: TokenKind): bool =
  not p.isAtEnd() and p.peek().kind == kind

proc match*(p: Parser, kinds: varargs[TokenKind]): bool =
  for kind in kinds:
    if p.check(kind):
      discard p.advance()
      return true
  false

proc expect*(p: Parser, kind: TokenKind, msg: string): Token =
  if p.check(kind): return p.advance()
  raise newException(ParseError,
    &"Line {p.peek().line}: Expected {msg}, got '{p.peek().lexeme}'")

# Forward declaration
proc expression*(p: Parser): Node
proc statement*(p: Parser): Node

proc primary*(p: Parser): Node =
  let tok = p.peek()
  
  if p.match(tkInt):
    return Node(kind: nkInt, intVal: parseInt(p.prev().lexeme), line: tok.line)
  
  if p.match(tkFloat):
    return Node(kind: nkFloat, floatVal: parseFloat(p.prev().lexeme), line: tok.line)
  
  if p.match(tkString):
    return Node(kind: nkString, strVal: p.prev().lexeme, line: tok.line)
  
  if p.match(tkTrue):
    return Node(kind: nkBool, boolVal: true, line: tok.line)
  
  if p.match(tkFalse):
    return Node(kind: nkBool, boolVal: false, line: tok.line)
  
  if p.match(tkNil):
    return Node(kind: nkNil, line: tok.line)
  
  if p.match(tkIdent):
    return Node(kind: nkIdent, name: p.prev().lexeme, line: tok.line)
  
  if p.match(tkLParen):
    let expr = p.expression()
    discard p.expect(tkRParen, ")")
    return expr
  
  if p.match(tkLBracket):
    var elements: seq[Node]
    if not p.check(tkRBracket):
      elements.add(p.expression())
      while p.match(tkComma):
        elements.add(p.expression())
    discard p.expect(tkRBracket, "]")
    return Node(kind: nkArray, elements: elements, line: tok.line)
  
  raise newException(ParseError, &"Line {tok.line}: Unexpected token '{tok.lexeme}'")

proc callExpr*(p: Parser): Node =
  var node = p.primary()
  
  while true:
    if p.match(tkLParen):
      var args: seq[Node]
      if not p.check(tkRParen):
        args.add(p.expression())
        while p.match(tkComma):
          args.add(p.expression())
      discard p.expect(tkRParen, ")")
      node = Node(kind: nkCall, callee: node, args: args, line: node.line)
    elif p.match(tkLBracket):
      let idx = p.expression()
      discard p.expect(tkRBracket, "]")
      node = Node(kind: nkIndex, indexTarget: node, index: idx, line: node.line)
    else:
      break
  
  node

proc unary*(p: Parser): Node =
  if p.match(tkMinus, tkNot):
    let op = p.prev().lexeme
    let line = p.prev().line
    let operand = p.unary()
    return Node(kind: nkUnOp, unOp: op, operand: operand, line: line)
  p.callExpr()

proc binary*(p: Parser, ops: seq[TokenKind], next: proc(): Node): Node =
  var left = next()
  while p.peek().kind in ops:
    let op = p.advance().lexeme
    let right = next()
    left = Node(kind: nkBinOp, op: op, left: left, right: right, line: left.line)
  left

proc expression*(p: Parser): Node =
  # Precedence: or < and < eq < cmp < add < mul < unary < call < primary
  proc orExpr(): Node =
    proc andExpr(): Node =
      proc eqExpr(): Node =
        proc cmpExpr(): Node =
          proc addExpr(): Node =
            proc mulExpr(): Node =
              p.binary(@[tkStar, tkSlash, tkPercent], proc(): Node = p.unary())
            p.binary(@[tkPlus, tkMinus], mulExpr)
          p.binary(@[tkLt, tkLe, tkGt, tkGe], addExpr)
        p.binary(@[tkEq, tkNeq], cmpExpr)
      p.binary(@[tkAnd], eqExpr)
    p.binary(@[tkOr], andExpr)
  orExpr()

proc block*(p: Parser): Node =
  discard p.expect(tkLBrace, "{")
  var stmts: seq[Node]
  while not p.check(tkRBrace) and not p.isAtEnd():
    stmts.add(p.statement())
  discard p.expect(tkRBrace, "}")
  Node(kind: nkBlock, stmts: stmts)

proc statement*(p: Parser): Node =
  let tok = p.peek()
  
  if p.match(tkLet):
    let name = p.expect(tkIdent, "identifier").lexeme
    discard p.expect(tkAssign, "=")
    let value = p.expression()
    discard p.match(tkSemicolon)
    return Node(kind: nkLet, letName: name, letValue: value, line: tok.line)
  
  if p.match(tkFn):
    let name = p.expect(tkIdent, "function name").lexeme
    discard p.expect(tkLParen, "(")
    var params: seq[string]
    if not p.check(tkRParen):
      params.add(p.expect(tkIdent, "parameter").lexeme)
      while p.match(tkComma):
        params.add(p.expect(tkIdent, "parameter").lexeme)
    discard p.expect(tkRParen, ")")
    let body = p.block()
    return Node(kind: nkFn, fnName: name, params: params, body: body, line: tok.line)
  
  if p.match(tkIf):
    let cond = p.expression()
    let then = p.block()
    var els: Node
    if p.match(tkElse):
      els = if p.check(tkIf): p.statement() else: p.block()
    return Node(kind: nkIf, cond: cond, then: then, els: els, line: tok.line)
  
  if p.match(tkWhile):
    let cond = p.expression()
    let body = p.block()
    return Node(kind: nkWhile, whileCond: cond, whileBody: body, line: tok.line)
  
  if p.match(tkReturn):
    let val = if not p.check(tkSemicolon) and not p.check(tkRBrace):
      p.expression()
    else:
      Node(kind: nkNil)
    discard p.match(tkSemicolon)
    return Node(kind: nkReturn, retVal: val, line: tok.line)
  
  let expr = p.expression()
  
  # Assignment
  if p.match(tkAssign):
    let value = p.expression()
    discard p.match(tkSemicolon)
    return Node(kind: nkAssign, target: expr, value: value, line: tok.line)
  
  discard p.match(tkSemicolon)
  expr

proc parse*(source: string): Node =
  let lexer = newLexer(source)
  let tokens = lexer.tokenize()
  let parser = newParser(tokens)
  
  var stmts: seq[Node]
  while not parser.isAtEnd():
    stmts.add(parser.statement())
  
  Node(kind: nkProgram, program: stmts)
```

---

## 3. Tree-Walking Evaluator

```nim
# eval.nim
# Tree-walking interpreter สำหรับ MiniLang

import parser, std/[tables, strformat, math]

type
  ValueKind* = enum
    vkInt, vkFloat, vkString, vkBool, vkNil
    vkArray, vkFunction, vkBuiltin

  Value* = ref object
    case kind*: ValueKind
    of vkInt: intVal*: int64
    of vkFloat: floatVal*: float64
    of vkString: strVal*: string
    of vkBool: boolVal*: bool
    of vkNil: discard
    of vkArray: elements*: seq[Value]
    of vkFunction:
      params*: seq[string]
      body*: Node
      closure*: Env
    of vkBuiltin:
      name*: string
      fn*: proc(args: seq[Value]): Value

  Env* = ref object
    vars*: Table[string, Value]
    parent*: Env

  ReturnSignal* = object of CatchableError
    value*: Value

  EvalError* = object of CatchableError

proc newEnv*(parent: Env = nil): Env =
  Env(vars: initTable[string, Value](), parent: parent)

proc get*(env: Env, name: string): Value =
  if name in env.vars: return env.vars[name]
  if not env.parent.isNil: return env.parent.get(name)
  raise newException(EvalError, &"Undefined variable: {name}")

proc set*(env: Env, name: string, val: Value) =
  if name in env.vars:
    env.vars[name] = val
    return
  if not env.parent.isNil:
    env.parent.set(name, val)
    return
  raise newException(EvalError, &"Undefined variable: {name}")

proc define*(env: Env, name: string, val: Value) =
  env.vars[name] = val

proc `$`*(v: Value): string =
  if v.isNil: return "nil"
  case v.kind
  of vkInt: $v.intVal
  of vkFloat: $v.floatVal
  of vkString: v.strVal
  of vkBool: $v.boolVal
  of vkNil: "nil"
  of vkArray: "[" & v.elements.mapIt($it).join(", ") & "]"
  of vkFunction: &"<fn({v.params.join(\", \")})>"
  of vkBuiltin: &"<builtin:{v.name}>"

proc isTruthy*(v: Value): bool =
  if v.isNil: return false
  case v.kind
  of vkBool: v.boolVal
  of vkNil: false
  of vkInt: v.intVal != 0
  of vkFloat: v.floatVal != 0.0
  of vkString: v.strVal.len > 0
  of vkArray: v.elements.len > 0
  else: true

proc makeBuiltins*(env: Env) =
  env.define("print", Value(kind: vkBuiltin, name: "print", fn: proc(args: seq[Value]): Value =
    echo args.mapIt($it).join(" ")
    Value(kind: vkNil)
  ))
  
  env.define("len", Value(kind: vkBuiltin, name: "len", fn: proc(args: seq[Value]): Value =
    if args.len != 1: raise newException(EvalError, "len() takes 1 argument")
    case args[0].kind
    of vkString: Value(kind: vkInt, intVal: args[0].strVal.len)
    of vkArray: Value(kind: vkInt, intVal: args[0].elements.len)
    else: raise newException(EvalError, "len() requires string or array")
  ))
  
  env.define("push", Value(kind: vkBuiltin, name: "push", fn: proc(args: seq[Value]): Value =
    if args.len != 2: raise newException(EvalError, "push() takes 2 arguments")
    if args[0].kind != vkArray: raise newException(EvalError, "push() requires array")
    let arr = Value(kind: vkArray, elements: args[0].elements & @[args[1]])
    arr
  ))

proc eval*(node: Node, env: Env): Value

proc evalBinOp*(op: string, left, right: Value): Value =
  # Arithmetic
  if left.kind == vkInt and right.kind == vkInt:
    let a = left.intVal; let b = right.intVal
    case op
    of "+": return Value(kind: vkInt, intVal: a + b)
    of "-": return Value(kind: vkInt, intVal: a - b)
    of "*": return Value(kind: vkInt, intVal: a * b)
    of "/": return Value(kind: vkFloat, floatVal: a.float / b.float)
    of "%": return Value(kind: vkInt, intVal: a mod b)
    of "==": return Value(kind: vkBool, boolVal: a == b)
    of "!=": return Value(kind: vkBool, boolVal: a != b)
    of "<": return Value(kind: vkBool, boolVal: a < b)
    of "<=": return Value(kind: vkBool, boolVal: a <= b)
    of ">": return Value(kind: vkBool, boolVal: a > b)
    of ">=": return Value(kind: vkBool, boolVal: a >= b)
    else: discard
  
  if op == "+" and left.kind == vkString and right.kind == vkString:
    return Value(kind: vkString, strVal: left.strVal & right.strVal)
  
  if op == "==" :
    return Value(kind: vkBool, boolVal: $left == $right)
  if op == "!=":
    return Value(kind: vkBool, boolVal: $left != $right)
  
  raise newException(EvalError, &"Invalid operation: {left.kind} {op} {right.kind}")

proc eval*(node: Node, env: Env): Value =
  if node.isNil:
    return Value(kind: vkNil)
  
  case node.kind
  of nkInt: Value(kind: vkInt, intVal: node.intVal)
  of nkFloat: Value(kind: vkFloat, floatVal: node.floatVal)
  of nkString: Value(kind: vkString, strVal: node.strVal)
  of nkBool: Value(kind: vkBool, boolVal: node.boolVal)
  of nkNil: Value(kind: vkNil)
  
  of nkIdent:
    env.get(node.name)
  
  of nkBinOp:
    if node.op == "and":
      let l = eval(node.left, env)
      if not l.isTruthy(): return l
      return eval(node.right, env)
    elif node.op == "or":
      let l = eval(node.left, env)
      if l.isTruthy(): return l
      return eval(node.right, env)
    
    let left = eval(node.left, env)
    let right = eval(node.right, env)
    evalBinOp(node.op, left, right)
  
  of nkUnOp:
    let val = eval(node.operand, env)
    case node.unOp
    of "-":
      if val.kind == vkInt: Value(kind: vkInt, intVal: -val.intVal)
      elif val.kind == vkFloat: Value(kind: vkFloat, floatVal: -val.floatVal)
      else: raise newException(EvalError, "Cannot negate non-number")
    of "not": Value(kind: vkBool, boolVal: not val.isTruthy())
    else: raise newException(EvalError, &"Unknown unary op: {node.unOp}")
  
  of nkLet:
    let val = eval(node.letValue, env)
    env.define(node.letName, val)
    val
  
  of nkAssign:
    let val = eval(node.value, env)
    if node.target.kind == nkIdent:
      env.set(node.target.name, val)
    elif node.target.kind == nkIndex:
      let arr = eval(node.target.indexTarget, env)
      let idx = eval(node.target.index, env)
      if arr.kind == vkArray and idx.kind == vkInt:
        arr.elements[idx.intVal] = val
      else:
        raise newException(EvalError, "Invalid assignment target")
    val
  
  of nkBlock:
    let blockEnv = newEnv(env)
    var last: Value = Value(kind: vkNil)
    for stmt in node.stmts:
      last = eval(stmt, blockEnv)
    last
  
  of nkIf:
    let cond = eval(node.cond, env)
    if cond.isTruthy():
      eval(node.then, env)
    elif not node.els.isNil:
      eval(node.els, env)
    else:
      Value(kind: vkNil)
  
  of nkWhile:
    while eval(node.whileCond, env).isTruthy():
      discard eval(node.whileBody, env)
    Value(kind: vkNil)
  
  of nkFn:
    let fn = Value(kind: vkFunction, params: node.params, body: node.body, closure: env)
    if node.fnName.len > 0:
      env.define(node.fnName, fn)
    fn
  
  of nkCall:
    let callee = eval(node.callee, env)
    let args = node.args.mapIt(eval(it, env))
    
    case callee.kind
    of vkBuiltin:
      callee.fn(args)
    of vkFunction:
      if args.len != callee.params.len:
        raise newException(EvalError,
          &"Expected {callee.params.len} args, got {args.len}")
      let fnEnv = newEnv(callee.closure)
      for i, param in callee.params:
        fnEnv.define(param, args[i])
      try:
        eval(callee.body, fnEnv)
      except ReturnSignal as ret:
        ret.value
    else:
      raise newException(EvalError, &"Cannot call {callee.kind}")
  
  of nkReturn:
    let val = eval(node.retVal, env)
    var ex = newException(ReturnSignal, "return")
    ex.value = val
    raise ex
  
  of nkArray:
    Value(kind: vkArray, elements: node.elements.mapIt(eval(it, env)))
  
  of nkIndex:
    let target = eval(node.indexTarget, env)
    let idx = eval(node.index, env)
    case target.kind
    of vkArray:
      if idx.kind != vkInt: raise newException(EvalError, "Array index must be int")
      target.elements[idx.intVal]
    of vkString:
      if idx.kind != vkInt: raise newException(EvalError, "String index must be int")
      Value(kind: vkString, strVal: $target.strVal[idx.intVal])
    else:
      raise newException(EvalError, &"Cannot index {target.kind}")
  
  of nkProgram:
    var last: Value = Value(kind: vkNil)
    for stmt in node.program:
      last = eval(stmt, env)
    last

proc run*(source: string): Value =
  let ast = parse(source)
  let env = newEnv()
  env.makeBuiltins()
  eval(ast, env)

# REPL
when isMainModule:
  import std/rdstdin
  
  let env = newEnv()
  env.makeBuiltins()
  
  echo "MiniLang REPL v1.0. Type 'quit' to exit."
  
  while true:
    let line = readLineFromStdin(">> ")
    if line == "quit": break
    
    try:
      let ast = parse(line)
      let result = eval(ast, env)
      if result.kind != vkNil:
        echo $result
    except ParseError as e:
      echo &"Parse error: {e.msg}"
    except EvalError as e:
      echo &"Runtime error: {e.msg}"
```

---

## สรุป

| Phase | Role |
|-------|------|
| Lexer | Source text → tokens |
| Parser | Tokens → AST (recursive descent) |
| Evaluator | AST → values (tree-walking) |
| Environment | Variable scoping + closures |
| ReturnSignal | Exception-based control flow |

---

**Next**: [Part 80 - Binary Serialization & Protocol Buffers](../advanced/part80_serialization.md)
