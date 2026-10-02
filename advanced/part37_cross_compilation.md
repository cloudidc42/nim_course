# Part 37: Cross-Compilation และการ Build สำหรับหลาย Platform

## Cross-Compilation คืออะไร

```
Cross-compilation = เขียนโค้ดบน platform A แล้ว compile เป็น binary สำหรับ platform B

Nim รองรับ:
- Linux  -> Windows (x64)
- Linux  -> macOS   (x64, arm64)
- Linux  -> ARM/RISC-V (embedded)
- macOS  -> Windows
- Any    -> WebAssembly
- Any    -> JavaScript/Node.js
```

## การติดตั้ง Cross-Compiler

```bash
# Ubuntu/Debian: ติดตั้ง mingw-w64 สำหรับ Linux -> Windows
sudo apt install mingw-w64

# macOS:
brew install mingw-w64

# Verify installation
x86_64-w64-mingw32-gcc --version
i686-w64-mingw32-gcc --version

# ARM cross-compiler
sudo apt install gcc-aarch64-linux-gnu

# RISC-V
sudo apt install gcc-riscv64-linux-gnu
```

## Compile สำหรับ Windows จาก Linux

```bash
# Compile Nim for Windows x64
nim c \
  --cpu:amd64 \
  --os:windows \
  --cc:gcc \
  --passC:"-target x86_64-w64-mingw32" \
  --passL:"-target x86_64-w64-mingw32" \
  --gcc.exe:x86_64-w64-mingw32-gcc \
  --gcc.linkerexe:x86_64-w64-mingw32-gcc \
  -o:output.exe \
  myapp.nim

# Shorthand with nimcross tool
# nimble install nimcross
nimcross windows myapp.nim

# Build release Windows binary
nim c \
  -d:release \
  --cpu:amd64 \
  --os:windows \
  --cc:gcc \
  --gcc.exe:x86_64-w64-mingw32-gcc \
  --gcc.linkerexe:x86_64-w64-mingw32-gcc \
  -o:myapp.exe \
  myapp.nim

# Build 32-bit Windows
nim c \
  --cpu:i386 \
  --os:windows \
  --gcc.exe:i686-w64-mingw32-gcc \
  --gcc.linkerexe:i686-w64-mingw32-gcc \
  -o:myapp32.exe \
  myapp.nim
```

## Platform Detection ในโค้ด

```nim
# platform_detect.nim
# เขียนโค้ดที่ทำงานได้บนทุก platform

when defined(windows):
  echo "Running on Windows"
  # Windows-specific code
elif defined(linux):
  echo "Running on Linux"
  # Linux-specific code
elif defined(macosx):
  echo "Running on macOS"
else:
  echo "Unknown platform"

# Architecture
when defined(amd64) or defined(x86_64):
  echo "64-bit x86"
elif defined(i386):
  echo "32-bit x86"
elif defined(arm64) or defined(aarch64):
  echo "ARM 64-bit"
elif defined(arm):
  echo "ARM 32-bit"

# Cross-platform file paths
when defined(windows):
  const pathSep = '\\'
  const exeSuffix = ".exe"
else:
  const pathSep = '/'
  const exeSuffix = ""

proc joinPath(parts: varargs[string]): string =
  parts.join($pathSep)

# Cross-platform process management
proc runCommand(cmd: string): int =
  when defined(windows):
    import winlean
    # Use WinAPI
    0
  else:
    import posix
    WEXITSTATUS(system(cmd.cstring))

# Platform-specific constants
const
  maxPath* = when defined(windows): 260
              elif defined(macosx): 1024
              else: 4096
  
  pathDelim* = when defined(windows): ';'
                else: ':'
```

## Compile สำหรับ ARM

```bash
# Compile for ARM64 (Raspberry Pi 4, Apple Silicon, etc.)
nim c \
  --cpu:arm64 \
  --os:linux \
  --cc:gcc \
  --gcc.exe:aarch64-linux-gnu-gcc \
  --gcc.linkerexe:aarch64-linux-gnu-gcc \
  -o:myapp_arm64 \
  myapp.nim

# Test with QEMU
qemu-aarch64-static ./myapp_arm64

# ARM32 (Raspberry Pi 2/3)
nim c \
  --cpu:arm \
  --os:linux \
  --gcc.exe:arm-linux-gnueabihf-gcc \
  --gcc.linkerexe:arm-linux-gnueabihf-gcc \
  -o:myapp_arm32 \
  myapp.nim

# RISC-V 64
nim c \
  --cpu:riscv64 \
  --os:linux \
  --gcc.exe:riscv64-linux-gnu-gcc \
  --gcc.linkerexe:riscv64-linux-gnu-gcc \
  -o:myapp_riscv64 \
  myapp.nim
```

## คอมไพล์เป็น JavaScript / WebAssembly

```bash
# Compile to JavaScript (Node.js/browser)
nim js -o:myapp.js myapp.nim

# Run with Node.js
node myapp.js

# Compile to WebAssembly
# ต้อง install emscripten ก่อน
nimble install nimc-wasm
nim c \
  --cc:clang \
  --os:emscripten \
  --cpu:wasm32 \
  --passC:"-Os" \
  -o:myapp.js \
  myapp.nim
```

## Cross-platform Nim Code Examples

```nim
# cross_platform.nim
# สร้าง library ที่ cross-platform

import os, strutils

# Cross-platform home directory
proc getHomeDir*(): string =
  when defined(windows):
    let userProfile = getEnv("USERPROFILE", "")
    if userProfile.len > 0: return userProfile
    return getEnv("HOMEDRIVE", "") & getEnv("HOMEPATH", "")
  else:
    return getEnv("HOME", expandTilde("~"))

# Cross-platform config directory
proc getConfigDir*(appName: string): string =
  when defined(windows):
    let appData = getEnv("APPDATA", getHomeDir())
    result = appData / appName
  elif defined(macosx):
    result = getHomeDir() / "Library" / "Application Support" / appName
  else:
    let xdgConfig = getEnv("XDG_CONFIG_HOME", getHomeDir() / ".config")
    result = xdgConfig / appName
  createDir(result)

# Cross-platform temp directory
proc getTempDir*(): string =
  when defined(windows):
    getEnv("TEMP", getEnv("TMP", "C:\\Temp"))
  else:
    getEnv("TMPDIR", "/tmp")

# Test
echo "Home: ", getHomeDir()
echo "Config: ", getConfigDir("MyApp")
echo "Temp: ", getTempDir()

# Cross-platform shared library
when defined(windows):
  const libExt = ".dll"
elif defined(macosx):
  const libExt = ".dylib"
else:
  const libExt = ".so"

const libName = "mylib" & libExt
echo "Looking for: ", libName

# Cross-platform thread-safe initialization
import locks

var gLock: Lock
initLock(gLock)

proc threadSafeInit() =
  withLock(gLock):
    # Initialize global state
    discard
```

## nimble.lock และการสร้าง Reproducible Build

```bash
# nimble.lock ensures same deps across machines
nimble lock        # create lock file
nimble install     # install from lock file

# .nimble project file for cross-compilation
# myapp.nimble:
# 
task buildWindows, "Build for Windows":
  exec "nim c --os:windows --cpu:amd64 " &
       "--gcc.exe:x86_64-w64-mingw32-gcc " &
       "--gcc.linkerexe:x86_64-w64-mingw32-gcc " &
       "-d:release -o:bin/myapp.exe src/myapp.nim"

task buildLinux, "Build for Linux":
  exec "nim c -d:release -o:bin/myapp src/myapp.nim"

task buildAll, "Build all platforms":
  exec "nimble buildWindows"
  exec "nimble buildLinux"

# Run with: nimble buildWindows
```

## สรุป Part 37

ในบทนี้เราได้เรียนรู้:
- ✅ การติดตั้ง cross-compiler
- ✅ Compile Linux -> Windows
- ✅ Compile สำหรับ ARM/RISC-V
- ␅ Compile เป็น JavaScript/WebAssembly
- ✅ Cross-platform code patterns
- ✅ Reproducible builds ด้วย nimble.lock

---

**Previous**: [Part 36 - Direct Syscalls](../security/part36_direct_syscalls.md)
**Next**: [Part 38 - Advanced Macros](part38_advanced_macros.md)
