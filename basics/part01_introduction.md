# Part 01: Introduction to Nim - รู้จักภาษา Nim

## ภาษา Nim คืออะไร?

Nim เป็นภาษาโปรแกรมที่ถูกออกแบบมาเพื่อความ **ประสิทธิภาพสูง**, **ความยืดหยุ่น** และ **ความสามารถในการแสดงออก** ภาษา Nim สร้างโดย Andreas Rumpf ในปี 2008 และเปิดตัวเวอร์ชัน 1.0 ในปี 2019

### จุดเด่นของ Nim

```
┌─────────────────────────────────────────────────────────────┐
│                    Nim Language Features                      │
├─────────────────────────────────────────────────────────────┤
│  ✅ Compiled (แปลงเป็น C/C++/JavaScript)                    │
│  ✅ Statically Typed (ตรวจสอบประเภทตอน compile)            │
│  ✅ Memory Safe (ปลอดภัยจาก memory errors)                  │
│  ✅ Fast (เร็วเทียบเท่า C/C++)                              │
│  ✅ Expressive (เขียนโค้ดสั้น อ่านง่าย)                    │
│  ✅ Metaprogramming (Macros, Templates)                      │
│  ✅ Cross-platform (Windows, Linux, macOS)                   │
│  ✅ Multiple backends (C, C++, JS, WASM)                     │
└─────────────────────────────────────────────────────────────┘
```

## ทำไมต้องเรียน Nim?

### 1. ประสิทธิภาพระดับ C/C++
Nim คอมไพล์ไปเป็นภาษา C ก่อน แล้วค่อย compile เป็น native code ทำให้ได้ประสิทธิภาพสูงมาก

### 2. ไวยากรณ์สวยงาม เหมือน Python
```nim
# Nim code ที่อ่านง่ายเหมือน Python
var name = "World"
echo "Hello, " & name & "!"

# Loop แบบง่ายๆ
for i in 1..10:
  echo i
```

### 3. ระบบ Type ที่ทรงพลัง
```nim
# Type inference อัตโนมัติ
let x = 42          # ถูกอนุมานเป็น int
let y = 3.14        # ถูกอนุมานเป็น float
let s = "hello"     # ถูกอนุมานเป็น string

# หรือกำหนด type เองได้
var count: int = 0
var pi: float64 = 3.14159
```

### 4. Memory Management อัตโนมัติ
```nim
# Nim จัดการ memory ให้อัตโนมัติ
# ไม่ต้อง malloc/free เหมือน C
var list = @[1, 2, 3, 4, 5]
list.add(6)
# memory จะถูก free เองเมื่อไม่ใช้แล้ว
```

### 5. Metaprogramming อันทรงพลัง
```nim
# Macro ที่รันตอน compile time
macro repeat(n: static int, body: untyped): untyped =
  result = newStmtList()
  for i in 0..<n:
    result.add(body)

repeat(3):
  echo "Hello!"
# Output:
# Hello!
# Hello!
# Hello!
```

## ประวัติและวิวัฒนาการ

| ปี | เหตุการณ์ |
|----|-----------|
| 2005 | Andreas Rumpf เริ่มพัฒนา (ชื่อเดิม: Nimrod) |
| 2008 | เปิดตัวต่อสาธารณะครั้งแรก |
| 2014 | เปลี่ยนชื่อจาก Nimrod เป็น Nim |
| 2019 | Nim 1.0 เปิดตัวอย่างเป็นทางการ |
| 2021 | Nim 1.6 พร้อม ORC memory management |
| 2023 | Nim 2.0 เปิดตัวพร้อม breaking changes |
| 2024 | Nim 2.2 ปัจจุบัน |

## เปรียบเทียบกับภาษาอื่น

### Nim vs Python
```nim
# Python
def factorial(n):
    if n == 0:
        return 1
    return n * factorial(n-1)

# Nim - ไวยากรณ์คล้ายกันมาก แต่เร็วกว่า 10-100x
proc factorial(n: int): int =
  if n == 0:
    return 1
  return n * factorial(n-1)
```

### Nim vs C
```c
// C - verbose และเสี่ยง memory error
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char* concat(const char* a, const char* b) {
    char* result = malloc(strlen(a) + strlen(b) + 1);
    if (!result) return NULL;
    strcpy(result, a);
    strcat(result, b);
    return result;
}

int main() {
    char* s = concat("Hello, ", "World!");
    printf("%s\n", s);
    free(s);  // ต้อง free เอง!
    return 0;
}
```

```nim
# Nim - สั้นกว่า ปลอดภัยกว่า เร็วพอๆ กัน
proc concat(a, b: string): string =
  a & b

echo concat("Hello, ", "World!")
# ไม่ต้อง free! Nim จัดการให้
```

### Nim vs Rust
```rust
// Rust - ปลอดภัยมาก แต่ syntax ซับซ้อน
fn fibonacci(n: u64) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}
```

```nim
# Nim - ง่ายกว่า อ่านง่ายกว่า เร็วพอๆ กัน
proc fibonacci(n: uint64): uint64 =
  case n
  of 0: 0
  of 1: 1
  else: fibonacci(n-1) + fibonacci(n-2)
```

## Use Cases ของ Nim

### 1. Systems Programming
```nim
# เขียน System tools ประสิทธิภาพสูง
import os, strutils

# อ่านและประมวลผลไฟล์ขนาดใหญ่
proc processFile(path: string) =
  var count = 0
  for line in lines(path):
    if line.contains("error"):
      inc count
  echo "Found ", count, " errors"
```

### 2. Game Development
```nim
# Game logic ที่รันเร็ว
type
  Vector2 = object
    x, y: float32

proc `+`(a, b: Vector2): Vector2 =
  Vector2(x: a.x + b.x, y: a.y + b.y)

proc length(v: Vector2): float32 =
  sqrt(v.x * v.x + v.y * v.y)
```

### 3. Web Backend
```nim
# Web server ด้วย Jester framework
import jester

routes:
  get "/":
    resp "Hello, World!"
  
  get "/api/users":
    resp """{"users": ["Alice", "Bob"]}"""
```

### 4. Scripting & Automation
```nim
# Scripts ที่รันเร็วกว่า Python
import osproc, strutils

# รัน command และดึง output
let result = execProcess("git log --oneline -5")
for line in result.splitLines():
  echo "Commit: ", line
```

### 5. Security & Malware Research (Educational)
```nim
# Low-level programming สำหรับ security research
import winim/lean

# เรียกใช้ Windows API โดยตรง
proc getProcesses() =
  var pe32: PROCESSENTRY32
  pe32.dwSize = sizeof(PROCESSENTRY32).DWORD
  let snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0)
  if Process32First(snapshot, addr pe32):
    while Process32Next(snapshot, addr pe32):
      echo pe32.szExeFile
  CloseHandle(snapshot)
```

## โปรแกรม Hello World แรก

```nim
# hello.nim - โปรแกรมแรกของเรา

echo "สวัสดีชาวโลก!"
echo "Hello, World!"
echo "Bonjour le monde!"
echo "Hola Mundo!"

# แสดงข้อมูลพื้นฐาน
let language = "Nim"
let version = 2
echo "ยินดีต้อนรับสู่ ", language, " เวอร์ชัน ", version
```

### วิธี Compile และ Run

```bash
# บันทึกไฟล์เป็น hello.nim แล้วรัน:

# วิธีที่ 1: Compile แล้วรัน
nim compile --run hello.nim

# วิธีที่ 2: ย่อ
nim c -r hello.nim

# วิธีที่ 3: Compile อย่างเดียว
nim c hello.nim
./hello  # บน Linux/macOS
hello.exe  # บน Windows

# Compile แบบ optimized
nim c -d:release hello.nim

# Compile สำหรับ debug
nim c -d:debug hello.nim
```

## โครงสร้างโปรเจกต์ Nim

```
myproject/
├── myproject.nimble    # Package configuration
├── src/
│   ├── myproject.nim   # Main source file
│   └── modules/        # Sub-modules
│       ├── utils.nim
│       └── types.nim
├── tests/
│   ├── test_utils.nim
│   └── test_types.nim
└── README.md
```

### ไฟล์ .nimble
```nimble
# myproject.nimble
version = "0.1.0"
author = "Your Name"
description = "My awesome Nim project"
license = "MIT"

bin = @["myproject"]

# Dependencies
requires "nim >= 2.0.0"
requires "jester >= 0.5.0"
```

## Nim Compiler ทำงานอย่างไร?

```
┌─────────────────────────────────────────────────────────────┐
│                    Nim Compilation Process                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  source.nim  ──►  AST  ──►  C code  ──►  native binary       │
│                                                               │
│  1. Parser: แปลง source code เป็น AST                       │
│  2. Semantic Analysis: ตรวจสอบ types, scopes                │
│  3. Transformation: Macro expansion, optimization            │
│  4. C Code Generation: แปลง AST เป็น C                      │
│  5. C Compilation: ใช้ GCC/Clang แปลงเป็น binary            │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

## ตัวอย่างโปรแกรมแรก: Calculator

```nim
# calculator.nim - เครื่องคิดเลขง่ายๆ

import strutils, strformat

proc add(a, b: float): float = a + b
proc subtract(a, b: float): float = a - b
proc multiply(a, b: float): float = a * b
proc divide(a, b: float): float =
  if b == 0:
    raise newException(DivByZeroDefect, "Cannot divide by zero!")
  a / b

proc calculate(a, b: float, op: char): float =
  case op
  of '+': add(a, b)
  of '-': subtract(a, b)
  of '*': multiply(a, b)
  of '/': divide(a, b)
  else:
    raise newException(ValueError, &"Unknown operator: {op}")

echo "=== Nim Calculator ==="
echo ""

let pairs = [(10.0, 5.0), (100.0, 7.0), (3.14, 2.0)]
let ops = ['+', '-', '*', '/']

for (a, b) in pairs:
  for op in ops:
    try:
      let result = calculate(a, b, op)
      echo &"{a} {op} {b} = {result}"
    except DivByZeroDefect as e:
      echo &"Error: {e.msg}"
```

## ตัวอย่างโปรแกรมที่ 2: FizzBuzz

```nim
# fizzbuzz.nim

proc fizzBuzz(n: int): string =
  if n mod 15 == 0: "FizzBuzz"
  elif n mod 3 == 0: "Fizz"
  elif n mod 5 == 0: "Buzz"
  else: $n

for i in 1..100:
  echo fizzBuzz(i)
```

## สรุป Part 1

ในบทนี้เราได้เรียนรู้:
- ✅ Nim คืออะไร และทำไมต้องเรียน
- ✅ ประวัติและวิวัฒนาการของ Nim
- ✅ จุดเด่นของ Nim เมื่อเทียบกับภาษาอื่น
- ✅ Use cases ของ Nim
- ✅ วิธี compile และ run โปรแกรม
- ✅ โครงสร้างโปรเจกต์เบื้องต้น
- ✅ Standard Library overview

---

**Next**: [Part 02 - การติดตั้งและตั้งค่า Environment](part02_installation.md)
