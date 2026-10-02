# Part 17: Generics - โปรแกรมแบบ Generic

## Generic Functions

```nim
# Generic function ที่ทำงานได้กับทุก type
proc identity[T](x: T): T = x

echo identity(42)         # 42
echo identity("hello")    # hello
echo identity(3.14)       # 3.14
echo identity(true)       # true

# Generic swap
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

# Generic min/max
proc myMin[T](a, b: T): T =
  if a < b: a else: b

proc myMax[T](a, b: T): T =
  if a > b: a else: b

echo myMin(3, 5)      # 3
echo myMin(3.14, 2.71) # 2.71
echo myMin("apple", "banana")  # apple
```

## Type Constraints

```nim
# Constrain to specific types
proc sum[T: SomeNumber](arr: openArray[T]): T =
  result = T(0)
  for x in arr:
    result += x

echo sum([1, 2, 3, 4, 5])           # 15
echo sum([1.5, 2.5, 3.5])           # 7.5
# echo sum(["a", "b"])              # Error! string is not SomeNumber

# Multiple constraints
proc minMax[T: SomeOrdinal](arr: openArray[T]): (T, T) =
  var mn = arr[0]
  var mx = arr[0]
  for x in arr:
    if x < mn: mn = x
    if x > mx: mx = x
  (mn, mx)

echo minMax([3, 1, 4, 1, 5, 9])  # (1, 9)
echo minMax(['a', 'z', 'm'])     # (a, z)

# OR constraints (|)
proc printValue[T: int | float | string](x: T) =
  echo "Value: ", x

printValue(42)
printValue(3.14)
printValue("hello")
# printValue(true)  # Error! bool not in constraint

# Concept-based constraints
type Printable = concept x
  $x is string

proc display[T: Printable](x: T) =
  echo "Display: ", $x

display(42)      # works (int has $ defined)
display("hello") # works
display(@[1,2,3])  # works

# Comparable constraint
type Comparable = concept x, y
  (x < y) is bool

proc clamp[T: Comparable](x, lo, hi: T): T =
  if x < lo: lo
  elif hi < x: hi
  else: x

echo clamp(5, 1, 10)    # 5
echo clamp(15, 1, 10)   # 10
echo clamp(-5, 1, 10)   # 1
```

## Generic Types

```nim
# Generic object
type
  Box[T] = object
    value: T
    label: string

proc newBox[T](value: T, label: string = ""): Box[T] =
  Box[T](value: value, label: label)

proc get[T](b: Box[T]): T = b.value

proc map[T, U](b: Box[T], f: proc(x: T): U): Box[U] =
  Box[U](value: f(b.value), label: b.label)

let intBox = newBox(42, "number")
let strBox = newBox("hello", "greeting")

echo intBox.get()  # 42
echo strBox.get()  # hello

let doubledBox = intBox.map(x => x * 2)
echo doubledBox.get()  # 84

# Generic Stack
type Stack[T] = object
  items: seq[T]

proc newStack[T](): Stack[T] =
  Stack[T](items: @[])

proc push[T](s: var Stack[T], item: T) =
  s.items.add(item)

proc pop[T](s: var Stack[T]): T =
  if s.items.len == 0:
    raise newException(IndexDefect, "Stack is empty")
  let item = s.items[^1]
  s.items.del(s.items.high)
  item

proc peek[T](s: Stack[T]): T =
  if s.items.len == 0:
    raise newException(IndexDefect, "Stack is empty")
  s.items[^1]

proc isEmpty[T](s: Stack[T]): bool = s.items.len == 0
proc size[T](s: Stack[T]): int = s.items.len

# Test Stack
var intStack = newStack[int]()
intStack.push(1)
intStack.push(2)
intStack.push(3)

echo "Size: ", intStack.size()  # 3
echo "Top: ", intStack.peek()   # 3
echo "Pop: ", intStack.pop()    # 3
echo "Pop: ", intStack.pop()    # 2
echo "Size: ", intStack.size()  # 1

# Generic Queue
type Queue[T] = object
  items: seq[T]

proc newQueue[T](): Queue[T] = Queue[T](items: @[])
proc enqueue[T](q: var Queue[T], item: T) = q.items.add(item)
proc dequeue[T](q: var Queue[T]): T =
  if q.items.len == 0: raise newException(IndexDefect, "Queue empty")
  result = q.items[0]
  q.items.del(0)
proc front[T](q: Queue[T]): T = q.items[0]
proc isEmpty[T](q: Queue[T]): bool = q.items.len == 0

var strQueue = newQueue[string]()
strQueue.enqueue("first")
strQueue.enqueue("second")
strQueue.enqueue("third")
echo strQueue.dequeue()  # first
echo strQueue.dequeue()  # second
```

## Generic Data Structures

```nim
# Generic Binary Search Tree
type
  BST[T] = ref object
    value: T
    left, right: BST[T]

proc insert[T](t: var BST[T], value: T) =
  if t == nil:
    t = BST[T](value: value)
    return
  if value < t.value:
    insert(t.left, value)
  elif value > t.value:
    insert(t.right, value)

proc contains[T](t: BST[T], value: T): bool =
  if t == nil: return false
  if value == t.value: return true
  if value < t.value: return contains(t.left, value)
  return contains(t.right, value)

proc inorder[T](t: BST[T]): seq[T] =
  if t == nil: return @[]
  result = inorder(t.left) & @[t.value] & inorder(t.right)

# Test BST
var tree: BST[int]
for n in [5, 3, 7, 1, 4, 6, 8]:
  tree.insert(n)

echo tree.inorder()     # @[1, 3, 4, 5, 6, 7, 8]
echo tree.contains(4)   # true
echo tree.contains(9)   # false

# Generic Pair / Either
type
  Either[L, R] = object
    case isRight: bool
    of true:  right: R
    of false: left: L

proc Left[L, R](value: L): Either[L, R] =
  Either[L, R](isRight: false, left: value)

proc Right[L, R](value: R): Either[L, R] =
  Either[L, R](isRight: true, right: value)

proc map[L, R, U](e: Either[L, R], f: proc(x: R): U): Either[L, U] =
  if e.isRight: Right[L, U](f(e.right))
  else: Left[L, U](e.left)

proc getOrElse[L, R](e: Either[L, R], default: R): R =
  if e.isRight: e.right else: default

# Example
proc safeDivide(a, b: float): Either[string, float] =
  if b == 0.0: Left[string, float]("Division by zero")
  else: Right[string, float](a / b)

let r1 = safeDivide(10.0, 2.0)
let r2 = safeDivide(10.0, 0.0)

echo r1.getOrElse(0.0)   # 5.0
echo r2.getOrElse(0.0)   # 0.0

let doubled = r1.map(x => x * 2)
echo doubled.getOrElse(0.0)  # 10.0
```

## Generic Algorithms

```nim
# Generic sorting
proc insertionSort[T](arr: var seq[T]) =
  for i in 1..<arr.len:
    let key = arr[i]
    var j = i - 1
    while j >= 0 and arr[j] > key:
      arr[j+1] = arr[j]
      dec j
    arr[j+1] = key

var ints = @[5, 3, 8, 1, 9, 2, 7, 4, 6]
insertionSort(ints)
echo ints  # @[1, 2, 3, 4, 5, 6, 7, 8, 9]

var strs = @["banana", "apple", "cherry", "date"]
insertionSort(strs)
echo strs  # @[apple, banana, cherry, date]

# Generic binary search
proc binarySearch[T](arr: openArray[T], target: T): int =
  var lo = 0
  var hi = arr.high
  
  while lo <= hi:
    let mid = (lo + hi) div 2
    if arr[mid] == target: return mid
    elif arr[mid] < target: lo = mid + 1
    else: hi = mid - 1
  
  -1

let sorted = [1, 3, 5, 7, 9, 11, 13, 15]
echo binarySearch(sorted, 7)   # 3
echo binarySearch(sorted, 10)  # -1
```

## Practical: Generic Cache

```nim
# Generic LRU Cache
import tables, lists

type
  CacheEntry[V] = object
    key: string
    value: V

  LRUCache[V] = object
    capacity: int
    cache: Table[string, V]
    order: DoublyLinkedList[string]

proc newLRUCache[V](capacity: int): LRUCache[V] =
  LRUCache[V](
    capacity: capacity,
    cache: initTable[string, V](),
    order: initDoublyLinkedList[string]()
  )

proc get[V](lru: var LRUCache[V], key: string): Option[V] =
  import options
  if key in lru.cache:
    some(lru.cache[key])
  else:
    none(V)

proc put[V](lru: var LRUCache[V], key: string, value: V) =
  if key in lru.cache:
    lru.cache[key] = value
  else:
    if lru.cache.len >= lru.capacity:
      # Remove oldest (simple implementation)
      for k in lru.cache.keys:
        lru.cache.del(k)
        break
    lru.cache[key] = value

# Test
var cache = newLRUCache[int](3)
cache.put("a", 1)
cache.put("b", 2)
cache.put("c", 3)

echo cache.get("a")  # Some(1)
cache.put("d", 4)    # Evicts oldest
echo cache.get("a")  # might be None (evicted)
```

## แบบฝึกหัด Part 17

### แบบฝึกหัดที่ 1: Generic Pair
```nim
type Pair[A, B] = tuple[first: A, second: B]

proc makePair[A, B](a: A, b: B): Pair[A, B] =
  (first: a, second: b)

proc fst[A, B](p: Pair[A, B]): A = p.first
proc snd[A, B](p: Pair[A, B]): B = p.second

proc swap[A, B](p: Pair[A, B]): Pair[B, A] =
  makePair(p.second, p.first)

proc map[A, B, C](p: Pair[A, B], f: proc(a: A): C): Pair[C, B] =
  makePair(f(p.first), p.second)

let p = makePair(1, "hello")
echo p.fst()   # 1
echo p.snd()   # hello
echo p.swap()  # (first: "hello", second: 1)
echo p.map(x => x * 2)  # (first: 2, second: "hello")
```

## สรุป Part 17

ในบทนี้เราได้เรียนรู้:
- ✅ Generic functions
- ✅ Type constraints: SomeNumber, concept
- ✅ Generic types: Box, Stack, Queue
- ✅ Generic data structures: BST, Either
- ✅ Generic algorithms
- ✅ Practical: LRU Cache

---

**Previous**: [Part 16 - OOP](part16_oop.md)
**Next**: [Part 18 - Templates](part18_templates.md)
