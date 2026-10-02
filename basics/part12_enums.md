# Part 12: Enums - ชนิดข้อมูลที่ระบุค่าได้ล่วงหน้า

## Basic Enums

```nim
# Basic enum
type Direction = enum
  North, South, East, West

var dir = North
echo dir          # North
echo dir.ord      # 0 (ordinal value)
echo succ(dir)    # South
echo pred(West)   # East

# Enum กับ custom values
type HttpStatus = enum
  Ok       = 200
  Created  = 201
  NotFound = 404
  Error    = 500

echo Ok.ord    # 200
echo NotFound  # NotFound

# Enum กับ string values
type Color = enum
  Red   = "red"
  Green = "green"
  Blue  = "blue"

echo Red    # red
echo $Red   # red

# Iteration
for d in Direction:
  echo d

for d in North..East:
  echo d

# Comparison
echo North < South  # true (ordinal order)
echo East > West    # false
echo North == North  # true

# Conversion
let n: int = North.ord
let d2: Direction = Direction(2)  # East
echo d2  # East
```

## Enum Sets

```nim
type
  Permission = enum
    Read, Write, Execute, Admin

type Permissions = set[Permission]

# Set literal
let userPerms: Permissions = {Read, Write}
let adminPerms: Permissions = {Read, Write, Execute, Admin}
let readOnly: Permissions = {Read}

# Check permissions
proc canRead(p: Permissions): bool = Read in p
proc canWrite(p: Permissions): bool = Write in p
proc isAdmin(p: Permissions): bool = Admin in p

echo canRead(userPerms)    # true
echo isAdmin(userPerms)    # false
echo isAdmin(adminPerms)   # true

# Operations
let combined = userPerms + {Execute}
echo combined  # {Read, Write, Execute}

let restricted = adminPerms - {Admin}
echo restricted  # {Read, Write, Execute}

# Real-world example: File permissions
type FileMode = set[char]

proc formatMode(mode: FileMode): string =
  result = ""
  result &= if 'r' in mode: "r" else: "-"
  result &= if 'w' in mode: "w" else: "-"
  result &= if 'x' in mode: "x" else: "-"

let ownerMode: FileMode = {'r', 'w'}
let groupMode: FileMode = {'r'}
let otherMode: FileMode = {}

echo formatMode(ownerMode)  # rw-
echo formatMode(groupMode)  # r--
echo formatMode(otherMode)  # ---
```

## Advanced Enum Patterns

```nim
# Enum กับ methods (using case)
type Season = enum Spring, Summer, Autumn, Winter

proc temperature(s: Season): string =
  case s
  of Spring: "warm"
  of Summer: "hot"
  of Autumn: "cool"
  of Winter: "cold"

proc nextSeason(s: Season): Season =
  Season((s.ord + 1) mod 4)

for s in Season:
  echo s, " is ", s.temperature()
  echo "  Next season: ", s.nextSeason()

# Enum ใน variant objects
type
  NodeKind = enum
    Leaf, Inner

  TreeNode = ref object
    case kind: NodeKind
    of Leaf:
      value: int
    of Inner:
      left, right: TreeNode

proc newLeaf(v: int): TreeNode =
  TreeNode(kind: Leaf, value: v)

proc newInner(l, r: TreeNode): TreeNode =
  TreeNode(kind: Inner, left: l, right: r)

proc sum(t: TreeNode): int =
  case t.kind
  of Leaf: t.value
  of Inner: sum(t.left) + sum(t.right)

let tree = newInner(
  newInner(newLeaf(1), newLeaf(2)),
  newInner(newLeaf(3), newLeaf(4))
)
echo sum(tree)  # 10
```

## Practical: State Machine with Enums

```nim
# state_machine.nim - Order processing state machine

type
  OrderState = enum
    Pending, Confirmed, Processing, Shipped, Delivered, Cancelled

  OrderEvent = enum
    Confirm, StartProcessing, Ship, Deliver, Cancel

type
  Order = object
    id: string
    state: OrderState
    history: seq[string]

proc canTransition(current: OrderState, event: OrderEvent): bool =
  case current
  of Pending:
    event in {Confirm, Cancel}
  of Confirmed:
    event in {StartProcessing, Cancel}
  of Processing:
    event in {Ship, Cancel}
  of Shipped:
    event in {Deliver}
  of Delivered, Cancelled:
    false

proc transition(o: var Order, event: OrderEvent): bool =
  if not canTransition(o.state, event):
    return false
  
  let oldState = o.state
  
  o.state = case event
    of Confirm:         Confirmed
    of StartProcessing: Processing
    of Ship:            Shipped
    of Deliver:         Delivered
    of Cancel:          Cancelled
  
  o.history.add($oldState & " -> " & $o.state)
  true

proc printOrder(o: Order) =
  import strformat
  echo &"\nOrder {o.id}:"
  echo &"  State: {o.state}"
  echo "  History:"
  for h in o.history:
    echo "    ", h

var order = Order(id: "ORD-001", state: Pending)

let events = [Confirm, StartProcessing, Ship, Deliver]
for event in events:
  if order.transition(event):
    echo "Processed: ", event
  else:
    echo "Cannot process: ", event

printOrder(order)

# Try invalid transition
var order2 = Order(id: "ORD-002", state: Pending)
discard order2.transition(Confirm)
if not order2.transition(Deliver):  # invalid!
  echo "\nCannot deliver from Confirmed state"
```

## แบบฝึกหัด Part 12

### แบบฝึกหัดที่ 1: Days of Week
```nim
type
  DayOfWeek = enum
    Monday = 1, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday

proc isWeekend(d: DayOfWeek): bool =
  d in {Saturday, Sunday}

proc isWorkday(d: DayOfWeek): bool =
  not d.isWeekend()

proc nextWorkday(d: DayOfWeek): DayOfWeek =
  var next = DayOfWeek((d.ord mod 7) + 1)
  while next.isWeekend():
    next = DayOfWeek((next.ord mod 7) + 1)
  next

for day in DayOfWeek:
  let dayType = if day.isWeekend(): "Weekend" else: "Workday"
  echo day, ": ", dayType
```

## สรุป Part 12

ในบทนี้เราได้เรียนรู้:
- ✅ Basic enums
- ✅ Enums กับ custom values (int, string)
- ✅ Enum sets (bit fields)
- ✅ Enum methods (case expressions)
- ✅ Variant objects กับ enums
- ✅ State machine pattern

---

**Previous**: [Part 11 - Tuples](part11_tuples.md)
**Next**: [Part 13 - File I/O](part13_file_io.md)
