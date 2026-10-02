# Part 51: Privilege Escalation เพื่อการศึกษา

> **คำเตือน**: เนื้อหานี้เป็นการศึกษาเพื่อ authorized pentesting, CTF,
> และทำความเข้าใจ Windows security model เท่านั้น

## Privilege Escalation Overview

```
Windows Privilege Levels:
  NT AUTHORITY\SYSTEM     (highest)
  Administrator
  Standard User
  Low Integrity (sandbox)

Privesc Methods:
1. Unquoted Service Paths
2. Weak Service Permissions
3. DLL Hijacking
4. Always Install Elevated
5. Token Impersonation (SeImpersonatePrivilege)
6. UAC Bypass
7. Kernel Exploits
8. Credential dumping

Tools:
- WinPEAS: automated enumeration
- PowerUp: PowerShell privesc
- PrivescCheck: PowerShell checks
```

## System Enumeration

```nim
# system_enum.nim
# เก็บข้อมูลระบบเพื่อหา privesc vectors

import os, osproc, strutils, strformat, tables

type SystemInfo = object
  hostname: string
  osVersion: string
  architecture: string
  currentUser: string
  privileges: seq[string]
  groups: seq[string]
  installedPatches: seq[string]
  networkInfo: seq[string]
  runningServices: seq[string]
  unquotedPaths: seq[string]

proc getSystemInfo(): SystemInfo =
  result.hostname = getHostname()
  result.currentUser = getEnv("USERNAME", getEnv("USER", "unknown"))
  
  when defined(windows):
    # OS version
    let (osVer, _) = execCmdEx("ver")
    result.osVersion = osVer.strip()
    
    # Privileges
    let (privs, _) = execCmdEx("whoami /priv")
    for line in privs.splitLines():
      if "Enabled" in line or "Disabled" in line:
        result.privileges.add(line.strip())
    
    # Groups
    let (groups, _) = execCmdEx("whoami /groups")
    for line in groups.splitLines():
      if line.len > 0 and not line.startsWith("Group"):
        result.groups.add(line.strip())
    
    # Installed patches
    let (patches, _) = execCmdEx("wmic qfe list brief")
    for line in patches.splitLines()[1..^1]:
      let parts = line.split()
      if parts.len > 0:
        result.installedPatches.add(parts.join(" "))
    
    # Network
    let (netinfo, _) = execCmdEx("ipconfig /all")
    result.networkInfo = netinfo.splitLines().filterIt(it.len > 0)

  result

proc findUnquotedServicePaths(): seq[string] =
  when defined(windows):
    result = @[]
    let (output, _) = execCmdEx(
      "wmic service get name,pathname,startmode /format:csv"
    )
    
    for line in output.splitLines():
      let parts = line.split(",")
      if parts.len >= 4:
        let path = parts[2].strip()
        let startMode = parts[3].strip()
        
        # Unquoted path with spaces = possible hijack
        if ' ' in path and not path.startsWith('"') and startMode == "Auto":
          result.add(&"Service: {parts[1]} | Path: {path}")
  else:
    result = @[]

# Check for weak file permissions
proc checkFilePermissions(path: string): bool =
  when defined(windows):
    let (output, exitCode) = execCmdEx(&"icacls \"{path}\"")
    if exitCode != 0: return false
    
    # Check if everyone/users can write
    "Everyone:(F)" in output or 
    "Everyone:(W)" in output or
    "BUILTIN\\Users:(F)" in output or
    "BUILTIN\\Users:(W)" in output
  else:
    false

proc runEnum() =
  echo "=== System Enumeration ==="
  let info = getSystemInfo()
  echo "Hostname: ", info.hostname
  echo "User: ", info.currentUser
  echo "OS: ", info.osVersion
  
  echo "\n=== Privileges ==="
  for p in info.privileges:
    echo "  ", p
  
  echo "\n=== Unquoted Service Paths ==="
  let unquoted = findUnquotedServicePaths()
  if unquoted.len == 0:
    echo "  None found"
  else:
    for path in unquoted:
      echo "  [!] ", path

runEnum()
```

## Token Impersonation

```nim
# token_impersonation.nim
# SeImpersonatePrivilege - Potato attacks concept

when defined(windows):
  import winlean
  
  type
    TOKEN_TYPE = enum ttPrimary = 1, ttImpersonation
    SECURITY_IMPERSONATION_LEVEL = enum
      slAnonymous, slIdentification, slImpersonation, slDelegation

  proc OpenProcessToken*(hProcess: HANDLE, dwDesiredAccess: DWORD,
                         phToken: ptr HANDLE): BOOL
    {.importc, dynlib: "advapi32".}
  
  proc DuplicateTokenEx*(hExistingToken: HANDLE, dwDesiredAccess: DWORD,
                          lpTokenAttributes: pointer,
                          ImpersonationLevel: SECURITY_IMPERSONATION_LEVEL,
                          TokenType: TOKEN_TYPE,
                          phNewToken: ptr HANDLE): BOOL
    {.importc, dynlib: "advapi32".}
  
  proc ImpersonateLoggedOnUser*(hToken: HANDLE): BOOL
    {.importc, dynlib: "advapi32".}
  
  proc RevertToSelf*(): BOOL
    {.importc, dynlib: "advapi32".}
  
  proc GetCurrentProcess*(): HANDLE
    {.importc, dynlib: "kernel32".}
  
  const
    TOKEN_ALL_ACCESS = 0xF01FF.DWORD
    TOKEN_DUPLICATE = 0x0002.DWORD
    TOKEN_IMPERSONATE = 0x0004.DWORD
    TOKEN_QUERY = 0x0008.DWORD

  ## Potato-style: ImpersonateNamedPipeClient
  ## Concept only - for CTF/security research
  proc impersonateNamedPipeClient*(hNamedPipe: HANDLE): BOOL
    {.importc, dynlib: "advapi32".}

  proc getCurrentTokenInfo(): string =
    var hToken: HANDLE
    if OpenProcessToken(GetCurrentProcess(), TOKEN_QUERY, addr hToken) == FALSE:
      return "Failed to open token"
    defer: discard CloseHandle(hToken)
    
    # In real code: GetTokenInformation + LookupAccountSid
    "Token opened (would query user SID here)"

  echo getCurrentTokenInfo()

## ความรู้เกี่ยวกับ Potato attacks:
## - HotPotato: NBNS spoofing + NTLM relay
## - RottenPotato/JuicyPotato: COM server + token stealing
## - PrintSpoofer: Print spooler pipe impersonation
## - SweetPotato: combination technique
## ทั้งหมดใช้ SeImpersonatePrivilege
```

## UAC Bypass Concepts

```nim
# uac_concepts.nim
# User Account Control bypass เพื่อการศึกษา

## UAC Architecture:
## Standard user wants elevation
##   -> UAC prompt
##   -> User approves
##   -> Token duplicated with high integrity

## UAC Bypass Methods:
## 1. Auto-elevation: บาง executables auto-elevate (เช่น eventvwr.exe)
## 2. fodhelper.exe: registry hijack
## 3. cmstp.exe: COM scriptlet
## 4. SilentCleanup: scheduled task DLL sideload
## 5. UACME project: 70+ bypass methods

## fodhelper bypass (classic, patched in newer Windows):
## HKCU:\Software\Classes\ms-settings\shell\open\command
## DefaultIcon + DelegateExecute

when defined(windows):
  import osproc, strutils
  
  ## ความรู้เพื่อ detect:
  proc checkUACLevel(): string =
    import winlean
    let (output, _) = execCmdEx(
      "reg query HKLM\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Policies\\System /v EnableLUA"
    )
    if "0x0" in output:
      "UAC: Disabled"
    elif "0x1" in output:
      let (lvl, _) = execCmdEx(
        "reg query HKLM\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Policies\\System /v ConsentPromptBehaviorAdmin"
      )
      if "0x5" in lvl: "UAC: Prompt for credentials"
      elif "0x2" in lvl: "UAC: Always notify (max)"
      else: "UAC: Enabled (level " & lvl & ")"
    else:
      "UAC: Unknown state"
  
  echo checkUACLevel()

echo "\nDetection of UAC bypasses:"
echo "- Monitor registry writes to HKCU:Software\\Classes"
echo "- Unexpected auto-elevated child processes"
echo "- Process creation with high integrity without UAC prompt"
echo "- eventvwr, fodhelper, cmstp child process anomalies"
```

## สรุป Part 51

- ␅ Privilege escalation overview
- ␅ System enumeration tools
- ␅ Token impersonation concepts
- ␅ UAC bypass concepts
- ␅ Blue Team detection indicators

---
**Next**: [Part 52 - Network Recon](part52_network_recon.md)
