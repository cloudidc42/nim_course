# Part 04: Operators - ตัวดำเนินการ

## Arithmetic Operators (ตัวดำเนินการคำนวณ)

```nim
# Arithmetic operators พื้นฐาน
let a = 10
let b = 3

echo a + b    # 13 - บวก
echo a - b    # 7  - ลบ
echo a * b    # 30 - คูณ
echo a / b    # 3.3333... - หาร (ได้ float)
echo a div b  # 3  - integer division (ได้ int)
echo a mod b  # 1  - modulo
echo a ^ b    # 1000 - ยกกำลัง (แต่ใช้ pow() ดีกว่า)

# Float arithmetic
let x = 10.0
let y = 3.0
echo x + y    # 13.0
echo x - y    # 7.0
echo x * y    # 30.0
echo x / y    # 3.3333...
# echo x div y  # Error! div ใช้กับ int เท่านั้น
echo x mod y  # 1.0 - float modulo

import math
echo pow(2.0, 10)   # 1024.0 - power function
echo sqrt(16.0)     # 4.0
echo abs(-5)        # 5
echo abs(-3.14)     # 3.14
```

## Comparison Operators (ตัวดำเนินการเปรียบเทียบ)

```nim
let x = 5
let y = 10

echo x == y   # false - เท่ากัน
echo x != y   # true  - ไม่เท่ากัน
echo x < y    # true  - น้อยกว่า
echo x > y    # false - มากกว่า
echo x <= y   # true  - น้อยกว่าหรือเท่ากัน
echo x >= y   # false - มากกว่าหรือเท่ากัน

# เปรียบเทียบ string
let s1 = "apple"
let s2 = "banana"
echo s1 == s2   # false
echo s1 < s2    # true (lexicographic)
echo s1 > s2    # false

# เปรียบเทียบ char
echo 'a' < 'b'  # true
echo 'Z' < 'a'  # true (uppercase < lowercase in ASCII)

# Chained comparisons (ต้องใช้ and)
let n = 5
echo n > 0 and n < 10   # true - แบบ manual
# Nim ไม่รองรับ 0 < n < 10 โดยตรง
```

## Logical Operators (ตัวดำเนินการตรรกะ)

```nim
# Logical operators
echo true and true   # true
echo true and false  # false
echo false and true  # false
echo false and false # false

echo true or true    # true
echo true or false   # true
echo false or true   # true
echo false or false  # false

echo not true        # false
echo not false       # true

echo true xor true   # false (exclusive or)
echo true xor false  # true

# Short-circuit evaluation
var x = 0
echo (x != 0) and (100 div x > 5)  # false - ไม่คำนวณ 100 div x
echo (x == 0) or (100 div x > 5)   # true  - ไม่คำนวณ 100 div x

# Truth table
import strformat
echo "\n=== Truth Table ==="
for a in [false, true]:
  for b in [false, true]:
    echo &"  {a} AND {b} = {a and b}"
    echo &"  {a} OR  {b} = {a or b}"
    echo &"  {a} XOR {b} = {a xor b}"
    echo ""
```

## Bitwise Operators (ตัวดำเนินการ Bitwise)

```nim
# Bitwise operators
let a: int = 0b1010  # 10
let b: int = 0b1100  # 12

echo a and b    # 0b1000 = 8  - bitwise AND
echo a or b     # 0b1110 = 14 - bitwise OR
echo a xor b    # 0b0110 = 6  - bitwise XOR
echo not a      # bitwise NOT (complement)
echo a shl 2    # 0b101000 = 40 - shift left
echo a shr 1    # 0b0101 = 5   - shift right

# Practical examples
let flags = 0b0000_0000
let FLAG_A = 0b0000_0001  # bit 0
let FLAG_B = 0b0000_0010  # bit 1
let FLAG_C = 0b0000_0100  # bit 2

# Set flag
var myFlags = flags
myFlags = myFlags or FLAG_A   # set bit 0
myFlags = myFlags or FLAG_C   # set bit 2
echo myFlags.toBin(8)         # 00000101

# Check flag
echo bool(myFlags and FLAG_A)  # true - FLAG_A set
echo bool(myFlags and FLAG_B)  # false - FLAG_B not set

# Clear flag
myFlags = myFlags and not FLAG_A  # clear bit 0
echo myFlags.toBin(8)             # 00000100

# Toggle flag
myFlags = myFlags xor FLAG_C  # toggle bit 2
echo myFlags.toBin(8)          # 00000000

# Bit manipulation utils
proc setBit(value: int, bit: int): int =
  value or (1 shl bit)

proc clearBit(value: int, bit: int): int =
  value and not (1 shl bit)

proc toggleBit(value: int, bit: int): int =
  value xor (1 shl bit)

proc testBit(value: int, bit: int): bool =
  bool(value and (1 shl bit))

var v = 0
v = v.setBit(3)
v = v.setBit(5)
echo v.toBin(8)      # 00101000
echo v.testBit(3)    # true
echo v.testBit(4)    # false
v = v.clearBit(3)
echo v.toBin(8)      # 00100000
```

## Assignment Operators

```nim
var x = 10

# Compound assignment
x += 5     # x = x + 5 = 15
x -= 3     # x = x - 3 = 12
x *= 2     # x = x * 2 = 24
x /= 4     # x = x / 4 = 6.0 (Float!)
x = 10     # reset

var y = 10
y = y div 3  # integer division (ไม่มี /=)
y = 10
y mod= 3     # y = y mod 3 = 1
y = 10

# Bitwise compound assignment
y = 0b1111
y = y and 0b1010    # 0b1010 = 10 (ไม่มี and=)
y = 0b1111
# ใน Nim ไม่มี and=, or=, xor= โดยตรง
# ต้องเขียน explicit

# String concatenation
var s = "Hello"
s &= ", World!"  # s = "Hello, World!"
echo s

s.add(" Nim!")   # เพิ่มใน place
echo s

# Sequence append
var nums = @[1, 2, 3]
nums &= @[4, 5, 6]  # append sequence
nums.add(7)          # append single
echo nums
```

## String Operators

```nim
# String concatenation
let s1 = "Hello"
let s2 = "World"
let s3 = s1 & ", " & s2 & "!"
echo s3  # Hello, World!

# String comparison
echo "apple" == "apple"   # true
echo "apple" < "banana"   # true (lexicographic)
echo "Banana" < "apple"   # true (uppercase < lowercase)

# String indexing
let str = "Hello"
echo str[0]      # H
echo str[^1]     # o (last element)
echo str[1..3]   # ell
echo str[1..<3]  # el (exclusive end)
echo str[..2]    # Hel
echo str[2..]    # llo

# String multiplication (repeat)
import strutils
echo "abc".repeat(3)   # abcabcabc
echo "-".repeat(20)    # --------------------

# String in/contains
echo "World" in "Hello, World!"  # true
echo "hello" in "Hello, World!"  # false (case sensitive)
```

## Comparison Operators สำหรับ Sequences

```nim
# Sequence comparison
let a = @[1, 2, 3]
let b = @[1, 2, 3]
let c = @[1, 2, 4]

echo a == b   # true  - same elements
echo a == c   # false
echo a < c    # true  - lexicographic
echo a <= b   # true

# in operator
echo 2 in a     # true
echo 5 in a     # false
echo 5 notin a  # true

# Array
let arr = [1, 2, 3, 4, 5]
echo 3 in arr     # true
echo 6 notin arr  # true
```

## Special Operators

```nim
# is operator - type checking
let x = 42
echo x is int      # true
echo x is float    # false
echo x is string   # false

# isnot operator
echo x isnot float  # true

# of operator - object type checking
type
  Animal = ref object of RootObj
    name: string
  Dog = ref object of Animal
    breed: string
  Cat = ref object of Animal
    indoor: bool

let d: Animal = Dog(name: "Rex", breed: "Lab")

echo d of Dog  # true
echo d of Cat  # false
echo d of Animal  # true

# addr operator - get address
var n = 42
let p = addr n  # pointer to n
echo p[]  # dereference: 42

# [] operator - dereference
p[] = 100
echo n  # 100

# .. operator - range
for i in 1..10:
  echo i

# ..< operator - exclusive range
for i in 0..<5:
  echo i  # 0, 1, 2, 3, 4
```

## Operator Precedence (ลำดับการประเมินผล)

```nim
# ลำดับสูงสุดไปต่ำสุด:
# 1. $ (string conversion)
# 2. *, /, div, mod, shl, shr, and
# 3. +, -, or, xor, &
# 4. ==, <=, <, >=, >, !=, in, notin, is, isnot, of
# 5. not
# 6. and
# 7. or, xor
# 8. .. (range)
# 9. if (ternary)
# 10. :=

# ตัวอย่าง
echo 2 + 3 * 4    # 14 (ไม่ใช่ 20)
echo (2 + 3) * 4  # 20

echo 10 - 3 + 2   # 9  (left to right)
echo 2 ^ 3 ^ 2    # 512 (right to left? NO - ^ is left to right in Nim)

# ใช้วงเล็บเสมอเมื่อไม่แน่ใจ
echo (2 + 3) * (4 - 1)  # 15
echo not true or false   # false (not true = false, then false or false)
echo not (true or false) # false
```

## Custom Operators

```nim
# Nim อนุญาตให้สร้าง operator เอง

# Operator สำหรับ Vector
type Vec2 = object
  x, y: float

proc `+`(a, b: Vec2): Vec2 =
  Vec2(x: a.x + b.x, y: a.y + b.y)

proc `-`(a, b: Vec2): Vec2 =
  Vec2(x: a.x - b.x, y: a.y - b.y)

proc `*`(v: Vec2, scalar: float): Vec2 =
  Vec2(x: v.x * scalar, y: v.y * scalar)

proc `==`(a, b: Vec2): bool =
  a.x == b.x and a.y == b.y

proc `$`(v: Vec2): string =
  "Vec2(" & $v.x & ", " & $v.y & ")"

let v1 = Vec2(x: 1.0, y: 2.0)
let v2 = Vec2(x: 3.0, y: 4.0)

echo v1 + v2       # Vec2(4.0, 6.0)
echo v1 - v2       # Vec2(-2.0, -2.0)
echo v1 * 2.0      # Vec2(2.0, 4.0)
echo v1 == v1      # true
echo v1 == v2      # false

# Custom operators ที่ไม่มีใน Nim มาตรฐาน
proc `**`(base, exp: int): int =
  var result = 1
  for _ in 0..<exp:
    result *= base
  result

echo 2 ** 10  # 1024
echo 3 ** 3   # 27

# Operator สำหรับ string
proc `*`(s: string, n: int): string =
  var result = ""
  for _ in 0..<n:
    result &= s
  result

echo "abc" * 3   # abcabcabc
echo "-" * 20    # --------------------
```

## Ternary Expression (if expression)

```nim
# Nim มี if expression แทน ternary operator
let x = 10
let sign = if x > 0: "positive" elif x < 0: "negative" else: "zero"
echo sign  # positive

# ใช้ใน assignment
let abs_x = if x >= 0: x else: -x
echo abs_x  # 10

# ใช้ใน function call
import math
proc clamp(val, minVal, maxVal: float): float =
  if val < minVal: minVal
  elif val > maxVal: maxVal
  else: val

echo clamp(5.0, 0.0, 10.0)   # 5.0
echo clamp(-5.0, 0.0, 10.0)  # 0.0
echo clamp(15.0, 0.0, 10.0)  # 10.0
```

## Operator Overloading ขั้นสูง

```nim
# Matrix operations
type Matrix2x2 = array[2, array[2, float]]

proc `+`(a, b: Matrix2x2): Matrix2x2 =
  [[a[0][0] + b[0][0], a[0][1] + b[0][1]],
   [a[1][0] + b[1][0], a[1][1] + b[1][1]]]

proc `*`(a, b: Matrix2x2): Matrix2x2 =
  [[a[0][0]*b[0][0] + a[0][1]*b[1][0],
    a[0][0]*b[0][1] + a[0][1]*b[1][1]],
   [a[1][0]*b[0][0] + a[1][1]*b[1][0],
    a[1][0]*b[0][1] + a[1][1]*b[1][1]]]

proc `$`(m: Matrix2x2): string =
  import strformat
  &"[{m[0][0]:.1f}, {m[0][1]:.1f}]\n[{m[1][0]:.1f}, {m[1][1]:.1f}]"

let m1: Matrix2x2 = [[1.0, 2.0], [3.0, 4.0]]
let m2: Matrix2x2 = [[5.0, 6.0], [7.0, 8.0]]

echo "M1:"
echo m1
echo "\nM2:"
echo m2
echo "\nM1 + M2:"
echo m1 + m2
echo "\nM1 * M2:"
echo m1 * m2
```

## Practical: Calculator with All Operators

```nim
# advanced_calculator.nim

import strformat, math, strutils

type
  Token = object
    kind: string  # "num", "op"
    value: string

proc calculate(expr: string): float =
  # Simple single-operation calculator
  var parts = expr.strip().splitWhitespace()
  
  if parts.len != 3:
    raise newException(ValueError, "Invalid expression: " & expr)
  
  let a = parseFloat(parts[0])
  let op = parts[1]
  let b = parseFloat(parts[2])
  
  case op
  of "+":  a + b
  of "-":  a - b
  of "*":  a * b
  of "/":
    if b == 0: raise newException(DivByZeroDefect, "Division by zero!")
    a / b
  of "%":  a mod b
  of "**": pow(a, b)
  of "//": float(int(a) div int(b))  # integer division
  else:
    raise newException(ValueError, "Unknown operator: " & op)

# Test cases
let expressions = [
  "10 + 5",
  "10 - 3",
  "4 * 7",
  "15 / 4",
  "15 // 4",
  "15 % 4",
  "2 ** 8",
]

echo "=== Calculator Results ==="
for expr in expressions:
  try:
    let result = calculate(expr)
    echo &"  {expr} = {result}"
  except:
    echo &"  {expr} -> Error: {getCurrentExceptionMsg()}"
```

## แบบฝึกหัด Part 4

### แบบฝึกหัดที่ 1: Bitwise Operations
```nim
# Implement a simple RGB color using bit manipulation
type Color = uint32

proc makeColor(r, g, b: uint8): Color =
  (uint32(r) shl 16) or (uint32(g) shl 8) or uint32(b)

proc getRed(c: Color): uint8   = uint8(c shr 16)
proc getGreen(c: Color): uint8 = uint8((c shr 8) and 0xFF)
proc getBlue(c: Color): uint8  = uint8(c and 0xFF)

# เฉลย
let red = makeColor(255, 0, 0)
let green = makeColor(0, 255, 0)
let blue = makeColor(0, 0, 255)
let white = makeColor(255, 255, 255)

import strformat
echo &"Red:   #{red:06X} -> R={getRed(red)}, G={getGreen(red)}, B={getBlue(red)}"
echo &"Green: #{green:06X} -> R={getRed(green)}, G={getGreen(green)}, B={getBlue(green)}"
echo &"Blue:  #{blue:06X} -> R={getRed(blue)}, G={getGreen(blue)}, B={getBlue(blue)}"
echo &"White: #{white:06X} -> R={getRed(white)}, G={getGreen(white)}, B={getBlue(white)}"
```

### แบบฝึกหัดที่ 2: Custom Vector Operations
```nim
type Vec3 = object
  x, y, z: float

proc `+`(a, b: Vec3): Vec3 = Vec3(x: a.x+b.x, y: a.y+b.y, z: a.z+b.z)
proc `*`(v: Vec3, s: float): Vec3 = Vec3(x: v.x*s, y: v.y*s, z: v.z*s)
proc dot(a, b: Vec3): float = a.x*b.x + a.y*b.y + a.z*b.z

import math
proc length(v: Vec3): float = sqrt(v.x*v.x + v.y*v.y + v.z*v.z)
proc normalize(v: Vec3): Vec3 =
  let l = v.length()
  Vec3(x: v.x/l, y: v.y/l, z: v.z/l)

proc `$`(v: Vec3): string =
  import strformat
  &"Vec3({v.x:.2f}, {v.y:.2f}, {v.z:.2f})"

# เฉลย
let v1 = Vec3(x: 1.0, y: 2.0, z: 3.0)
let v2 = Vec3(x: 4.0, y: 5.0, z: 6.0)

echo "v1 = ", v1
echo "v2 = ", v2
echo "v1 + v2 = ", v1 + v2
echo "v1 * 2 = ", v1 * 2.0
echo "v1 · v2 = ", dot(v1, v2)
echo "|v1| = ", v1.length()
echo "normalize(v1) = ", v1.normalize()
```

## สรุป Part 4

ในบทนี้เราได้เรียนรู้:
- ✅ Arithmetic operators (+, -, *, /, div, mod, ^)
- ✅ Comparison operators (==, !=, <, >, <=, >=)
- ✅ Logical operators (and, or, not, xor)
- ✅ Bitwise operators (and, or, xor, not, shl, shr)
- ✅ Assignment operators (=, +=, -=, *=, &=)
- ✅ Special operators (is, isnot, in, notin, of)
- ✅ Operator overloading
- ✅ Operator precedence

---

**Previous**: [Part 03 - Variables และ Types](part03_variables_types.md)
**Next**: [Part 05 - Control Flow](part05_control_flow.md)
