# Part 50: Windows Persistence เพื่อการศึกษา

> **คำเตือน**: เนื้อหานี้สำหรับการศึกษาเท่านั้น: Blue Team, Red Team awareness,
> CTF และ authorized penetration testing สำหรับการตรวจสอบระบบเท่านั้น

## Persistence Mechanisms Overview

```
วิธี persistence หลักใน Windows:
1. Registry Run keys
2. Scheduled Tasks
3. Windows Services
4. Startup Folder
5. DLL Hijacking
6. COM Object Hijacking
7. WMI Subscriptions
8. Boot/Pre-OS methods (UEFI, bootkit)

วัตถุประสงค์ในการศึกษา:
- เข้าใจวิธีการเพื่อ detect/huntรายประจำ
- สร้าง detection rules (Sigma, YARA)
- Blue Team: รู้ว่าผู้โจมตีลงทำอะไร
```

## Registry Persistence

```nim
# registry_persistence.nim
# ทำความเข้าใจเพื่อ detect

when defined(windows):
  import winlean, strutils, strformat

  # Registry persistence locations
  const persistenceLocations = [
    # HKCU Run keys (no admin needed)
    ("HKCU", r"SOFTWARE\Microsoft\Windows\CurrentVersion\Run"),
    ("HKCU", r"SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce"),
    # HKLM Run keys (admin needed)
    ("HKLM", r"SOFTWARE\Microsoft\Windows\CurrentVersion\Run"),
    ("HKLM", r"SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce"),
    # Additional locations
    ("HKLM", r"SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"),
    ("HKLM", r"SYSTEM\CurrentControlSet\Services"),
  ]

  # Check registry for persistence (defensive)
  proc checkRegistryPersistence(): seq[string] =
    result = @[]
    let hives = [(HKEY_CURRENT_USER, "HKCU"), (HKEY_LOCAL_MACHINE, "HKLM")]
    
    let runKeys = @[
      r"SOFTWARE\Microsoft\Windows\CurrentVersion\Run",
      r"SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce",
      r"SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Run"
    ]
    
    for (hive, hiveName) in hives:
      for key in runKeys:
        var hKey: HKEY
        if RegOpenKeyExA(hive, key, 0, KEY_READ, addr hKey) == ERROR_SUCCESS:
          defer: discard RegCloseKey(hKey)
          
          var index = 0.DWORD
          var valueName = newString(256)
          var valueNameSize = 256.DWORD
          var valueType = 0.DWORD
          var valueData = newString(1024)
          var valueDataSize = 1024.DWORD
          
          while true:
            valueNameSize = 256
            valueDataSize = 1024
            let status = RegEnumValueA(
              hKey, index,
              addr valueName[0], addr valueNameSize,
              nil,
              addr valueType, cast[ptr BYTE](addr valueData[0]), addr valueDataSize
            )
            if status != ERROR_SUCCESS: break
            
            let name = valueName[0..<valueNameSize.int]
            let data = valueData[0..<valueDataSize.int - 1]
            result.add(&"[{hiveName}] {key}\\{name} -> {data}")
            inc index

  # Display persistence entries
  let entries = checkRegistryPersistence()
  echo &"Found {entries.len} run key entries:"
  for entry in entries:
    echo "  ", entry

  # Add entry (educational - for authorized testing)
  proc addRegistryRun(name, path: string, hive: HKEY = HKEY_CURRENT_USER): bool =
    let key = r"SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
    writeRegistryString(hive, key, name, path)

  # Remove entry
  proc removeRegistryRun(name: string, hive: HKEY = HKEY_CURRENT_USER): bool =
    var hKey: HKEY
    let key = r"SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
    if RegOpenKeyExA(hive, key, 0, KEY_WRITE, addr hKey) != ERROR_SUCCESS:
      return false
    defer: discard RegCloseKey(hKey)
    RegDeleteValueA(hKey, name) == ERROR_SUCCESS
```

## Scheduled Task Persistence

```nim
# scheduled_task.nim
# ตรวจ scheduled tasks เพื่อตรวจหา persistence

when defined(windows):
  import os, osproc, strutils, strformat

  # List scheduled tasks using schtasks.exe
  proc listScheduledTasks(): seq[string] =
    let (output, exitCode) = execCmdEx("schtasks /query /fo CSV /nh")
    if exitCode != 0:
      return @[]
    
    result = @[]
    for line in output.splitLines():
      let parts = line.split(",")
      if parts.len >= 3:
        let taskName = parts[0].strip(chars = {'"'})
        let status = parts[2].strip(chars = {'"'})
        result.add(&"{taskName} [{status}]")

  # Create scheduled task (for testing)
  proc createScheduledTask(name, path, trigger: string): bool =
    ## trigger: ONLOGON, ONSTART, DAILY, MINUTE
    let cmd = &"schtasks /create /tn \"{name}\" /tr \"{path}\" /sc {trigger} /f"
    let (_, exitCode) = execCmdEx(cmd)
    exitCode == 0

  # Delete scheduled task
  proc deleteScheduledTask(name: string): bool =
    let (_, exitCode) = execCmdEx(&"schtasks /delete /tn \"{name}\" /f")
    exitCode == 0

  # Check for suspicious scheduled tasks
  proc findSuspiciousTasks(): seq[string] =
    result = @[]
    let tasks = listScheduledTasks()
    
    let suspiciousKeywords = @[
      "temp", "tmp", "appdata", "public",
      "powershell -enc", "cmd /c", "wscript"
    ]
    
    for task in tasks:
      for kw in suspiciousKeywords:
        if kw.toLowerAscii() in task.toLowerAscii():
          result.add("[SUSPICIOUS] " & task)
          break

  echo "Scheduled Tasks:"
  for t in listScheduledTasks():
    echo "  ", t

  echo "\nSuspicious Tasks:"
  for t in findSuspiciousTasks():
    echo "  ", t
```

## Windows Service Persistence

```nim
# service_persistence.nim
# Windows services เป็น persistence method

when defined(windows):
  import winlean, strutils, strformat

  type
    SC_HANDLE = pointer
  
  proc OpenSCManagerA*(lpMachineName, lpDatabaseName: cstring,
                       dwDesiredAccess: DWORD): SC_HANDLE
    {.importc, dynlib: "advapi32".}
  
  proc CreateServiceA*(hSCManager: SC_HANDLE, lpServiceName, lpDisplayName: cstring,
                       dwDesiredAccess, dwServiceType, dwStartType, dwErrorControl: DWORD,
                       lpBinaryPathName, lpLoadOrderGroup: cstring, lpdwTagId: ptr DWORD,
                       lpDependencies, lpServiceStartName, lpPassword: cstring): SC_HANDLE
    {.importc, dynlib: "advapi32".}
  
  proc OpenServiceA*(hSCManager: SC_HANDLE, lpServiceName: cstring,
                     dwDesiredAccess: DWORD): SC_HANDLE
    {.importc, dynlib: "advapi32".}
  
  proc StartServiceA*(hService: SC_HANDLE, dwNumServiceArgs: DWORD,
                      lpServiceArgVectors: pointer): BOOL
    {.importc, dynlib: "advapi32".}
  
  proc DeleteService*(hService: SC_HANDLE): BOOL
    {.importc, dynlib: "advapi32".}
  
  proc CloseServiceHandle*(hSCObject: SC_HANDLE): BOOL
    {.importc, dynlib: "advapi32".}

  const
    SC_MANAGER_ALL_ACCESS = 0x000F003F.DWORD
    SERVICE_ALL_ACCESS = 0xF01FF.DWORD
    SERVICE_WIN32_OWN_PROCESS = 0x00000010.DWORD
    SERVICE_AUTO_START = 0x00000002.DWORD
    SERVICE_ERROR_NORMAL = 0x00000001.DWORD

  # Create Windows service
  proc createService(name, displayName, binaryPath: string): bool =
    let scm = OpenSCManagerA(nil, nil, SC_MANAGER_ALL_ACCESS)
    if scm == nil: return false
    defer: discard CloseServiceHandle(scm)
    
    let svc = CreateServiceA(
      scm, name, displayName,
      SERVICE_ALL_ACCESS,
      SERVICE_WIN32_OWN_PROCESS,
      SERVICE_AUTO_START,
      SERVICE_ERROR_NORMAL,
      binaryPath, nil, nil, nil, nil, nil
    )
    
    if svc == nil: return false
    defer: discard CloseServiceHandle(svc)
    
    # Start service immediately
    discard StartServiceA(svc, 0, nil)
    true

  # Detection: List services with unusual paths
  proc listServicesWithPaths(): seq[(string, string)] =
    import osproc
    let (output, _) = execCmdEx(
      "sc query type= all state= all | findstr SERVICE_NAME"
    )
    # Parse output...
    result = @[]
```

## Defense และ Detection

```nim
# persistence_defense.nim
# Blue Team: ตรวจหา persistence indicators

import os, osproc, strutils, strformat, json

type PersistenceReport = object
  registryRun: seq[string]
  scheduledTasks: seq[string]
  startupFolder: seq[string]
  services: seq[string]
  suspiciousFound: bool

proc checkStartupFolder(): seq[string] =
  result = @[]
  let startupPaths = [
    getEnv("APPDATA") & r"\Microsoft\Windows\Start Menu\Programs\Startup",
    r"C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup"
  ]
  
  for path in startupPaths:
    if dirExists(path):
      for kind, file in walkDir(path):
        if kind == pcFile:
          result.add(file)

proc buildPersistenceReport(): PersistenceReport =
  result.startupFolder = checkStartupFolder()
  
  # Look for suspicious executables
  let suspiciousPaths = @["temp", "tmp", "appdata\\local", "public"]
  
  for entry in result.registryRun & result.scheduledTasks & result.startupFolder:
    for path in suspiciousPaths:
      if path in entry.toLowerAscii():
        result.suspiciousFound = true
        break

proc exportReport(report: PersistenceReport, path: string) =
  let data = %*{
    "registryRun": report.registryRun,
    "scheduledTasks": report.scheduledTasks,
    "startupFolder": report.startupFolder,
    "suspiciousFound": report.suspiciousFound
  }
  writeFile(path, $data)
  echo "Report saved to: ", path

echo "=== Persistence Check ==="
echo "Startup folder:"
for f in checkStartupFolder():
  echo "  ", f
```

## สรุป Part 50

- ␅ Persistence mechanism overview
- ␅ Registry Run keys (detection + creation)
- ␅ Scheduled Tasks (enumerate + suspicious check)
- ␅ Windows Services
- ␅ Blue Team detection tools

---
**Next**: [Part 51 - Privilege Escalation](part51_privesc.md)
