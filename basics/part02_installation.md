# Part 02: การติดตั้งและตั้งค่า Environment

## ระบบที่รองรับ

Nim รองรับระบบปฏิบัติการหลัก:
- **Windows** 10/11 (x86, x64, ARM64)
- **macOS** 10.12+ (Intel, Apple Silicon)
- **Linux** (x86, x64, ARM, ARM64, RISC-V)
- **FreeBSD**, **OpenBSD**, **NetBSD**

## วิธีติดตั้ง Nim

### วิธีที่ 1: ใช้ choosenim (แนะนำ)

`choosenim` เป็น version manager สำหรับ Nim เหมือน `nvm` สำหรับ Node.js

#### Linux/macOS
```bash
# ดาวน์โหลดและติดตั้ง choosenim
curl https://nim-lang.org/choosenim/init.sh -sSf | sh

# เพิ่ม PATH
export PATH=$HOME/.nimble/bin:$PATH

# ตรวจสอบการติดตั้ง
nim --version
nimble --version
choosenin --version
```

#### Windows (PowerShell)
```powershell
# ดาวน์โหลด choosenim จาก GitHub releases
# https://github.com/dom96/choosenim/releases

# หรือใช้ winget
winget install nim

# หรือ Scoop
scoop install nim

# ตรวจสอบ
nim --version
```

### วิธีที่ 2: ดาวน์โหลดโดยตรง

```bash
# ดาวน์โหลด pre-built binary จาก nim-lang.org
wget https://nim-lang.org/download/nim-2.0.0-linux_x64.tar.xz
tar xf nim-2.0.0-linux_x64.tar.xz
cd nim-2.0.0

# เพิ่ม PATH
echo 'export PATH=$HOME/nim-2.0.0/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### วิธีที่ 3: Package Manager

```bash
# Ubuntu/Debian
sudo apt-get install nim

# Fedora/RHEL
sudo dnf install nim

# Arch Linux
sudo pacman -S nim

# macOS Homebrew
brew install nim

# NixOS
nix-env -i nim
```

## การจัดการเวอร์ชันด้วย choosenim

```bash
# ติดตั้งเวอร์ชันใหม่
choosenin 2.0.0
choosenin 1.6.14

# ดูเวอร์ชันที่ติดตั้ง
choosenin show

# เลือกเวอร์ชัน
choosenin 2.0.0

# อัปเดตเป็นเวอร์ชันล่าสุด
choosenin update stable
```

## Compiler Options สำคัญ

```bash
# พื้นฐาน
nim c file.nim              # Compile
nim c -r file.nim           # Compile และ Run
nim cpp file.nim            # Compile เป็น C++
nim js file.nim             # Compile เป็น JavaScript

# Optimization
nim c -d:release file.nim   # Release build (optimized)
nim c -d:debug file.nim     # Debug build
nim c -d:danger file.nim    # Maximum performance

# Output
nim c -o:output file.nim    # กำหนดชื่อ output
nim c --outdir:./bin file.nim  # กำหนด output directory

# Cross-compilation
nim c --os:windows --cpu:amd64 file.nim  # Compile สำหรับ Windows

# Threads
nim c --threads:on file.nim  # เปิด multi-threading
```

## ไฟล์ nim.cfg (Project Configuration)

```ini
# nim.cfg - ไฟล์ config สำหรับโปรเจกต์

# เพิ่ม search paths
path = "src"
path = "vendor"

# Compiler options
cc = gcc

# Thread support
threads = on

# Platform-specific settings
@if windows:
  define = "windows_target"
@end

@if linux:
  passL = "-lm"  # Link math library
@end
```

## สร้างโปรเจกต์แรก

```bash
# สร้างโปรเจกต์ด้วย nimble
mkdir myapp
cd myapp
nimble init
```

### โครงสร้างโปรเจกต์
```
myapp/
├── .gitignore
├── myapp.nimble       # Package configuration
├── src/
│   └── myapp.nim      # Main source
└── tests/
    └── test1.nim      # Tests
```

### myapp.nimble
```nimble
# Package
version       = "0.1.0"
author        = "YourName"
description   = "My first Nim application"
license       = "MIT"
srcDir        = "src"

# Binary executable
bin           = @["myapp"]

# Dependencies
requires "nim >= 2.0.0"
```

### src/myapp.nim
```nim
import std/[strformat, os, strutils]

const AppVersion = "0.1.0"
const AppName = "MyApp"

proc printWelcome() =
  echo &"Welcome to {AppName} v{AppVersion}!"
  echo "Built with Nim ", NimVersion

proc main() =
  printWelcome()
  
  let args = commandLineParams()
  
  if args.len == 0:
    echo "Usage: myapp <name>"
  else:
    let name = args[0]
    echo &"Hello, {name}!"

when isMainModule:
  main()
```

### Build และ Run
```bash
# Build
nimble build

# Run
./myapp "World"

# Build and run
nimble run -- "World"

# Run tests
nimble test
```

## Nimble Package Manager

```bash
# ค้นหา package
nimble search jester

# ติดตั้ง package
nimble install jester
nimble install norm
nimble install nimcrypto

# ดู packages ที่ติดตั้ง
nimble list --installed

# อัปเดต packages
nimble refresh
nimble upgrade
```

## การตั้งค่า CI/CD

### GitHub Actions
```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        nim: [2.0.0, 1.6.14]
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Nim
        uses: jiro4989/setup-nim-action@v1
        with:
          nim-version: ${{ matrix.nim }}
      
      - name: Install dependencies
        run: nimble install -d -y
      
      - name: Run tests
        run: nimble test
      
      - name: Build
        run: nimble build -d:release
```

## สรุป Part 2

ในบทนี้เราได้เรียนรู้:
- ✅ วิธีติดตั้ง Nim ทุกวิธี
- ✅ การใช้ choosenim จัดการเวอร์ชัน
- ✅ Compiler options สำคัญ
- ✅ การตั้งค่า IDE
- ✅ สร้างโปรเจกต์ด้วย nimble
- ✅ Package management

---

**Previous**: [Part 01 - Introduction to Nim](part01_introduction.md)
**Next**: [Part 03 - Variables และ Types](part03_variables_types.md)
