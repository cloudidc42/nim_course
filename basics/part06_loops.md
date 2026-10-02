# Part 06: Loops ขั้นสูง - การวนซ้ำ

## for Loop รูปแบบต่างๆ

```nim
# 1. Range loop
for i in 1..10:
  echo i

# 2. Exclusive range
for i in 0..<10:
  echo i  # 0-9

# 3. Countdown
for i in countdown(10, 1):
  echo i  # 10..1

# 4. countup with step
for i in countup(0, 100, 10):
  echo i  # 0, 10, 20, ..., 100

# 5. Sequence
let fruits = @["apple", "banana", "cherry"]
for fruit in fruits:
  echo fruit

# 6. Array
let primes = [2, 3, 5, 7, 11, 13]
for p in primes:
  echo p

# 7. String characters
for ch in "Hello, World!":
  echo ch

# 8. With index (pairs)
for i, fruit in fruits:
  echo i, ": ", fruit

# 9. Reversed
for fruit in fruits.reversed():
  echo fruit

# 10. Items iterator
for item in items(fruits):
  echo item

# 11. mitems (mutable items)
var numbers = @[1, 2, 3, 4, 5]
for n in mitems(numbers):
  n *= 2
echo numbers  # @[2, 4, 6, 8, 10]
```

## Iterator ใน Nim

```nim
# Iterator คล้าย generator ใน Python
iterator countTo(n: int): int =
  var i = 0
  while i <= n:
    yield i
    inc i

for n in countTo(5):
  echo n  # 0, 1, 2, 3, 4, 5

# Iterator กับ multiple values
iterator enumerate[T](s: seq[T]): (int, T) =
  for i, item in s:
    yield (i, item)

for (i, fruit) in enumerate(@["apple", "banana", "cherry"]):
  echo i, ": ", fruit

# Infinite iterator
iterator naturals(): int =
  var n = 0
  while true:
    yield n
    inc n

# ใช้ตัดด้วย break
var count = 0
for n in naturals():
  echo n
  inc count
  if count >= 5: break

# Closure iterator
proc makeCounter(start: int = 0): iterator(): int =
  return iterator(): int =
    var i = start
    while true:
      yield i
      inc i

let counter = makeCounter(10)
for _ in 0..<5:
  echo counter()  # 10, 11, 12, 13, 14

# Iterator กับ filter
iterator filter[T](s: seq[T], pred: proc(x: T): bool): T =
  for item in s:
    if pred(item):
      yield item

let evens = toSeq(filter(@[1,2,3,4,5,6,7,8], proc(x: int): bool = x mod 2 == 0))
echo evens  # @[2, 4, 6, 8]
```

## while Loop รูปแบบต่างๆ

```nim
# 1. Basic while
var i = 0
while i < 5:
  echo i
  inc i

# 2. Do-while pattern
var attempts = 0
while true:
  inc attempts
  echo "Attempt: ", attempts
  if attempts >= 3: break

# 3. While กับ multiple conditions
var x = 100
var y = 0
while x > 0 and y < 50:
  x -= 7
  y += 3
echo "Final: x=", x, " y=", y

# 4. While กับ complex state
type
  ParseState = enum
    Start, InWord, InSpace, Done

var state = Start
var words: seq[string] = @[]
var current = ""
let text = "  hello   world  nim  "

var pos = 0
while pos <= text.len:
  let ch = if pos < text.len: text[pos] else: '\0'
  
  case state
  of Start:
    if ch == ' ': state = InSpace
    elif ch != '\0':
      current &= ch
      state = InWord
  of InWord:
    if ch == ' ' or ch == '\0':
      words.add(current)
      current = ""
      state = if ch == '\0': Done else: InSpace
    else:
      current &= ch
  of InSpace:
    if ch == '\0': state = Done
    elif ch != ' ':
      current &= ch
      state = InWord
  of Done:
    break
  
  inc pos

echo words  # @["hello", "world", "nim"]
```

## Loop Control: break, continue, return

```nim
# break ใน for loop
for i in 1..100:
  if i * i > 50:
    echo "First i where i^2 > 50: ", i
    break

# continue - ข้ามไป iteration ถัดไป
var sum = 0
for i in 1..20:
  if i mod 3 == 0:
    continue
  sum += i
echo "Sum of numbers 1-20 not divisible by 3: ", sum

# Nested loop control ด้วย block
block search:
  for i in 1..10:
    for j in 1..10:
      if i * j == 42:
        echo "Found: ", i, " * ", j, " = 42"
        break search

# Loop labels (ใช้ block)
block outer:
  for i in 0..<3:
    block inner:
      for j in 0..<3:
        if i == 1 and j == 1:
          echo "Skipping inner at i=1,j=1"
          break inner  # ออกจาก inner เท่านั้น
        echo i, ",", j
```

## Functional Loop Patterns

```nim
import sequtils, sugar

let numbers = @[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# map - แปลงค่า
let doubled = numbers.map(x => x * 2)
echo doubled  # @[2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

# filter - กรอง
let evens = numbers.filter(x => x mod 2 == 0)
echo evens  # @[2, 4, 6, 8, 10]

# foldl/foldr - reduce
let sum = numbers.foldl(a + b)
echo sum  # 55

let product = numbers.foldl(a * b)
echo product  # 3628800

# mapIt (simpler syntax)
let squared = numbers.mapIt(it * it)
echo squared  # @[1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

# filterIt
let odds = numbers.filterIt(it mod 2 != 0)
echo odds  # @[1, 3, 5, 7, 9]

# keepIf (in-place filter)
var mutable_nums = @[1, 2, 3, 4, 5, 6]
mutable_nums.keepIf(x => x mod 2 == 0)
echo mutable_nums  # @[2, 4, 6]

# apply (in-place map)
var nums2 = @[1, 2, 3, 4, 5]
apply(nums2, x => x * x)
echo nums2  # @[1, 4, 9, 16, 25]

# zip - รวมสอง sequences
let names = @["Alice", "Bob", "Carol"]
let ages = @[30, 25, 28]
let zipped = zip(names, ages)
for (name, age) in zipped:
  echo name, " is ", age

# all, any, none
echo numbers.all(x => x > 0)    # true
echo numbers.any(x => x > 9)    # true
echo numbers.none(x => x > 10)  # true

# count
echo numbers.count(x => x mod 2 == 0)  # 5 (even numbers)

# find
echo numbers.find(x => x > 7)  # 7 (index of first match)
```

## Comprehension-like Patterns

```nim
import sequtils, sugar

# List comprehension (Python: [x*2 for x in range(1,11) if x%2==0])
let result = toSeq(1..10).filterIt(it mod 2 == 0).mapIt(it * 2)
echo result  # @[4, 8, 12, 16, 20]

# Nested comprehension
let matrix2 = collect(newSeq):
  for i in 1..3:
    collect(newSeq):
      for j in 1..3:
        i * j

for row in matrix2:
  echo row

# collect macro
import std/sugar

let evensSquared = collect(newSeq):
  for x in 1..20:
    if x mod 2 == 0:
      x * x

echo evensSquared  # @[4, 16, 36, 64, 100, 144, 196, 256, 324, 400]

# collect เป็น Table
import tables
let wordLengths = collect(initTable[string, int]()):
  for word in @["hello", "world", "nim"]:
    {word: word.len}

echo wordLengths  # {"hello": 5, "world": 5, "nim": 3}
```

## Loop Performance

```nim
# เปรียบเทียบ performance ของ loop patterns

import times, strformat

let n = 10_000_000

# 1. Classic for loop
let t1 = cpuTime()
var sum1 = 0
for i in 0..<n:
  sum1 += i
echo &"Classic loop: {(cpuTime()-t1)*1000:.2f}ms, sum={sum1}"

# 2. While loop
let t2 = cpuTime()
var sum2 = 0
var i2 = 0
while i2 < n:
  sum2 += i2
  inc i2
echo &"While loop: {(cpuTime()-t2)*1000:.2f}ms, sum={sum2}"

# 3. Functional (slower due to seq allocation)
import sequtils
let t3 = cpuTime()
let sum3 = toSeq(0..<n).foldl(a + b)
echo &"Functional: {(cpuTime()-t3)*1000:.2f}ms, sum={sum3}"

# Loop unrolling (manual optimization)
proc sumRange(n: int): int64 =
  result = 0
  var i = 0
  # Process 4 at a time
  while i + 3 < n:
    result += i + (i+1) + (i+2) + (i+3)
    i += 4
  # Handle remainder
  while i < n:
    result += i
    inc i
```

## Advanced Iterator Patterns

```nim
# Fibonacci iterator
iterator fibonacci(): int =
  var a, b = 0, 1
  while true:
    yield a
    (a, b) = (b, a + b)

echo "First 10 Fibonacci numbers:"
var count = 0
for f in fibonacci():
  echo f
  inc count
  if count >= 10: break

# Tree traversal iterator
type
  TreeNode = ref object
    value: int
    left, right: TreeNode

proc newNode(v: int, l, r: TreeNode = nil): TreeNode =
  TreeNode(value: v, left: l, right: r)

iterator inorder(node: TreeNode): int =
  # Iterative in-order traversal using a stack
  var stack: seq[TreeNode] = @[]
  var current = node
  
  while current != nil or stack.len > 0:
    while current != nil:
      stack.add(current)
      current = current.left
    
    current = stack.pop()
    yield current.value
    current = current.right

# Build a BST
let tree = newNode(5,
  newNode(3, newNode(1), newNode(4)),
  newNode(8, newNode(7), newNode(9))
)

echo "In-order traversal:"
for v in inorder(tree):
  echo v  # 1, 3, 4, 5, 7, 8, 9
```

## Loop Patterns: Common Algorithms

```nim
# Binary Search with loop
proc binarySearch[T](arr: seq[T], target: T): int =
  var lo = 0
  var hi = arr.high
  
  while lo <= hi:
    let mid = (lo + hi) div 2
    if arr[mid] == target:
      return mid
    elif arr[mid] < target:
      lo = mid + 1
    else:
      hi = mid - 1
  
  return -1

let sorted = @[1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
echo binarySearch(sorted, 7)   # 3
echo binarySearch(sorted, 10)  # -1

# Bubble Sort
proc bubbleSort(arr: var seq[int]) =
  let n = arr.len
  var swapped = true
  while swapped:
    swapped = false
    for i in 0..<n-1:
      if arr[i] > arr[i+1]:
        swap(arr[i], arr[i+1])
        swapped = true

var data = @[64, 34, 25, 12, 22, 11, 90]
bubbleSort(data)
echo data  # @[11, 12, 22, 25, 34, 64, 90]

# Sieve of Eratosthenes
proc sieve(n: int): seq[int] =
  var isPrime = newSeq[bool](n + 1)
  for i in 2..n:
    isPrime[i] = true
  
  var p = 2
  while p * p <= n:
    if isPrime[p]:
      var multiple = p * p
      while multiple <= n:
        isPrime[multiple] = false
        multiple += p
    inc p
  
  result = @[]
  for i in 2..n:
    if isPrime[i]:
      result.add(i)

echo "Primes up to 50: ", sieve(50)

# Floyd's Cycle Detection
type ListNode = ref object
  val: int
  next: ListNode

proc hasCycle(head: ListNode): bool =
  var slow = head
  var fast = head
  
  while fast != nil and fast.next != nil:
    slow = slow.next
    fast = fast.next.next
    if slow == fast:
      return true
  
  return false
```

## Practical Example: CSV Parser

```nim
# csv_parser.nim - Parse CSV files ด้วย loops

import strutils, strformat

type CsvTable = seq[seq[string]]

proc parseCsv(content: string, delimiter: char = ','): CsvTable =
  result = @[]
  for line in content.splitLines():
    if line.len == 0: continue
    var row: seq[string] = @[]
    var field = ""
    var inQuotes = false
    
    for ch in line:
      if ch == '"':
        inQuotes = not inQuotes
      elif ch == delimiter and not inQuotes:
        row.add(field.strip())
        field = ""
      else:
        field &= ch
    
    row.add(field.strip())
    result.add(row)

proc printTable(table: CsvTable) =
  if table.len == 0: return
  
  # Calculate column widths
  let cols = table[0].len
  var widths = newSeq[int](cols)
  
  for row in table:
    for i, cell in row:
      if i < cols:
        widths[i] = max(widths[i], cell.len)
  
  # Print separator
  let sep = "+" & widths.mapIt("-".repeat(it + 2)).join("+") & "+"
  
  for rowIdx, row in table:
    echo sep
    var line = "|"
    for i, cell in row:
      if i < cols:
        line &= " " & cell.alignLeft(widths[i]) & " |"
    echo line
    if rowIdx == 0:
      echo sep  # Double separator after header
  echo sep

# Test
let csvData = """
Name,Age,City,Score
Alice,30,Bangkok,95.5
Bob,25,Chiang Mai,87.0
Carol,28,Phuket,92.3
Dave,35,Khon Kaen,78.8
"""

let table = parseCsv(csvData)
printTable(table)

# Process data
echo "\n=== Statistics ==="
var totalScore = 0.0
var count = 0
for row in table[1..^1]:  # skip header
  totalScore += parseFloat(row[3])
  inc count

echo &"Average score: {totalScore/count.float:.2f}"
```

## แบบฝึกหัด Part 6

### แบบฝึกหัดที่ 1: Pattern Printing
```nim
# Print patterns using loops

# Triangle
proc printTriangle(n: int) =
  for i in 1..n:
    echo "*".repeat(i)

# Diamond
proc printDiamond(n: int) =
  # Upper half
  for i in 1..n:
    let spaces = " ".repeat(n - i)
    let stars = "*".repeat(2*i - 1)
    echo spaces & stars
  # Lower half
  for i in countdown(n-1, 1):
    let spaces = " ".repeat(n - i)
    let stars = "*".repeat(2*i - 1)
    echo spaces & stars

echo "=== Triangle ==="
printTriangle(5)
echo "\n=== Diamond ==="
printDiamond(5)
```

### แบบฝึกหัดที่ 2: Prime Generator
```nim
# Generate primes using iterator
iterator primes(): int =
  yield 2
  var candidates = toSeq(countup(3, int.high, 2))
  var found: seq[int] = @[2]
  
  for candidate in candidates:
    var isPrime = true
    for p in found:
      if p * p > candidate: break
      if candidate mod p == 0:
        isPrime = false
        break
    if isPrime:
      found.add(candidate)
      yield candidate

echo "First 20 primes:"
var count = 0
for p in primes():
  echo p
  inc count
  if count >= 20: break
```

## สรุป Part 6

ในบทนี้เราได้เรียนรู้:
- ✅ for loop ทุกรูปแบบ
- ✅ Iterator และ yield
- ✅ while loop รูปแบบต่างๆ
- ✅ break, continue, block labels
- ✅ Functional patterns: map, filter, fold
- ✅ collect macro
- ✅ Loop performance
- ✅ Common algorithms ด้วย loops

---

**Previous**: [Part 05 - Control Flow](part05_control_flow.md)
**Next**: [Part 07 - Functions](part07_functions.md)
