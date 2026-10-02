# Part 16: OOP - Object-Oriented Programming ใน Nim

## Objects และ Methods

```nim
# Object types
type
  Animal = ref object of RootObj
    name: string
    sound: string
    legs: int

  Dog = ref object of Animal
    breed: string

  Cat = ref object of Animal
    indoor: bool

# Constructor
proc newAnimal*(name, sound: string, legs: int): Animal =
  Animal(name: name, sound: sound, legs: legs)

proc newDog*(name, breed: string): Dog =
  Dog(name: name, sound: "Woof", legs: 4, breed: breed)

proc newCat*(name: string, indoor: bool = true): Cat =
  Cat(name: name, sound: "Meow", legs: 4, indoor: indoor)

# Methods
proc speak*(a: Animal): string =
  a.name & " says: " & a.sound

proc info*(a: Animal): string =
  import strformat
  &"Animal: {a.name} ({a.legs} legs)"

proc info*(d: Dog): string =
  import strformat
  &"Dog: {d.name}, Breed: {d.breed}"

proc info*(c: Cat): string =
  import strformat
  let location = if c.indoor: "indoor" else: "outdoor"
  &"Cat: {c.name} ({location})"

# Test
let a = newAnimal("Generic", "...", 4)
let d = newDog("Rex", "Labrador")
let c = newCat("Whiskers")

echo a.speak()  # Generic says: ...
echo d.speak()  # Rex says: Woof
echo c.speak()  # Whiskers says: Meow

echo a.info()   # Animal: Generic (4 legs)
echo d.info()   # Dog: Rex, Breed: Labrador
echo c.info()   # Cat: Whiskers (indoor)
```

## Dynamic Dispatch (Polymorphism)

```nim
# method สำหรับ dynamic dispatch
type Shape = ref object of RootObj

method area*(s: Shape): float {.base.} =
  0.0

method perimeter*(s: Shape): float {.base.} =
  0.0

method describe*(s: Shape): string {.base.} =
  "Unknown shape"

type Circle = ref object of Shape
  radius: float

method area*(c: Circle): float =
  import math
  PI * c.radius * c.radius

method perimeter*(c: Circle): float =
  import math
  2 * PI * c.radius

method describe*(c: Circle): string =
  import strformat
  &"Circle(r={c.radius:.1f})"

type Rectangle = ref object of Shape
  width, height: float

method area*(r: Rectangle): float =
  r.width * r.height

method perimeter*(r: Rectangle): float =
  2 * (r.width + r.height)

method describe*(r: Rectangle): string =
  import strformat
  &"Rect({r.width:.1f}x{r.height:.1f})"

type Triangle = ref object of Shape
  a, b, c: float

method area*(t: Triangle): float =
  import math
  let s = (t.a + t.b + t.c) / 2.0
  sqrt(s * (s-t.a) * (s-t.b) * (s-t.c))

method perimeter*(t: Triangle): float =
  t.a + t.b + t.c

method describe*(t: Triangle): string =
  import strformat
  &"Triangle({t.a:.1f}, {t.b:.1f}, {t.c:.1f})"

# Polymorphism in action
let shapes: seq[Shape] = @[
  Circle(radius: 5.0),
  Rectangle(width: 4.0, height: 6.0),
  Triangle(a: 3.0, b: 4.0, c: 5.0)
]

echo "=== Shapes ==="
for s in shapes:
  echo s.describe()
  echo &"  Area: {s.area():.2f}"
  echo &"  Perimeter: {s.perimeter():.2f}"

# Type checking
for s in shapes:
  if s of Circle:
    echo "Found a circle with radius: ", Circle(s).radius
  elif s of Rectangle:
    let r = Rectangle(s)
    echo "Found a rectangle: ", r.width, "x", r.height
```

## Inheritance Chain

```nim
# Multi-level inheritance
type
  Vehicle = ref object of RootObj
    make: string
    model: string
    year: int
    speed: float

  Car = ref object of Vehicle
    doors: int
    transmission: string

  ElectricCar = ref object of Car
    batteryCapacity: float
    range: float

  Truck = ref object of Vehicle
    payload: float
    axles: int

# Base methods
proc info*(v: Vehicle): string =
  import strformat
  &"{v.year} {v.make} {v.model}"

proc accelerate*(v: var Vehicle, amount: float) =
  v.speed += amount
  echo v.info(), " now at ", v.speed, " km/h"

# Car methods
proc info*(c: Car): string =
  import strformat
  &"{procCall Vehicle(c).info()} ({c.doors}-door, {c.transmission})"

# ElectricCar methods
proc info*(e: ElectricCar): string =
  import strformat
  &"{procCall Car(e).info()}, Battery: {e.batteryCapacity}kWh, Range: {e.range}km"

proc charge*(e: ElectricCar) =
  echo "Charging ", e.info()

# Truck methods
proc info*(t: Truck): string =
  import strformat
  &"{procCall Vehicle(t).info()}, Payload: {t.payload}t"

# Test
let tesla = ElectricCar(
  make: "Tesla", model: "Model 3", year: 2024,
  doors: 4, transmission: "Automatic",
  batteryCapacity: 75.0, range: 500.0
)

let volvo = Truck(
  make: "Volvo", model: "FH", year: 2023,
  payload: 25.0, axles: 3
)

echo tesla.info()
echo volvo.info()

# Type checking
proc processVehicle(v: Vehicle) =
  echo "Processing: ", v.info()
  if v of ElectricCar:
    ElectricCar(v).charge()
  elif v of Car:
    echo "It's a car"
  elif v of Truck:
    echo "It's a truck"
```

## Mixins (Composition over Inheritance)

```nim
# Mixin pattern ใน Nim (ไม่มี mixin โดยตรง แต่ทำได้ด้วย templates)

type
  Printable = concept x
    `$`(x) is string

type Serializable = concept x
  x.serialize() is string
  # deserialize isn't checked here

# Implementing mixins with templates
template withLogging*(T: typedesc): untyped =
  var log: seq[string] = @[]
  
  proc addLog(obj: T, msg: string) =
    log.add(msg)
  
  proc getLogs(obj: T): seq[string] =
    log

# Composition pattern (better than deep inheritance)
type
  Logger = object
    entries: seq[string]

  Cache[T] = object
    store: seq[(string, T)]
    maxSize: int

  UserService = object
    logger: Logger
    cache: Cache[string]

proc log(l: var Logger, msg: string) =
  l.entries.add(msg)

proc get[T](c: var Cache[T], key: string): Option[T] =
  import options
  for (k, v) in c.store:
    if k == key: return some(v)
  none(T)

proc set[T](c: var Cache[T], key: string, value: T) =
  if c.store.len >= c.maxSize:
    c.store.del(0)  # evict oldest
  c.store.add((key, value))

proc getUser(svc: var UserService, id: string): string =
  svc.logger.log("Getting user: " & id)
  
  let cached = svc.cache.get(id)
  if cached.isSome:
    svc.logger.log("Cache hit for: " & id)
    return cached.get()
  
  # Simulate DB fetch
  let user = "User(" & id & ")"
  svc.cache.set(id, user)
  svc.logger.log("DB fetch for: " & id)
  user

var svc = UserService(
  logger: Logger(entries: @[]),
  cache: Cache[string](store: @[], maxSize: 10)
)

echo svc.getUser("1")
echo svc.getUser("2")
echo svc.getUser("1")  # cache hit
echo "Logs: ", svc.logger.entries
```

## Practical: Game Entities

```nim
# game_entities.nim - OOP game entities

import math, strformat

type
  Vector2 = object
    x, y: float

  Entity = ref object of RootObj
    id: int
    pos: Vector2
    vel: Vector2
    alive: bool

  Player = ref object of Entity
    health: int
    maxHealth: int
    score: int
    name: string

  Enemy = ref object of Entity
    health: int
    damage: int
    enemyType: string

  Bullet = ref object of Entity
    damage: int
    owner: string

var nextEntityId = 1

proc newEntity*(x, y: float): Entity =
  result = Entity(id: nextEntityId, pos: Vector2(x: x, y: y), 
                  vel: Vector2(x: 0, y: 0), alive: true)
  inc nextEntityId

proc newPlayer*(name: string, x, y: float): Player =
  result = Player(id: nextEntityId, pos: Vector2(x: x, y: y),
                  vel: Vector2(x: 0, y: 0), alive: true,
                  health: 100, maxHealth: 100, score: 0, name: name)
  inc nextEntityId

proc newEnemy*(kind: string, x, y: float): Enemy =
  let (hp, dmg) = case kind
    of "weak":   (20, 5)
    of "normal": (50, 10)
    of "strong": (100, 20)
    else:        (30, 8)
  
  result = Enemy(id: nextEntityId, pos: Vector2(x: x, y: y),
                 vel: Vector2(x: 0, y: 0), alive: true,
                 health: hp, damage: dmg, enemyType: kind)
  inc nextEntityId

proc distance(a, b: Vector2): float =
  let dx = a.x - b.x
  let dy = a.y - b.y
  sqrt(dx*dx + dy*dy)

method update*(e: Entity, dt: float) {.base.} =
  e.pos.x += e.vel.x * dt
  e.pos.y += e.vel.y * dt

method update*(p: Player, dt: float) =
  procCall Entity(p).update(dt)
  # Player-specific update logic

method update*(e: Enemy, dt: float) =
  procCall Entity(e).update(dt)
  # Enemy AI could go here

method takeDamage*(e: Entity, damage: int) {.base.} =
  discard  # base entities don't take damage

method takeDamage*(p: Player, damage: int) =
  p.health = max(0, p.health - damage)
  if p.health == 0:
    p.alive = false
    echo &"{p.name} has died!"

method takeDamage*(e: Enemy, damage: int) =
  e.health = max(0, e.health - damage)
  if e.health == 0:
    e.alive = false

method describe*(e: Entity): string {.base.} =
  &"Entity#{e.id} at ({e.pos.x:.0f},{e.pos.y:.0f})"

method describe*(p: Player): string =
  &"Player({p.name}) HP:{p.health}/{p.maxHealth} Score:{p.score}"

method describe*(e: Enemy): string =
  &"Enemy({e.enemyType}) HP:{e.health}"

# Game simulation
var entities: seq[Entity] = @[]

let player = newPlayer("Hero", 0.0, 0.0)
entities.add(player)
entities.add(newEnemy("weak", 10.0, 5.0))
entities.add(newEnemy("normal", -5.0, 15.0))
entities.add(newEnemy("strong", 20.0, 0.0))

echo "=== Game State ==="
for e in entities:
  echo e.describe()

echo "\n=== Combat ==="
for e in entities:
  if e of Enemy:
    let enemy = Enemy(e)
    let dist = distance(player.pos, enemy.pos)
    echo &"\nFighting {enemy.describe()} (distance: {dist:.1f})"
    player.takeDamage(enemy.damage)
    enemy.takeDamage(30)
    echo "After: ", player.describe()
    echo "Enemy: ", if enemy.alive: enemy.describe() else: "DEAD"
```

## แบบฝึกหัด Part 16

### แบบฝึกหัดที่ 1: Bank Account Hierarchy
```nim
type
  Account = ref object of RootObj
    owner: string
    balance: float

  SavingsAccount = ref object of Account
    interestRate: float

  CheckingAccount = ref object of Account
    overdraftLimit: float

method deposit*(a: Account, amount: float) {.base.} =
  a.balance += amount
  echo a.owner, " deposited ", amount

method withdraw*(a: Account, amount: float): bool {.base.} =
  if amount > a.balance:
    echo "Insufficient funds"
    return false
  a.balance -= amount
  echo a.owner, " withdrew ", amount
  true

method withdraw*(c: CheckingAccount, amount: float): bool =
  if amount > c.balance + c.overdraftLimit:
    echo "Exceeds overdraft limit"
    return false
  c.balance -= amount
  true

method applyInterest*(s: SavingsAccount) =
  let interest = s.balance * s.interestRate
  s.balance += interest
  echo "Interest applied: ", interest

# Test
let savings = SavingsAccount(owner: "Alice", balance: 1000.0, interestRate: 0.05)
let checking = CheckingAccount(owner: "Bob", balance: 500.0, overdraftLimit: 200.0)

savings.deposit(500.0)
savings.applyInterest()
echo "Balance: ", savings.balance

discard checking.withdraw(600.0)  # uses overdraft
echo "Checking balance: ", checking.balance
```

## สรุป Part 16

ในบทนี้เราได้เรียนรู้:
- ✅ Object inheritance ด้วย `ref object of`
- ✅ Dynamic dispatch ด้วย `method`
- ✅ `{.base.}` pragma
- ✅ `procCall` สำหรับเรียก parent method
- ✅ Type checking: `of`, casting
- ✅ Composition over inheritance
- ✅ Practical: Game entity system

---

**Previous**: [Part 15 - Modules](../basics/part15_modules.md)
**Next**: [Part 17 - Generics](part17_generics.md)
