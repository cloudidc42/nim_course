# Part 96 - Advanced FFI

## บทนำ

Foreign Function Interface ขั้นสูง:
- C++ interop
- Platform-specific APIs
- COM interfaces (Windows)
- Dynamic library loading

---

## 1. C++ Interop

```nim
# cpp_interop.nim
# Calling C++ from Nim

{.passC: "-std=c++17".}
{.passL: "-lstdc++".}

# Import C++ string
type
  CppString* {.importcpp: "std::string", header: "<string>".} = object

proc newCppString*(s: cstring): CppString
  {.importcpp: "std::string(@)", header: "<string>".}

proc cppLen*(s: CppString): int
  {.importcpp: "#.size()", header: "<string>".}

proc cppCStr*(s: CppString): cstring
  {.importcpp: "#.c_str()", header: "<string>".}

proc cppAppend*(s: var CppString, other: CppString)
  {.importcpp: "#.append(@)", header: "<string>".}

# Import C++ vector
type
  CppVector*[T] {.importcpp: "std::vector<'0>", header: "<vector>".} = object

proc newCppVector*[T](): CppVector[T]
  {.importcpp: "std::vector<'*0>()", header: "<vector>".}

proc push_back*[T](v: var CppVector[T], val: T)
  {.importcpp: "#.push_back(@)", header: "<vector>".}

proc `[]`*[T](v: CppVector[T], idx: int): T
  {.importcpp: "#[#]", header: "<vector>".}

proc size*[T](v: CppVector[T]): int
  {.importcpp: "#.size()", header: "<vector>".}

# Import C++ unordered_map
type
  CppMap*[K, V] {.importcpp: "std::unordered_map<'0,'1>",
                   header: "<unordered_map>".} = object

proc newCppMap*[K, V](): CppMap[K, V]
  {.importcpp: "std::unordered_map<'*0,'*1>()", header: "<unordered_map>".}

proc `[]=`*[K, V](m: var CppMap[K, V], key: K, val: V)
  {.importcpp: "#[#] = #", header: "<unordered_map>".}

proc `[]`*[K, V](m: CppMap[K, V], key: K): V
  {.importcpp: "#[#]", header: "<unordered_map>".}

proc count*[K, V](m: CppMap[K, V], key: K): int
  {.importcpp: "#.count(@)", header: "<unordered_map>".}

# Wrapping a C++ class
{.emit: """
#include <string>
#include <vector>
class NimCppClass {
public:
  std::string name;
  std::vector<int> data;
  
  NimCppClass(const std::string& n) : name(n) {}
  
  void add(int v) { data.push_back(v); }
  int sum() const {
    int s = 0;
    for (auto x : data) s += x;
    return s;
  }
  const char* getName() const { return name.c_str(); }
};
""".}

type
  NimCppClass* {.importcpp: "NimCppClass", nodecl.} = object

proc newNimCppClass*(name: cstring): NimCppClass
  {.importcpp: "NimCppClass(@)", nodecl.}

proc add*(obj: var NimCppClass, v: int)
  {.importcpp: "#.add(@)", nodecl.}

proc sum*(obj: NimCppClass): int
  {.importcpp: "#.sum()", nodecl.}

proc getName*(obj: NimCppClass): cstring
  {.importcpp: "#.getName()", nodecl.}

when isMainModule:
  # C++ string
  var s = newCppString("Hello")
  echo "Length: ", s.cppLen()
  echo "C str: ", s.cppCStr()
  
  # C++ vector
  var v = newCppVector[int]()
  v.push_back(1)
  v.push_back(2)
  v.push_back(3)
  echo "Size: ", v.size()
  echo "v[0]: ", v[0]
  
  # C++ class
  var obj = newNimCppClass("myobj")
  obj.add(10)
  obj.add(20)
  obj.add(30)
  echo "Name: ", obj.getName()
  echo "Sum: ", obj.sum()
```

---

## 2. Platform-Specific APIs

```nim
# platform_api.nim
# Linux system calls

import std/[posix, os, strformat]

# epoll (Linux) for high-performance I/O
when defined(linux):
  const
    EPOLLIN* = 0x00000001.cint
    EPOLLOUT* = 0x00000004.cint
    EPOLLERR* = 0x00000008.cint
    EPOLLHUP* = 0x00000010.cint
    EPOLLET* = 0x80000000.cint  # Edge-triggered
    EPOLL_CTL_ADD* = 1
    EPOLL_CTL_MOD* = 2
    EPOLL_CTL_DEL* = 3

  type
    EpollData* {.union.} = object
      p*: pointer
      fd*: cint
      u32*: uint32
      u64*: uint64

    EpollEvent* = object
      events*: uint32
      data*: EpollData

  proc epoll_create1*(flags: cint): cint
    {.importc, header: "<sys/epoll.h>".}
  
  proc epoll_ctl*(epfd, op, fd: cint, event: ptr EpollEvent): cint
    {.importc, header: "<sys/epoll.h>".}
  
  proc epoll_wait*(epfd: cint, events: ptr EpollEvent, maxevents, timeout: cint): cint
    {.importc, header: "<sys/epoll.h>".}

  # High-performance event loop using epoll
  type
    EventHandler* = proc(fd: cint, events: uint32)
    EpollLoop* = ref object
      epfd*: cint
      handlers*: seq[(cint, EventHandler)]

  proc newEpollLoop*(): EpollLoop =
    let epfd = epoll_create1(0)
    if epfd == -1: raiseOSError(osLastError())
    EpollLoop(epfd: epfd)

  proc add*(loop: EpollLoop, fd: cint, events: uint32, handler: EventHandler) =
    var ev = EpollEvent(events: events)
    ev.data.fd = fd
    if epoll_ctl(loop.epfd, EPOLL_CTL_ADD, fd, addr ev) == -1:
      raiseOSError(osLastError())
    loop.handlers.add((fd, handler))

  proc run*(loop: EpollLoop, timeout = -1) =
    var events: array[64, EpollEvent]
    let n = epoll_wait(loop.epfd, addr events[0], 64, timeout.cint)
    if n == -1: raiseOSError(osLastError())
    
    for i in 0..<n:
      let fd = events[i].data.fd
      for (hfd, handler) in loop.handlers:
        if hfd == fd: handler(fd, events[i].events)

  proc close*(loop: EpollLoop) =
    discard posix.close(loop.epfd)

# inotify (Linux) for file watching  
when defined(linux):
  const
    IN_CREATE* = 0x00000100'u32
    IN_MODIFY* = 0x00000002'u32
    IN_DELETE* = 0x00000200'u32
    IN_MOVED_FROM* = 0x00000040'u32
    IN_MOVED_TO* = 0x00000080'u32

  type
    InotifyEvent* = object
      wd*: cint
      mask*: uint32
      cookie*: uint32
      len*: uint32
      name*: UncheckedArray[char]

  proc inotify_init1*(flags: cint): cint
    {.importc, header: "<sys/inotify.h>".}
  
  proc inotify_add_watch*(fd: cint, pathname: cstring, mask: uint32): cint
    {.importc, header: "<sys/inotify.h>".}
  
  proc inotify_rm_watch*(fd: cint, wd: cint): cint
    {.importc, header: "<sys/inotify.h>".}

  type
    FileWatcher* = ref object
      fd*: cint
      watches*: seq[(cint, string)]  # wd -> path

  proc newFileWatcher*(): FileWatcher =
    let fd = inotify_init1(0)
    if fd == -1: raiseOSError(osLastError())
    FileWatcher(fd: fd)

  proc watch*(fw: FileWatcher, path: string,
              mask = IN_CREATE or IN_MODIFY or IN_DELETE): cint =
    result = inotify_add_watch(fw.fd, path.cstring, mask)
    if result == -1: raiseOSError(osLastError())
    fw.watches.add((result, path))

  proc readEvents*(fw: FileWatcher): seq[tuple[path, name: string, mask: uint32]] =
    var buf: array[4096, byte]
    let n = posix.read(fw.fd, addr buf[0], buf.len)
    if n <= 0: return
    
    var i = 0
    while i < n:
      let ev = cast[ptr InotifyEvent](addr buf[i])
      let name = if ev.len > 0: $cast[cstring](addr ev.name[0]) else: ""
      
      for (wd, path) in fw.watches:
        if wd == ev.wd:
          result.add((path, name, ev.mask))
          break
      
      i += sizeof(InotifyEvent) + ev.len.int

when isMainModule:
  when defined(linux):
    # Test file watcher
    let fw = newFileWatcher()
    let wd = fw.watch("/tmp")
    echo &"Watching /tmp (wd={wd})"
    echo "Creating test file..."
    writeFile("/tmp/test_inotify.txt", "test")
    
    let events = fw.readEvents()
    for ev in events:
      echo &"Event: {ev.path}/{ev.name} mask=0x{ev.mask:08x}"
```

---

## 3. Dynamic Library Loading

```nim
# dynlib_advanced.nim
# Dynamic library management

import std/[dynlib, strformat, os, tables]

# Generic plugin loader with version checking
type
  PluginVersion* = object
    major*, minor*, patch*: int
  
  PluginMetadata* = object
    name*: string
    version*: PluginVersion
    description*: string
    apiVersion*: int

  DynPlugin* = ref object
    handle*: LibHandle
    path*: string
    meta*: PluginMetadata
    symbols*: Table[string, pointer]

const CURRENT_API_VERSION* = 1

proc loadPlugin*(path: string): DynPlugin =
  let handle = loadLib(path)
  if handle == nil:
    raise newException(IOError, &"Failed to load {path}")
  
  # Load metadata
  let getMetaFn = cast[proc(): ptr PluginMetadata {.cdecl.}](
    handle.symAddr("plugin_get_metadata")
  )
  if getMetaFn == nil:
    handle.unloadLib()
    raise newException(IOError, "Missing plugin_get_metadata")
  
  let meta = getMetaFn()
  if meta.apiVersion != CURRENT_API_VERSION:
    handle.unloadLib()
    raise newException(IOError, &"API version mismatch: got {meta.apiVersion}, expected {CURRENT_API_VERSION}")
  
  DynPlugin(handle: handle, path: path, meta: meta[])

proc getSymbol*[T](p: DynPlugin, name: string): T =
  if name in p.symbols:
    return cast[T](p.symbols[name])
  
  let sym = p.handle.symAddr(name)
  if sym == nil:
    raise newException(KeyError, &"Symbol not found: {name}")
  
  p.symbols[name] = sym
  cast[T](sym)

proc close*(p: DynPlugin) =
  p.handle.unloadLib()

# Example plugin interface
type
  TransformFn* = proc(input: cstring, output: ptr cstring, outLen: ptr int): bool {.cdecl.}
  
  Transformer* = ref object
    plugin*: DynPlugin
    transform*: TransformFn

proc newTransformer*(pluginPath: string): Transformer =
  let plugin = loadPlugin(pluginPath)
  let fn = plugin.getSymbol[TransformFn]("transform")
  Transformer(plugin: plugin, transform: fn)

proc apply*(t: Transformer, input: string): string =
  var output: cstring
  var outLen: int
  
  if t.transform(input.cstring, addr output, addr outLen):
    result = newString(outLen)
    copyMem(addr result[0], output, outLen)
  else:
    raise newException(IOError, "Transform failed")

# Lazy symbol resolution
type
  LazyLib* = ref object
    path*: string
    handle*: LibHandle
    loaded*: bool

proc newLazyLib*(path: string): LazyLib =
  LazyLib(path: path, loaded: false)

proc resolve*(lib: LazyLib, sym: string): pointer =
  if not lib.loaded:
    lib.handle = loadLib(lib.path)
    if lib.handle == nil:
      raise newException(IOError, &"Failed to load {lib.path}")
    lib.loaded = true
  
  result = lib.handle.symAddr(sym)
  if result == nil:
    raise newException(KeyError, &"Symbol not found: {sym}")

when isMainModule:
  echo "Dynamic library loading example"
  echo "In production: load shared libraries (.so/.dll) with plugin interface"
  
  # Show available standard libs
  when defined(linux):
    echo "Testing libm.so..."
    let libm = loadLib("libm.so.6")
    if libm != nil:
      let sinFn = cast[proc(x: float64): float64 {.cdecl.}](libm.symAddr("sin"))
      if sinFn != nil:
        echo &"sin(3.14159) = {sinFn(3.14159):.5f}"
      libm.unloadLib()
```

---

## สรุป

| Topic | Key Concept |
|-------|-------------|
| C++ interop | `importcpp`, `header` pragmas |
| epoll | Edge-triggered async I/O |
| inotify | File system events |
| dynlib | Runtime symbol resolution |

---

**Next**: [Part 97 - Resilience Engineering](../advanced/part97_resilience.md)
