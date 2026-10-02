# Part 05: Control Flow - การควบคุมการทำงาน

## if / elif / else

```nim
# พื้นฐาน if/elif/else
let x = 15

if x > 0:
  echo "Positive"
elif x < 0:
  echo "Negative"
else:
  echo "Zero"

# if expression (ternary-like)
let sign = if x > 0: "+" elif x < 0: "-" else: "0"
echo sign  # +

# Nested if
let score = 85
if score >= 90:
  echo "A"
elif score >= 80:
  if score >= 85:
    echo "B+"
  else:
    echo "B"
elif score >= 70:
  echo "C"
else:
  echo "F"

# Multiple conditions
let age = 25
let hasID = true

if age >= 18 and hasID:
  echo "Can enter"
elif age >= 18 and not hasID:
  echo "Need ID"
else:
  echo "Too young"
```

## case Statement

```nim
# case เหมือน switch ใน C แต่ทรงพลังกว่ามาก

let day = 3

case day
of 1: echo "Monday"
of 2: echo "Tuesday"
of 3: echo "Wednesday"
of 4: echo "Thursday"
of 5: echo "Friday"
of 6, 7: echo "Weekend"
else: echo "Invalid day"

# case กับ range
let grade = 'B'
case grade
of 'A': echo "Excellent"
of 'B', 'C': echo "Good"
of 'D': echo "Passing"
of 'F': echo "Failing"
else: echo "Unknown grade"

# case กับ int range
let n = 42
case n
of 1..10:   echo "Small"
of 11..50:  echo "Medium"
of 51..100: echo "Large"
else:       echo "Very large"

# case expression
import strformat
let month = 4
let monthName = case month
  of 1:  "January"
  of 2:  "February"
  of 3:  "March"
  of 4:  "April"
  of 5:  "May"
  of 6:  "June"
  of 7:  "July"
  of 8:  "August"
  of 9:  "September"
  of 10: "October"
  of 11: "November"
  of 12: "December"
  else:  "Unknown"

echo &"Month {month} is {monthName}"
```

## case กับ Enums

```nim
type
  Direction = enum
    North, South, East, West

  Color = enum
    Red = "red"
    Green = "green"
    Blue = "blue"

let dir = North

case dir
of North: echo "Going North"
of South: echo "Going South"
of East:  echo "Going East"
of West:  echo "Going West"
# ไม่ต้อง else เพราะ enum ครบทุก case แล้ว

# case กับ string
let colorName = "blue"
let c = case colorName
  of "red":   Red
  of "green": Green
  of "blue":  Blue
  else: raise newException(ValueError, "Unknown color: " & colorName)
  
echo c  # Blue
```

## when Statement (Compile-time if)

```nim
# when ทำงานตอน compile time
when defined(windows):
  echo "Running on Windows"
elif defined(macosx):
  echo "Running on macOS"
elif defined(linux):
  echo "Running on Linux"
else:
  echo "Unknown OS"

# when กับ NimVersion
when NimVersion >= "2.0.0":
  echo "Nim 2.0+ features available"
else:
  echo "Legacy Nim"

# when สำหรับ CPU architecture
when sizeof(int) == 8:
  echo "64-bit system"
else:
  echo "32-bit system"

# when ใน procedures
proc debugPrint(msg: string) =
  when defined(debug):
    echo "[DEBUG] ", msg
  # else: ไม่ทำอะไร

# when สำหรับ platform-specific code
type
  OSHandle =
    when defined(windows): uint
    else: cint
```

## for Loops

```nim
# for loop พื้นฐาน
for i in 1..5:
  echo i  # 1, 2, 3, 4, 5

# exclusive range
for i in 0..<5:
  echo i  # 0, 1, 2, 3, 4

# countdown
for i in countdown(10, 1):
  echo i  # 10, 9, 8, ..., 1

# Step
for i in countup(0, 10, 2):
  echo i  # 0, 2, 4, 6, 8, 10

# for กับ sequence
let fruits = ["apple", "banana", "cherry"]
for fruit in fruits:
  echo fruit

# for กับ index
for i, fruit in fruits:
  echo i, ": ", fruit
  # 0: apple
  # 1: banana
  # 2: cherry

# for กับ string (แต่ละ character)
for ch in "Hello":
  echo ch

# for กับ string index
for i, ch in "Hello":
  echo i, ": ", ch

# for กับ pairs
import tables
let scores = {"Alice": 95, "Bob": 87, "Carol": 92}
for name, score in scores:
  echo name, ": ", score
```

## while Loops

```nim
# while loop พื้นฐาน
var n = 1
while n <= 5:
  echo n
  inc n

# while กับ condition
import random
randomize()

var count = 0
var total = 0
while total < 100:
  let roll = rand(1..6)
  total += roll
  inc count
  echo &"Roll {count}: {roll} (total: {total})"

echo &"Needed {count} rolls to reach 100"

# Infinite loop
var x = 0
while true:
  x += 1
  if x >= 5:
    break
echo "Broke out at: ", x

# Do-while pattern (Nim ไม่มี do-while โดยตรง)
var attempts = 0
while true:
  inc attempts
  let input = attempts  # simulate input
  if input >= 3:
    break
echo "Attempts: ", attempts
```

## break และ continue

```nim
# break - หยุด loop
for i in 1..10:
  if i == 5:
    break
  echo i  # 1, 2, 3, 4

# continue - ข้ามไป iteration ถัดไป
for i in 1..10:
  if i mod 2 == 0:
    continue
  echo i  # 1, 3, 5, 7, 9

# break จาก nested loops
block outer:
  for i in 1..3:
    for j in 1..3:
      if i == 2 and j == 2:
        break outer  # break จาก outer loop
      echo i, ",", j
# Output: 1,1 1,2 1,3 2,1

# Named blocks
block myLoop:
  var i = 0
  while true:
    i += 1
    if i == 5:
      break myLoop
  echo "unreachable"
echo "After myLoop"  # แสดง

# continue ใน nested loops
for i in 1..3:
  for j in 1..3:
    if j == 2:
      continue
    echo i, ",", j
```

## return Statement

```nim
# return จาก function
proc findFirst(nums: seq[int], target: int): int =
  for i, n in nums:
    if n == target:
      return i
  return -1  # ไม่พบ

let nums = @[3, 7, 2, 9, 5]
echo findFirst(nums, 9)   # 3
echo findFirst(nums, 10)  # -1

# Early return
proc processData(data: seq[int]): string =
  if data.len == 0:
    return "No data"
  
  if data.len < 3:
    return "Too little data"
  
  var sum = 0
  for n in data:
    sum += n
  
  return "Sum: " & $sum

echo processData(@[])           # No data
echo processData(@[1, 2])       # Too little data
echo processData(@[1, 2, 3, 4]) # Sum: 10
```

## Exceptions (Error Handling)

```nim
# try/except/finally
try:
  let n = parseInt("not a number")
  echo "Parsed: ", n
except ValueError as e:
  echo "ValueError: ", e.msg
except:
  echo "Unknown error: ", getCurrentExceptionMsg()
finally:
  echo "This always runs"

# raise exception
proc divide(a, b: int): int =
  if b == 0:
    raise newException(DivByZeroDefect, "Cannot divide by zero!")
  a div b

try:
  echo divide(10, 2)   # 5
  echo divide(10, 0)   # raises exception
except DivByZeroDefect as e:
  echo "Error: ", e.msg

# Custom exceptions
type
  ValidationError = object of ValueError
    field: string

proc validateAge(age: int) =
  if age < 0 or age > 150:
    var e = newException(ValidationError, "Invalid age: " & $age)
    e.field = "age"
    raise e

try:
  validateAge(200)
except ValidationError as e:
  echo "Validation failed for field: ", e.field
  echo "Message: ", e.msg
```

## Defer Statement

```nim
# defer - รันโค้ดเมื่อออกจาก scope (เหมือน finally)
proc processFile(path: string) =
  let f = open(path, fmRead)
  defer: f.close()  # จะรันเมื่อออกจาก proc ไม่ว่าจะ return ยังไง
  
  for line in f.lines:
    echo line

# หลาย defer - รันแบบ LIFO (Last In, First Out)
proc demo() =
  echo "Start"
  defer: echo "First defer (runs last)"
  defer: echo "Second defer (runs second)"
  defer: echo "Third defer (runs first)"
  echo "End"
  # Output:
  # Start
  # End
  # Third defer (runs first)
  # Second defer (runs second)
  # First defer (runs last)

demo()
```

## Comprehensive Example: State Machine

```nim
# state_machine.nim - Traffic Light State Machine

type
  LightState = enum
    Red, Yellow, Green

proc nextState(current: LightState): LightState =
  case current
  of Red:    Green
  of Green:  Yellow
  of Yellow: Red

proc getWaitTime(state: LightState): int =
  case state
  of Red:    60   # seconds
  of Green:  45
  of Yellow: 5

proc printState(state: LightState, step: int) =
  import strformat
  let time = getWaitTime(state)
  let symbol = case state
    of Red:    "🔴"
    of Green:  "🟢"
    of Yellow: "🟡"
  echo &"Step {step}: {symbol} {state} ({time}s)"

# Simulate traffic light
var currentState = Red
for step in 1..9:
  printState(currentState, step)
  currentState = nextState(currentState)
```

## Practical: FizzBuzz ขั้นสูง

```nim
# fizzbuzz_advanced.nim

# วิธีที่ 1: Simple
proc fizzBuzz1(n: int): string =
  if n mod 15 == 0: "FizzBuzz"
  elif n mod 3 == 0: "Fizz"
  elif n mod 5 == 0: "Buzz"
  else: $n

# วิธีที่ 2: Using case
proc fizzBuzz2(n: int): string =
  case (n mod 3, n mod 5)
  of (0, 0): "FizzBuzz"
  of (0, _): "Fizz"
  of (_, 0): "Buzz"
  else: $n

# วิธีที่ 3: Generalized - รองรับ rules ใดก็ได้
type Rule = tuple[divisor: int, word: string]

proc fizzBuzz3(n: int, rules: seq[Rule]): string =
  result = ""
  for rule in rules:
    if n mod rule.divisor == 0:
      result &= rule.word
  if result.len == 0:
    result = $n

let defaultRules: seq[Rule] = @[(3, "Fizz"), (5, "Buzz")]
let customRules: seq[Rule] = @[(3, "Fizz"), (5, "Buzz"), (7, "Bazz")]

echo "=== FizzBuzz 1-30 ==="
for i in 1..30:
  echo fizzBuzz3(i, defaultRules)

echo "\n=== Custom FizzBuzz (3=Fizz, 5=Buzz, 7=Bazz) 1-21 ==="
for i in 1..21:
  echo fizzBuzz3(i, customRules)
```

## Pattern: Guard Clauses

```nim
# Guard clauses - ตรวจสอบ conditions ก่อน แล้ว return เร็ว
# ทำให้โค้ดอ่านง่ายขึ้น

# แบบเก่า (nested)
proc processUserOld(name: string, age: int, email: string): string =
  if name.len > 0:
    if age >= 18:
      if email.contains("@"):
        return "User " & name & " registered successfully"
      else:
        return "Invalid email"
    else:
      return "Too young"
  else:
    return "Name required"

# แบบใหม่ (guard clauses)
proc processUser(name: string, age: int, email: string): string =
  if name.len == 0:
    return "Name required"
  
  if age < 18:
    return "Too young"
  
  if not email.contains("@"):
    return "Invalid email"
  
  "User " & name & " registered successfully"

echo processUser("", 25, "a@b.com")        # Name required
echo processUser("Alice", 15, "a@b.com")   # Too young
echo processUser("Alice", 25, "invalid")   # Invalid email
echo processUser("Alice", 25, "a@b.com")  # Success
```

## Pattern: Command Dispatcher

```nim
# Command pattern ด้วย case
import strutils, strformat

type Command = enum
  Help, List, Add, Remove, Quit

proc parseCommand(input: string): (Command, seq[string]) =
  let parts = input.strip().splitWhitespace()
  if parts.len == 0:
    return (Help, @[])
  
  let cmd = case parts[0].toLower()
    of "help", "h", "?": Help
    of "list", "ls", "l": List
    of "add", "a": Add
    of "remove", "rm", "r": Remove
    of "quit", "exit", "q": Quit
    else: Help
  
  (cmd, if parts.len > 1: parts[1..^1] else: @[])

var items: seq[string] = @[]
var running = true

proc handleCommand(cmd: Command, args: seq[string]) =
  case cmd
  of Help:
    echo "Commands: help, list, add <item>, remove <item>, quit"
  of List:
    if items.len == 0:
      echo "No items"
    else:
      for i, item in items:
        echo &"  {i+1}. {item}"
  of Add:
    if args.len == 0:
      echo "Usage: add <item>"
    else:
      let item = args.join(" ")
      items.add(item)
      echo &"Added: {item}"
  of Remove:
    if args.len == 0:
      echo "Usage: remove <item>"
    else:
      let item = args.join(" ")
      let idx = items.find(item)
      if idx >= 0:
        items.del(idx)
        echo &"Removed: {item}"
      else:
        echo &"Not found: {item}"
  of Quit:
    running = false
    echo "Goodbye!"

# Simulate commands
let commands = ["help", "add apple", "add banana", "add cherry", 
                "list", "remove banana", "list", "quit"]

for input in commands:
  echo "\n> ", input
  let (cmd, args) = parseCommand(input)
  handleCommand(cmd, args)
  if not running:
    break
```

## แบบฝึกหัด Part 5

### แบบฝึกหัดที่ 1: Grade Calculator
```nim
proc letterGrade(score: float): string =
  # เฉลย
  case int(score)
  of 90..100: "A"
  of 80..89:  "B"
  of 70..79:  "C"
  of 60..69:  "D"
  else:       "F"

proc gradePoints(letter: string): float =
  case letter
  of "A": 4.0
  of "B": 3.0
  of "C": 2.0
  of "D": 1.0
  else:   0.0

let scores = [95.0, 87.5, 73.0, 62.0, 45.0]
for score in scores:
  let grade = letterGrade(score)
  let points = gradePoints(grade)
  echo score, " -> ", grade, " (", points, " points)"
```

### แบบฝึกหัดที่ 2: Fizz-Buzz-Bazz
```nim
# สร้าง program ที่:
# - หาร 3 ได้: Fizz
# - หาร 5 ได้: Buzz  
# - หาร 7 ได้: Bazz
# - หาร 15 ได้: FizzBuzz
# - หาร 21 ได้: FizzBazz
# - หาร 35 ได้: BuzzBazz
# - หาร 105 ได้: FizzBuzzBazz
for i in 1..105:
  var s = ""
  if i mod 3 == 0: s &= "Fizz"
  if i mod 5 == 0: s &= "Buzz"
  if i mod 7 == 0: s &= "Bazz"
  echo if s.len > 0: s else: $i
```

## สรุป Part 5

ในบทนี้เราได้เรียนรู้:
- ✅ if/elif/else สำหรับ branching
- ✅ case statement ที่ทรงพลัง
- ✅ when สำหรับ compile-time conditions
- ✅ for loops ทุกรูปแบบ
- ✅ while loops
- ✅ break และ continue
- ✅ return สำหรับ early exit
- ✅ defer สำหรับ cleanup
- ✅ Pattern: Guard Clauses
- ✅ Pattern: Command Dispatcher

---

**Previous**: [Part 04 - Operators](part04_operators.md)
**Next**: [Part 06 - Loops ขั้นสูง](part06_loops.md)
