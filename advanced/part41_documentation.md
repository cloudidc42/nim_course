# Part 41: Documentation และ API Reference

## nimdoc - Nim's Documentation Generator

```bash
# สร้าง docs จาก source code
nim doc mymodule.nim              # single file
nim doc --project --index:on src/   # entire project
nim doc --outdir:docs src/main.nim  # custom output dir

# Options
--docInternal       # include private symbols (##:)
--index:on          # create search index
--git.url:url       # link to source on GitHub
--git.commit:hash   # link to specific commit

# nimdoc generates:
# - HTML files with syntax highlighting
# - Search index
# - module graph
```

## Doc Comments

```nim
# documented_module.nim

## โมดูลย์สำหรับจัดการงานเกี่ยวกับ ผู้ใช้
## 
## Example:
## ```nim
## import documented_module
## let u = newUser("Alice", "alice@example.com")
## echo u.name
## ```

type
  User* = object
    ## เอนติตี้ผู้ใช้ในระบบ
    name*: string      ## ชื่อผู้ใช้ (required)
    email*: string     ## email (unique)
    age*: int          ## อายุ (18+)

proc newUser*(name, email: string, age: int = 0): User =
  ## สร้าง User object ใหม่
  ##
  ## Parameters:
  ##   name  - ชื่อผู้ใช้ (1-50 ตัวอักษร)
  ##   email - อีเมล ต้อง unique ในระบบ
  ##   age   - อายุ (default: 0 = unknown)
  ##
  ## Returns: User object
  ##
  ## Raises:
  ##   ValueError: ถ้า name ว่าง
  ##
  ## Example:
  ## ```nim
  ## let u = newUser("Alice", "alice@example.com", 30)
  ## assert u.name == "Alice"
  ## ```
  if name.len == 0:
    raise newException(ValueError, "Name cannot be empty")
  User(name: name, email: email, age: age)

proc greet*(user: User): string =
  ## สร้าง greeting message
  runnableExamples:
    let u = newUser("Bob", "bob@mail.com")
    assert greet(u) == "Hello, Bob!"
  
  "Hello, " & user.name & "!"

proc `$`*(user: User): string =
  ## String representation
  user.name & " <" & user.email & ">"

# Private proc (ไม่อยู่ใน public docs)
proc validate(user: User): bool =
  ## :private:
  user.name.len > 0 and '@' in user.email
```

## runnableExamples

```nim
# ตัวอย่างที่เรียกใช้ได้จริงและเป็นส่วนหนึ่งของ docs

proc sortSeq*(s: seq[int]): seq[int] =
  ## Sort a sequence in ascending order
  runnableExamples:
    let s = @[3, 1, 4, 1, 5, 9, 2, 6]
    let sorted = sortSeq(s)
    assert sorted == @[1, 1, 2, 3, 4, 5, 6, 9]
    assert sortSeq(@[]) == @[]
    assert sortSeq(@[1]) == @[1]
  
  import algorithm
  s.sorted()

# nimdoc will:
# 1. Extract runnableExamples
# 2. Compile and run them
# 3. Display in documentation
# 4. Fail doc generation if examples fail
```

## รูปแบบ โปรเจคต์ (docs ครบถ้วน)

```nim
# src/mylib.nim - main library module

## mylib: ไลบรารีสำหรับ...
##
## Getting Started:
## ```nim
## import mylib
## ```

when isMainModule:
  # Test code here won't appear in docs
  discard
```

```bash
# nimble.nimble tasks for docs
task docs, "Generate and serve docs":
  exec "nim doc --project --index:on --outdir:docs src/mylib.nim"
  exec "cd docs && python3 -m http.server 8000"

task doctest, "Run doc examples":
  exec "nim doc --project src/mylib.nim"  # compiles examples
```

## ลายลักษณ์การเขียน Docs ที่ดี

```nim
# best_practice.nim

## โมดูลย์สำหรับจัดการ collection
##
## สรุปการเปลี่ยนแปลงระหว่างเวอร์ชัน:
## - 1.0.0: เริ่มต้น
## - 1.1.0: เพิ่ม filter/map
## - 2.0.0: Breaking change: เปลี่ยน API

type
  Stack*[T] = object
    ## Stack (LIFO) data structure.
    ##
    ## Type parameters:
    ##   T - element type
    ##
    ## Thread safety: **NOT thread-safe**.
    ## Use `Mutex` if sharing between threads.
    items: seq[T]

proc newStack*[T](): Stack[T] =
  ## Create empty stack
  Stack[T](items: @[])

proc push*[T](s: var Stack[T], item: T) =
  ## Push item onto top of stack.
  ##
  ## Complexity: O(1) amortized
  ##
  ## Example:
  ## ```nim
  ## var s = newStack[int]()
  ## s.push(1)
  ## s.push(2)
  ## assert s.peek() == 2
  ## ```
  s.items.add(item)

proc pop*[T](s: var Stack[T]): T =
  ## Remove and return top item.
  ##
  ## Raises: IndexDefect if stack is empty
  if s.items.len == 0:
    raise newException(IndexDefect, "Stack is empty")
  s.items.pop()

proc peek*[T](s: Stack[T]): T =
  ## Return top item without removing it.
  if s.items.len == 0:
    raise newException(IndexDefect, "Stack is empty")
  s.items[^1]

proc isEmpty*[T](s: Stack[T]): bool =
  ## Return true if stack has no items
  s.items.len == 0

proc len*[T](s: Stack[T]): int =
  ## Return number of items in stack
  s.items.len
```

## สรุป Part 41

ในบทนี้เราได้เรียนรู้:
- ␅ nimdoc และการสร้าง API docs
- ␅ doc comments syntax (##)
- ␅ runnableExamples
- ␅ Project documentation layout
- ␅ Best practices สำหรับ doc comments

---

**Previous**: [Part 40 - DSL Design](part40_dsl_design.md)
**Next**: [Part 42 - Tooling](part42_tooling.md)
