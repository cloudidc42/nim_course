# Part 82 - WebAssembly with Nim

## บทนำ

Nim สามารถ compile เป็น WebAssembly ผ่าน:
1. Emscripten (ผ่าน C backend)
2. Direct WASM target (experimental)
3. JavaScript backend (สำหรับ browser)

---

## 1. Nim → JavaScript (JS Backend)

```nim
# jsapp.nim
# Compile ด้วย: nim js -d:release -o:app.js jsapp.nim

import std/[jsffi, dom, asyncjs, strformat]

# JavaScript FFI
proc alert*(msg: cstring) {.importjs: "alert(#)".}
proc console_log*(msg: cstring) {.importjs: "console.log(#)".}

# DOM manipulation
proc getElementById*(id: cstring): Element {.importjs: "document.getElementById(#)".}
proc querySelector*(sel: cstring): Element {.importjs: "document.querySelector(#)".}
proc createElement*(tag: cstring): Element {.importjs: "document.createElement(#)".}

proc setInnerHTML*(el: Element, html: cstring) {.importjs: "#.innerHTML = #".}
proc getInnerHTML*(el: Element): cstring {.importjs: "#.innerHTML".}
proc appendChild*(parent, child: Element) {.importjs: "#.appendChild(#)".}
proc addEventListener*(el: Element, event, handler: cstring) {.importjs: "#.addEventListener(#, #)".}
proc setAttribute*(el: Element, name, value: cstring) {.importjs: "#.setAttribute(#, #)".}
proc getValue*(el: Element): cstring {.importjs: "#.value".}
proc setValue*(el: Element, v: cstring) {.importjs: "#.value = #".}
proc addClass*(el: Element, cls: cstring) {.importjs: "#.classList.add(#)".}
proc removeClass*(el: Element, cls: cstring) {.importjs: "#.classList.remove(#)".}

# Fetch API
type
  Response* = ref object of JsObject
  FetchOptions* = ref object of JsObject

proc fetch*(url: cstring): Future[Response] {.importjs: "fetch(#)".}
proc json*(r: Response): Future[JsObject] {.importjs: "#.json()".}
proc text*(r: Response): Future[cstring] {.importjs: "#.text()".}

# LocalStorage
proc localStorageSet*(key, value: cstring) {.importjs: "localStorage.setItem(#, #)".}
proc localStorageGet*(key: cstring): cstring {.importjs: "localStorage.getItem(#)".}

# Timer
proc setTimeout*(fn: proc(), ms: int) {.importjs: "setTimeout(#, #)".}
proc setInterval*(fn: proc(), ms: int): int {.importjs: "setInterval(#, #)".}
proc clearInterval*(id: int) {.importjs: "clearInterval(#)".}

# Example: Counter app
var count = 0

proc updateDisplay() =
  let el = getElementById("counter")
  if not el.isNil:
    setInnerHTML(el, cstring($count))

proc increment() =
  inc count
  updateDisplay()
  localStorageSet("count", cstring($count))

proc decrement() =
  dec count
  updateDisplay()

proc reset() =
  count = 0
  updateDisplay()

proc initApp() =
  # Restore from localStorage
  let saved = localStorageGet("count")
  if not saved.isNil and $saved != "null":
    try: count = parseInt($saved)
    except: count = 0
  
  updateDisplay()
  
  # Wire up buttons
  let incBtn = getElementById("inc")
  let decBtn = getElementById("dec")
  let resetBtn = getElementById("reset")
  
  if not incBtn.isNil:
    incBtn.addEventListener("click", "nim_increment")
  if not decBtn.isNil:
    decBtn.addEventListener("click", "nim_decrement")
  if not resetBtn.isNil:
    resetBtn.addEventListener("click", "nim_reset")

# Export functions for JavaScript
proc nim_increment*() {.exportc.} = increment()
proc nim_decrement*() {.exportc.} = decrement()
proc nim_reset*() {.exportc.} = reset()

# Async fetch example
proc fetchData*(url: cstring) {.async.} =
  try:
    let response = await fetch(url)
    let data = await response.text()
    console_log(data)
  except JsError as e:
    console_log(cstring("Fetch error: " & $e.message))

when isMainModule:
  initApp()
```

---

## 2. HTML Template สำหรับ Nim JS App

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Nim Counter App</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      min-height: 100vh;
      background: #1a1a2e;
      color: #eee;
    }
    
    .counter {
      font-size: 5rem;
      font-weight: bold;
      color: #00d2ff;
      margin: 2rem;
      min-width: 6rem;
      text-align: center;
    }
    
    .buttons {
      display: flex;
      gap: 1rem;
    }
    
    button {
      padding: 0.8rem 2rem;
      font-size: 1.2rem;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      transition: transform 0.1s;
    }
    
    button:active { transform: scale(0.95); }
    
    #dec { background: #e74c3c; color: white; }
    #inc { background: #2ecc71; color: white; }
    #reset { background: #3498db; color: white; }
  </style>
</head>
<body>
  <h1>Nim Counter</h1>
  <div id="counter" class="counter">0</div>
  <div class="buttons">
    <button id="dec" onclick="nim_decrement()">-</button>
    <button id="reset" onclick="nim_reset()">Reset</button>
    <button id="inc" onclick="nim_increment()">+</button>
  </div>
  <script src="app.js"></script>
</body>
</html>
```

---

## 3. Nim → WASM ผ่าน Emscripten

```nim
# wasm_lib.nim
# Compile ด้วย:
# nim c --cpu:wasm32 --os:emscripten -d:release \
#   --passC:"-O3 -s EXPORTED_FUNCTIONS=['_add','_fib','_sort']" \
#   -o:lib.js wasm_lib.nim

# Export functions สำหรับ WASM
proc add*(a, b: int32): int32 {.exportc, cdecl.} =
  a + b

proc fib*(n: int32): int32 {.exportc, cdecl.} =
  if n <= 1: return n
  var a, b = 1
  for _ in 2..<n:
    let c = a + b
    a = b
    b = c
  b

proc sortArray*(arr: ptr UncheckedArray[int32], len: int32) {.exportc, cdecl.} =
  # In-place bubble sort (simple example)
  for i in 0..<len:
    for j in 0..<len-i-1:
      if arr[j] > arr[j+1]:
        swap(arr[j], arr[j+1])

# Compute-heavy: Mandelbrot set
proc mandelbrot*(x0, y0: float64, maxIter: int32): int32 {.exportc, cdecl.} =
  var x = 0.0
  var y = 0.0
  result = 0
  while x*x + y*y <= 4.0 and result < maxIter:
    let xtemp = x*x - y*y + x0
    y = 2.0*x*y + y0
    x = xtemp
    inc result

proc renderMandelbrot*(
    pixelsPtr: ptr UncheckedArray[uint32],
    width, height, maxIter: int32,
    xMin, xMax, yMin, yMax: float64
) {.exportc, cdecl.} =
  for py in 0..<height:
    for px in 0..<width:
      let x0 = xMin + (xMax - xMin) * px.float / width.float
      let y0 = yMin + (yMax - yMin) * py.float / height.float
      let iter = mandelbrot(x0, y0, maxIter)
      
      # Color based on iteration count
      let pct = iter.float / maxIter.float
      let r = uint32(9.0 * (1.0 - pct) * pct * pct * pct * 255.0)
      let g = uint32(15.0 * (1.0 - pct) * (1.0 - pct) * pct * pct * 255.0)
      let b = uint32(8.5 * (1.0 - pct) * (1.0 - pct) * (1.0 - pct) * pct * 255.0)
      
      pixelsPtr[py * width + px] = (0xFF shl 24) or (b shl 16) or (g shl 8) or r
```

---

## 4. JavaScript Glue Code

```javascript
// wasm_loader.js
// โหลดและใช้ WASM module จาก Nim

async function loadNimWasm() {
  const response = await fetch('lib.wasm');
  const bytes = await response.arrayBuffer();
  
  const memory = new WebAssembly.Memory({ initial: 16, maximum: 256 });
  
  const importObject = {
    env: {
      memory,
      // Nim runtime functions
      __stack_pointer: new WebAssembly.Global({ value: 'i32', mutable: true }, 65536),
    }
  };
  
  const { instance } = await WebAssembly.instantiate(bytes, importObject);
  const { exports } = instance;
  
  return {
    add: (a, b) => exports.add(a, b),
    fib: (n) => exports.fib(n),
    
    renderMandelbrot: (width, height, maxIter, bounds) => {
      const pixels = new Uint32Array(memory.buffer, 0, width * height);
      exports.renderMandelbrot(
        0, width, height, maxIter,
        bounds.xMin, bounds.xMax, bounds.yMin, bounds.yMax
      );
      return new Uint8ClampedArray(memory.buffer, 0, width * height * 4);
    }
  };
}

// Canvas rendering
async function renderToCanvas(canvasId) {
  const nim = await loadNimWasm();
  const canvas = document.getElementById(canvasId);
  const ctx = canvas.getContext('2d');
  const { width, height } = canvas;
  
  const pixelData = nim.renderMandelbrot(width, height, 256, {
    xMin: -2.5, xMax: 1.0,
    yMin: -1.25, yMax: 1.25
  });
  
  const imageData = new ImageData(pixelData, width, height);
  ctx.putImageData(imageData, 0, 0);
  
  console.log(`Rendered ${width}x${height} Mandelbrot set`);
}
```

---

## 5. Nim JS: React-like Component

```nim
# component.nim
# Simple component system สำหรับ Nim JS

import std/[jsffi, dom, strutils, tables]

type
  Component* = ref object of JsObject
    render*: proc(): string
    state*: JsObject
    el*: Element

  VNode* = object
    tag*: string
    attrs*: seq[(string, string)]
    children*: seq[VNode]
    text*: string

proc h*(tag: string, attrs: seq[(string, string)] = @[],
    children: seq[VNode] = @[]): VNode =
  VNode(tag: tag, attrs: attrs, children: children)

proc text*(content: string): VNode =
  VNode(tag: "#text", text: content)

proc renderVNode*(vnode: VNode): cstring =
  if vnode.tag == "#text":
    return cstring(vnode.text)
  
  var html = &"<{vnode.tag}"
  for (k, v) in vnode.attrs:
    html &= &" {k}=\"{v}\""
  html &= ">"
  
  for child in vnode.children:
    html &= $renderVNode(child)
  
  html &= &"</{vnode.tag}>"
  cstring(html)

# Todo list component example
type
  Todo* = object
    id*: int
    text*: string
    done*: bool

var todos: seq[Todo]
var nextId = 1

proc renderTodos*(): cstring =
  var html = "<ul class='todo-list'>"
  for todo in todos:
    let doneClass = if todo.done: "done" else: ""
    let checked = if todo.done: "checked" else: ""
    html &= &"""<li class='{doneClass}'>
      <input type='checkbox' {checked} 
        onchange='nim_toggleTodo({todo.id})'>
      <span>{todo.text}</span>
      <button onclick='nim_deleteTodo({todo.id})'>×</button>
    </li>"""
  html &= "</ul>"
  cstring(html)

proc addTodo*(text: string) {.exportc.} =
  todos.add(Todo(id: nextId, text: text, done: false))
  inc nextId
  
  let container = getElementById("app")
  if not container.isNil:
    setInnerHTML(container, renderTodos())

proc toggleTodo*(id: int) {.exportc.} =
  for i, todo in todos:
    if todo.id == id:
      todos[i].done = not todos[i].done
      break
  
  let container = getElementById("app")
  if not container.isNil:
    setInnerHTML(container, renderTodos())

proc deleteTodo*(id: int) {.exportc.} =
  todos = todos.filterIt(it.id != id)
  let container = getElementById("app")
  if not container.isNil:
    setInnerHTML(container, renderTodos())

when isMainModule:
  addTodo("Learn Nim")
  addTodo("Build a WASM app")
  addTodo("Deploy to production")
```

---

## Build Script

```bash
#!/bin/bash
# build_wasm.sh

# JS backend (easiest)
nim js -d:release --opt:size -o:dist/app.js src/jsapp.nim

# WASM ผ่าน Emscripten (ต้องติดตั้ง emsdk)
# source ~/emsdk/emsdk_env.sh
# nim c \
#   --cpu:wasm32 \
#   --os:emscripten \
#   -d:release \
#   --opt:size \
#   --passL:"-s EXPORTED_FUNCTIONS=['_add','_fib','_renderMandelbrot']" \
#   --passL:"-s ALLOW_MEMORY_GROWTH=1" \
#   --passL:"-s MODULARIZE=1" \
#   -o:dist/lib.js \
#   src/wasm_lib.nim

echo "Build complete!"
```

---

## สรุป

| Target | วิธี | ใช้เมื่อ |
|--------|------|---------|
| JS backend | `nim js` | Browser apps, DOM |
| WASM/Emscripten | `nim c --cpu:wasm32` | High-performance compute |
| Node.js | `nim js --backend:js` | Server-side JS |

---

**Next**: [Part 83 - Plugin Architecture & Dynamic Loading](../advanced/part83_plugins.md)
