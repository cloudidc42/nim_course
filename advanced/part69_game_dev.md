# Part 69 - Game Development with Nim

## บทนำ

Nim มี library หลายตัวสำหรับ game development:
- **SDL2** (bindings: `sdl2` package) — low-level graphics, input, audio
- **SFML** (bindings: `csfml`) — higher-level 2D game library
- **Raylib** (bindings: `naylib`) — simple, cross-platform game library
- **Godot** — Nim GDNative bindings

ในบทนี้ใช้ SDL2 เป็นหลัก เพราะมี bindings ที่ดีและ portable

---

## 1. Entity-Component-System (ECS) Architecture

```nim
# ecs.nim
# Pure Nim ECS ไม่ต้องการ external library

import std/[tables, sets, typetraits, hashes, bitops]

type
  EntityId* = distinct uint32
  ComponentId* = uint32
  Signature* = uint64  # ใช้ bitmask สำหรับ component types

  Entity* = object
    id*: EntityId
    signature*: Signature

  ComponentStorage*[T] = ref object
    data*: Table[EntityId, T]
    componentId*: ComponentId

  World* = ref object
    entities*: Table[EntityId, Entity]
    nextId*: uint32
    componentIds*: Table[string, ComponentId]
    nextComponentId*: ComponentId
    storages*: Table[ComponentId, pointer]  # ComponentStorage[T]
    systems*: seq[proc(world: World, dt: float)]
    deadEntities*: seq[EntityId]

proc `==`*(a, b: EntityId): bool {.borrow.}
proc hash*(e: EntityId): Hash {.borrow.}

proc newWorld*(): World =
  World(
    entities: initTable[EntityId, Entity](),
    componentIds: initTable[string, ComponentId]()
  )

proc createEntity*(world: World): EntityId =
  let id = EntityId(world.nextId)
  inc world.nextId
  world.entities[id] = Entity(id: id, signature: 0)
  id

proc destroyEntity*(world: World, id: EntityId) =
  world.deadEntities.add(id)

proc flushDead*(world: World) =
  for id in world.deadEntities:
    world.entities.del(id)
    # TODO: remove from all component storages
  world.deadEntities.setLen(0)

proc getComponentId*(world: World, T: typedesc): ComponentId =
  let name = $T
  if not world.componentIds.hasKey(name):
    world.componentIds[name] = world.nextComponentId
    inc world.nextComponentId
  world.componentIds[name]

proc getStorage*[T](world: World): ComponentStorage[T] =
  let cid = world.getComponentId(T)
  if world.storages.hasKey(cid):
    return cast[ComponentStorage[T]](world.storages[cid])
  let storage = ComponentStorage[T](
    data: initTable[EntityId, T](),
    componentId: cid
  )
  world.storages[cid] = cast[pointer](storage)
  storage

proc addComponent*[T](world: World, entity: EntityId, component: T) =
  let cid = world.getComponentId(T)
  let storage = world.getStorage(T)
  storage.data[entity] = component
  world.entities[entity].signature = world.entities[entity].signature or (1'u64 shl cid)

proc getComponent*[T](world: World, entity: EntityId): var T =
  let storage = world.getStorage(T)
  storage.data[entity]

proc hasComponent*[T](world: World, entity: EntityId): bool =
  let cid = world.getComponentId(T)
  (world.entities[entity].signature and (1'u64 shl cid)) != 0

proc removeComponent*[T](world: World, entity: EntityId) =
  let cid = world.getComponentId(T)
  let storage = world.getStorage(T)
  storage.data.del(entity)
  world.entities[entity].signature = world.entities[entity].signature and not(1'u64 shl cid)

iterator query*(world: World, T: typedesc): tuple[entity: EntityId, comp: var T] =
  let storage = world.getStorage(T)
  for id, comp in storage.data.mpairs:
    yield (entity: id, comp: comp)

iterator query2*[A, B](world: World, ta: typedesc[A], tb: typedesc[B]): 
    tuple[entity: EntityId, a: var A, b: var B] =
  let storageA = world.getStorage(A)
  let storageB = world.getStorage(B)
  for id, a in storageA.data.mpairs:
    if storageB.data.hasKey(id):
      yield (entity: id, a: a, b: storageB.data[id])

proc registerSystem*(world: World, system: proc(world: World, dt: float)) =
  world.systems.add(system)

proc update*(world: World, dt: float) =
  for system in world.systems:
    system(world, dt)
  world.flushDead()

# Game components
type
  Transform* = object
    x*, y*: float
    rotation*: float
    scaleX*, scaleY*: float

  Velocity* = object
    dx*, dy*: float
    maxSpeed*: float

  Health* = object
    current*, max*: int
    invincibleTimer*: float

  Sprite* = object
    textureId*: string
    width*, height*: float
    srcX*, srcY*: float
    srcW*, srcH*: float
    flipX*, flipY*: bool
    layer*: int

  Collider* = object
    width*, height*: float
    offsetX*, offsetY*: float
    isTrigger*: bool

  Tag* = object
    value*: string

# Built-in systems
proc movementSystem*(world: World, dt: float) =
  for (entity, vel) in world.query(Velocity):
    if world.hasComponent[Transform](entity):
      var transform = world.getComponent[Transform](entity)
      transform.x += vel.dx * dt
      transform.y += vel.dy * dt

proc healthSystem*(world: World, dt: float) =
  var dead: seq[EntityId]
  for (entity, hp) in world.query(Health):
    if hp.invincibleTimer > 0:
      hp.invincibleTimer -= dt
    if hp.current <= 0:
      dead.add(entity)
  for e in dead:
    world.destroyEntity(e)
```

---

## 2. SDL2 Rendering

```nim
# sdl2_game.nim
# SDL2 game loop สำหรับ 2D games

import sdl2
import sdl2/[image, ttf, mixer]
import std/[tables, strformat, math, options]

type
  GameState* = enum
    gsMenu
    gsPlaying
    gsPaused
    gsGameOver

  InputState* = object
    keys*: array[SDL_NUM_SCANCODES, bool]
    keysDown*: array[SDL_NUM_SCANCODES, bool]  # Just pressed this frame
    keysUp*: array[SDL_NUM_SCANCODES, bool]    # Just released this frame
    mouseX*, mouseY*: int
    mouseButtons*: uint32
    mouseButtonsDown*: uint32

  Camera2D* = object
    x*, y*: float
    zoom*: float
    screenW*, screenH*: float

  Game* = ref object
    window*: WindowPtr
    renderer*: RendererPtr
    running*: bool
    state*: GameState
    dt*: float
    fps*: float
    targetFps*: float
    input*: InputState
    camera*: Camera2D
    world*: World
    textures*: Table[string, TexturePtr]
    fonts*: Table[string, FontPtr]

proc newGame*(title: string, width, height: int, fps = 60.0): Game =
  if sdl2.init(INIT_EVERYTHING) != SdlSuccess:
    raise newException(IOError, &"SDL2 init failed: {sdl2.getError()}")
  
  if not image.init(IMG_INIT_PNG or IMG_INIT_JPG):
    raise newException(IOError, "SDL2_image init failed")
  
  if ttf.init() != 0:
    raise newException(IOError, "SDL2_ttf init failed")
  
  let window = createWindow(title, SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
    cint(width), cint(height), SDL_WINDOW_SHOWN or SDL_WINDOW_RESIZABLE)
  
  let renderer = createRenderer(window, -1,
    Renderer_Accelerated or Renderer_PresentVsync)
  
  renderer.setDrawBlendMode(BlendMode_Blend)
  
  Game(
    window: window,
    renderer: renderer,
    running: true,
    targetFps: fps,
    state: gsMenu,
    camera: Camera2D(zoom: 1.0, screenW: float(width), screenH: float(height)),
    world: newWorld(),
    textures: initTable[string, TexturePtr](),
    fonts: initTable[string, FontPtr]()
  )

proc loadTexture*(game: Game, name, path: string): TexturePtr =
  let surface = image.load(path)
  if surface == nil:
    raise newException(IOError, &"Cannot load image: {path}")
  let tex = game.renderer.createTextureFromSurface(surface)
  freeSurface(surface)
  game.textures[name] = tex
  tex

proc loadFont*(game: Game, name, path: string, size: int): FontPtr =
  let font = openFont(path, cint(size))
  if font == nil:
    raise newException(IOError, &"Cannot load font: {path}")
  game.fonts[name] = font
  font

proc worldToScreen*(cam: Camera2D, x, y: float): tuple[sx, sy: float] =
  (
    sx: (x - cam.x) * cam.zoom + cam.screenW / 2,
    sy: (y - cam.y) * cam.zoom + cam.screenH / 2
  )

proc drawTexture*(game: Game, texName: string, x, y, w, h: float,
    angle = 0.0, flipX = false, flipY = false) =
  let (sx, sy) = worldToScreen(game.camera, x, y)
  let dstRect = rect(
    cint(sx - w * game.camera.zoom / 2),
    cint(sy - h * game.camera.zoom / 2),
    cint(w * game.camera.zoom),
    cint(h * game.camera.zoom)
  )
  
  if game.textures.hasKey(texName):
    let tex = game.textures[texName]
    let flip = if flipX and flipY: SDL_FLIP_BOTH
               elif flipX: SDL_FLIP_HORIZONTAL
               elif flipY: SDL_FLIP_VERTICAL
               else: SDL_FLIP_NONE
    game.renderer.copyEx(tex, nil, addr dstRect, angle, nil, flip)

proc drawText*(game: Game, fontName, text: string, x, y: float,
    color = color(255, 255, 255, 255)) =
  if not game.fonts.hasKey(fontName): return
  let font = game.fonts[fontName]
  let surface = font.renderTextSolid(text, color)
  if surface == nil: return
  let tex = game.renderer.createTextureFromSurface(surface)
  defer:
    freeSurface(surface)
    destroyTexture(tex)
  
  var w, h: cint
  tex.queryTexture(nil, nil, addr w, addr h)
  let dstRect = rect(cint(x), cint(y), w, h)
  game.renderer.copy(tex, nil, addr dstRect)

proc processInput*(game: Game) =
  # Clear frame-only states
  for i in 0..<SDL_NUM_SCANCODES:
    game.input.keysDown[i] = false
    game.input.keysUp[i] = false
  game.input.mouseButtonsDown = 0
  
  var event = defaultEvent
  while pollEvent(event):
    case event.kind
    of QuitEvent:
      game.running = false
    of KeyDown:
      let sc = event.key.keysym.scancode.int
      if not game.input.keys[sc]:
        game.input.keysDown[sc] = true
      game.input.keys[sc] = true
    of KeyUp:
      let sc = event.key.keysym.scancode.int
      game.input.keys[sc] = false
      game.input.keysUp[sc] = true
    of MouseMotion:
      game.input.mouseX = event.motion.x
      game.input.mouseY = event.motion.y
    of MouseButtonDown:
      game.input.mouseButtonsDown = game.input.mouseButtonsDown or (1'u32 shl event.button.button)
      game.input.mouseButtons = game.input.mouseButtons or (1'u32 shl event.button.button)
    of MouseButtonUp:
      game.input.mouseButtons = game.input.mouseButtons and not(1'u32 shl event.button.button)
    else: discard

proc isKeyHeld*(game: Game, scancode: Scancode): bool =
  game.input.keys[scancode.int]

proc isKeyPressed*(game: Game, scancode: Scancode): bool =
  game.input.keysDown[scancode.int]

proc run*(game: Game, update: proc(game: Game, dt: float),
    draw: proc(game: Game)) =
  var lastTime = getTicks()
  
  while game.running:
    let currentTime = getTicks()
    game.dt = float(currentTime - lastTime) / 1000.0
    lastTime = currentTime
    game.fps = if game.dt > 0: 1.0 / game.dt else: 0.0
    
    game.processInput()
    update(game, game.dt)
    
    game.renderer.setDrawColor(0, 0, 0, 255)
    game.renderer.clear()
    
    draw(game)
    
    game.renderer.present()

proc destroy*(game: Game) =
  for tex in game.textures.values:
    destroyTexture(tex)
  for font in game.fonts.values:
    close(font)
  game.renderer.destroy()
  game.window.destroy()
  ttf.quit()
  image.quit()
  sdl2.quit()
```

---

## 3. Collision Detection

```nim
# collision.nim
# AABB และ Circle collision detection

import std/[math, options]

type
  AABB* = object
    x*, y*: float     # Center
    halfW*, halfH*: float

  Circle* = object
    x*, y*: float
    radius*: float

  CollisionInfo* = object
    colliding*: bool
    normalX*, normalY*: float  # Collision normal
    depth*: float              # Penetration depth
    contactX*, contactY*: float

proc aabbVsAabb*(a, b: AABB): CollisionInfo =
  let dx = b.x - a.x
  let dy = b.y - a.y
  let overlapX = a.halfW + b.halfW - abs(dx)
  let overlapY = a.halfH + b.halfH - abs(dy)
  
  if overlapX <= 0 or overlapY <= 0:
    return CollisionInfo(colliding: false)
  
  if overlapX < overlapY:
    let nx = if dx < 0: -1.0 else: 1.0
    CollisionInfo(
      colliding: true,
      normalX: nx, normalY: 0.0,
      depth: overlapX
    )
  else:
    let ny = if dy < 0: -1.0 else: 1.0
    CollisionInfo(
      colliding: true,
      normalX: 0.0, normalY: ny,
      depth: overlapY
    )

proc circleVsCircle*(a, b: Circle): CollisionInfo =
  let dx = b.x - a.x
  let dy = b.y - a.y
  let distSq = dx * dx + dy * dy
  let radiusSum = a.radius + b.radius
  
  if distSq >= radiusSum * radiusSum:
    return CollisionInfo(colliding: false)
  
  let dist = sqrt(distSq)
  let depth = radiusSum - dist
  
  if dist < 0.0001:
    return CollisionInfo(colliding: true, normalX: 1.0, normalY: 0.0, depth: depth)
  
  CollisionInfo(
    colliding: true,
    normalX: dx / dist,
    normalY: dy / dist,
    depth: depth
  )

proc aabbVsCircle*(box: AABB, circle: Circle): CollisionInfo =
  ## AABB vs Circle — find closest point on AABB to circle center
  let closestX = max(box.x - box.halfW, min(circle.x, box.x + box.halfW))
  let closestY = max(box.y - box.halfH, min(circle.y, box.y + box.halfH))
  
  let dx = circle.x - closestX
  let dy = circle.y - closestY
  let distSq = dx * dx + dy * dy
  
  if distSq >= circle.radius * circle.radius:
    return CollisionInfo(colliding: false)
  
  let dist = sqrt(distSq)
  if dist < 0.0001:
    return CollisionInfo(colliding: true, normalX: 0.0, normalY: -1.0,
      depth: circle.radius)
  
  CollisionInfo(
    colliding: true,
    normalX: dx / dist,
    normalY: dy / dist,
    depth: circle.radius - dist
  )

proc resolveCollision*(aPos: var tuple[x, y: float], bPos: var tuple[x, y: float],
    info: CollisionInfo, massA, massB: float) =
  ## Apply position correction based on collision info
  if not info.colliding: return
  
  let totalMass = massA + massB
  let correctionA = info.depth * (massB / totalMass)
  let correctionB = info.depth * (massA / totalMass)
  
  aPos.x -= info.normalX * correctionA
  aPos.y -= info.normalY * correctionA
  bPos.x += info.normalX * correctionB
  bPos.y += info.normalY * correctionB

# Spatial hash grid for broad-phase collision detection
type
  SpatialHash*[T] = ref object
    cellSize*: float
    cells*: Table[tuple[gx, gy: int], seq[T]]

proc newSpatialHash*[T](cellSize: float): SpatialHash[T] =
  SpatialHash[T](cellSize: cellSize, cells: initTable[tuple[gx, gy: int], seq[T]]())

proc cell*(sh: SpatialHash, x, y: float): tuple[gx, gy: int] =
  (gx: int(x / sh.cellSize), gy: int(y / sh.cellSize))

proc insert*[T](sh: SpatialHash[T], x, y: float, obj: T) =
  let c = sh.cell(x, y)
  if not sh.cells.hasKey(c):
    sh.cells[c] = @[]
  sh.cells[c].add(obj)

proc query*[T](sh: SpatialHash[T], x, y, radius: float): seq[T] =
  let minCell = sh.cell(x - radius, y - radius)
  let maxCell = sh.cell(x + radius, y + radius)
  
  for gx in minCell.gx..maxCell.gx:
    for gy in minCell.gy..maxCell.gy:
      let c = (gx: gx, gy: gy)
      if sh.cells.hasKey(c):
        result.add(sh.cells[c])

proc clear*[T](sh: SpatialHash[T]) =
  sh.cells.clear()
```

---

## 4. Example: Simple Platformer

```nim
# platformer.nim
# ตัวอย่าง simple platformer game

import sdl2
import std/[math, strformat]

# Player constants
const
  PLAYER_SPEED = 200.0
  JUMP_FORCE   = -500.0
  GRAVITY      = 1000.0
  TILE_SIZE    = 32

type
  PlayerState* = enum
    psIdle
    psRunning
    psJumping
    psFalling

  Player* = object
    x*, y*: float
    velX*, velY*: float
    state*: PlayerState
    facingRight*: bool
    onGround*: bool
    jumpBuffer*: float  # Coyote time
    health*: int

  Tile* = enum
    tEmpty
    tSolid
    tPlatform  # One-way platform (pass through from below)
    tSpike

  Level* = ref object
    tiles*: seq[seq[Tile]]
    width*, height*: int
    spawnX*, spawnY*: float

proc getTile*(level: Level, gx, gy: int): Tile =
  if gx < 0 or gx >= level.width or gy < 0 or gy >= level.height:
    return tSolid  # Out of bounds = solid
  level.tiles[gy][gx]

proc isSolid*(tile: Tile): bool =
  tile == tSolid

proc updatePlayer*(player: var Player, level: Level, input: InputState, dt: float) =
  # Horizontal movement
  let moveX = (if isKeyHeld(input, SDL_SCANCODE_RIGHT): 1.0 else: 0.0) -
              (if isKeyHeld(input, SDL_SCANCODE_LEFT): 1.0 else: 0.0)
  
  player.velX = moveX * PLAYER_SPEED
  
  if moveX > 0: player.facingRight = true
  elif moveX < 0: player.facingRight = false
  
  # Coyote time (allow jumping briefly after walking off edge)
  if player.onGround:
    player.jumpBuffer = 0.15
  elif player.jumpBuffer > 0:
    player.jumpBuffer -= dt
  
  # Jump
  if isKeyPressed(input, SDL_SCANCODE_SPACE) or isKeyPressed(input, SDL_SCANCODE_Z):
    if player.jumpBuffer > 0:
      player.velY = JUMP_FORCE
      player.jumpBuffer = 0
      player.onGround = false
  
  # Variable jump height (release to fall faster)
  if not isKeyHeld(input, SDL_SCANCODE_SPACE) and player.velY < 0:
    player.velY += GRAVITY * 2.0 * dt  # Extra gravity when falling
  else:
    player.velY += GRAVITY * dt
  
  # Clamp fall speed
  player.velY = min(player.velY, 1000.0)
  
  # Move horizontally with collision
  player.x += player.velX * dt
  let tileX = int(player.x / TILE_SIZE)
  let tileY = int(player.y / TILE_SIZE)
  
  # Check horizontal collisions
  if player.velX > 0:
    let rightTile = getTile(level, int((player.x + 8) / TILE_SIZE), tileY)
    if rightTile.isSolid:
      player.x = float(tileX * TILE_SIZE + TILE_SIZE - 8) - 0.1
      player.velX = 0
  elif player.velX < 0:
    let leftTile = getTile(level, int((player.x - 8) / TILE_SIZE), tileY)
    if leftTile.isSolid:
      player.x = float((tileX + 1) * TILE_SIZE + 8) + 0.1
      player.velX = 0
  
  # Move vertically with collision
  player.onGround = false
  player.y += player.velY * dt
  
  if player.velY > 0:  # Falling
    let belowTile = getTile(level, int(player.x / TILE_SIZE), int((player.y + 16) / TILE_SIZE))
    if belowTile.isSolid:
      player.y = float(int((player.y + 16) / TILE_SIZE) * TILE_SIZE - 16) - 0.1
      player.velY = 0
      player.onGround = true
  elif player.velY < 0:  # Rising
    let aboveTile = getTile(level, int(player.x / TILE_SIZE), int((player.y - 16) / TILE_SIZE))
    if aboveTile.isSolid:
      player.y = float((int((player.y - 16) / TILE_SIZE) + 1) * TILE_SIZE + 16) + 0.1
      player.velY = 0
  
  # Update state
  if not player.onGround:
    player.state = if player.velY < 0: psJumping else: psFalling
  elif abs(player.velX) > 0.1:
    player.state = psRunning
  else:
    player.state = psIdle
  
  # Spike damage
  let currentTile = getTile(level, int(player.x / TILE_SIZE), int(player.y / TILE_SIZE))
  if currentTile == tSpike:
    dec player.health

proc drawLevel*(game: Game, level: Level) =
  for y in 0..<level.height:
    for x in 0..<level.width:
      let tile = level.tiles[y][x]
      if tile == tEmpty: continue
      
      let color = case tile
        of tSolid: color(100, 100, 200, 255)
        of tPlatform: color(150, 100, 50, 255)
        of tSpike: color(200, 50, 50, 255)
        else: color(80, 80, 80, 255)
      
      game.renderer.setDrawColor(color.r, color.g, color.b, color.a)
      var r = rect(cint(x * TILE_SIZE), cint(y * TILE_SIZE), cint(TILE_SIZE), cint(TILE_SIZE))
      game.renderer.fillRect(addr r)

proc drawPlayer*(game: Game, player: Player) =
  game.renderer.setDrawColor(0, 200, 100, 255)
  var r = rect(cint(player.x - 8), cint(player.y - 16), 16, 32)
  game.renderer.fillRect(addr r)
  
  # Eyes
  game.renderer.setDrawColor(255, 255, 255, 255)
  let eyeX = if player.facingRight: cint(player.x + 2) else: cint(player.x - 6)
  var eye = rect(eyeX, cint(player.y - 12), 4, 4)
  game.renderer.fillRect(addr eye)
```

---

## 5. Audio System

```nim
# audio.nim
# SDL2_mixer สำหรับ sound effects และ music

import sdl2/mixer
import std/[tables, strformat]

type
  AudioSystem* = ref object
    sounds*: Table[string, ChunkPtr]
    music*: Table[string, MusicPtr]
    sfxVolume*: int
    musicVolume*: int

proc newAudioSystem*(numChannels = 16): AudioSystem =
  if openAudio(44100, MIX_DEFAULT_FORMAT, 2, 2048) != 0:
    raise newException(IOError, "SDL_mixer init failed")
  allocateChannels(cint(numChannels))
  AudioSystem(sfxVolume: 80, musicVolume: 60)

proc loadSound*(audio: AudioSystem, name, path: string) =
  let chunk = loadWAV(path)
  if chunk == nil:
    raise newException(IOError, &"Cannot load sound: {path}")
  audio.sounds[name] = chunk

proc loadMusic*(audio: AudioSystem, name, path: string) =
  let music = loadMUS(path)
  if music == nil:
    raise newException(IOError, &"Cannot load music: {path}")
  audio.music[name] = music

proc playSound*(audio: AudioSystem, name: string, loops = 0, channel = -1): int =
  if audio.sounds.hasKey(name):
    let chunk = audio.sounds[name]
    chunk.volume(cint(audio.sfxVolume))
    return playChannel(cint(channel), chunk, cint(loops))
  -1

proc playMusic*(audio: AudioSystem, name: string, loops = -1) =
  if audio.music.hasKey(name):
    let music = audio.music[name]
    volumeMusic(cint(audio.musicVolume))
    discard playMusic(music, cint(loops))

proc stopMusic*() = haltMusic()
proc pauseMusic*() = pauseMusic()
proc resumeMusic*() = resumeMusic()

proc destroy*(audio: AudioSystem) =
  for chunk in audio.sounds.values:
    freeChunk(chunk)
  for music in audio.music.values:
    freeMusic(music)
  closeAudio()
```

---

## สรุป

| Pattern | Use Case |
|---------|---------|
| ECS | Complex games with many entity types |
| SDL2 | 2D games, cross-platform |
| Collision AABB | Fast simple collision |
| Spatial Hash | Many objects, broad-phase |
| Coyote Time | Better platformer feel |
| Variable Jump | Responsive jump control |

---

**Next**: [Part 70 - Nim Package Development](../advanced/part70_packages.md)
