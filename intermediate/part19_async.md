# Part 19: Async/Await - การเขียนโปรแกรมแบบ Asynchronous

## Async เบื้องต้น

```nim
import asyncdispatch, asynchttpclient, times

# proc async เบื้องต้น
proc hello(): Future[string] {.async.} =
  await sleepAsync(100)  # non-blocking sleep 100ms
  return "Hello, Async World!"

proc main() {.async.} =
  let msg = await hello()
  echo msg

waitFor main()

# หลาย async procs พร้อมกัน
proc task1(): Future[string] {.async.} =
  await sleepAsync(200)
  return "Task 1 done"

proc task2(): Future[string] {.async.} =
  await sleepAsync(100)
  return "Task 2 done"

proc runParallel() {.async.} =
  # รันพร้อมกัน
  let (r1, r2) = await (task1(), task2())
  echo r1, ", ", r2

waitFor runParallel()
```

## Futures

```nim
import asyncdispatch, futures

# สร้าง Future manually
proc fetchData(id: int): Future[string] {.async.} =
  # simulate network call
  await sleepAsync(id * 50)
  return "Data for id=" & $id

# Callback style
proc fetchWithCallback(id: int): Future[string] =
  result = newFuture[string]("fetchWithCallback")
  let fut = fetchData(id)
  fut.addCallback(proc(f: Future[string]) =
    if f.failed:
      result.fail(f.readError())
    else:
      result.complete(f.read())
  )

# Future chaining
proc processData(data: string): Future[string] {.async.} =
  await sleepAsync(10)
  return "Processed: " & data.toUpper()

proc pipeline() {.async.} =
  let data = await fetchData(1)
  let processed = await processData(data)
  echo processed

waitFor pipeline()

# all - รอทุก futures
proc fetchMany() {.async.} =
  var futures: seq[Future[string]] = @[]
  for i in 1..5:
    futures.add(fetchData(i))
  
  let results = await all(futures)
  for r in results:
    echo r

waitFor fetchMany()
```

## Async HTTP Client

```nim
import asyncdispatch, asynchttpclient, json

proc fetchJson(url: string): Future[JsonNode] {.async.} =
  let client = newAsyncHttpClient()
  try:
    let response = await client.get(url)
    let body = await response.body
    return parseJson(body)
  finally:
    client.close()

proc fetchMultipleUrls(urls: seq[string]) {.async.} =
  var futs: seq[Future[JsonNode]] = @[]
  for url in urls:
    futs.add(fetchJson(url))
  
  let results = await all(futs)
  for r in results:
    echo r["title"].getStr()  # example field

# Async HTTP with timeout
proc fetchWithTimeout(url: string, ms: int): Future[string] {.async.} =
  let client = newAsyncHttpClient()
  client.headers = newHttpHeaders({"User-Agent": "NimBot/1.0"})
  
  try:
    let responseFuture = client.get(url)
    let timeoutFuture = sleepAsync(ms)
    
    # Wait for whichever comes first
    let winner = await race(responseFuture, timeoutFuture)
    
    if responseFuture.finished:
      let resp = await responseFuture
      return await resp.body
    else:
      return "Timeout!"
  finally:
    client.close()
```

## Async Server

```nim
import asyncdispatch, asyncnet, strutils, strformat

type
  Client = object
    socket: AsyncSocket
    id: int

var clients: seq[Client] = @[]
var nextClientId = 1

proc handleClient(client: Client) {.async.} =
  let socket = client.socket
  echo &"Client {client.id} connected"
  
  try:
    while true:
      let line = await socket.recvLine()
      if line.len == 0:
        break  # disconnected
      
      echo &"Client {client.id}: {line}"
      
      # Echo back
      await socket.send("Echo: " & line & "\r\n")
      
      # Broadcast to all clients
      for c in clients:
        if c.id != client.id:
          try:
            await c.socket.send(&"[{client.id}]: {line}\r\n")
          except:
            discard
  except:
    echo &"Client {client.id} error: {getCurrentExceptionMsg()}"
  finally:
    socket.close()
    clients = clients.filterIt(it.id != client.id)
    echo &"Client {client.id} disconnected"

proc serve(port: Port) {.async.} =
  let server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(port)
  server.listen()
  
  echo &"Server listening on port {int(port)}"
  
  while true:
    let socket = await server.accept()
    let client = Client(socket: socket, id: nextClientId)
    inc nextClientId
    clients.add(client)
    asyncCheck handleClient(client)

# Run: waitFor serve(Port(8080))
```

## Async File I/O

```nim
import asyncdispatch, asyncio, os

proc readFileAsync(path: string): Future[string] {.async.} =
  # Nim's async file I/O
  let f = openAsync(path)
  try:
    result = await f.readAll()
  finally:
    f.close()

proc writeFileAsync(path, content: string): Future[void] {.async.} =
  let f = openAsync(path, fmWrite)
  try:
    await f.write(content)
  finally:
    f.close()

proc processFiles(paths: seq[string]) {.async.} =
  var readFuts: seq[Future[string]] = @[]
  for path in paths:
    readFuts.add(readFileAsync(path))
  
  let contents = await all(readFuts)
  
  for i, content in contents:
    echo &"File {paths[i]}: {content.len} bytes"

# waitFor processFiles(@["a.txt", "b.txt"])
```

## Error Handling ใน Async

```nim
import asyncdispatch

proc riskyAsync(): Future[int] {.async.} =
  await sleepAsync(50)
  raise newException(ValueError, "Something went wrong!")
  return 42

proc safeAsync() {.async.} =
  try:
    let val = await riskyAsync()
    echo "Got: ", val
  except ValueError as e:
    echo "Caught: ", e.msg

proc withFallback(primary, fallback: Future[string]) {.async.} =
  try:
    let r = await primary
    echo "Primary succeeded: ", r
  except:
    echo "Primary failed, trying fallback..."
    let r = await fallback
    echo "Fallback: ", r

# Timeout pattern
proc withTimeout[T](fut: Future[T], ms: int): Future[Option[T]] {.async.} =
  import options
  let timer = sleepAsync(ms)
  
  await race(fut, timer)
  
  if fut.finished:
    if fut.failed:
      raise fut.readError()
    return some(fut.read())
  else:
    return none(T)

waitFor safeAsync()
```

## Practical: Async Web Scraper

```nim
import asyncdispatch, asynchttpclient, htmlparser, 
       xmltree, strutils, sequtils, tables

type
  PageInfo = object
    url: string
    title: string
    links: seq[string]

proc extractTitle(html: string): string =
  # Simple title extraction
  let start = html.find("<title>")
  let finish = html.find("</title>")
  if start >= 0 and finish > start:
    html[start + 7 ..< finish].strip()
  else:
    "No title"

proc extractLinks(html: string): seq[string] =
  result = @[]
  var pos = 0
  while true:
    let hrefPos = html.find("href=\"", pos)
    if hrefPos < 0: break
    
    let start = hrefPos + 6
    let finish = html.find("\"", start)
    if finish < 0: break
    
    let link = html[start ..< finish]
    if link.startsWith("http"):
      result.add(link)
    pos = finish

proc scrapePage(client: AsyncHttpClient, url: string): Future[PageInfo] {.async.} =
  try:
    let resp = await client.get(url)
    let body = await resp.body
    
    return PageInfo(
      url: url,
      title: extractTitle(body),
      links: extractLinks(body)
    )
  except:
    return PageInfo(url: url, title: "Error: " & getCurrentExceptionMsg())

proc scrapeMany(urls: seq[string]): Future[seq[PageInfo]] {.async.} =
  let client = newAsyncHttpClient()
  client.headers = newHttpHeaders({
    "User-Agent": "NimScraper/1.0",
    "Accept": "text/html"
  })
  
  try:
    var futs: seq[Future[PageInfo]] = @[]
    for url in urls:
      futs.add(scrapePage(client, url))
    
    return await all(futs)
  finally:
    client.close()

# Usage
proc main() {.async.} =
  let urls = @[
    "https://httpbin.org/get",
    "https://httpbin.org/ip",
    "https://httpbin.org/headers"
  ]
  
  echo "Scraping ", urls.len, " pages concurrently..."
  let pages = await scrapeMany(urls)
  
  for page in pages:
    echo "\nURL: ", page.url
    echo "Title: ", page.title
    echo "Links found: ", page.links.len

# waitFor main()
```

## สรุป Part 19

ในบทนี้เราได้เรียนรู้:
- ✅ async/await พื้นฐาน
- ✅ Future และ Promise patterns
- ✅ Async HTTP client
- ✅ Async TCP server
- ✅ Async file I/O
- ✅ Error handling ใน async
- ✅ Practical: Async web scraper

---

**Previous**: [Part 18 - Templates/Macros](part18_templates_macros.md)
**Next**: [Part 20 - Threads](part20_threads.md)
