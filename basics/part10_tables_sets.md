# Part 10: Tables และ Sets - ตาราง Hash และเซต

## Tables (Hash Maps)

```nim
import tables

# สร้าง Table
var t1: Table[string, int]  # empty
var t2 = initTable[string, int]()
var t3 = {"Alice": 30, "Bob": 25, "Carol": 28}.toTable()

# เพิ่มข้อมูล
t2["one"] = 1
t2["two"] = 2
t2["three"] = 3

# ดึงข้อมูล
echo t3["Alice"]    # 30
echo t3.getOrDefault("Dave", 0)  # 0 (ไม่มีใน table)

# ตรวจสอบ key
echo t3.hasKey("Alice")   # true
echo t3.hasKey("Dave")    # false
echo "Alice" in t3        # true
echo "Dave" notin t3      # true

# ลบข้อมูล
t3.del("Carol")
echo t3  # {"Alice": 30, "Bob": 25}

# Length
echo t3.len  # 2

# Iterate
for key, value in t3:
  echo key, " -> ", value

for key in t3.keys:
  echo key

for value in t3.values:
  echo value

# mget - mutable reference
var scores = {"Alice": 95, "Bob": 87}.toTable()
scores.mget("Alice") += 5  # increment in-place
echo scores  # {"Alice": 100, "Bob": 87}
```

## Table Operations

```nim
import tables, sequtils, algorithm

var inventory = {
  "apple": 50,
  "banana": 30,
  "cherry": 100,
  "date": 20
}.toTable()

# Merge tables
var extra = {"elderberry": 15, "fig": 45}.toTable()
for k, v in extra:
  inventory[k] = v
echo inventory.len  # 6

# Filter table
proc filterTable[K, V](t: Table[K, V], pred: proc(k: K, v: V): bool): Table[K, V] =
  result = initTable[K, V]()
  for k, v in t:
    if pred(k, v):
      result[k] = v

let expensive = filterTable(inventory, proc(k: string, v: int): bool = v >= 40)
echo expensive  # items with count >= 40

# Sort by value
var sorted = toSeq(inventory.pairs)
sorted.sort(proc(a, b: (string, int)): int = a[1] - b[1])
echo "Sorted by count:"
for (k, v) in sorted:
  echo "  ", k, ": ", v

# Invert table
proc invertTable[K, V](t: Table[K, V]): Table[V, K] =
  result = initTable[V, K]()
  for k, v in t:
    result[v] = k

let numToName = {"Alice": 1, "Bob": 2, "Carol": 3}.toTable()
let nameToNum = invertTable(numToName)
echo nameToNum[1]  # Alice
```

## CountTable

```nim
import tables

# CountTable - สำหรับนับความถี่
var ct = initCountTable[string]()

ct.inc("apple")
ct.inc("banana")
ct.inc("apple")
ct.inc("cherry")
ct.inc("banana")
ct.inc("apple")

echo ct          # {apple: 3, banana: 2, cherry: 1}
echo ct["apple"]  # 3

# Most common
ct.sort()  # sort by count (descending)
for item, count in ct:
  echo item, ": ", count

# นับจาก sequence
let words = "the quick brown fox jumps over the lazy dog the".split()
let wordCount = words.toCountTable()
echo wordCount["the"]  # 3

# Combine
var ct2 = initCountTable[string]()
ct2.inc("apple", 10)
ct2.inc("banana", 5)
ct.merge(ct2)
echo ct["apple"]  # 13
```

## OrderedTable

```nim
import tables

# OrderedTable - รักษาลำดับการ insert
var ot = initOrderedTable[string, int]()
ot["c"] = 3
ot["a"] = 1
ot["b"] = 2

echo "Ordered table:"
for k, v in ot:
  echo k, ": ", v  # c, a, b (insertion order)

# Sort ordered table
ot.sort(proc(a, b: (string, int)): int = cmp(a[0], b[0]))
echo "\nSorted ordered table:"
for k, v in ot:
  echo k, ": ", v  # a, b, c
```

## Sets

```nim
import sets

# Set - unique elements
var s1 = initHashSet[int]()
var s2 = [1, 2, 3, 4, 5].toHashSet()

# เพิ่มข้อมูล
s1.incl(1)
s1.incl(2)
s1.incl(2)  # duplicate ไม่เพิ่ม
s1.incl(3)
echo s1  # {1, 2, 3}

# ลบข้อมูล
s1.excl(2)
echo s1  # {1, 3}

# ตรวจสอบ
echo 1 in s2      # true
echo 6 in s2      # false
echo 6 notin s2   # true

# Set operations
let a = [1, 2, 3, 4, 5].toHashSet()
let b = [3, 4, 5, 6, 7].toHashSet()

# Union (รวม)
echo a + b        # {1, 2, 3, 4, 5, 6, 7}
echo a.union(b)   # same

# Intersection (ตัดกัน)
echo a * b              # {3, 4, 5}
echo a.intersection(b)  # same

# Difference (ลบออก)
echo a - b             # {1, 2}
echo a.difference(b)   # same

# Symmetric Difference
echo a.symmetricDifference(b)  # {1, 2, 6, 7}

# Subset/Superset
let c = [3, 4].toHashSet()
echo c <= a     # true (c is subset of a)
echo a >= c     # true (a is superset of c)
echo a == b     # false
```

## Nim System Sets (Bit Sets)

```nim
# System sets - bit fields สำหรับ small enums
type
  Color = enum
    Red, Green, Blue, Yellow, Cyan, Magenta, White, Black

type ColorSet = set[Color]

var colors: ColorSet = {}
colors.incl(Red)
colors.incl(Green)
colors.incl(Blue)

echo colors          # {Red, Green, Blue}
echo Red in colors   # true
echo Yellow in colors  # false

# Set literals
let rgb: ColorSet = {Red, Green, Blue}
let warm: ColorSet = {Red, Yellow}
let cool: ColorSet = {Blue, Cyan}

# Operations
echo rgb + warm     # {Red, Green, Blue, Yellow}
echo rgb * warm     # {Red}
echo rgb - warm     # {Green, Blue}

# Char sets
let digits: set[char] = {'0'..'9'}
let letters: set[char] = {'a'..'z', 'A'..'Z'}
let alphanum: set[char] = digits + letters

echo '5' in digits    # true
echo 'a' in letters   # true
echo '!' in alphanum  # false
```

## Practical: Inventory Management

```nim
# inventory.nim - Inventory system ใช้ Tables

import tables, sets, sequtils, algorithm, strformat, options

type
  Product = object
    id: string
    name: string
    price: float
    quantity: int
    category: string

  Inventory = object
    products: Table[string, Product]
    categories: HashSet[string]

proc newInventory(): Inventory =
  Inventory(
    products: initTable[string, Product](),
    categories: initHashSet[string]()
  )

proc addProduct(inv: var Inventory, p: Product) =
  inv.products[p.id] = p
  inv.categories.incl(p.category)

proc getProduct(inv: Inventory, id: string): Option[Product] =
  if id in inv.products: some(inv.products[id])
  else: none(Product)

proc updateQuantity(inv: var Inventory, id: string, delta: int) =
  if id in inv.products:
    inv.products[id].quantity += delta

proc search(inv: Inventory, query: string): seq[Product] =
  result = @[]
  let lq = query.toLower()
  for _, p in inv.products:
    if lq in p.name.toLower() or lq in p.category.toLower():
      result.add(p)

proc totalValue(inv: Inventory): float =
  result = 0.0
  for _, p in inv.products:
    result += p.price * p.quantity.float

# Demo
var inv = newInventory()
for p in [
  Product(id: "P001", name: "MacBook Pro", price: 65000.0, quantity: 10, category: "Electronics"),
  Product(id: "P002", name: "iPhone 15", price: 35000.0, quantity: 25, category: "Electronics"),
  Product(id: "P003", name: "Nike Running", price: 3500.0, quantity: 50, category: "Shoes"),
  Product(id: "P005", name: "Python Book", price: 850.0, quantity: 100, category: "Books"),
  Product(id: "P006", name: "Nim Programming", price: 1200.0, quantity: 75, category: "Books"),
]:
  inv.addProduct(p)

echo &"Total Value: ฿{totalValue(inv):.2f}"
echo "Categories:"
for cat in inv.categories:
  echo "  - ", cat

echo "\nSearch 'book':"
for p in inv.search("book"):
  echo &"  {p.name}: ฿{p.price:.2f}"
```

## แบบฝึกหัด Part 10

### แบบฝึกหัดที่ 1: Phone Book
```nim
import tables, strutils

type PhoneBook = object
  contacts: Table[string, seq[string]]  # name -> phones

proc newPhoneBook(): PhoneBook =
  PhoneBook(contacts: initTable[string, seq[string]]())

proc addContact(pb: var PhoneBook, name, phone: string) =
  if name notin pb.contacts:
    pb.contacts[name] = @[]
  pb.contacts[name].add(phone)

proc findContact(pb: PhoneBook, name: string): seq[string] =
  pb.contacts.getOrDefault(name, @[])

proc searchContacts(pb: PhoneBook, query: string): seq[string] =
  result = @[]
  let lq = query.toLower()
  for name in pb.contacts.keys:
    if lq in name.toLower():
      result.add(name)
  result.sort()

var pb = newPhoneBook()
pb.addContact("Alice Smith", "081-234-5678")
pb.addContact("Alice Smith", "02-345-6789")
pb.addContact("Bob Jones", "089-876-5432")
pb.addContact("Carol White", "083-111-2222")

echo "Alice's numbers: ", pb.findContact("Alice Smith")
echo "Search 'alice': ", pb.searchContacts("alice")
```

## สรุป Part 10

ในบทนี้เราได้เรียนรู้:
- ✅ Table[K,V]: hash map พื้นฐาน
- ✅ CountTable: นับความถี่
- ✅ OrderedTable: รักษาลำดับ
- ✅ HashSet: set ที่ไม่ซ้ำ
- ✅ OrderedSet: set รักษาลำดับ
- ✅ System sets: bit fields สำหรับ enums
- ✅ Set operations: union, intersection, difference

---

**Previous**: [Part 09 - Strings](part09_strings.md)
**Next**: [Part 11 - Tuples](part11_tuples.md)
