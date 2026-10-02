# Part 21: FFI กับ C - Foreign Function Interface

## เรียกใช้ C Functions

```nim
# เรียก C standard library functions
proc printf(fmt: cstring): cint {.importc, varargs, header: "<stdio.h>".}
proc strlen(s: cstring): csize_t {.importc, header: "<string.h>".}
proc malloc(size: csize_t): pointer {.importc, header: "<stdlib.h>".}
proc free(p: pointer) {.importc, header: "<stdlib.h>".}

# ใช้งาน
printf("Hello from C!\n")
printf("Count: %d\n", 42)

let s: cstring = "Hello"
echo "C strlen: ", strlen(s)  # 5

# Math functions
proc c_sqrt(x: cdouble): cdouble {.importc: "sqrt", header: "<math.h>".}
proc c_pow(base, exp: cdouble): cdouble {.importc: "pow", header: "<math.h>".}
proc c_abs(x: cint): cint {.importc: "abs", header: "<stdlib.h>".}

echo c_sqrt(16.0)    # 4.0
echo c_pow(2.0, 10.0)  # 1024.0
echo c_abs(-42)      # 42

# Time functions
type TimeT = int64
proc c_time(t: ptr TimeT): TimeT {.importc: "time", header: "<time.h>".}

var t: TimeT
echo "Unix time: ", c_time(addr t)
```

## C Types และ Nim Types

```nim
# Type mapping
# C         -> Nim
# int        -> cint
# long       -> clong
# char       -> cchar
# float      -> cfloat
# double     -> cdouble
# char*      -> cstring
# void*      -> pointer
# size_t     -> csize_t
# int32_t    -> int32
# uint64_t   -> uint64
# bool       -> bool (C99)

# Struct mapping
type
  CPoint {.importc: "Point", header: "mylib.h".} = object
    x, y: cint
  
  CRect {.importc: "Rect", header: "mylib.h".} = object
    x, y, w, h: cint

# Union mapping
type
  MyUnion {.union.} = object
    i: cint
    f: cfloat
    b: array[4, uint8]

var u: MyUnion
u.i = 0x3F800000  # float 1.0 in hex
echo u.f  # ~1.0

# Enum mapping
type
  ColorEnum {.importc: "Color", header: "graphics.h".} = enum
    RED = 0, GREEN = 1, BLUE = 2
```

## Inline C Code

```nim
# Inline C ตรงๆ
{.emit: """
#include <stdio.h>

void sayHello(const char* name) {
    printf("Hello, %s!\n", name);
}

int add(int a, int b) {
    return a + b;
}
""".}

proc sayHello(name: cstring) {.importc.}
proc addC(a, b: cint): cint {.importc: "add".}

sayHello("World")
echo addC(3, 4)

# Emit กับ Nim variables
proc fastAbs(x: int): int =
  {.emit: "result = x < 0 ? -x : x;".}
```

## สร้าง C Library Wrapper

```nim
# Wrapper สำหรับ OpenSSL (example)
{.passL: "-lssl -lcrypto".}

type
  EVP_MD_CTX {.importc, header: "<openssl/evp.h>", incompleteStruct.} = object
  EVP_MD {.importc, header: "<openssl/evp.h>", incompleteStruct.} = object

proc EVP_MD_CTX_new(): ptr EVP_MD_CTX {.importc, header: "<openssl/evp.h>".}
proc EVP_MD_CTX_free(ctx: ptr EVP_MD_CTX) {.importc, header: "<openssl/evp.h>".}
proc EVP_sha256(): ptr EVP_MD {.importc, header: "<openssl/evp.h>".}
proc EVP_DigestInit_ex(ctx: ptr EVP_MD_CTX, md: ptr EVP_MD, impl: pointer): cint {.importc, header: "<openssl/evp.h>".}
proc EVP_DigestUpdate(ctx: ptr EVP_MD_CTX, d: pointer, cnt: csize_t): cint {.importc, header: "<openssl/evp.h>".}
proc EVP_DigestFinal_ex(ctx: ptr EVP_MD_CTX, md: ptr UncheckedArray[uint8], s: ptr cuint): cint {.importc, header: "<openssl/evp.h>".}

proc sha256(data: string): string =
  let ctx = EVP_MD_CTX_new()
  defer: EVP_MD_CTX_free(ctx)
  
  var digest: array[32, uint8]
  var digestLen: cuint = 32
  
  discard EVP_DigestInit_ex(ctx, EVP_sha256(), nil)
  discard EVP_DigestUpdate(ctx, data.cstring, data.len.csize_t)
  discard EVP_DigestFinal_ex(ctx, cast[ptr UncheckedArray[uint8]](addr digest[0]), addr digestLen)
  
  result = ""
  for b in digest:
    result &= b.toHex(2).toLower()

# echo sha256("hello")
```

## Windows API

```nim
# Windows API wrappers
when defined(windows):
  import winlean
  
  proc MessageBoxA(hwnd: pointer, text, caption: cstring, 
                   utype: cuint): cint 
    {.importc, header: "<windows.h>".}
  
  proc GetLastError(): culong {.importc, header: "<windows.h>".}
  
  proc VirtualAlloc(lpAddress: pointer, dwSize: csize_t, 
                    flAllocationType, flProtect: culong): pointer 
    {.importc, header: "<windows.h>".}
  
  proc VirtualFree(lpAddress: pointer, dwSize: csize_t, 
                   dwFreeType: culong): bool 
    {.importc, header: "<windows.h>".}
  
  const
    MB_OK = 0
    MB_YESNO = 4
    MEM_COMMIT = 0x1000
    MEM_RESERVE = 0x2000
    PAGE_READWRITE = 0x04
    MEM_RELEASE = 0x8000
  
  # ใช้งาน
  discard MessageBoxA(nil, "Hello from Nim!", "Test", MB_OK)
  
  # Allocate memory
  let mem = VirtualAlloc(nil, 4096, MEM_COMMIT or MEM_RESERVE, PAGE_READWRITE)
  if mem != nil:
    echo "Allocated 4096 bytes at: 0x", cast[int](mem).toHex()
    discard VirtualFree(mem, 0, MEM_RELEASE)
```

## Practical: Wrapper สำหรับ SQLite

```nim
# sqlite3_wrapper.nim

{.passL: "-lsqlite3".}

type
  Sqlite3 {.importc: "sqlite3", header: "<sqlite3.h>", incompleteStruct.} = object
  Sqlite3Stmt {.importc: "sqlite3_stmt", header: "<sqlite3.h>", incompleteStruct.} = object

proc sqlite3_open(filename: cstring, ppDb: ptr ptr Sqlite3): cint 
  {.importc, header: "<sqlite3.h>".}
proc sqlite3_close(db: ptr Sqlite3): cint 
  {.importc, header: "<sqlite3.h>".}
proc sqlite3_exec(db: ptr Sqlite3, sql, callback: cstring, 
                  arg: pointer, errmsg: ptr cstring): cint 
  {.importc, header: "<sqlite3.h>".}
proc sqlite3_errmsg(db: ptr Sqlite3): cstring 
  {.importc, header: "<sqlite3.h>".}
proc sqlite3_prepare_v2(db: ptr Sqlite3, zSql: cstring, nByte: cint,
                        ppStmt: ptr ptr Sqlite3Stmt, pzTail: ptr cstring): cint 
  {.importc, header: "<sqlite3.h>".}
proc sqlite3_step(pStmt: ptr Sqlite3Stmt): cint 
  {.importc, header: "<sqlite3.h>".}
proc sqlite3_finalize(pStmt: ptr Sqlite3Stmt): cint 
  {.importc, header: "<sqlite3.h>".}
proc sqlite3_column_text(pStmt: ptr Sqlite3Stmt, iCol: cint): cstring 
  {.importc, header: "<sqlite3.h>".}
proc sqlite3_column_int(pStmt: ptr Sqlite3Stmt, iCol: cint): cint 
  {.importc, header: "<sqlite3.h>".}

const
  SQLITE_OK = 0
  SQLITE_ROW = 100
  SQLITE_DONE = 101

# Nim wrapper
type
  Database = object
    handle: ptr Sqlite3

proc openDb(path: string): Database =
  var db: ptr Sqlite3
  let rc = sqlite3_open(path, addr db)
  if rc != SQLITE_OK:
    raise newException(IOError, "Cannot open DB: " & $sqlite3_errmsg(db))
  Database(handle: db)

proc close(db: var Database) =
  discard sqlite3_close(db.handle)

proc exec(db: Database, sql: string) =
  var errmsg: cstring
  let rc = sqlite3_exec(db.handle, sql, nil, nil, addr errmsg)
  if rc != SQLITE_OK:
    raise newException(Exception, "SQL error: " & $errmsg)

proc query(db: Database, sql: string): seq[seq[string]] =
  result = @[]
  var stmt: ptr Sqlite3Stmt
  
  let rc = sqlite3_prepare_v2(db.handle, sql, -1, addr stmt, nil)
  if rc != SQLITE_OK:
    raise newException(Exception, "SQL error")
  
  defer: discard sqlite3_finalize(stmt)
  
  while sqlite3_step(stmt) == SQLITE_ROW:
    var row: seq[string] = @[]
    var col = 0
    while true:
      let text = sqlite3_column_text(stmt, col.cint)
      if text == nil: break
      row.add($text)
      inc col
    result.add(row)

# Usage
var db = openDb(":memory:")
defer: db.close()

db.exec("""
  CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    name TEXT,
    age INTEGER
  )
""")

db.exec("INSERT INTO users VALUES (1, 'Alice', 30)")
db.exec("INSERT INTO users VALUES (2, 'Bob', 25)")
db.exec("INSERT INTO users VALUES (3, 'Carol', 35)")

let rows = db.query("SELECT * FROM users WHERE age > 26")
for row in rows:
  echo row.join(", ")
```

## สรุป Part 21

ในบทนี้เราได้เรียนรู้:
- ✅ Import C functions ด้วย `importc`
- ✅ C/Nim type mapping
- ✅ Inline C code ด้วย `emit`
- ✅ สร้าง C library wrapper
- ✅ Windows API
- ✅ Practical: SQLite wrapper

---

**Previous**: [Part 20 - Threads](part20_threads.md)
**Next**: [Part 22 - Memory Management](part22_memory.md)
