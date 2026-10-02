# Part 08: Arrays และ Sequences - อาร์เรย์และลำดับ

## Arrays (ขนาดตายตัว)

```nim
# Array declaration - ขนาดต้องรู้ตอน compile
var arr1: array[5, int]           # ประกาศ array ขนาด 5
var arr2 = [1, 2, 3, 4, 5]       # inferred type: array[5, int]
var arr3: array[3, string] = ["a", "b", "c"]

# Zero-indexed
echo arr2[0]   # 1
echo arr2[4]   # 5
echo arr2[^1]  # 5 (last element)
echo arr2[^2]  # 4 (second to last)

# Array สามารถมี index ที่กำหนดเองได้
type Direction = enum North, South, East, West
var dirVec: array[Direction, string]
dirVec[North] = "Up"
dirVec[South] = "Down"
dirVec[East] = "Right"
dirVec[West] = "Left"

for dir, vec in dirVec:
  echo dir, " -> ", vec

# Static array operations
var nums = [10, 20, 30, 40, 50]
echo nums.len         # 5
echo nums.high        # 4 (last index)
echo nums.low         # 0 (first index)
echo nums[1..3]       # [20, 30, 40]

# Iterate
for n in nums:
  echo n

for i, n in nums:
  echo i, ": ", n

# Multidimensional arrays
var matrix: array[3, array[3, int]]
for i in 0..2:
  for j in 0..2:
    matrix[i][j] = i * 3 + j

# หรือ initialize โดยตรง
let grid = [[1, 2, 3],
            [4, 5, 6],
            [7, 8, 9]]

echo grid[1][2]  # 6

# 3D array
var cube: array[2, array[2, array[2, int]]]
cube[0][0][0] = 1
cube[1][1][1] = 8
```

## Sequences (Dynamic Arrays)

```nim
# Sequences - dynamic, resizable
var s1: seq[int] = @[]             # empty sequence
var s2 = @[1, 2, 3, 4, 5]         # initialized
var s3 = newSeq[string](3)         # sequence of 3 empty strings

# Add elements
s1.add(10)
s1.add(20)
s1.add(30)
echo s1  # @[10, 20, 30]

# Append another sequence
s1 &= @[40, 50]
echo s1  # @[10, 20, 30, 40, 50]

# Insert
s2.insert(99, 2)  # insert 99 at index 2
echo s2  # @[1, 2, 99, 3, 4, 5]

# Delete
s2.delete(2)  # delete at index 2
echo s2  # @[1, 2, 3, 4, 5]

# Pop (remove and return last)
let last = s2.pop()
echo last  # 5
echo s2    # @[1, 2, 3, 4]

# Length
echo s2.len    # 4
echo s2.high   # 3

# Access
echo s2[0]     # 1
echo s2[^1]    # 4 (last)
echo s2[1..2]  # @[2, 3]

# Check if empty
echo s1.len == 0  # false
echo newSeq[int]().len == 0  # true
```

## Operations บน Sequences

```nim
import sequtils

let nums = @[3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]

# Find
echo nums.find(5)     # 4 (first occurrence index)
echo nums.contains(9) # true
echo 9 in nums        # true

# Count
echo nums.count(5)    # 3 (occurrences)

# Sort
import algorithm
var sorted = nums
sorted.sort()
echo sorted  # @[1, 1, 2, 3, 3, 4, 5, 5, 5, 6, 9]

# Reverse sort
sorted.sort(Descending)
echo sorted  # @[9, 6, 5, 5, 5, 4, 3, 3, 2, 1, 1]

# Custom sort
var people = @[("Bob", 25), ("Alice", 30), ("Carol", 22)]
people.sort(proc(a, b: (string, int)): int =
  cmp(a[1], b[1]))  # sort by age
echo people  # @[("Carol", 22), ("Bob", 25), ("Alice", 30)]

# Reverse
var r = @[1, 2, 3, 4, 5]
reverse(r)
echo r  # @[5, 4, 3, 2, 1]
echo r.reversed()  # @[1, 2, 3, 4, 5] (returns new)

# Deduplicate
var dup = @[1, 2, 2, 3, 3, 3, 4]
dup.deduplicate()
echo dup  # @[1, 2, 3, 4]

# Flatten nested sequences
let nested = @[@[1, 2], @[3, 4], @[5, 6]]
echo nested.concat()  # @[1, 2, 3, 4, 5, 6]

# Zip
let a = @[1, 2, 3]
let b = @["a", "b", "c"]
let zipped = zip(a, b)
echo zipped  # @[(1, "a"), (2, "b"), (3, "c")]

# Unzip
let (unzippedA, unzippedB) = unzip(zipped)
echo unzippedA  # @[1, 2, 3]
echo unzippedB  # @["a", "b", "c"]
```

## Functional Operations บน Sequences

```nim
import sequtils, sugar

let nums = @[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# map - แปลงค่า
let doubled = nums.map(x => x * 2)
echo doubled  # @[2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

# mapIt (simpler)
let squared = nums.mapIt(it * it)
echo squared  # @[1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

# filter
let evens = nums.filter(x => x mod 2 == 0)
echo evens  # @[2, 4, 6, 8, 10]

# filterIt
let odds = nums.filterIt(it mod 2 != 0)
echo odds  # @[1, 3, 5, 7, 9]

# foldl (left fold / reduce)
let sum = nums.foldl(a + b)
echo sum  # 55

let product = nums.foldl(a * b)
echo product  # 3628800

# apply (in-place map)
var mutable = @[1, 2, 3, 4, 5]
apply(mutable, x => x * 10)
echo mutable  # @[10, 20, 30, 40, 50]

# keepIf (in-place filter)
var data = @[1, 2, 3, 4, 5, 6, 7, 8]
data.keepIf(x => x mod 3 == 0)
echo data  # @[3, 6]

# all, any, none
echo nums.all(x => x > 0)     # true
echo nums.any(x => x > 9)     # true
echo nums.none(x => x > 10)   # true

# toSeq (convert iterator to seq)
import std/sugar
let evenSeq = collect(newSeq):
  for i in 0..20:
    if i mod 2 == 0: i

echo evenSeq  # @[0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```

## Slicing และ Manipulation

```nim
let data = @[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# Slice
echo data[2..5]     # @[2, 3, 4, 5]
echo data[2..<5]    # @[2, 3, 4]
echo data[..3]      # @[0, 1, 2, 3]
echo data[7..]      # @[7, 8, 9]
echo data[^3..^1]   # @[7, 8, 9]

# Head/Tail
proc head[T](s: seq[T], n: int = 1): seq[T] =
  s[0..<min(n, s.len)]

proc tail[T](s: seq[T], n: int = 1): seq[T] =
  s[max(0, s.len-n)..<s.len]

echo data.head(3)  # @[0, 1, 2]
echo data.tail(3)  # @[7, 8, 9]

# Chunking
proc chunks[T](s: seq[T], size: int): seq[seq[T]] =
  result = @[]
  var i = 0
  while i < s.len:
    result.add(s[i..<min(i+size, s.len)])
    i += size

echo data.chunks(3)  # @[@[0, 1, 2], @[3, 4, 5], @[6, 7, 8], @[9]]

# Windows (sliding window)
proc windows[T](s: seq[T], size: int): seq[seq[T]] =
  result = @[]
  for i in 0..s.len-size:
    result.add(s[i..<i+size])

echo @[1,2,3,4,5].windows(3)  # @[@[1,2,3], @[2,3,4], @[3,4,5]]
```

## Array/Sequence Algorithms

```nim
# Binary Search (on sorted array)
proc binarySearch[T](arr: openArray[T], target: T): int =
  var lo = 0
  var hi = arr.high
  
  while lo <= hi:
    let mid = (lo + hi) div 2
    if arr[mid] == target: return mid
    elif arr[mid] < target: lo = mid + 1
    else: hi = mid - 1
  
  -1

# Merge Sort
proc mergeSort(arr: seq[int]): seq[int] =
  if arr.len <= 1: return arr
  
  let mid = arr.len div 2
  let left = mergeSort(arr[0..<mid])
  let right = mergeSort(arr[mid..^1])
  
  var result: seq[int] = @[]
  var i, j = 0
  
  while i < left.len and j < right.len:
    if left[i] <= right[j]:
      result.add(left[i])
      inc i
    else:
      result.add(right[j])
      inc j
  
  result & left[i..^1] & right[j..^1]

let unsorted = @[38, 27, 43, 3, 9, 82, 10]
echo mergeSort(unsorted)  # @[3, 9, 10, 27, 38, 43, 82]

# Quick Sort
proc quickSort(arr: var seq[int], lo, hi: int) =
  if lo >= hi: return
  
  let pivot = arr[hi]
  var i = lo - 1
  
  for j in lo..<hi:
    if arr[j] <= pivot:
      inc i
      swap(arr[i], arr[j])
  
  swap(arr[i+1], arr[hi])
  let pi = i + 1
  
  quickSort(arr, lo, pi - 1)
  quickSort(arr, pi + 1, hi)

var qs_data = @[64, 25, 12, 22, 11]
quickSort(qs_data, 0, qs_data.high)
echo qs_data  # @[11, 12, 22, 25, 64]
```

## Practical: Matrix Operations

```nim
# matrix.nim - Matrix operations

type Matrix = seq[seq[float]]

proc newMatrix(rows, cols: int, fill: float = 0.0): Matrix =
  result = newSeq[seq[float]](rows)
  for i in 0..<rows:
    result[i] = newSeq[float](cols)
    for j in 0..<cols:
      result[i][j] = fill

proc `+`(a, b: Matrix): Matrix =
  assert a.len == b.len and a[0].len == b[0].len
  result = newMatrix(a.len, a[0].len)
  for i in 0..<a.len:
    for j in 0..<a[0].len:
      result[i][j] = a[i][j] + b[i][j]

proc `*`(a, b: Matrix): Matrix =
  assert a[0].len == b.len
  result = newMatrix(a.len, b[0].len)
  for i in 0..<a.len:
    for j in 0..<b[0].len:
      for k in 0..<a[0].len:
        result[i][j] += a[i][k] * b[k][j]

proc transpose(m: Matrix): Matrix =
  result = newMatrix(m[0].len, m.len)
  for i in 0..<m.len:
    for j in 0..<m[0].len:
      result[j][i] = m[i][j]

proc identity(n: int): Matrix =
  result = newMatrix(n, n)
  for i in 0..<n:
    result[i][i] = 1.0

# Test
let m1 = @[@[1.0, 2.0, 3.0], @[4.0, 5.0, 6.0]]
let m2 = @[@[7.0, 8.0], @[9.0, 10.0], @[11.0, 12.0]]

echo "M1 * M2:"
for row in m1 * m2:
  echo row
echo "\nIdentity(3):"
for row in identity(3):
  echo row
```

## แบบฝึกหัด Part 8

### แบบฝึกหัดที่ 1: Statistics
```nim
import math, sequtils, algorithm

proc mean(data: seq[float]): float =
  data.sum() / data.len.float

proc median(data: seq[float]): float =
  var sorted = data
  sorted.sort()
  let n = sorted.len
  if n mod 2 == 0:
    (sorted[n div 2 - 1] + sorted[n div 2]) / 2.0
  else:
    sorted[n div 2]

proc variance(data: seq[float]): float =
  let m = mean(data)
  data.mapIt((it - m) * (it - m)).sum() / data.len.float

proc stddev(data: seq[float]): float =
  sqrt(variance(data))

let scores = @[85.0, 92.0, 78.0, 96.0, 88.0, 73.0, 91.0, 84.0]
import strformat
echo &"Mean:   {mean(scores):.2f}"
echo &"Median: {median(scores):.2f}"
echo &"StdDev: {stddev(scores):.2f}"
echo &"Min:    {scores.min():.2f}"
echo &"Max:    {scores.max():.2f}"
```

## สรุป Part 8

ในบทนี้เราได้เรียนรู้:
- ✅ Arrays: fixed-size, stack-allocated
- ✅ Sequences: dynamic, heap-allocated
- ✅ Operations: add, insert, delete, pop, find, sort
- ✅ Functional: map, filter, fold, zip
- ✅ Slicing และ manipulation
- ✅ Sorting algorithms
- ✅ Matrix operations

---

**Previous**: [Part 07 - Functions](part07_functions.md)
**Next**: [Part 09 - Strings](part09_strings.md)
