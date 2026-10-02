# Part 53: Reverse Engineering Tools ใน Nim

## Binary Analysis

```nim
# binary_analyzer.nim
# วิเคราะห์ binary files

import os, strutils, strformat, tables, sequtils

type
  BinaryInfo = object
    path: string
    size: int
    magic: string
    fileType: string
    entropy: float
    strings: seq[string]
    isPacked: bool
    isSigned: bool
    suspicious: seq[string]

proc calcEntropy(data: string): float =
  import math
  if data.len == 0: return 0.0
  var freq: array[256, int]
  for c in data: inc freq[c.int]
  for f in freq:
    if f > 0:
      let p = float(f) / float(data.len)
      result -= p * log2(p)

proc detectFileType(header: string): string =
  if header.len < 4: return "unknown"
  let b = header
  
  if b[0] == 'M' and b[1] == 'Z': return "PE (Windows)"
  if b[0] == '\x7F' and b[1..3] == "ELF": return "ELF (Linux)"
  if b[0..3] == "\xCA\xFE\xBA\xBE": return "Mach-O FAT"
  if b[0..3] == "\xCF\xFA\xED\xFE": return "Mach-O x64"
  if b[0..1] == "PK": return "ZIP archive"
  if b[0..2] == "GIF": return "GIF image"
  if b[0..3] == "\x89PNG": return "PNG image"
  if b[0..1] == "\xFF\xD8": return "JPEG image"
  if b[0..3] == "%PDF": return "PDF document"
  return "unknown"

proc extractStrings(data: string, minLen: int = 6): seq[string] =
  result = @[]
  var current = ""
  let printable = {32.chr..126.chr}
  for c in data:
    if c in printable:
      current &= c
    else:
      if current.len >= minLen:
        result.add(current)
      current = ""
  if current.len >= minLen:
    result.add(current)

proc findSuspiciousStrings(strings: seq[string]): seq[string] =
  result = @[]
  let suspicious = @[
    # Network
    "CreateSocket", "WSAStartup", "connect", "send", "recv",
    "InternetOpen", "HttpSendRequest",
    # Process
    "CreateRemoteThread", "VirtualAllocEx", "WriteProcessMemory",
    "NtAllocateVirtualMemory", "NtWriteVirtualMemory",
    # Anti-analysis
    "IsDebuggerPresent", "CheckRemoteDebuggerPresent",
    "NtQueryInformationProcess", "GetTickCount",
    # Persistence
    "RegSetValueEx", "CreateService", "schtasks",
    # Shell
    "cmd.exe", "powershell", "wscript", "cscript"
  ]
  
  for s in strings:
    for suspect in suspicious:
      if suspect.toLowerAscii() in s.toLowerAscii():
        result.add(s)
        break

proc analyzeBinary(path: string): BinaryInfo =
  result.path = path
  if not fileExists(path): return
  
  let data = readFile(path)
  result.size = data.len
  result.magic = data[0..<min(16, data.len)].toHex()
  result.fileType = detectFileType(data)
  result.entropy = calcEntropy(data)
  result.strings = extractStrings(data)
  result.isPacked = result.entropy > 7.0  # High entropy = packed/encrypted
  result.suspicious = findSuspiciousStrings(result.strings)

proc printReport(info: BinaryInfo) =
  echo &"=== Binary Analysis: {info.path.extractFilename()} ==="
  echo &"Size: {info.size} bytes"
  echo &"Type: {info.fileType}"
  echo &"Magic: {info.magic}"
  echo &"Entropy: {info.entropy:.2f} {'(PACKED/ENCRYPTED)' if info.isPacked else ''}"
  echo &"Strings extracted: {info.strings.len}"
  
  if info.suspicious.len > 0:
    echo &"\n[!] Suspicious strings ({info.suspicious.len}):"
    for s in info.suspicious[0..<min(10, info.suspicious.len)]:
      echo &"    {s}"
  
  echo &"\nAll strings ({min(20, info.strings.len)} of {info.strings.len}):"
  for s in info.strings[0..<min(20, info.strings.len)]:
    echo &"  {s}"

if paramCount() > 0:
  let info = analyzeBinary(paramStr(1))
  printReport(info)
else:
  # Demo with self
  let info = analyzeBinary(getAppFilename())
  printReport(info)
```

## Disassembler (x86/x64)

```nim
# mini_disasm.nim
# แยก x86/x64 opcodes เบื้องต้น

import strutils, strformat, tables

# Simplified instruction decoder (concept)
# Real: use libcapstone via FFI

type
  Instruction = object
    offset: uint64
    bytes: seq[byte]
    mnemonic: string
    operands: string

# Common x86/x64 one-byte opcodes
const simpleOpcodes = {
  0x90.byte: "nop",
  0xC3.byte: "ret",
  0xCC.byte: "int3",
  0x50.byte: "push rax",
  0x51.byte: "push rcx",
  0x52.byte: "push rdx",
  0x53.byte: "push rbx",
  0x58.byte: "pop rax",
  0x59.byte: "pop rcx",
  0x5A.byte: "pop rdx",
  0x5B.byte: "pop rbx",
  0x48.byte: "REX.W",  # prefix
}.toTable()

proc disassembleSimple(bytes: seq[byte], baseAddr: uint64 = 0): seq[Instruction] =
  result = @[]
  var i = 0
  
  while i < bytes.len:
    let b = bytes[i]
    var instr = Instruction(offset: baseAddr + i.uint64)
    
    case b:
    of 0x90:  # NOP
      instr = Instruction(offset: baseAddr + i.uint64, bytes: @[b],
                           mnemonic: "nop", operands: "")
      inc i
    
    of 0xE9:  # JMP rel32
      if i + 4 < bytes.len:
        let rel = cast[int32](bytes[i+1] or (bytes[i+2] shl 8) or
                              (bytes[i+3] shl 16) or (bytes[i+4] shl 24))
        let target = (baseAddr + i.uint64 + 5).int64 + rel
        instr = Instruction(
          offset: baseAddr + i.uint64,
          bytes: bytes[i..i+4],
          mnemonic: "jmp",
          operands: &"0x{target:X}"
        )
        i += 5
      else: inc i
    
    of 0xE8:  # CALL rel32
      if i + 4 < bytes.len:
        let rel = cast[int32](bytes[i+1] or (bytes[i+2] shl 8) or
                              (bytes[i+3] shl 16) or (bytes[i+4] shl 24))
        let target = (baseAddr + i.uint64 + 5).int64 + rel
        instr = Instruction(
          offset: baseAddr + i.uint64,
          bytes: bytes[i..i+4],
          mnemonic: "call",
          operands: &"0x{target:X}"
        )
        i += 5
      else: inc i
    
    of 0xC3:  # RET
      instr = Instruction(offset: baseAddr + i.uint64, bytes: @[b],
                           mnemonic: "ret", operands: "")
      inc i
    
    of 0xB8..0xBF:  # MOV reg, imm32
      let regs = ["rax", "rcx", "rdx", "rbx", "rsp", "rbp", "rsi", "rdi"]
      let reg = regs[b - 0xB8]
      if i + 4 < bytes.len:
        let imm = bytes[i+1].uint32 or (bytes[i+2].uint32 shl 8) or
                  (bytes[i+3].uint32 shl 16) or (bytes[i+4].uint32 shl 24)
        instr = Instruction(
          offset: baseAddr + i.uint64,
          bytes: bytes[i..i+4],
          mnemonic: "mov",
          operands: &"{reg}, 0x{imm:X}"
        )
        i += 5
      else: inc i
    
    else:
      instr = Instruction(
        offset: baseAddr + i.uint64,
        bytes: @[b],
        mnemonic: &"db 0x{b:02X}",
        operands: ""
      )
      inc i
    
    result.add(instr)

proc printDisasm(instrs: seq[Instruction]) =
  for instr in instrs:
    let bytesHex = instr.bytes.mapIt(toHex(it.int, 2)).join(" ")
    echo &"  0x{instr.offset:08X}  {bytesHex:<20} {instr.mnemonic} {instr.operands}"

# Example
let shellcode: seq[byte] = @[
  0x48.byte, 0x31.byte, 0xC9.byte,  # (REX.W) xor ...
  0x48.byte, 0xFF.byte, 0xC1.byte,  # (REX.W) inc rcx  
  0xB8.byte, 0x3C.byte, 0x00.byte, 0x00.byte, 0x00.byte,  # mov eax, 0x3C
  0x0F.byte, 0x05.byte,             # syscall
  0xC3.byte                         # ret
]

echo "Disassembly:"
printDisasm(disassembleSimple(shellcode, 0x1000))

echo "\nNote: Use libcapstone (via FFI) for full disassembly"
```

## การใช้ Capstone via FFI

```nim
# capstone_wrapper.nim
# Capstone disassembly engine wrapper
# ติดตั้ง: nimble install capstone (or use system libcapstone)

{.passL: "-lcapstone".}

const
  CS_ARCH_X86 = 3
  CS_MODE_64 = 1 shl 3
  CS_OPT_SYNTAX_INTEL = 1

type
  csh = uint
  cs_insn {.pure.} = object
    id: cuint
    address: uint64
    size: uint16
    bytes: array[16, uint8]
    mnemonic: array[32, char]
    op_str: array[160, char]

proc cs_open*(arch, mode: cint, handle: ptr csh): cint
  {.importc, dynlib: "libcapstone.so".}

proc cs_disasm*(handle: csh, code: ptr uint8, code_size: uint,
                address: uint64, count: uint,
                insn: ptr ptr cs_insn): uint
  {.importc, dynlib: "libcapstone.so".}

proc cs_free*(insn: ptr cs_insn, count: uint)
  {.importc, dynlib: "libcapstone.so".}

proc cs_close*(handle: ptr csh): cint
  {.importc, dynlib: "libcapstone.so".}

proc disassemble(code: seq[byte], baseAddr: uint64 = 0) =
  var handle: csh
  if cs_open(CS_ARCH_X86, CS_MODE_64, addr handle) != 0:
    echo "Failed to initialize capstone"
    return
  defer: discard cs_close(addr handle)
  
  var insn: ptr cs_insn
  let count = cs_disasm(handle, unsafeAddr code[0], code.len.uint,
                         baseAddr, 0, addr insn)
  defer: cs_free(insn, count)
  
  for i in 0..<count:
    let instr = cast[ptr UncheckedArray[cs_insn]](insn)[i]
    let mnem = $cast[cstring](unsafeAddr instr.mnemonic[0])
    let ops = $cast[cstring](unsafeAddr instr.op_str[0])
    echo &"  0x{instr.address:08X}: {mnem} {ops}"

# Uncomment when libcapstone is available:
# let shellcode: seq[byte] = @[0x48, 0x31, 0xC0, 0xC3]
# disassemble(shellcode)
echo "Capstone wrapper ready (requires libcapstone)"
```

## สรุป Part 53

- ␅ Binary analyzer (magic, entropy, strings)
- ␅ Simple x86/x64 disassembler
- ␅ Capstone FFI wrapper
- ␅ Suspicious string detection

---
**Next**: [Part 54 - Fuzzing](part54_fuzzing.md)
