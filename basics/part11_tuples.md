# Part 11: Tuples และ Object Types - ทูเพิลและอ็อบเจกต์

## Tuples

```nim
# Tuple - fixed-size, heterogeneous collection
var t1: (int, string, float) = (42, "hello", 3.14)
var t2 = (1, "world", true)  # type inferred

# Access by index
echo t1[0]  # 42
echo t1[1]  # hello
echo t1[2]  # 3.14

# Named tuples
type Point = tuple[x, y: int]
type Color = tuple[r, g, b: uint8]

var p: Point = (x: 10, y: 20)
var c: Color = (r: 255'u8, g: 128'u8, b: 0'u8)

echo p.x  # 10
echo c.r  # 255

# Anonymous named tuple
var person = (name: "Alice", age: 30, active: true)
echo person.name   # Alice
echo person.age    # 30

# Destructuring
let (name, age, active) = person
echo name   # Alice
echo age    # 30

# Partial destructuring
let (x, y) = (10, 20)
echo x, " ", y  # 10 20

# Tuple comparison
let a = (1, 2, 3)
let b = (1, 2, 3)
let c = (1, 2, 4)

echo a == b  # true
echo a < c   # true (lexicographic)

# Swap using tuple
var x1 = 5
var y1 = 10
(x1, y1) = (y1, x1)  # swap
echo x1, " ", y1  # 10 5

# Return multiple values
proc minMax(nums: openArray[int]): (int, int) =
  var mn = nums[0]
  var mx = nums[0]
  for n in nums:
    if n < mn: mn = n
    if n > mx: mx = n
  (mn, mx)

let (mn, mx) = minMax([5, 3, 8, 1, 9, 2])
echo "min=", mn, " max=", mx
```

## Object Types

```nim
# Object - structured data
type
  Person = object
    name: string
    age: int
    email: string

# สร้าง object
var p1 = Person(name: "Alice", age: 30, email: "alice@mail.com")
var p2: Person
p2.name = "Bob"
p2.age = 25
p2.email = "bob@mail.com"

echo p1.name  # Alice
echo p2.age   # 25

# Default values (ต้องใช้ proc)
proc newPerson(name: string, age: int = 0, email: string = ""): Person =
  Person(name: name, age: age, email: email)

var p3 = newPerson("Carol", 28)

# Object fields
echo p1.name
p1.age = 31
echo p1.age  # 31

# Objects are value types
var p4 = p1  # copy
p4.name = "Dave"
echo p1.name  # Alice (unchanged)
echo p4.name  # Dave
```

## Ref Objects

```nim
# Ref object - reference type (heap allocated)
type
  Node = ref object
    value: int
    next: Node

# สร้าง
var n1 = Node(value: 1)
var n2 = Node(value: 2)
var n3 = Node(value: 3)

n1.next = n2
n2.next = n3

# Traverse
var current = n1
while current != nil:
  echo current.value
  current = current.next

# Ref objects are reference types
var r1 = Node(value: 100)
var r2 = r1  # reference copy (same object!)
r2.value = 200
echo r1.value  # 200 (changed!)
echo r2.value  # 200

# Check nil
if n1 != nil:
  echo "n1 is valid"

# new() function
var n = new(Node)
n.value = 42
```

## Methods บน Objects

```nim
type
  Rectangle = object
    width, height: float

proc area(r: Rectangle): float =
  r.width * r.height

proc perimeter(r: Rectangle): float =
  2 * (r.width + r.height)

proc isSquare(r: Rectangle): bool =
  r.width == r.height

proc scale(r: var Rectangle, factor: float) =
  r.width *= factor
  r.height *= factor

proc `$`(r: Rectangle): string =
  import strformat
  &"Rectangle({r.width:.1f} x {r.height:.1f})"

var rect = Rectangle(width: 5.0, height: 3.0)
echo rect         # Rectangle(5.0 x 3.0)
echo rect.area()  # 15.0
echo rect.perimeter()  # 16.0
echo rect.isSquare()   # false

rect.scale(2.0)
echo rect  # Rectangle(10.0 x 6.0)

# Method chaining pattern
type Builder = object
  parts: seq[string]

proc add(b: var Builder, part: string): var Builder =
  b.parts.add(part)
  b

proc build(b: Builder): string =
  b.parts.join(", ")

var builder: Builder
echo builder.add("A").add("B").add("C").build()
```

## Variant Objects (Union Types)

```nim
# Variant object - ทำงานเหมือน union ใน C
type
  ShapeKind = enum
    Circle, Rectangle, Triangle

  Shape = object
    case kind: ShapeKind
    of Circle:
      radius: float
    of Rectangle:
      width, height: float
    of Triangle:
      base, h: float  # h = height

proc area(s: Shape): float =
  import math
  case s.kind
  of Circle:
    PI * s.radius * s.radius
  of Rectangle:
    s.width * s.height
  of Triangle:
    0.5 * s.base * s.h

proc `$`(s: Shape): string =
  import strformat
  case s.kind
  of Circle:
    &"Circle(r={s.radius:.1f}, area={s.area():.2f})"
  of Rectangle:
    &"Rect({s.width:.1f}x{s.height:.1f}, area={s.area():.2f})"
  of Triangle:
    &"Triangle(b={s.base:.1f}, h={s.h:.1f}, area={s.area():.2f})"

let shapes = [
  Shape(kind: Circle, radius: 5.0),
  Shape(kind: Rectangle, width: 4.0, height: 6.0),
  Shape(kind: Triangle, base: 3.0, h: 8.0)
]

for s in shapes:
  echo s
```

## Practical: JSON-like Data Structure

```nim
# Implement a simple JSON-like variant type

type
  JsonKind = enum
    JNull, JBool, JInt, JFloat, JString, JArray, JObject

  JsonNode = ref object
    case kind: JsonKind
    of JNull: discard
    of JBool: boolVal: bool
    of JInt: intVal: int64
    of JFloat: floatVal: float64
    of JString: strVal: string
    of JArray: arrayVal: seq[JsonNode]
    of JObject: objectVal: seq[(string, JsonNode)]

proc newNull(): JsonNode = JsonNode(kind: JNull)
proc newBool(b: bool): JsonNode = JsonNode(kind: JBool, boolVal: b)
proc newInt(n: int64): JsonNode = JsonNode(kind: JInt, intVal: n)
proc newFloat(f: float64): JsonNode = JsonNode(kind: JFloat, floatVal: f)
proc newString(s: string): JsonNode = JsonNode(kind: JString, strVal: s)
proc newArray(): JsonNode = JsonNode(kind: JArray, arrayVal: @[])
proc newObject(): JsonNode = JsonNode(kind: JObject, objectVal: @[])

proc add(arr: JsonNode, item: JsonNode) =
  assert arr.kind == JArray
  arr.arrayVal.add(item)

proc set(obj: JsonNode, key: string, val: JsonNode) =
  assert obj.kind == JObject
  for i, (k, _) in obj.objectVal:
    if k == key:
      obj.objectVal[i] = (key, val)
      return
  obj.objectVal.add((key, val))

proc `$`(n: JsonNode): string =
  case n.kind
  of JNull:   "null"
  of JBool:   $n.boolVal
  of JInt:    $n.intVal
  of JFloat:  $n.floatVal
  of JString: "\"" & n.strVal & "\""
  of JArray:
    "[" & n.arrayVal.mapIt($it).join(", ") & "]"
  of JObject:
    "{" & n.objectVal.mapIt("\"“ & it[0] & "\": " & $it[1]).join(", ") & "}"

# Build JSON structure
let user = newObject()
user.set("name", newString("Alice"))
user.set("age", newInt(30))
user.set("active", newBool(true))

let skills = newArray()
skills.add(newString("Nim"))
skills.add(newString("Python"))
skills.add(newString("Rust"))
user.set("skills", skills)

echo user
```

## แบบฝึกหัด Part 11

### แบบฝึกหัดที่ 1: Student Record System
```nim
type
  Grade = enum GradeA = "A", GradeB = "B", GradeC = "C", GradeD = "D", GradeF = "F"

  Subject = tuple[name: string, score: float, grade: Grade]

  Student = object
    id: string
    name: string
    subjects: seq[Subject]

proc calcGrade(score: float): Grade =
  if score >= 80: GradeA
  elif score >= 70: GradeB
  elif score >= 60: GradeC
  elif score >= 50: GradeD
  else: GradeF

proc addSubject(s: var Student, name: string, score: float) =
  s.subjects.add((name: name, score: score, grade: calcGrade(score)))

proc gpa(s: Student): float =
  if s.subjects.len == 0: return 0.0
  var total = 0.0
  for sub in s.subjects:
    total += sub.score
  total / s.subjects.len.float

proc printReport(s: Student) =
  import strformat
  echo &"\n=== Student Report ==="
  echo &"ID: {s.id} | Name: {s.name}"
  echo "Subjects:"
  for sub in s.subjects:
    echo &"  {sub.name}: {sub.score:.1f} ({sub.grade})"
  echo &"GPA: {s.gpa():.2f}"

var student = Student(id: "ST001", name: "Alice")
student.addSubject("Math", 92.0)
student.addSubject("Science", 85.0)
student.addSubject("English", 78.0)
student.addSubject("History", 65.0)
student.printReport()
```

## สรุป Part 11

ในบทนี้เราได้เรียนรู้:
- ✅ Tuples: heterogeneous, fixed-size collections
- ✅ Named tuples
- ✅ Destructuring
- ✅ Object types: value types
- ✅ Ref objects: reference types
- ✅ Methods บน objects (UFCS)
- ✅ Variant objects (discriminated unions)

---

**Previous**: [Part 10 - Tables และ Sets](part10_tables_sets.md)
**Next**: [Part 12 - Enums](part12_enums.md)
