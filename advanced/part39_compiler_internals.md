# Part 39: Nim Compiler Internals และ NimScript

## Nim Compiler Architecture

```
Source Code (.nim)
    |
    v
Parser -> AST (Abstract Syntax Tree)
    |
    v
Semantic Analysis (sem*.nim)
    |
    v
Code Generator
    |
    +---> C Backend    (cgen.nim) -> .c files -> GCC/Clang -> binary
    +---> C++ Backend  -> .cpp files
    +---> JS Backend   -> .js file
    +---> LLVM Backend (via nlvm) -> LLVM IR

Nim source: https://github.com/nim-lang/Nim
Key files:
  compiler/parser.nim    - tokenizer + parser
  compiler/sem.nim       - semantic analysis
  compiler/cgen.nim      - C code generation
  compiler/macros.nim    - macro processing
```

## NimScript

```nim
# config.nims
# NimScript: Nim as a scripting language
# Used for: build scripts, nimble tasks, IDE configuration

# Compiler settings
switch("opt", "speed")
switch("define", "release")
switch("gc", "orc")
switch("threads", "on")

# Conditional configuration
if defined(windows):
  switch("passC", "-DWINDOWS")
  switch("passL", "-lws2_32")
elif defined(macosx):
  switch("passC", "-DMACOS")
  switch("passL", "-framework CoreFoundation")
else:
  switch("passC", "-DLINUX")

# Path configuration  
pathAdd("src")
pathAdd("vendor/libs")
```

## nimble Scripts แบบได้ผล

```nim
# myproject.nimble - full project config
version = "1.0.0"
author   = "Alice"
description = "My Nim Application"
license = "MIT"

# Dependencies
requires "nim >= 2.0.0"
requires "jester >= 0.5.0"
requires "db_connector >= 0.1.0"
requires "checksums >= 0.1.0"

# Source directories
srcDir = "src"
binDir = "bin"
bin = @["main"]

# Custom tasks
task test, "Run all tests":
  exec "nim c -r tests/test_all.nim"

task docs, "Generate documentation":
  exec "nim doc --project --index:on --outdir:docs src/main.nim"

task release, "Build release binaries":
  mkDir "bin"
  exec "nim c -d:release --opt:speed -o:bin/app src/main.nim"

task clean, "Clean build artifacts":
  rmDir "bin"
  exec "find . -name '*.c' -delete"
  exec "find . -name '*.o' -delete"

task lint, "Run Nim linter":
  exec "nim check src/main.nim"
  exec "nim c --hints:on --warnings:on src/main.nim"

task fmt, "Format source code (with nimpretty)":
  for f in listFiles("src"):
    if f.endsWith(".nim"):
      exec "nimpretty --indent:2 " & f

task bench, "Run benchmarks":
  exec "nim c -d:release -r benchmarks/bench.nim"

task docker, "Build Docker image":
  exec "nim c -d:release -o:bin/app_linux src/main.nim"
  exec "docker build -t myapp ."

task ci, "CI pipeline: test + docs + lint":
  exec "nimble test"
  exec "nimble docs"
  exec "nimble lint"
```

## การ Debug Nim

```nim
# debug_tools.nim

# วิธี debug ใน Nim
import sugar, strutils, macros

# 1. dump macro (ดูค่า)
import sugar
let x = 42
let y = "hello"
dump(x)         # x = 42
dump(x + 1)     # x + 1 = 43
dump(y.len)     # y.len = 5

# 2. Custom debug macro
macro dbg(expr: untyped): untyped =
  let exprStr = expr.toStrLit()
  quote do:
    let val = `expr`
    echo `exprStr`, " = ", val
    val

let result = dbg(2 + 2 * 3)  # 2 + 2 * 3 = 8

# 3. Debug build vs release
when not defined(release):
  proc debugLog(msg: string) =
    echo "[DEBUG] ", msg
else:
  template debugLog(msg: string) = discard  # No-op in release

# 4. Stack trace
proc foo() =
  raise newException(ValueError, "test error")

try:
  foo()
except ValueError as e:
  echo e.msg
  echo getStackTrace(e)

# 5. Nim verbosity flags
# nim c --verbosity:2 myfile.nim   # show all actions
# nim c --hints:on myfile.nim      # show hints
# nim c --warnings:on myfile.nim   # show warnings
```

## การใช้ Compiler Directives

```nim
# compiler_directives.nim

# {.pragma.} directives
proc fastProc() {.inline, noSideEffect.} =
  discard  # inlined, no side effects

proc deprecated_fn() {.deprecated: "Use newFn instead".} =
  discard

type Stackable {.inheritable.} = object
  data: int

{.push warning[UnusedImport]: off.}  # disable warning
import unused_module
{.pop.}  # restore

{.hint[Performance]: on.}  # enable performance hints

# Global variable initialization order
var g1 {.global.}: int = 0  # thread-local if {.threadvar.}

# Experimental features
{.experimental: "views".}
{.experimental: "strictNotNil".}

# Line directives (for source maps)
{.line: "myfile.nim", 100.}:
  echo "I appear to be on line 100"

# Platform-specific compilation
{.passC: "-mavx2".}   # Pass flag to C compiler
{.passL: "-lm".}      # Pass flag to linker

# Compile-time messages
{.hint: "Building for target: " & $hostOS.}
{.warning: "This is experimental".}
when false:
  {.fatal: "This should not be compiled".}
```

## NimVM และ Compile-time Execution

```nim
# nimvm.nim
# ข้อผิดพลาดเกี่ยวกับ NimVM

# NimVM = virtual machine ที่ Nim compiler ใช้รัน code ตอน compile time
# สำหรับ: const, static, when expressions, macros

# Compile-time computation
proc fib(n: int): int {.compileTime.} =
  if n <= 1: n
  else: fib(n-1) + fib(n-2)

const fib10 = fib(10)  # computed at compile time = 55
echo fib10

# Compile-time file reading
const cppCode = staticRead("template.cpp")  # read file at compile time
# embed into binary:
# const embeddedData = staticRead("data.bin")

# NimVM limitations:
# 1. Cannot call C FFI functions
# 2. No thread-local storage
# 3. Limited to pure Nim code
# 4. Cannot use unsafe memory operations
# 5. No I/O except staticRead/staticExec

# staticExec: run command at compile time
const gitHash = staticExec("git rev-parse --short HEAD")
echo "Build from: ", gitHash

# Compile-time type validation
proc validateType(T: typedesc): bool {.compileTime.} =
  when T is int or T is float or T is string:
    true
  else:
    false

static:
  assert validateType(int)
  assert validateType(string)
  # assert validateType(seq[int]) would fail
```

## สรุป Part 39

ในบทนี้เราได้เรียนรู้:
- ✅ Nim compiler architecture
- ✅ NimScript และ config.nims
- ✅ nimble tasks แบบได้ผล
- ␅ Debug techniques
- ✅ Compiler directives
- ✅ NimVM และ compile-time execution

---

**Previous**: [Part 38 - Advanced Macros](part38_advanced_macros.md)
**Next**: [Part 40 - DSL Design](part40_dsl_design.md)
