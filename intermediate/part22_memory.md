# Part 22: Memory Management - การจัดการหน่วยความจำ

## ARC/ORC - Automatic Reference Counting

```nim
# Nim ใช้ ARC (Automatic Reference Counting) หรือ ORC (cyclic GC)
# compile: nim c --gc:arc myfile.nim
# compile: nim c --gc:orc myfile.nim  (handles cycles)

# Ownership และ Moves
type Resource = object
  data: seq[int]
  name: string

proc newResource(name: string, size: int): Resource =
  Resource(name: name, data: newSeq[int](size))

proc `=destroy`(r: var Resource) =
  echo "Destroying: ", r.name
  # Custom cleanup here

proc `=copy`(dest: var Resource, src: Resource) =
  echo "Copying: ", src.name
  dest.data = src.data
  dest.name = src.name

proc `=dup`(src: Resource): Resource =
  echo "Duping: ", src.name
  Resource(data: src.data, name: src.name)

proc `=sink`(dest: var Resource, src: Resource) =
  echo "Moving: ", src.name
  `=destroy`(dest)
  wasMoved(dest)
  dest.data = src.data
  dest.name = src.name

var r1 = newResource("R1", 100)
var r2 = r1  # copy
let r3 = move(r1)  # move (r1 becomes empty)
```

## Manual Memory Management

```nim
import std/memory

# alloc/dealloc
let p = alloc(1024)  # allocate 1024 bytes
let ip = cast[ptr int](p)
ip[] = 42
echo ip[]  # 42
dealloc(p)

# alloc0 - zeroed memory
let p2 = alloc0(sizeof(int) * 10)
let arr = cast[ptr UncheckedArray[int]](p2)
for i in 0..<10:
  echo arr[i]  # all 0
dealloc(p2)

# resize
var buf = alloc(100)
buf = realloc(buf, 200)  # grow to 200
dealloc(buf)

# allocShared - thread-safe allocation
let shared = allocShared(sizeof(int))
let sharedInt = cast[ptr int](shared)
sharedInt[] = 100
# use in threads...
deallocShared(shared)

# create/destroy - typed allocation
let intPtr = create(int)  # allocate one int
intPtr[] = 42
echo intPtr[]
destroy(intPtr)

# createShared
let sharedArr = createShared(int, 10)  # array of 10 ints
sharedArr[0] = 1
sharedArr[9] = 99
destroyShared(sharedArr)
```

## Unsafe Pointers

```nim
# ptr - unsafe, untraced pointer
var x = 42
let p: ptr int = addr x
echo p[]     # 42
p[] = 100
echo x       # 100

# Pointer arithmetic (dangerous!)
var arr = [1, 2, 3, 4, 5]
let start = addr arr[0]
let second = cast[ptr int](cast[int](start) + sizeof(int))
echo second[]  # 2

# cast between pointer types
let bytePtr = cast[ptr uint8](addr arr[0])
echo bytePtr[]  # first byte of arr[0]

# UncheckedArray - array without bounds check
type BigArray = ptr UncheckedArray[int]

proc processArray(data: BigArray, len: int) =
  for i in 0..<len:
    echo data[i]

# ref vs ptr
type
  Node = ref object    # GC-traced, safer
    value: int
    next: Node
  
  CNode = ptr object   # untraced, manual management
    value: int
    next: ptr CNode
```

## Custom Allocators

```nim
# Pool Allocator
type
  PoolBlock = object
    data: array[64, byte]
    used: bool

  PoolAllocator = object
    blocks: seq[PoolBlock]
    capacity: int

proc newPool(capacity: int): PoolAllocator =
  PoolAllocator(
    blocks: newSeq[PoolBlock](capacity),
    capacity: capacity
  )

proc alloc(pool: var PoolAllocator): pointer =
  for i in 0..<pool.blocks.len:
    if not pool.blocks[i].used:
      pool.blocks[i].used = true
      return addr pool.blocks[i].data[0]
  raise newException(OutOfMemError, "Pool exhausted")

proc free(pool: var PoolAllocator, p: pointer) =
  for i in 0..<pool.blocks.len:
    if addr pool.blocks[i].data[0] == p:
      pool.blocks[i].used = false
      return
  raise newException(AccessViolationDefect, "Invalid pointer")

# Usage
var pool = newPool(10)
let p1 = pool.alloc()
let p2 = pool.alloc()
pool.free(p1)
pool.free(p2)
```

## Memory Safety Patterns

```nim
# RAII pattern with defer
proc withBuffer(size: int, body: proc(buf: pointer)) =
  let buf = alloc(size)
  defer: dealloc(buf)
  body(buf)

withBuffer(1024) do (buf: pointer):
  # buf is automatically freed after this block
  let arr = cast[ptr UncheckedArray[byte]](buf)
  arr[0] = 0xFF
  echo "Using buffer"

# Smart pointer pattern
type
  SmartPtr[T] = object
    p: ptr T
    refCount: ptr int

proc newSmartPtr[T](val: T): SmartPtr[T] =
  let p = create(T)
  let rc = create(int)
  p[] = val
  rc[] = 1
  SmartPtr[T](p: p, refCount: rc)

proc `=destroy`[T](sp: var SmartPtr[T]) =
  if sp.refCount != nil:
    dec sp.refCount[]
    if sp.refCount[] == 0:
      destroy(sp.p)
      destroy(sp.refCount)

proc `=copy`[T](dest: var SmartPtr[T], src: SmartPtr[T]) =
  dest.p = src.p
  dest.refCount = src.refCount
  if dest.refCount != nil:
    inc dest.refCount[]

proc get[T](sp: SmartPtr[T]): ptr T = sp.p
proc `[]`[T](sp: SmartPtr[T]): T = sp.p[]

# Zero-copy string operations
proc substr(s: string, start, finish: int): openArray[char] =
  s.toOpenArray(start, finish - 1)
```

## Memory Layout

```nim
import std/typetraits

# sizeof/alignof
echo sizeof(int)      # 8 (on 64-bit)
echo sizeof(int32)    # 4
echo sizeof(float64)  # 8
echo sizeof(char)     # 1
echo sizeof(bool)     # 1

type
  Compact = object  # packed layout
    a: int8    # 1 byte
    b: int32   # 4 bytes (3 bytes padding before)
    c: int8    # 1 byte (7 bytes padding after)
  
  Packed {.packed.} = object  # no padding
    a: int8
    b: int32
    c: int8

echo sizeof(Compact)  # 12 (with padding)
echo sizeof(Packed)   # 6 (no padding)

# offsetof
echo offsetof(Compact, a)  # 0
echo offsetof(Compact, b)  # 4 (after padding)
echo offsetOf(Compact, c)  # 8

# Bit manipulation
type
  Flags = uint32

proc setBit(flags: var Flags, bit: int) =
  flags = flags or (1.Flags shl bit)

proc clearBit(flags: var Flags, bit: int) =
  flags = flags and not (1.Flags shl bit)

proc testBit(flags: Flags, bit: int): bool =
  (flags and (1.Flags shl bit)) != 0

var f: Flags = 0
f.setBit(0)
f.setBit(3)
f.setBit(7)
echo f.toBin(8)  # 10001001
echo f.testBit(3)  # true
f.clearBit(3)
echo f.testBit(3)  # false
```

## Practical: Memory Arena

```nim
# Memory Arena - เร็วมากสำหรับ many small allocations
type
  Arena = object
    data: seq[byte]
    pos: int

proc newArena(size: int): Arena =
  Arena(data: newSeq[byte](size), pos: 0)

proc alloc(arena: var Arena, size: int, align: int = 8): pointer =
  # Align position
  let aligned = (arena.pos + align - 1) and not (align - 1)
  
  if aligned + size > arena.data.len:
    raise newException(OutOfMemError, "Arena exhausted")
  
  result = addr arena.data[aligned]
  arena.pos = aligned + size

proc reset(arena: var Arena) =
  arena.pos = 0  # Reset all at once (very fast!)

# Usage - parse JSON into arena for zero-allocation parsing
var arena = newArena(1024 * 1024)  # 1MB arena

type
  JsonValue = object
    case kind: int
    of 0: intVal: int64
    of 1: floatVal: float64
    of 2: strVal: cstring
    of 3: boolVal: bool
    else: discard

proc allocJsonInt(arena: var Arena, n: int64): ptr JsonValue =
  result = cast[ptr JsonValue](arena.alloc(sizeof(JsonValue)))
  result.kind = 0
  result.intVal = n

let jv1 = arena.allocJsonInt(42)
let jv2 = arena.allocJsonInt(100)
echo jv1.intVal  # 42
echo jv2.intVal  # 100

arena.reset()  # Free all in O(1)!
```

## สรุป Part 22

ในบทนี้เราได้เรียนรู้:
- ✅ ARC/ORC: =destroy, =copy, =sink hooks
- ✅ Manual memory: alloc/dealloc/create/destroy
- ✅ Unsafe pointers และ pointer arithmetic
- ✅ Custom allocators (pool)
- ✅ Memory safety patterns (RAII, SmartPtr)
- ✅ Memory layout: sizeof, alignof, packed
- ✅ Practical: Memory Arena

---

**Previous**: [Part 21 - FFI with C](part21_ffi.md)
**Next**: [Part 23 - Compile-time Programming](part23_compiletime.md)
