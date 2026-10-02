# Part 07: Functions และ Procedures - ฟังก์ชัน

## proc vs func vs method vs template

```nim
# proc - general-purpose procedure (can have side effects)
proc greet(name: string): string =
  "Hello, " & name & "!"

# func - pure function (no side effects guaranteed)
func add(a, b: int): int =
  a + b

# method - OOP method (dynamic dispatch)
type Animal = ref object of RootObj
  name: string

method speak(a: Animal): string {.base.} =
  "..."

# template - compile-time substitution (no overhead)
template square(x: untyped): untyped =
  x * x

echo greet("World")   # Hello, World!
echo add(3, 4)        # 7
echo square(5)        # 25
```

## การประกาศ proc

```nim
# Basic syntax
proc name(params): returnType =
  body

# Examples
proc hello() =
  echo "Hello!"

proc add(a, b: int): int =
  a + b

proc greet(name: string, times: int = 1): string =
  var result = ""
  for _ in 0..<times:
    result &= "Hello, " & name & "!\n"
  result.strip()

# Multiple return types (ใช้ tuple)
proc divmod(a, b: int): (int, int) =
  (a div b, a mod b)

let (q, r) = divmod(17, 5)
echo "17 div 5 = ", q, " remainder ", r
```

## Parameters

```nim
# Value parameters (ค่าถูก copy)
proc double(x: int): int =
  var y = x  # ต้อง copy ก่อน ถ้าจะแก้
  y *= 2
  y

# var parameters (mutable reference)
proc increment(x: var int) =
  x += 1

var n = 5
increment(n)
echo n  # 6

# In the case of ref types:
proc addItem(s: var seq[int], item: int) =
  s.add(item)

var nums = @[1, 2, 3]
addItem(nums, 4)
echo nums  # @[1, 2, 3, 4]

# sink parameters (ownership transfer)
proc takesOwnership(s: sink string): string =
  s & " processed"

var myStr = "hello"
let result = takesOwnership(myStr)
# myStr อาจจะ moved แล้ว

# openArray - รับ array หรือ seq
proc sum(arr: openArray[int]): int =
  result = 0
  for x in arr:
    result += x

echo sum([1, 2, 3, 4, 5])          # array
echo sum(@[1, 2, 3, 4, 5])         # sequence
echo sum([1, 2, 3])                 # static array

# varargs - variable arguments
proc myPrint(args: varargs[string]) =
  for i, arg in args:
    if i > 0: stdout.write(", ")
    stdout.write(arg)
  stdout.write("\n")

myPrint("hello", "world", "nim")  # hello, world, nim

# varargs กับ conversion
proc sum2(args: varargs[int, `$`]): string =
  # จะแปลงทุก argument เป็น string ก่อน
  args.join(", ")

echo sum2(1, 2, 3)  # 1, 2, 3
```

## Default Parameters

```nim
proc createProfile(
  name: string,
  age: int = 0,
  email: string = "",
  active: bool = true
): string =
  import strformat
  &"Profile: {name}, age={age}, email={email}, active={active}"

echo createProfile("Alice")                       # Default all
echo createProfile("Bob", age = 25)               # Named args
echo createProfile("Carol", 28, "carol@mail.com") # Positional
echo createProfile("Dave", active = false)        # Named skipping
```

## Named Arguments

```nim
proc setup(width: int, height: int, fps: int = 60, fullscreen: bool = false) =
  echo "Setup: ", width, "x", height, "@", fps, "fps fullscreen=", fullscreen

# Positional
setup(1920, 1080)

# Named (ลำดับไม่สำคัญ)
setup(fps = 30, height = 1080, width = 1920)

# Mixed (positional ต้องมาก่อน named)
setup(1920, 1080, fullscreen = true)
```

## Return Values

```nim
# Explicit return
proc abs1(x: int): int =
  if x < 0:
    return -x
  return x

# Implicit return (last expression)
proc abs2(x: int): int =
  if x < 0: -x
  else: x

# result variable
proc abs3(x: int): int =
  result = x
  if result < 0:
    result = -result

# Multiple returns with tuple
proc minMax(nums: seq[int]): (int, int) =
  var min_v = nums[0]
  var max_v = nums[0]
  for n in nums:
    if n < min_v: min_v = n
    if n > max_v: max_v = n
  (min_v, max_v)

let (mn, mx) = minMax(@[3, 1, 4, 1, 5, 9, 2, 6])
echo "Min: ", mn, " Max: ", mx

# Named tuple return
proc stats(nums: seq[float]): tuple[min, max, mean: float] =
  var sum = 0.0
  result.min = nums[0]
  result.max = nums[0]
  for n in nums:
    if n < result.min: result.min = n
    if n > result.max: result.max = n
    sum += n
  result.mean = sum / nums.len.float

let data = @[3.0, 1.0, 4.0, 1.0, 5.0, 9.0, 2.0, 6.0]
let s = stats(data)
import strformat
echo &"Min={s.min}, Max={s.max}, Mean={s.mean:.2f}"
```

## Higher-Order Functions

```nim
# Function เป็น parameter
proc applyToAll(nums: seq[int], f: proc(x: int): int): seq[int] =
  result = newSeq[int](nums.len)
  for i, n in nums:
    result[i] = f(n)

let nums = @[1, 2, 3, 4, 5]
echo applyToAll(nums, proc(x: int): int = x * 2)  # @[2, 4, 6, 8, 10]
echo applyToAll(nums, proc(x: int): int = x * x)  # @[1, 4, 9, 16, 25]

# Arrow syntax (sugar)
import sugar
echo applyToAll(nums, x => x * 3)     # @[3, 6, 9, 12, 15]
echo applyToAll(nums, x => x * x + 1) # @[2, 5, 10, 17, 26]

# Returning functions
proc makeMultiplier(factor: int): proc(x: int): int =
  proc multiply(x: int): int = x * factor
  multiply

let triple = makeMultiplier(3)
let quintuple = makeMultiplier(5)

echo triple(7)      # 21
echo quintuple(4)   # 20

# Closure
proc makeCounter(start: int = 0): proc(): int =
  var count = start
  proc increment(): int =
    inc count
    count
  increment

let counter1 = makeCounter()
let counter2 = makeCounter(100)

echo counter1()  # 1
echo counter1()  # 2
echo counter1()  # 3
echo counter2()  # 101
echo counter2()  # 102
echo counter1()  # 4 (independent from counter2)
```

## Closures

```nim
# Closures capture variables from outer scope

proc makeAdder(x: int): proc(y: int): int =
  proc adder(y: int): int =
    x + y  # x ถูก capture จาก outer scope
  adder

let add5 = makeAdder(5)
let add10 = makeAdder(10)

echo add5(3)   # 8
echo add10(3)  # 13

# Mutable closure
proc makeAccumulator(): proc(x: int): int =
  var total = 0
  proc add(x: int): int =
    total += x
    total
  add

let acc = makeAccumulator()
echo acc(10)   # 10
echo acc(5)    # 15
echo acc(3)    # 18

# Closure สำหรับ event handlers
type EventHandler = proc(event: string)

proc createLogger(prefix: string): EventHandler =
  proc log(event: string) =
    echo prefix, ": ", event
  log

let errorLogger = createLogger("[ERROR]")
let infoLogger  = createLogger("[INFO]")

errorLogger("Something went wrong")  # [ERROR]: Something went wrong
infoLogger("User logged in")         # [INFO]: User logged in
```

## Overloading

```nim
# Nim รองรับ function overloading

proc format(x: int): string = "int: " & $x
proc format(x: float): string = "float: " & $x
proc format(x: string): string = "string: " & x
proc format(x: bool): string = "bool: " & $x

echo format(42)          # int: 42
echo format(3.14)        # float: 3.14
echo format("hello")     # string: hello
echo format(true)        # bool: true

# Overloading กับ different number of args
proc log(msg: string) = echo "[LOG] ", msg
proc log(level: string, msg: string) = echo "[", level, "] ", msg
proc log(level: string, msg: string, time: int) = 
  echo "[", level, "@", time, "] ", msg

log("Hello")                    # [LOG] Hello
log("ERROR", "File not found")  # [ERROR] File not found
log("INFO", "Started", 1234)    # [INFO@1234] Started
```

## Recursive Functions

```nim
# Basic recursion
proc factorial(n: int): int =
  if n <= 1: 1
  else: n * factorial(n - 1)

echo factorial(10)  # 3628800

# Fibonacci (naive - O(2^n))
proc fib(n: int): int =
  if n <= 1: n
  else: fib(n-1) + fib(n-2)

echo fib(10)  # 55

# Fibonacci (memoized - O(n))
import tables
var memo = initTable[int, int]()
proc fibMemo(n: int): int =
  if n in memo: return memo[n]
  if n <= 1: return n
  result = fibMemo(n-1) + fibMemo(n-2)
  memo[n] = result

echo fibMemo(40)  # 102334155

# Tail recursion
proc factorialTail(n: int, acc: int = 1): int =
  if n <= 1: acc
  else: factorialTail(n - 1, n * acc)

echo factorialTail(10)  # 3628800

# Mutual recursion
proc isEven(n: int): bool
proc isOdd(n: int): bool

proc isEven(n: int): bool =
  if n == 0: true
  else: isOdd(n - 1)

proc isOdd(n: int): bool =
  if n == 0: false
  else: isEven(n - 1)

echo isEven(10)  # true
echo isOdd(7)   # true

# Tree traversal
type
  Tree = ref object
    value: int
    left, right: Tree

proc sum(t: Tree): int =
  if t == nil: 0
  else: t.value + sum(t.left) + sum(t.right)

proc height(t: Tree): int =
  if t == nil: 0
  else: 1 + max(height(t.left), height(t.right))

let tree = Tree(
  value: 1,
  left: Tree(
    value: 2,
    left: Tree(value: 4),
    right: Tree(value: 5)
  ),
  right: Tree(
    value: 3,
    right: Tree(value: 6)
  )
)

echo "Sum: ", sum(tree)       # 21
echo "Height: ", height(tree) # 3
```

## Pragmas สำหรับ Functions

```nim
# {.noReturn.} - function ที่ไม่ return
proc panic(msg: string) {.noReturn.} =
  echo "FATAL: ", msg
  quit(1)

# {.inline.} - hint ให้ compiler inline function
proc square(x: int): int {.inline.} =
  x * x

# {.noinline.} - ห้าม inline
proc complexCalc(x: int): int {.noinline.} =
  x * x + x + 1

# {.discardable.} - return value ทิ้งได้
proc doSomething(): int {.discardable.} =
  42

doSomething()  # ไม่ต้อง discard

# {.raises.} - declare what exceptions can be raised
proc riskyOp() {.raises: [ValueError, IOError].} =
  discard

# {.deprecated.} - แจ้ง warning
proc oldFunction() {.deprecated: "Use newFunction() instead".} =
  discard

# {.exportc.} - export เป็น C function
proc myFunction(x: int): int {.exportc.} =
  x * 2
```

## Method Call Syntax (UFCS)

```nim
# Uniform Function Call Syntax
# f(x) เท่ากับ x.f()

let s = "hello world"

# Traditional
echo toUpper(s)
echo replace(s, "world", "nim")

# Method call (UFCS)
echo s.toUpper()
echo s.replace("world", "nim")

# Chaining
echo s.toUpper().replace("WORLD", "NIM").strip()

# Custom proc ก็ใช้ได้
proc addPrefix(s: string, prefix: string): string =
  prefix & s

echo addPrefix(s, "PREFIX_")      # Traditional
echo s.addPrefix("PREFIX_")       # UFCS
echo s.addPrefix("A_").addPrefix("B_")  # Chained
```

## Generic Functions

```nim
# Generic function ที่ทำงานกับ type ใดก็ได้
proc swap[T](a, b: var T) =
  let temp = a
  a = b
  b = temp

var x = 10
var y = 20
swap(x, y)
echo x, " ", y  # 20 10

var s1 = "hello"
var s2 = "world"
swap(s1, s2)
echo s1, " ", s2  # world hello

# Generic function กับ constraints
proc max[T: Ordinal](a, b: T): T =
  if a > b: a else: b

echo max(10, 20)    # 20
echo max('a', 'z')  # z

# Generic กับ multiple types
proc pair[T, U](a: T, b: U): (T, U) =
  (a, b)

let p1 = pair(1, "hello")
let p2 = pair(3.14, true)
echo p1  # (1, hello)
echo p2  # (3.14, true)
```

## Practical: Functional Pipeline

```nim
# functional_pipeline.nim - Pipeline pattern

import sugar, sequtils, strutils, strformat

# Pipeline functions
proc pipeline[T](initial: T, funcs: varargs[proc(x: T): T]): T =
  result = initial
  for f in funcs:
    result = f(result)

# สำหรับ string processing
let process = pipeline(
  "  Hello, World! This is Nim.  ",
  proc(s: string): string = s.strip(),
  proc(s: string): string = s.toLower(),
  proc(s: string): string = s.replace(",", "").replace(".", "").replace("!", ""),
  proc(s: string): string = s.split().join("_")
)
echo process  # hello_world_this_is_nim

# สำหรับ number processing
import math

let calcResult = pipeline(
  5.0,
  x => x * x,      # 25
  x => sqrt(x),    # 5
  x => x + 1,      # 6
  x => x * 2       # 12
)
echo calcResult  # 12.0

# Chain of responsibility
type Validator = proc(input: string): (bool, string)

proc minLength(n: int): Validator =
  proc(input: string): (bool, string) =
    if input.len < n: (false, &"Must be at least {n} chars")
    else: (true, "")

proc maxLength(n: int): Validator =
  proc(input: string): (bool, string) =
    if input.len > n: (false, &"Must be at most {n} chars")
    else: (true, "")

proc noSpaces(): Validator =
  proc(input: string): (bool, string) =
    if ' ' in input: (false, "Must not contain spaces")
    else: (true, "")

proc validate(input: string, validators: varargs[Validator]): (bool, string) =
  for v in validators:
    let (ok, msg) = v(input)
    if not ok: return (false, msg)
  (true, "Valid")

echo validate("hi",
  minLength(3),    # fail
  maxLength(20),
  noSpaces()
)  # (false, Must be at least 3 chars)

echo validate("helloworld",
  minLength(3),
  maxLength(20),
  noSpaces()
)  # (true, Valid)
```

## แบบฝึกหัด Part 7

### แบบฝึกหัดที่ 1: Currying
```nim
# Implement currying
proc curry[A, B, C](f: proc(a: A, b: B): C): proc(a: A): proc(b: B): C =
  proc outer(a: A): proc(b: B): C =
    proc inner(b: B): C =
      f(a, b)
    inner
  outer

let add = proc(a, b: int): int = a + b
let curriedAdd = curry(add)
let add5 = curriedAdd(5)

echo add5(3)   # 8
echo add5(10)  # 15
echo add5(-2)  # 3
```

### แบบฝึกหัดที่ 2: Function Composition
```nim
proc compose[A, B, C](f: proc(b: B): C, g: proc(a: A): B): proc(a: A): C =
  proc h(a: A): C = f(g(a))
  h

import strutils, math
let doubleAndSqrt = compose(
  proc(x: float): float = sqrt(x),
  proc(x: int): float = float(x * 2)
)
echo doubleAndSqrt(8)   # sqrt(16) = 4.0
echo doubleAndSqrt(50)  # sqrt(100) = 10.0
```

## สรุป Part 7

ในบทนี้เราได้เรียนรู้:
- ✅ proc, func, method, template ความแตกต่าง
- ✅ Parameters: value, var, sink, openArray, varargs
- ✅ Default parameters และ named arguments
- ✅ Return values: explicit, implicit, result
- ✅ Higher-order functions
- ✅ Closures
- ✅ Function overloading
- ✅ Recursive functions
- ✅ Pragmas สำหรับ functions
- ✅ Generic functions
- ✅ UFCS (Method call syntax)

---

**Previous**: [Part 06 - Loops](part06_loops.md)
**Next**: [Part 08 - Arrays และ Sequences](part08_arrays_sequences.md)
