# Part 32: Shellcode - พื้นฐานและการ Execute

> **คำเตือน**: เนื้อหานี้ใช้สำหรับการศึกษา, CTF, pentesting ที่ได้รับอนุญาต,
> และ defensive security research เท่านั้น ห้ามใช้โจมตีระบบที่ไม่ได้รับอนุญาต

## Shellcode คืออะไร

```
Shellcode = ชุดคำสั่ง machine code ที่ inject เข้าไปใน process แล้ว execute
โดยตรง ส่วนใหญ่เขียนด้วย Assembly แล้ว convert เป็น raw bytes

ใช้ใน:
- Exploit development (security research)
- Penetration testing (post-exploitation)
- Red teaming exercises
- Malware analysis (reverse engineering)
- CTF competitions

ต้องมี:
- ความรู้ x86/x64 assembly
- ความเข้าใจ Windows API internals
- ใบอนุญาต/การอนุมัติจากเจ้าของระบบ
```

## x64 Assembly พื้นฐาน

```asm
; Registers (x64)
; RAX, RBX, RCX, RDX - general purpose
; RSI, RDI - source/destination index
; RSP - stack pointer
; RBP - base pointer
; RIP - instruction pointer (read-only)
; R8-R15 - additional registers

; Windows x64 calling convention:
; Parameters: RCX, RDX, R8, R9, stack
; Return value: RAX

; Simple hello world shellcode in assembly
; (NASM syntax)
section .text
global _start

_start:
    xor rcx, rcx    ; hWnd = NULL
    lea rdx, [rel msg]  ; lpText
    lea r8, [rel cap]   ; lpCaption
    xor r9, r9      ; uType = 0
    sub rsp, 0x28   ; shadow space
    call MessageBoxA
    add rsp, 0x28
    
    xor rcx, rcx    ; exit code
    call ExitProcess

msg: db "Hello from Shellcode!", 0
cap: db "Test", 0
```

## Shellcode Execution ใน Nim

```nim
# shellcode_runner.nim
# Educational shellcode execution example
# สำหรับ CTF, authorized testing, security research

when defined(windows):
  import winlean

  proc VirtualAlloc*(
    lpAddress: pointer, dwSize: SIZE_T,
    flAllocationType, flProtect: DWORD
  ): pointer {.importc, dynlib: "kernel32".}
  
  proc VirtualFree*(
    lpAddress: pointer, dwSize: SIZE_T, dwFreeType: DWORD
  ): BOOL {.importc, dynlib: "kernel32".}
  
  proc RtlCopyMemory*(dest, src: pointer, size: SIZE_T)
    {.importc, dynlib: "ntdll".}
  
  proc VirtualProtect*(
    lpAddress: pointer, dwSize: SIZE_T,
    flNewProtect: DWORD, lpflOldProtect: ptr DWORD
  ): BOOL {.importc, dynlib: "kernel32".}
  
  proc CreateThread*(
    lpThreadAttributes: pointer, dwStackSize: SIZE_T,
    lpStartAddress: pointer, lpParameter: pointer,
    dwCreationFlags: DWORD, lpThreadId: ptr DWORD
  ): HANDLE {.importc, dynlib: "kernel32".}
  
  proc WaitForSingleObject*(hHandle: HANDLE, dwMilliseconds: DWORD): DWORD
    {.importc, dynlib: "kernel32".}
  
  const
    MEM_COMMIT = 0x1000.DWORD
    MEM_RESERVE = 0x2000.DWORD
    MEM_RELEASE = 0x8000.DWORD
    PAGE_READWRITE = 0x04.DWORD
    PAGE_EXECUTE_READ = 0x20.DWORD
    INFINITE = 0xFFFFFFFF.DWORD

  proc executeShellcode(shellcode: seq[byte]) =
    # 1. Allocate RW memory
    let mem = VirtualAlloc(nil, shellcode.len.SIZE_T,
                           MEM_COMMIT or MEM_RESERVE, PAGE_READWRITE)
    if mem == nil:
      echo "VirtualAlloc failed"
      return
    defer: discard VirtualFree(mem, 0, MEM_RELEASE)
    # 2. Copy shellcode to memory
    RtlCopyMemory(mem, unsafeAddr shellcode[0], shellcode.len.SIZE_T)
    # 3. Change to RX permission
    var oldProtect: DWORD
    if VirtualProtect(mem, shellcode.len.SIZE_T, 
                      PAGE_EXECUTE_READ, addr oldProtect) == 0:
      echo "VirtualProtect failed"
      return
    # 4. Execute in new thread
    var threadId: DWORD
    let hThread = CreateThread(nil, 0, mem, nil, 0, addr threadId)
    if hThread == nil:
      echo "CreateThread failed"
      return
    # 5. Wait for completion
    discard WaitForSingleObject(hThread, INFINITE)
    discard CloseHandle(hThread)
  
  # NOP sled placeholder (for testing)
  let testShellcode: seq[byte] = @[
    0x90.byte, 0x90.byte, 0x90.byte, 0xC3.byte  # NOP NOP NOP RET
  ]
  
  echo "Shellcode runner example (educational)"
```

## XOR Encryption

```nim
# XOR encryption/obfuscation for shellcode research
proc xorEncrypt(data: var seq[byte], key: byte) =
  for b in data.mitems:
    b = b xor key

proc xorDecryptAndExec(encrypted: seq[byte], key: byte) {.used.} =
  var decrypted = encrypted
  xorEncrypt(decrypted, key)
  # then execute...

# Anti-analysis detection (understanding for defensive purposes)
proc checkDebugger(): bool =
  when defined(windows):
    proc IsDebuggerPresent*(): BOOL {.importc, dynlib: "kernel32".}
    IsDebuggerPresent() == TRUE
  else:
    false

if checkDebugger():
  echo "Running under debugger"
```

## Shellcode Analyzer

```nim
# shellcode_analyzer.nim - Analyze shellcode bytes
import strutils, strformat, math

proc analyzeShellcode(shellcode: seq[byte]) =
  echo "=== Shellcode Analysis ==="
  echo "Size: ", shellcode.len, " bytes"
  
  echo "\n--- Hex Dump ---"
  for i in 0..<shellcode.len:
    if i mod 16 == 0:
      if i > 0: echo ""
      stdout.write &"{i:04X}  "
    stdout.write &"{shellcode[i]:02X} "
    if i mod 16 == 7: stdout.write " "
  echo ""
  
  echo "\n--- Pattern Detection ---"
  var nullCount = 0
  for b in shellcode:
    if b == 0: inc nullCount
  echo "Null bytes: ", nullCount
  
  var callCount = 0
  var jmpCount = 0
  for i, b in shellcode:
    case b
    of 0xE8: inc callCount
    of 0xEB, 0xE9: inc jmpCount
    else: discard
  echo "CALL instructions: ", callCount
  echo "JMP instructions: ", jmpCount
  
  var freq: array[256, int]
  for b in shellcode: inc freq[b]
  var entropy = 0.0
  for f in freq:
    if f > 0:
      let p = float(f) / float(shellcode.len)
      entropy -= p * log2(p)
  echo &"Entropy: {entropy:.2f} (7.0+ = likely encrypted/compressed)"

let sampleShellcode: seq[byte] = @[
  0x48.byte, 0x31.byte, 0xC9.byte,  # xor rcx, rcx
  0x48.byte, 0xFF.byte, 0xC1.byte,  # inc rcx
  0x48.byte, 0x31.byte, 0xC0.byte,  # xor rax, rax
  0xB0.byte, 0x3C.byte,             # mov al, 60 (sys_exit)
  0x0F.byte, 0x05.byte              # syscall
]

analyzeShellcode(sampleShellcode)
```

## สรุป Part 32

ในบทนี้เราได้เรียนรู้:
- ✅ Shellcode คืออะไร และใช้ทำอะไรใน security research
- ✅ x64 Assembly พื้นฐาน
- ✅ การ allocate executable memory (VirtualAlloc, CreateThread)
- ✅ XOR encryption เบื้องต้น
- ✅ Shellcode analyzer tool

---

**Previous**: [Part 31 - Windows API Advanced](part31_winapi.md)
**Next**: [Part 33 - Process Injection](part33_injection.md)
