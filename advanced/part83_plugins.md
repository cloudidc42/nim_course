# Part 83 - Plugin Architecture & Dynamic Loading

## บทนำ

Plugin system ช่วยให้ application extensible โดยไม่ต้อง recompile:
- Dynamic loading ด้วย `dlopen`/`LoadLibrary`
- Shared library interface
- Plugin discovery และ lifecycle management

---

## 1. Plugin Interface Definition

```nim
# plugin_api.nim
# Interface ที่ plugins ต้องใช้

const PluginApiVersion* = 1

type
  PluginInfo* = object
    name*: cstring
    version*: cstring
    author*: cstring
    description*: cstring
    apiVersion*: int32

  PluginContext* = object
    logFn*: proc(level: int32, msg: cstring) {.cdecl.}
    configGet*: proc(key: cstring): cstring {.cdecl.}
    emitEvent*: proc(event: cstring, data: cstring) {.cdecl.}

  # Functions every plugin must export
  PluginInitFn* = proc(ctx: ptr PluginContext): int32 {.cdecl.}
  PluginShutdownFn* = proc() {.cdecl.}
  PluginGetInfoFn* = proc(): ptr PluginInfo {.cdecl.}
  PluginHandleEventFn* = proc(event: cstring, data: cstring): int32 {.cdecl.}

  Plugin* = ref object
    info*: PluginInfo
    handle*: pointer      # dlopen handle
    init*: PluginInitFn
    shutdown*: PluginShutdownFn
    getInfo*: PluginGetInfoFn
    handleEvent*: PluginHandleEventFn
    loaded*: bool
```

---

## 2. Plugin Loader (Linux/macOS)

```nim
# plugin_loader.nim
# โหลด .so / .dylib plugins

import std/[os, tables, strformat, dynlib, strutils]
import plugin_api

type
  PluginError* = object of CatchableError

  PluginManager* = ref object
    plugins*: Table[string, Plugin]
    pluginDir*: string
    context*: PluginContext
    eventHandlers*: Table[string, seq[proc(data: string)]]

proc log(level: int32, msg: cstring) {.cdecl.} =
  let prefix = case level
    of 0: "[DEBUG]"
    of 1: "[INFO]"
    of 2: "[WARN]"
    of 3: "[ERROR]"
    else: "[???]"
  echo &"{prefix} Plugin: {msg}"

proc configGet(key: cstring): cstring {.cdecl.} =
  # Could read from config file/env
  let k = $key
  case k
  of "debug": cstring("false")
  of "version": cstring("1.0.0")
  else: cstring("")

proc newPluginManager*(pluginDir: string): PluginManager =
  result = PluginManager(
    plugins: initTable[string, Plugin](),
    pluginDir: pluginDir,
    eventHandlers: initTable[string, seq[proc(data: string)]]()
  )
  result.context = PluginContext(
    logFn: log,
    configGet: configGet,
    emitEvent: proc(event, data: cstring) {.cdecl.} =
      let evStr = $event
      let dataStr = $data
      if evStr in result.eventHandlers:
        for handler in result.eventHandlers[evStr]:
          handler(dataStr)
  )

proc loadPlugin*(pm: PluginManager, path: string): Plugin =
  let handle = loadLib(path)
  if handle.isNil:
    raise newException(PluginError, &"Failed to load plugin: {path}")
  
  let getInfo = cast[PluginGetInfoFn](symAddr(handle, "plugin_get_info"))
  let init = cast[PluginInitFn](symAddr(handle, "plugin_init"))
  let shutdown = cast[PluginShutdownFn](symAddr(handle, "plugin_shutdown"))
  let handleEvent = cast[PluginHandleEventFn](symAddr(handle, "plugin_handle_event"))
  
  if getInfo.isNil or init.isNil or shutdown.isNil:
    unloadLib(handle)
    raise newException(PluginError, &"Invalid plugin: missing required exports in {path}")
  
  let info = getInfo()[]
  
  if info.apiVersion != PluginApiVersion:
    unloadLib(handle)
    raise newException(PluginError,
      &"API version mismatch: plugin={info.apiVersion}, host={PluginApiVersion}")
  
  let plugin = Plugin(
    info: info,
    handle: handle,
    init: init,
    shutdown: shutdown,
    getInfo: getInfo,
    handleEvent: handleEvent
  )
  
  let rc = init(addr pm.context)
  if rc != 0:
    unloadLib(handle)
    raise newException(PluginError, &"Plugin init failed with code: {rc}")
  
  plugin.loaded = true
  echo &"Loaded plugin: {info.name} v{info.version} by {info.author}"
  plugin

proc loadAll*(pm: PluginManager) =
  if not dirExists(pm.pluginDir):
    echo &"Plugin dir not found: {pm.pluginDir}"
    return
  
  for kind, path in walkDir(pm.pluginDir):
    if kind != pcFile: continue
    let ext = path.splitFile().ext
    when defined(windows):
      if ext != ".dll": continue
    elif defined(macosx):
      if ext != ".dylib": continue
    else:
      if ext != ".so": continue
    
    try:
      let plugin = pm.loadPlugin(path)
      pm.plugins[path.splitFile().name] = plugin
    except PluginError as e:
      echo &"Failed to load {path}: {e.msg}"

proc unloadPlugin*(pm: PluginManager, name: string) =
  if name in pm.plugins:
    let plugin = pm.plugins[name]
    if plugin.loaded and not plugin.shutdown.isNil:
      plugin.shutdown()
    unloadLib(plugin.handle)
    pm.plugins.del(name)
    echo &"Unloaded plugin: {name}"

proc unloadAll*(pm: PluginManager) =
  for name in pm.plugins.keys.toSeq():
    pm.unloadPlugin(name)

proc emit*(pm: PluginManager, event: string, data = "") =
  # Notify all plugins
  for name, plugin in pm.plugins:
    if plugin.loaded and not plugin.handleEvent.isNil:
      discard plugin.handleEvent(cstring(event), cstring(data))
  
  # Notify host handlers
  if event in pm.eventHandlers:
    for handler in pm.eventHandlers[event]:
      handler(data)

proc on*(pm: PluginManager, event: string, handler: proc(data: string)) =
  if event notin pm.eventHandlers:
    pm.eventHandlers[event] = @[]
  pm.eventHandlers[event].add(handler)

proc listPlugins*(pm: PluginManager) =
  echo &"\nLoaded plugins ({pm.plugins.len}):"
  for name, plugin in pm.plugins:
    echo &"  {plugin.info.name} v{plugin.info.version} - {plugin.info.description}"
```

---

## 3. Example Plugin Implementation

```nim
# plugins/greeter.nim
# Compile ด้วย:
# nim c --app:lib -d:release --noMain -o:plugins/greeter.so plugins/greeter.nim

import plugin_api

var ctx: ptr PluginContext

var info = PluginInfo(
  name: cstring("greeter"),
  version: cstring("1.0.0"),
  author: cstring("Example"),
  description: cstring("A simple greeter plugin"),
  apiVersion: PluginApiVersion
)

proc plugin_get_info*(): ptr PluginInfo {.exportc, cdecl.} =
  addr info

proc plugin_init*(context: ptr PluginContext): int32 {.exportc, cdecl.} =
  ctx = context
  ctx.logFn(1, cstring("Greeter plugin initialized"))
  return 0

proc plugin_shutdown*() {.exportc, cdecl.} =
  if not ctx.isNil:
    ctx.logFn(1, cstring("Greeter plugin shutting down"))

proc plugin_handle_event*(event, data: cstring): int32 {.exportc, cdecl.} =
  let ev = $event
  let d = $data
  
  case ev
  of "greet":
    let name = if d.len > 0: d else: "World"
    let msg = cstring("Hello, " & name & "!")
    ctx.logFn(1, msg)
    ctx.emitEvent(cstring("greeted"), cstring(name))
    return 0
  of "farewell":
    ctx.logFn(1, cstring("Goodbye!"))
    return 0
  else:
    return -1  # Unhandled event
```

---

## 4. Scripting Engine Integration

```nim
# scripting.nim
# Plugin system ด้วย embedded scripting (Lua-style)

import std/[tables, strformat, strutils]

type
  ScriptValue* = object
    case kind*: enum
      svNil, svBool, svInt, svFloat, svString, svFunction
    boolVal*: bool
    intVal*: int64
    floatVal*: float64
    strVal*: string
    fnVal*: proc(args: seq[ScriptValue]): ScriptValue

  ScriptEnv* = ref object
    vars*: Table[string, ScriptValue]
    parent*: ScriptEnv

  Script* = ref object
    source*: string
    path*: string
    env*: ScriptEnv

proc newScriptEnv*(parent: ScriptEnv = nil): ScriptEnv =
  ScriptEnv(vars: initTable[string, ScriptValue](), parent: parent)

proc nilVal*(): ScriptValue = ScriptValue(kind: svNil)
proc boolVal*(v: bool): ScriptValue = ScriptValue(kind: svBool, boolVal: v)
proc intVal*(v: int64): ScriptValue = ScriptValue(kind: svInt, intVal: v)
proc strVal*(v: string): ScriptValue = ScriptValue(kind: svString, strVal: v)
proc fnVal*(v: proc(args: seq[ScriptValue]): ScriptValue): ScriptValue =
  ScriptValue(kind: svFunction, fnVal: v)

proc get*(env: ScriptEnv, name: string): ScriptValue =
  if name in env.vars: return env.vars[name]
  if not env.parent.isNil: return env.parent.get(name)
  nilVal()

proc set*(env: ScriptEnv, name: string, val: ScriptValue) =
  env.vars[name] = val

# Host API registration
type
  ScriptPlugin* = ref object
    name*: string
    env*: ScriptEnv
    hooks*: Table[string, proc(args: seq[ScriptValue]): ScriptValue]

proc newScriptPlugin*(name: string): ScriptPlugin =
  ScriptPlugin(
    name: name,
    env: newScriptEnv(),
    hooks: initTable[string, proc(args: seq[ScriptValue]): ScriptValue]()
  )

proc registerFunction*(plugin: ScriptPlugin, name: string,
    fn: proc(args: seq[ScriptValue]): ScriptValue) =
  plugin.env.set(name, fnVal(fn))
  plugin.hooks[name] = fn

proc call*(plugin: ScriptPlugin, hookName: string, args: seq[ScriptValue] = @[]): ScriptValue =
  if hookName in plugin.hooks:
    return plugin.hooks[hookName](args)
  nilVal()

# Example: Script-based transform pipeline
type
  TransformPlugin* = ref object
    transforms*: seq[proc(input: string): string]

proc addTransform*(p: TransformPlugin, fn: proc(input: string): string) =
  p.transforms.add(fn)

proc process*(p: TransformPlugin, input: string): string =
  result = input
  for transform in p.transforms:
    result = transform(result)

proc loadTransformScript*(path: string): TransformPlugin =
  let plugin = TransformPlugin(transforms: @[])
  
  # Simple scripted transforms (in real impl, use embedded Lua/JS)
  let source = readFile(path)
  
  for line in source.splitLines():
    let line = line.strip()
    if line.startsWith("transform:"):
      let rule = line[10..^1].strip()
      if rule.startsWith("upper"):
        plugin.addTransform(proc(s: string): string = s.toUpperAscii())
      elif rule.startsWith("lower"):
        plugin.addTransform(proc(s: string): string = s.toLowerAscii())
      elif rule.startsWith("trim"):
        plugin.addTransform(proc(s: string): string = s.strip())
      elif rule.startsWith("replace:"):
        let parts = rule[8..^1].split("->")
        if parts.len == 2:
          let from = parts[0].strip()
          let to = parts[1].strip()
          plugin.addTransform(proc(s: string): string = s.replace(from, to))
  
  plugin

when isMainModule:
  # Demonstrate plugin system
  let pm = newPluginManager("./plugins")
  
  pm.on("greeted") do(data: string):
    echo &"Host received: greeted event for '{data}'"
  
  pm.loadAll()
  pm.listPlugins()
  
  pm.emit("greet", "Nim")
  pm.emit("farewell")
  
  pm.unloadAll()
```

---

## 5. Hot Reload System

```nim
# hot_reload.nim
# Hot reload: watch plugin directory, reload on change

import std/[os, times, tables, strformat, asyncdispatch]
import plugin_loader

type
  WatchEntry* = object
    path*: string
    lastMtime*: Time
    pluginName*: string

  HotReloader* = ref object
    pm*: PluginManager
    watched*: Table[string, WatchEntry]
    interval*: int  # milliseconds

proc newHotReloader*(pm: PluginManager, interval = 1000): HotReloader =
  HotReloader(pm: pm, watched: initTable[string, WatchEntry](), interval: interval)

proc watch*(hr: HotReloader, path: string) =
  if fileExists(path):
    let info = getFileInfo(path)
    hr.watched[path] = WatchEntry(
      path: path,
      lastMtime: info.lastWriteTime,
      pluginName: path.splitFile().name
    )

proc checkChanges*(hr: HotReloader) =
  for path, entry in hr.watched.mpairs:
    if not fileExists(path): continue
    
    let info = getFileInfo(path)
    if info.lastWriteTime > entry.lastMtime:
      echo &"Plugin changed: {path}, reloading..."
      entry.lastMtime = info.lastWriteTime
      
      # Unload old version
      hr.pm.unloadPlugin(entry.pluginName)
      
      # Load new version
      try:
        let plugin = hr.pm.loadPlugin(path)
        hr.pm.plugins[entry.pluginName] = plugin
        echo &"Reloaded: {plugin.info.name} v{plugin.info.version}"
        hr.pm.emit("plugin_reloaded", entry.pluginName)
      except PluginError as e:
        echo &"Reload failed: {e.msg}"

proc startWatching*(hr: HotReloader) {.async.} =
  echo &"Hot reload watching {hr.watched.len} plugins every {hr.interval}ms"
  while true:
    hr.checkChanges()
    await sleepAsync(hr.interval)
```

---

## สรุป

| Pattern | ใช้เมื่อ |
|---------|---------|
| dynlib + cdecl | Native plugin (.so/.dll) |
| Script embedding | User-scriptable behavior |
| Transform pipeline | Data processing plugins |
| Hot reload | Development workflow |

---

**Next**: [Part 84 - Microservices & Service Mesh](../advanced/part84_microservices.md)
