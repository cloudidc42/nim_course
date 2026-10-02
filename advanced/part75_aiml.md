# Part 75 - AI/ML Integration in Nim

## บทนำ

Nim เหมาะกับ AI/ML เพราะ performance ใกล้เคียง C แต่มี syntax สะอาดกว่า
เรียนรู้: neural networks จาก scratch, tensor operations, LLM API integration,
ONNX model inference, และ embedding search

---

## 1. Neural Network จาก Scratch

```nim
# neural_net.nim
# Feedforward neural network ด้วย backpropagation

import std/[math, random, strformat, sequtils]

type
  Tensor* = seq[float64]  # Flat 1D tensor
  
  Layer* = object
    weights*: seq[seq[float64]]  # [out_features][in_features]
    biases*: seq[float64]
    output*: Tensor
    delta*: Tensor

  ActivationFn* = enum
    afReLU
    afSigmoid
    afTanh
    afSoftmax
    afLinear

  NeuralNetwork* = ref object
    layers*: seq[Layer]
    activations*: seq[ActivationFn]
    learningRate*: float64
    losses*: seq[float64]

# Activation functions
proc relu*(x: float64): float64 = max(0.0, x)
proc reluGrad*(x: float64): float64 = if x > 0: 1.0 else: 0.0

proc sigmoid*(x: float64): float64 = 1.0 / (1.0 + exp(-x))
proc sigmoidGrad*(x: float64): float64 = sigmoid(x) * (1.0 - sigmoid(x))

proc tanhAct*(x: float64): float64 = tanh(x)
proc tanhGrad*(x: float64): float64 = 1.0 - tanh(x) * tanh(x)

proc softmax*(x: Tensor): Tensor =
  let maxVal = x.foldl(max(a, b))
  let exps = x.mapIt(exp(it - maxVal))
  let sum = exps.foldl(a + b)
  exps.mapIt(it / sum)

proc apply*(fn: ActivationFn, x: float64): float64 =
  case fn
  of afReLU: relu(x)
  of afSigmoid: sigmoid(x)
  of afTanh: tanhAct(x)
  of afLinear: x
  of afSoftmax: x  # Applied separately

proc gradient*(fn: ActivationFn, x: float64): float64 =
  case fn
  of afReLU: reluGrad(x)
  of afSigmoid: sigmoidGrad(x)
  of afTanh: tanhGrad(x)
  of afLinear, afSoftmax: 1.0

# Layer creation
proc newLayer*(inFeatures, outFeatures: int): Layer =
  randomize()
  # Xavier initialization
  let std = sqrt(2.0 / float(inFeatures + outFeatures))
  var weights = newSeq[seq[float64]](outFeatures)
  for i in 0..<outFeatures:
    weights[i] = newSeq[float64](inFeatures)
    for j in 0..<inFeatures:
      weights[i][j] = rand(1.0) * 2 * std - std
  
  Layer(
    weights: weights,
    biases: newSeq[float64](outFeatures),
    output: newSeq[float64](outFeatures),
    delta: newSeq[float64](outFeatures)
  )

proc newNetwork*(layerSizes: seq[int], activations: seq[ActivationFn],
    learningRate = 0.01): NeuralNetwork =
  var layers: seq[Layer]
  for i in 0..<layerSizes.len - 1:
    layers.add(newLayer(layerSizes[i], layerSizes[i+1]))
  
  NeuralNetwork(layers: layers, activations: activations, learningRate: learningRate)

# Forward pass
proc forward*(net: NeuralNetwork, input: Tensor): Tensor =
  var current = input
  
  for i, layer in net.layers:
    var output = newSeq[float64](layer.biases.len)
    
    for j in 0..<output.len:
      var sum = layer.biases[j]
      for k in 0..<current.len:
        sum += layer.weights[j][k] * current[k]
      output[j] = sum
    
    # Apply activation
    if net.activations[i] == afSoftmax:
      output = softmax(output)
    else:
      output = output.mapIt(net.activations[i].apply(it))
    
    net.layers[i].output = output
    current = output
  
  current

# Backward pass (backpropagation)
proc backward*(net: NeuralNetwork, input: Tensor, target: Tensor) =
  # Compute output layer delta
  let lastIdx = net.layers.len - 1
  let output = net.layers[lastIdx].output
  
  for j in 0..<output.len:
    let error = output[j] - target[j]
    let grad = net.activations[lastIdx].gradient(output[j])
    net.layers[lastIdx].delta[j] = error * grad
  
  # Propagate deltas backwards
  for i in countdown(lastIdx - 1, 0):
    let nextLayer = net.layers[i + 1]
    let currLayer = net.layers[i]
    
    for j in 0..<currLayer.output.len:
      var sum = 0.0
      for k in 0..<nextLayer.delta.len:
        sum += nextLayer.weights[k][j] * nextLayer.delta[k]
      currLayer.delta[j] = sum * net.activations[i].gradient(currLayer.output[j])
  
  # Update weights
  var prev = input
  for i, layer in net.layers:
    for j in 0..<layer.biases.len:
      layer.biases[j] -= net.learningRate * layer.delta[j]
      for k in 0..<prev.len:
        layer.weights[j][k] -= net.learningRate * layer.delta[j] * prev[k]
    prev = layer.output

proc mseLoss*(output, target: Tensor): float64 =
  var sum = 0.0
  for i in 0..<output.len:
    let diff = output[i] - target[i]
    sum += diff * diff
  sum / float(output.len)

proc train*(net: NeuralNetwork, inputs, targets: seq[Tensor], epochs = 100) =
  for epoch in 1..epochs:
    var totalLoss = 0.0
    
    for i in 0..<inputs.len:
      let output = net.forward(inputs[i])
      totalLoss += mseLoss(output, targets[i])
      net.backward(inputs[i], targets[i])
    
    totalLoss /= float(inputs.len)
    net.losses.add(totalLoss)
    
    if epoch mod 10 == 0:
      echo &"Epoch {epoch}: loss={totalLoss:.6f}"

# Example: XOR problem
when isMainModule:
  let net = newNetwork(
    @[2, 4, 4, 1],  # 2 inputs, 2 hidden layers of 4, 1 output
    @[afReLU, afReLU, afSigmoid],
    learningRate = 0.1
  )
  
  let inputs  = @[@[0.0, 0.0], @[0.0, 1.0], @[1.0, 0.0], @[1.0, 1.0]]
  let targets = @[@[0.0], @[1.0], @[1.0], @[0.0]]
  
  net.train(inputs, targets, epochs = 1000)
  
  for i in 0..<inputs.len:
    let pred = net.forward(inputs[i])
    echo &"XOR({inputs[i][0]}, {inputs[i][1]}) = {pred[0]:.3f}"
```

---

## 2. LLM API Integration (OpenAI / Claude)

```nim
# llm_client.nim
# API client สำหรับ LLM services

import std/[asyncdispatch, httpclient, json, strformat, options, tables]

type
  Message* = object
    role*: string    # "system", "user", "assistant"
    content*: string

  ChatRequest* = object
    model*: string
    messages*: seq[Message]
    maxTokens*: int
    temperature*: float
    stream*: bool

  ChatResponse* = object
    id*: string
    model*: string
    content*: string
    inputTokens*: int
    outputTokens*: int
    stopReason*: string

  LLMClient* = ref object
    apiKey*: string
    baseUrl*: string
    model*: string
    httpClient*: AsyncHttpClient

proc newOpenAIClient*(apiKey: string, model = "gpt-4o"): LLMClient =
  let httpClient = newAsyncHttpClient()
  httpClient.headers = newHttpHeaders({
    "Authorization": &"Bearer {apiKey}",
    "Content-Type": "application/json"
  })
  LLMClient(
    apiKey: apiKey,
    baseUrl: "https://api.openai.com/v1",
    model: model,
    httpClient: httpClient
  )

proc newAnthropicClient*(apiKey: string, model = "claude-sonnet-4-6"): LLMClient =
  let httpClient = newAsyncHttpClient()
  httpClient.headers = newHttpHeaders({
    "x-api-key": apiKey,
    "anthropic-version": "2023-06-01",
    "Content-Type": "application/json"
  })
  LLMClient(
    apiKey: apiKey,
    baseUrl: "https://api.anthropic.com/v1",
    model: model,
    httpClient: httpClient
  )

proc chat*(client: LLMClient, messages: seq[Message],
    maxTokens = 1024, temperature = 0.7): Future[ChatResponse] {.async.} =
  let body = %*{
    "model": client.model,
    "max_tokens": maxTokens,
    "temperature": temperature,
    "messages": messages.mapIt(%*{"role": it.role, "content": it.content})
  }
  
  let url = if "anthropic" in client.baseUrl:
    &"{client.baseUrl}/messages"
  else:
    &"{client.baseUrl}/chat/completions"
  
  let resp = await client.httpClient.post(url, body = $body)
  let data = parseJson(await resp.body)
  
  if "anthropic" in client.baseUrl:
    # Anthropic response format
    return ChatResponse(
      id: data["id"].getStr(),
      model: data["model"].getStr(),
      content: data["content"][0]["text"].getStr(),
      inputTokens: data["usage"]["input_tokens"].getInt(),
      outputTokens: data["usage"]["output_tokens"].getInt(),
      stopReason: data["stop_reason"].getStr()
    )
  else:
    # OpenAI response format
    return ChatResponse(
      id: data["id"].getStr(),
      model: data["model"].getStr(),
      content: data["choices"][0]["message"]["content"].getStr(),
      inputTokens: data["usage"]["prompt_tokens"].getInt(),
      outputTokens: data["usage"]["completion_tokens"].getInt(),
      stopReason: data["choices"][0]["finish_reason"].getStr()
    )

# Streaming response
proc chatStream*(client: LLMClient, messages: seq[Message],
    onChunk: proc(chunk: string)): Future[void] {.async.} =
  let body = %*{
    "model": client.model,
    "max_tokens": 2048,
    "stream": true,
    "messages": messages.mapIt(%*{"role": it.role, "content": it.content})
  }
  
  let url = &"{client.baseUrl}/messages"
  let resp = await client.httpClient.post(url, body = $body)
  
  while true:
    let line = await resp.bodyStream.readLine()
    if line.len == 0: break
    
    if line.startsWith("data: "):
      let data = line[6..^1]
      if data == "[DONE]": break
      
      try:
        let json = parseJson(data)
        if json.hasKey("delta") and json["delta"].hasKey("text"):
          onChunk(json["delta"]["text"].getStr())
      except: discard

# Conversation manager
type
  Conversation* = ref object
    client*: LLMClient
    messages*: seq[Message]
    systemPrompt*: string

proc newConversation*(client: LLMClient, systemPrompt = ""): Conversation =
  var messages: seq[Message]
  if systemPrompt.len > 0:
    messages.add(Message(role: "system", content: systemPrompt))
  
  Conversation(client: client, messages: messages, systemPrompt: systemPrompt)

proc say*(conv: Conversation, userMessage: string,
    maxTokens = 1024): Future[string] {.async.} =
  conv.messages.add(Message(role: "user", content: userMessage))
  
  let resp = await conv.client.chat(conv.messages, maxTokens = maxTokens)
  
  conv.messages.add(Message(role: "assistant", content: resp.content))
  return resp.content

# RAG (Retrieval Augmented Generation)
type
  Document* = object
    id*: string
    content*: string
    embedding*: seq[float64]
    metadata*: Table[string, string]

  VectorStore* = ref object
    documents*: seq[Document]
    client*: LLMClient

proc embed*(client: LLMClient, text: string): Future[seq[float64]] {.async.} =
  ## Get text embedding from API
  let body = %*{"model": "text-embedding-3-small", "input": text}
  let url = &"{client.baseUrl}/embeddings"
  let resp = await client.httpClient.post(url, body = $body)
  let data = parseJson(await resp.body)
  
  data["data"][0]["embedding"].getElems().mapIt(it.getFloat())

proc cosineSimilarity*(a, b: seq[float64]): float64 =
  var dotProduct = 0.0
  var normA = 0.0
  var normB = 0.0
  
  for i in 0..<min(a.len, b.len):
    dotProduct += a[i] * b[i]
    normA += a[i] * a[i]
    normB += b[i] * b[i]
  
  if normA == 0.0 or normB == 0.0: return 0.0
  dotProduct / (sqrt(normA) * sqrt(normB))

proc search*(store: VectorStore, query: string,
    topK = 3): Future[seq[Document]] {.async.} =
  let queryEmbed = await store.client.embed(query)
  
  var scored: seq[tuple[score: float64, doc: Document]]
  for doc in store.documents:
    let score = cosineSimilarity(queryEmbed, doc.embedding)
    scored.add((score: score, doc: doc))
  
  scored.sort(proc(a, b: tuple[score: float64, doc: Document]): int =
    cmp(b.score, a.score)
  )
  
  scored[0..<min(topK, scored.len)].mapIt(it.doc)

proc ragQuery*(store: VectorStore, question: string): Future[string] {.async.} =
  let docs = await store.search(question, topK = 3)
  
  let context = docs.mapIt(it.content).join("\n\n---\n\n")
  
  let messages = @[
    Message(role: "system", content: "Answer based on the provided context."),
    Message(role: "user", content: &"Context:\n{context}\n\nQuestion: {question}")
  ]
  
  let resp = await store.client.chat(messages)
  resp.content
```

---

## 3. ONNX Model Inference

```nim
# onnx_inference.nim
# รัน ONNX models ด้วย C FFI (onnxruntime)

import std/[strformat, os, sequtils]

# ONNX Runtime C API bindings
type
  OrtEnv* = pointer
  OrtSession* = pointer
  OrtSessionOptions* = pointer
  OrtMemoryInfo* = pointer
  OrtValue* = pointer
  OrtAllocator* = pointer
  OrtStatus* = pointer

const ORT_LOGGING_LEVEL_WARNING* = 2'i32

proc OrtGetApiBase*(): pointer {.importc, dynlib: "libonnxruntime.so".}

# Simplified wrapper
type
  ONNXSession* = ref object
    env*: OrtEnv
    session*: OrtSession
    options*: OrtSessionOptions

proc loadModel*(modelPath: string): ONNXSession =
  # In real code: use OrtGetApiBase()->CreateEnv, CreateSession, etc.
  echo &"[ONNX] Loading model: {modelPath}"
  ONNXSession()

proc runInference*(session: ONNXSession, inputs: seq[seq[float32]]): seq[float32] =
  ## Run model inference
  echo &"[ONNX] Running inference on {inputs.len} inputs"
  # Placeholder — real impl would call OrtSession_Run
  @[]

# Example: Image classification
proc classifyImage*(session: ONNXSession, pixels: seq[float32],
    width, height, channels: int): tuple[classId: int, confidence: float32] =
  ## Classify image using loaded ONNX model
  ## pixels: flat array [height * width * channels]
  
  # Normalize if needed
  let normalized = pixels.mapIt(it / 255.0)
  
  let output = session.runInference(@[normalized])
  
  # Argmax to find predicted class
  var maxIdx = 0
  var maxVal = output[0]
  for i in 1..<output.len:
    if output[i] > maxVal:
      maxVal = output[i]
      maxIdx = i
  
  (classId: maxIdx, confidence: maxVal)

# tokenizer for LLM inference
type
  Tokenizer* = ref object
    vocab*: seq[string]
    vocabIndex*: Table[string, int]
    bos*: int  # Beginning of sequence token ID
    eos*: int  # End of sequence token ID
    unk*: int  # Unknown token ID

proc encode*(tok: Tokenizer, text: string): seq[int] =
  ## Simple whitespace tokenizer (real tokenizers use BPE/WordPiece)
  result = @[tok.bos]
  for word in text.split(' '):
    if tok.vocabIndex.hasKey(word):
      result.add(tok.vocabIndex[word])
    else:
      result.add(tok.unk)

proc decode*(tok: Tokenizer, ids: seq[int]): string =
  var words: seq[string]
  for id in ids:
    if id == tok.bos or id == tok.eos: continue
    if id < tok.vocab.len:
      words.add(tok.vocab[id])
    else:
      words.add("<unk>")
  words.join(" ")
```

---

## 4. Simple K-Means Clustering

```nim
# kmeans.nim
# K-Means clustering algorithm

import std/[math, random, strformat, sequtils, algorithm]

type
  Point* = seq[float64]
  
  KMeansResult* = object
    centroids*: seq[Point]
    labels*: seq[int]
    inertia*: float64
    iterations*: int

proc distance*(a, b: Point): float64 =
  var sum = 0.0
  for i in 0..<min(a.len, b.len):
    let diff = a[i] - b[i]
    sum += diff * diff
  sqrt(sum)

proc mean*(points: seq[Point]): Point =
  if points.len == 0: return @[]
  var result = newSeq[float64](points[0].len)
  for p in points:
    for i in 0..<p.len:
      result[i] += p[i]
  result.mapIt(it / float(points.len))

proc kMeans*(data: seq[Point], k: int, maxIter = 100,
    tolerance = 1e-4): KMeansResult =
  randomize()
  var centroids = newSeq[Point](k)
  
  # K-Means++ initialization
  centroids[0] = data[rand(data.len - 1)]
  
  for i in 1..<k:
    var distances = data.mapIt(
      centroids[0..<i].mapIt(distance(it, data[centroids[0..<i].mapIt(distance(it, data.mapIt(distance(it, centroids[0])))).minIndex()])).min()
    )
    let totalDist = distances.foldl(a + b)
    var rnd = rand(totalDist)
    
    for j, d in distances:
      rnd -= d
      if rnd <= 0:
        centroids[i] = data[j]
        break
    
    if centroids[i].len == 0:
      centroids[i] = data[^1]
  
  var labels = newSeq[int](data.len)
  var inertia = 0.0
  
  for iter in 0..<maxIter:
    # Assign points to nearest centroid
    for i, point in data:
      var minDist = float64.high
      var minIdx = 0
      for j, centroid in centroids:
        let d = distance(point, centroid)
        if d < minDist:
          minDist = d
          minIdx = j
      labels[i] = minIdx
      inertia += minDist * minDist
    
    # Update centroids
    var newCentroids = newSeq[Point](k)
    var moved = false
    
    for j in 0..<k:
      let clusterPoints = data.filterIt(labels[data.find(it)] == j)
      if clusterPoints.len > 0:
        let newCentroid = mean(clusterPoints)
        if distance(newCentroid, centroids[j]) > tolerance:
          moved = true
        newCentroids[j] = newCentroid
      else:
        newCentroids[j] = centroids[j]
    
    centroids = newCentroids
    
    if not moved:
      return KMeansResult(
        centroids: centroids,
        labels: labels,
        inertia: inertia,
        iterations: iter + 1
      )
  
  KMeansResult(centroids: centroids, labels: labels,
    inertia: inertia, iterations: maxIter)

when isMainModule:
  # Generate synthetic clusters
  randomize()
  var data: seq[Point]
  
  for _ in 0..49:
    data.add(@[rand(1.0), rand(1.0)])              # Cluster 1: top-left
  for _ in 0..49:
    data.add(@[rand(1.0) + 3.0, rand(1.0) + 3.0]) # Cluster 2: bottom-right
  for _ in 0..49:
    data.add(@[rand(1.0) + 6.0, rand(1.0)])        # Cluster 3: top-right
  
  let result = kMeans(data, k = 3)
  
  echo &"K-Means completed in {result.iterations} iterations"
  echo &"Inertia: {result.inertia:.2f}"
  for i, c in result.centroids:
    echo &"Centroid {i}: ({c[0]:.2f}, {c[1]:.2f})"
```

---

## สรุป

| Topic | Library / Approach |
|-------|-------------------|
| Neural Network | Custom Nim implementation |
| LLM API | httpclient + JSON |
| Embeddings | API + cosine similarity |
| RAG | VectorStore + LLM |
| ONNX Inference | onnxruntime C FFI |
| K-Means | Pure Nim |
| Tokenizer | BPE/WordPiece (custom or FFI) |

---

**Next**: [Part 76 - Real-Time Data Processing](../advanced/part76_realtime.md)
