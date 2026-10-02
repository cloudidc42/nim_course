# Part 03: Variables และ Types - ตัวแปรและประเภทข้อมูล

## การประกาศตัวแปร

Nim มีคีย์เวิร์ดสามตัวสำหรับประกาศตัวแปร: `var`, `let`, `const`

```nim
# var - ตัวแปรที่เปลี่ยนแปลงได้ (mutable)
var x = 10
x = 20  # OK - เปลี่ยนได้

# let - ตัวแปรที่เปลี่ยนแปลงไม่ได้ (immutable) - รู้ค่าตอน runtime
let y = 30
# y = 40  # Error! cannot assign to 'y'

# const - ค่าคงที่ที่รู้ค่าตอน compile time
const PI = 3.14159265358979
const AppName = "MyApp"
const MaxSize = 1024
# PI = 3.0  # Error!
```

### ความแตกต่าง var/let/const

```nim
# var - เปลี่ยนได้ทั้ง reference และ value
var count = 0
count = 1    # OK
count += 1   # OK

# let - เปลี่ยนไม่ได้ แต่ object ที่ชี้ไปเปลี่ยนได้
let numbers = @[1, 2, 3]
# numbers = @[4, 5, 6]  # Error!
# แต่ถ้าเป็น object:
let obj = MyObject(x: 1)
# obj.x = 2  # Error ถ้า let

var obj2 = MyObject(x: 1)
obj2.x = 2   # OK เพราะเป็น var

# const - ต้องรู้ค่าตอน compile
const BUFFER_SIZE = 4096
const GREETING = "Hello"
const IS_DEBUG = defined(debug)  # compile-time condition
```

## Type System ใน Nim

### Integer Types

```nim
# Integer types ทั้งหมด
var i8:   int8   = 127           # -128 ถึง 127
var i16:  int16  = 32767         # -32,768 ถึง 32,767
var i32:  int32  = 2147483647    # -2^31 ถึง 2^31-1
var i64:  int64  = 9223372036854775807  # -2^63 ถึง 2^63-1
var i:    int    = 9999          # platform-dependent (32 หรือ 64 bit)

# Unsigned integers
var u8:   uint8  = 255           # 0 ถึง 255
var u16:  uint16 = 65535         # 0 ถึง 65,535
var u32:  uint32 = 4294967295'u32
var u64:  uint64 = 18446744073709551615'u64
var u:    uint   = 1000          # platform-dependent

# Hexadecimal, Octal, Binary literals
var hex  = 0xFF          # 255
var oct  = 0o377         # 255
var bin  = 0b11111111    # 255

# Underscores สำหรับอ่านง่าย
var million  = 1_000_000
var bigBin   = 0b1111_0000_1010_0101
var bigHex   = 0xDEAD_BEEF

# Type suffixes
var x1 = 42'i8    # int8
var x2 = 42'u    # uint
var x3 = 42'i64  # int64

echo i8, " ", i16, " ", i32
echo u8, " ", u16, " ", u32
echo hex, " ", oct, " ", bin
```

### Float Types

```nim
# Float types
var f32:  float32 = 3.14'f32
var f64:  float64 = 3.141592653589793
var f:    float   = 3.14      # = float64 บน 64-bit

# Float literals
var a = 1.0
var b = 1.0e10    # 10,000,000,000.0
var c = 1.5e-3    # 0.0015
var d = 0.5
var e = .5        # ได้เช่นกัน
var f2 = 1'f32   # float32 ด้วย suffix

# Special values
import math
echo Inf           # Infinity
echo NegInf        # -Infinity
echo NaN           # Not a Number
echo isNaN(NaN)    # true
echo isInf(Inf)    # true
echo isFinite(1.0) # true

# Arithmetic
echo 1.0 / 0.0   # Inf
echo -1.0 / 0.0  # -Inf
echo 0.0 / 0.0   # NaN
```

### Boolean Type

```nim
# Boolean
var b1: bool = true
var b2: bool = false
var b3 = true   # type inference

# Boolean operations
echo true and false   # false
echo true or false    # true
echo not true         # false
echo true xor true    # false
echo true xor false   # true

# Comparison ได้ bool
echo 5 > 3     # true
echo 5 == 3    # false
echo 5 != 3    # true
echo 5 >= 5    # true
echo 5 <= 4    # false

# Short-circuit evaluation
var x = 0
echo (x != 0) and (10 div x > 5)  # false - ไม่ตรวจ x div 0
```

### Character Type

```nim
# char - single character
var c1: char = 'A'
var c2 = 'B'
var newline = '\n'
var tab = '\t'
var backslash = '\\'
var quote = '\''
var null_char = '\0'

# Unicode character (ใช้ Rune จาก unicode module)
import unicode
var rune1: Rune = "A".runeAt(0)
var rune2 = "ก".runeAt(0)  # Thai character

# Character operations
echo ord('A')        # 65 - ASCII code
echo chr(65)         # A - ASCII to char
echo 'A' < 'B'       # true
echo 'A'.isUpperAscii  # true
echo 'a'.isLowerAscii  # true
echo '5'.isDigit     # true
echo ' '.isSpaceAscii  # true

# Convert
let ch = 'X'
echo ch.ord          # 88
echo ch.toLowerAscii # x
echo ch.toUpperAscii # X (already upper)
```

### String Type

```nim
# String - immutable sequence of bytes (UTF-8)
var s1: string = "Hello, World!"
var s2 = "สวัสดีชาวโลก"  # UTF-8 supported
var s3 = ""              # empty string

# String literals
var multi = """
This is a
multi-line string
with "quotes" and 'apostrophes'
"""

var raw = r"C:\Users\Name\file.txt"  # raw string, no escaping

# String operations
echo s1.len          # 13 (bytes, not chars)
echo s1[0]           # H (byte, char type)
echo s1[0..4]        # Hello (slice)
echo s1 & " Nim!"    # concatenation
echo s1.contains("World")   # true
echo s1.startsWith("Hello") # true
echo s1.endsWith("!")       # true
echo s1.toUpper()           # HELLO, WORLD!
echo s1.toLower()           # hello, world!
echo s1.replace("World", "Nim")  # Hello, Nim!
echo s1.strip()             # ตัด whitespace
echo "  hello  ".strip()    # "hello"

# String interpolation
import strformat
let name = "Alice"
let age = 30
echo &"Name: {name}, Age: {age}"
echo &"Pi = {3.14159:.2f}"
echo &"Hex: {255:#x}"  # 0xff
```

## Type Inference (การอนุมานประเภท)

```nim
# Nim สามารถอนุมาน type ได้อัตโนมัติ
let a = 42         # int
let b = 3.14       # float
let c = "hello"    # string
let d = true       # bool
let e = 'x'        # char
let f = @[1, 2, 3] # seq[int]
let g = (1, "hi")  # tuple[int, string]

# ตรวจสอบ type ด้วย typeof
echo typeof(a)  # int
echo typeof(b)  # float
echo typeof(c)  # string
echo typeof(f)  # seq[int]

# Type annotations (เมื่อ Nim ไม่สามารถอนุมานได้)
var x: int = 0
var y: float64 = 0
var z: seq[string] = @[]
```

## Ordinal Types

```nim
# Ordinal types: int, char, bool, enum
# สามารถใช้ ord() และ chr() ได้

echo ord(true)   # 1
echo ord(false)  # 0
echo ord('A')    # 65

# inc และ dec
var n = 5
inc n     # n = 6
dec n     # n = 5
inc n, 3  # n = 8
dec n, 2  # n = 6

# pred และ succ (predecessor, successor)
echo pred(5)   # 4
echo succ(5)   # 6
echo pred('B') # A
echo succ('A') # B
```

## Subrange Types

```nim
# กำหนด range ของ type
type
  Percentage = range[0..100]
  DiceValue  = range[1..6]
  AsciiChar  = range['\0'..'\127']

var p: Percentage = 75
var d: DiceValue = 3
# var p2: Percentage = 101  # Error at runtime!

# Useful สำหรับ parameter validation
proc setVolume(vol: range[0..100]) =
  echo "Volume set to: ", vol

setVolume(50)    # OK
# setVolume(150) # Error! out of range
```

## Nil และ Option Types

```nim
# nil - ใช้กับ reference types
type Node = ref object
  value: int
  next: Node

var n: Node = nil  # Node ที่ยังไม่มี value

# เช็ค nil
if n == nil:
  echo "Node is nil"
elif n != nil:
  echo "Node has value: ", n.value

# Option type - แทน nil ได้ดีกว่า
import std/options

proc findUser(id: int): Option[string] =
  if id == 1:
    some("Alice")
  else:
    none(string)

let user = findUser(1)
if user.isSome:
  echo "Found: ", user.get()
else:
  echo "User not found"

# Pattern matching กับ Option
let result = findUser(99)
case result.isSome
of true:
  echo "User: ", result.get()
of false:
  echo "Not found"
```

## Type Casting และ Conversion

```nim
# Explicit type conversion
var i = 42
var f = float(i)    # int -> float
var i2 = int(3.7)   # float -> int (truncate, = 3)
var s = $i          # int -> string
var b = bool(1)     # int -> bool

# parseInt, parseFloat
import strutils
var n = parseInt("42")       # string -> int
var f2 = parseFloat("3.14")  # string -> float

# tryParseInt (ปลอดภัยกว่า)
var ok: bool
var result: int
(ok, result) = ("123".parseInt, true)

# รูปแบบที่แนะนำ
try:
  let n2 = parseInt("not_a_number")
except ValueError as e:
  echo "Error: ", e.msg

# cast[] - unsafe type casting
var x: int32 = 0x41424344
var bytes = cast[array[4, char]](x)
echo bytes  # ABCD (ขึ้นอยู่กับ endianness)

# converter - implicit conversion
converter toFloat(x: int): float = float(x)
var result2: float = 5  # auto convert int -> float
```

## Type Aliases

```nim
# Type aliases - ชื่อเล่นของ type
type
  Age = int
  Name = string
  Score = float

var age: Age = 25
var name: Name = "Alice"
var score: Score = 95.5

# Distinct types - type ใหม่ที่แตกต่างจาก base type
type
  Meters = distinct float
  Seconds = distinct float

var distance: Meters = 100.0.Meters
var time: Seconds = 9.58.Seconds

# ไม่สามารถบวกกันได้โดยตรง
# var result = distance + time  # Error! type mismatch

# ต้องแปลงก่อน
var speed = distance.float / time.float  # OK
```

## Complex Types Preview

```nim
# Tuples
var point = (x: 10, y: 20)
var pair = (1, "hello")
echo point.x  # 10
echo pair[0]  # 1

# Objects
type Person = object
  name: string
  age: int

var p = Person(name: "Alice", age: 30)
echo p.name  # Alice

# Sequences (dynamic arrays)
var nums = @[1, 2, 3, 4, 5]
nums.add(6)
echo nums[0]  # 1
echo nums.len  # 6

# Arrays (fixed size)
var arr: array[5, int] = [1, 2, 3, 4, 5]
echo arr[0]  # 1
echo arr.len # 5
```

## ตัวอย่างโปรแกรม: Type System Demo

```nim
# types_demo.nim - แสดง type system ทั้งหมด

import strformat, math

# ===== Integer Operations =====
echo "=== Integer Operations ==="
let a: int = 100
let b: int = 7

echo &"{a} + {b} = {a + b}"
echo &"{a} - {b} = {a - b}"
echo &"{a} * {b} = {a * b}"
echo &"{a} div {b} = {a div b}"  # integer division
echo &"{a} mod {b} = {a mod b}"  # modulo
echo &"{a} ^ {b} = {a ^ b}"     # power

# ===== Float Operations =====
echo "\n=== Float Operations ==="
let pi = 3.14159265358979
let r = 5.0

let area = pi * r * r
let circumference = 2 * pi * r

echo &"Circle radius: {r}"
echo &"Area: {area:.4f}"
echo &"Circumference: {circumference:.4f}"
echo &"sqrt({r}) = {sqrt(r):.4f}"
echo &"sin(π/2) = {sin(PI/2):.4f}"
echo &"cos(0) = {cos(0.0):.4f}"

# ===== String Operations =====
echo "\n=== String Operations ==="
let greeting = "Hello, World!"
echo &"Original: {greeting}"
echo &"Upper: {greeting.toUpper()}"
echo &"Lower: {greeting.toLower()}"
echo &"Length: {greeting.len}"
echo &"Reversed: {greeting.reversed()}"

import strutils
let words = greeting.split(", ")
for word in words:
  echo &"  Word: '{word}'"

# ===== Boolean Logic =====
echo "\n=== Boolean Logic ==="
let t = true
let f = false
echo &"T AND F = {t and f}"
echo &"T OR F = {t or f}"
echo &"NOT T = {not t}"
echo &"T XOR F = {t xor f}"

# ===== Type Conversions =====
echo "\n=== Type Conversions ==="
let n = 42
echo &"int: {n}"
echo &"float: {float(n)}"
echo &"string: {$n}"
echo &"hex: {n:#x}"
echo &"binary: {n:#b}"
echo &"octal: {n:#o}"
```

## ตัวอย่าง: Unit System (ใช้ distinct types)

```nim
# units.nim - ระบบหน่วยที่ปลอดภัย

type
  Celsius    = distinct float
  Fahrenheit = distinct float
  Kelvin     = distinct float

# Converter procedures
proc toFahrenheit(c: Celsius): Fahrenheit =
  Fahrenheit(c.float * 9.0/5.0 + 32.0)

proc toCelsius(f: Fahrenheit): Celsius =
  Celsius((f.float - 32.0) * 5.0/9.0)

proc toKelvin(c: Celsius): Kelvin =
  Kelvin(c.float + 273.15)

proc toCelsiusFromK(k: Kelvin): Celsius =
  Celsius(k.float - 273.15)

# Pretty printing
proc `$`(c: Celsius): string    = $c.float & "°C"
proc `$`(f: Fahrenheit): string = $f.float & "°F"
proc `$`(k: Kelvin): string     = $k.float & "K"

# Test
let bodyTemp = 37.0.Celsius
echo "Body temperature:"
echo "  ", bodyTemp
echo "  ", bodyTemp.toFahrenheit()
echo "  ", bodyTemp.toKelvin()

let freezing = 0.0.Celsius
echo "\nFreezing point:"
echo "  ", freezing
echo "  ", freezing.toFahrenheit()
echo "  ", freezing.toKelvin()

# ไม่สามารถบวก Celsius กับ Fahrenheit
# let wrong = bodyTemp + freezing.toFahrenheit()  # Error!
```

## แบบฝึกหัด Part 3

### แบบฝึกหัดที่ 1: Temperature Converter
```nim
# สร้าง temperature converter ที่รับค่าจาก user

import strutils, strformat

proc celsiusToFahrenheit(c: float): float =
  c * 9.0/5.0 + 32.0

proc fahrenheitToCelsius(f: float): float =
  (f - 32.0) * 5.0/9.0

# เฉลย
let celsius = 100.0
echo &"{celsius}°C = {celsiusToFahrenheit(celsius):.1f}°F"
let fahrenheit = 98.6
echo &"{fahrenheit}°F = {fahrenheitToCelsius(fahrenheit):.1f}°C"
```

### แบบฝึกหัดที่ 2: Type Sizes
```nim
# แสดงขนาดของ types ทั้งหมด
import strformat

echo "Type sizes:"
echo &"  int8:    {sizeof(int8)} bytes = {sizeof(int8)*8} bits"
echo &"  int16:   {sizeof(int16)} bytes = {sizeof(int16)*8} bits"
echo &"  int32:   {sizeof(int32)} bytes = {sizeof(int32)*8} bits"
echo &"  int64:   {sizeof(int64)} bytes = {sizeof(int64)*8} bits"
echo &"  float32: {sizeof(float32)} bytes = {sizeof(float32)*8} bits"
echo &"  float64: {sizeof(float64)} bytes = {sizeof(float64)*8} bits"
echo &"  char:    {sizeof(char)} bytes = {sizeof(char)*8} bits"
echo &"  bool:    {sizeof(bool)} bytes = {sizeof(bool)*8} bits"
```

### แบบฝึกหัดที่ 3: String Builder
```nim
# สร้าง string จาก array ของ words
import strutils

let words = ["Nim", "is", "a", "great", "language"]
let sentence = words.join(" ")
echo sentence
echo sentence.toUpper()
echo sentence.len, " characters"
echo sentence.count(' ') + 1, " words"
```

## สรุป Part 3

ในบทนี้เราได้เรียนรู้:
- ✅ `var`, `let`, `const` และความแตกต่าง
- ✅ Integer types: int8, int16, int32, int64, uint
- ✅ Float types: float32, float64
- ✅ Boolean, Char, String
- ✅ Type inference
- ✅ Type casting และ conversion
- ✅ Type aliases และ distinct types
- ✅ Subrange types

---

**Previous**: [Part 02 - การติดตั้ง](part02_installation.md)
**Next**: [Part 04 - Operators](part04_operators.md)
