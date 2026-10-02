# Part 45: WebSocket Real-time Apps

## WebSocket คืออะไร

```
HTTP: request -> response (จบเชื่อมต่อ)
WebSocket: full-duplex connection (ไม่จบเชื่อมต่อ)

HTTP Upgrade:
Client: GET /ws HTTP/1.1
        Upgrade: websocket
        Connection: Upgrade
Server: 101 Switching Protocols

ใช้สำหรับ:
- Chat applications
- Live dashboards
- Collaborative editing
- Gaming
- Real-time notifications
- Trading platforms
```

## WebSocket Server

```nim
# websocket_server.nim
import asynchttpserver, asyncdispatch, asyncnet, strutils, json
import websockets  # nimble install websockets

type
  Client = ref object
    socket: WebSocket
    id: string
    username: string
    room: string

var clients: seq[Client] = @[]

proc broadcast(msg: string, room: string = "", exclude: Client = nil) {.async.} =
  var disconnected: seq[int] = @[]
  for i, client in clients:
    if client == exclude: continue
    if room.len > 0 and client.room != room: continue
    try:
      await client.socket.send(msg)
    except:
      disconnected.add(i)
  
  # Remove disconnected clients
  for i in disconnected.reversed():
    clients.del(i)

proc handleClient(ws: WebSocket, req: Request) {.async.} =
  import random
  let client = Client(
    socket: ws,
    id: toHex(rand(0xFFFF), 4),
    room: "general"
  )
  clients.add(client)
  
  echo "[+] Client connected: ", client.id
  
  try:
    while ws.readyState == Open:
      let (opcode, data) = await ws.receivePacket()
      
      case opcode
      of Text:
        let msg = parseJson(data)
        let msgType = msg["type"].getStr()
        
        case msgType
        of "join":
          client.username = msg["username"].getStr()
          client.room = msg.getOrDefault("room", %"general").getStr()
          
          let notify = %*{
            "type": "system",
            "message": client.username & " joined " & client.room
          }
          await broadcast($notify, client.room, client)
        
        of "message":
          let outMsg = %*{
            "type": "message",
            "from": client.username,
            "message": msg["message"].getStr(),
            "timestamp": $now()
          }
          await broadcast($outMsg, client.room)
        
        of "ping":
          await ws.send($(%*{"type": "pong"}))
        
        of "typing":
          let notify = %*{
            "type": "typing",
            "username": client.username
          }
          await broadcast($notify, client.room, client)
        
        else:
          echo "Unknown message type: ", msgType
      
      of Binary:
        echo "Binary data received: ", data.len, " bytes"
      
      of Ping:
        await ws.send(data, Pong)
      
      of Close:
        break
      
      else: discard
  except:
    echo "[!] Client error: ", getCurrentExceptionMsg()
  
  # Remove client
  clients.keepItIf(it != client)
  
  let notify = %*{
    "type": "system",
    "message": client.username & " left"
  }
  await broadcast($notify, client.room)
  echo "[-] Client disconnected: ", client.id

proc handler(req: Request) {.async.} =
  if req.url.path == "/ws":
    let ws = await newWebSocket(req)
    await handleClient(ws, req)
  elif req.url.path == "/":
    let html = """
<!DOCTYPE html>
<html>
<head><title>Nim WebSocket Chat</title></head>
<body>
<h1>Real-time Chat</h1>
<input id="username" placeholder="Username" />
<input id="room" placeholder="Room" value="general" />
<button onclick="connect()">Connect</button>
<div id="messages" style="height:300px;overflow:auto;border:1px solid #ccc;padding:10px"></div>
<input id="msg" placeholder="Message" />
<button onclick="send()">Send</button>
<script>
let ws;
function connect() {
  const username = document.getElementById('username').value;
  const room = document.getElementById('room').value;
  ws = new WebSocket('ws://' + location.host + '/ws');
  ws.onopen = () => {
    ws.send(JSON.stringify({type:'join', username, room}));
  };
  ws.onmessage = (e) => {
    const data = JSON.parse(e.data);
    const div = document.getElementById('messages');
    div.innerHTML += '<p><b>' + (data.from||'System') + ':</b> ' + data.message + '</p>';
    div.scrollTop = div.scrollHeight;
  };
}
function send() {
  const msg = document.getElementById('msg');
  ws.send(JSON.stringify({type:'message', message:msg.value}));
  msg.value = '';
}
document.getElementById('msg').addEventListener('keypress', e => {
  if(e.key==='Enter') send();
});
</script>
</body>
</html>"""
    await req.respond(Http200, html, newHttpHeaders({"Content-Type": "text/html"}))
  else:
    await req.respond(Http404, "Not Found")

let server = newAsyncHttpServer()
waitFor server.serve(Port(8080), handler)
```

## WebSocket Client

```nim
# ws_client.nim
# WebSocket client สำหรับ test

import asyncdispatch, json, strutils
import websockets

proc wsClient(url: string) {.async.} =
  let ws = await newWebSocket(url)
  echo "Connected to: ", url
  
  # Send join message
  await ws.send($(%*{"type": "join", "username": "TestClient"}))
  
  # Read messages in background
  proc readLoop() {.async.} =
    while ws.readyState == Open:
      try:
        let msg = await ws.receiveStrPacket()
        echo "Received: ", msg
      except:
        break
  
  asyncCheck readLoop()
  
  # Send messages
  for i in 1..5:
    await ws.send($(%*{"type": "message", "message": "Hello #" & $i}))
    await sleepAsync(1000)
  
  await ws.close()

waitFor wsClient("ws://localhost:8080/ws")
```

## Live Dashboard

```nim
# live_dashboard.nim
# Real-time dashboard ด้วย WebSocket

import asynchttpserver, asyncdispatch, json, times, strutils, os
import websockets

var dashboardClients: seq[WebSocket] = @[]

proc sendMetrics() {.async.} =
  while true:
    let metrics = %*{
      "type": "metrics",
      "timestamp": $getTime(),
      "cpu": 45,    # In real app: get from /proc/stat or psutil
      "memory": 62,
      "requests": 1234,
      "latency": 23.5
    }
    
    var disconnected: seq[int] = @[]
    for i, ws in dashboardClients:
      try:
        if ws.readyState == Open:
          await ws.send($metrics)
        else:
          disconnected.add(i)
      except:
        disconnected.add(i)
    
    for i in disconnected.reversed():
      dashboardClients.del(i)
    
    await sleepAsync(2000)  # Update every 2 seconds

proc dashboardHandler(req: Request) {.async.} =
  if req.url.path == "/ws":
    let ws = await newWebSocket(req)
    dashboardClients.add(ws)
    
    while ws.readyState == Open:
      try:
        discard await ws.receiveStrPacket()
      except:
        break
    
    dashboardClients.keepItIf(it != ws)
  
  elif req.url.path == "/":
    let html = """
<!DOCTYPE html><html><head><title>Live Dashboard</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head><body>
<h1>Live Server Metrics</h1>
<canvas id="cpuChart" width="400" height="200"></canvas>
<div id="stats"></div>
<script>
const ws = new WebSocket('ws://' + location.host + '/ws');
const cpuData = [];
const ctx = document.getElementById('cpuChart').getContext('2d');
const chart = new Chart(ctx, {
  type: 'line',
  data: { labels: [], datasets: [{label: 'CPU %', data: cpuData, borderColor: 'rgb(75,192,192)'}] },
  options: { animation: false }
});
ws.onmessage = (e) => {
  const d = JSON.parse(e.data);
  if(d.type === 'metrics') {
    chart.data.labels.push(new Date().toLocaleTimeString());
    cpuData.push(d.cpu);
    if(cpuData.length > 20) { cpuData.shift(); chart.data.labels.shift(); }
    chart.update();
    document.getElementById('stats').innerHTML =
      `CPU: ${d.cpu}% | Memory: ${d.memory}% | Req: ${d.requests} | Latency: ${d.latency}ms`;
  }
};
</script></body></html>"""
    await req.respond(Http200, html, newHttpHeaders({"Content-Type": "text/html"}))

asyncCheck sendMetrics()
let server = newAsyncHttpServer()
waitFor server.serve(Port(8081), dashboardHandler)
```

## สรุป Part 45

ในบทนี้เราได้เรียนรู้:
- ␅ WebSocket protocol สรุป
- ␅ Full WebSocket chat server
- ␅ WebSocket client
- ␅ Live dashboard ด้วย Chart.js
- ␅ Multi-room support

---

**Previous**: [Part 44 - Malware Analysis](../security/part44_malware_analysis.md)
**Next**: [Part 46 - Microservices](part46_microservices.md)
