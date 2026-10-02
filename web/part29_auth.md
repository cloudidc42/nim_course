# Part 29: Authentication และ JWT

## Password Hashing

```nim
# nimble install bcrypt
import bcrypt, strutils

# Hash password
proc hashPassword(password: string): string =
  hash(password, genSalt(12))

# Verify password
proc verifyPassword(password, hash: string): bool =
  compare(password, hash)

# Usage
let password = "mysecretpassword"
let hashed = hashPassword(password)
echo "Hashed: ", hashed

echo verifyPassword(password, hashed)    # true
echo verifyPassword("wrongpass", hashed) # false

# Manual PBKDF2 (no external dep)
import sha, strutils

proc pbkdf2(password, salt: string, iterations: int = 100000): string =
  var key = password & salt
  for i in 0..<iterations:
    let sha = newSHA256()
    sha.update(key)
    key = sha.hexdigest()
  return key

let salt = "random_salt_here"
let key = pbkdf2("password123", salt)
echo "PBKDF2: ", key
```

## JWT (JSON Web Tokens)

```nim
import base64, json, hmac, sha, times, strutils, tables

type
  JWTHeader = object
    alg: string
    typ: string
  
  JWTPayload = object
    sub: string
    iss: string
    aud: string
    exp: int64
    iat: int64
    data: JsonNode

proc base64url_encode(s: string): string =
  s.encode(true).replace("=", "").replace("+", "-").replace("/", "_")

proc base64url_decode(s: string): string =
  var padded = s.replace("-", "+").replace("_", "/")
  while padded.len mod 4 != 0:
    padded &= "="
  decode(padded)

proc createJWT(secret: string, payload: JWTPayload): string =
  let header = %*{"alg": "HS256", "typ": "JWT"}
  let headerEncoded = base64url_encode($header)
  let payloadJson = %*{
    "sub": payload.sub, "iss": payload.iss, "aud": payload.aud,
    "exp": payload.exp, "iat": payload.iat, "data": payload.data
  }
  let payloadEncoded = base64url_encode($payloadJson)
  let message = headerEncoded & "." & payloadEncoded
  let sig = hmac_sha256(secret, message)
  let sigEncoded = base64url_encode(sig.toHex())
  message & "." & sigEncoded

proc verifyJWT(secret, token: string): Option[JsonNode] =
  import options
  let parts = token.split(".")
  if parts.len != 3:
    return none(JsonNode)
  let message = parts[0] & "." & parts[1]
  let expectedSig = base64url_encode(hmac_sha256(secret, message).toHex())
  if expectedSig != parts[2]:
    return none(JsonNode)
  let payload = parseJson(base64url_decode(parts[1]))
  if "exp" in payload:
    if payload["exp"].getInt() < epochTime().int64:
      return none(JsonNode)
  some(payload)

# Usage
let secret = "my-super-secret-key-change-in-production"
let payload = JWTPayload(
  sub: "user123", iss: "myapp.com", aud: "myapp.com",
  exp: (epochTime() + 3600).int64, iat: epochTime().int64,
  data: %*{"role": "admin", "name": "Alice"}
)
let token = createJWT(secret, payload)
echo "Token: ", token[0..50], "..."

let verified = verifyJWT(secret, token)
if verified.isSome:
  echo "Valid! User: ", verified.get()["sub"].getStr()
```

## Session Management

```nim
import tables, times, random, strutils

type
  Session = object
    id: string
    userId: string
    data: Table[string, string]
    createdAt: Time
    expiresAt: Time

var sessions = initTable[string, Session]()

proc generateSessionId(): string =
  randomize()
  var id = ""
  for i in 0..<32:
    id &= toHex(rand(255), 2)
  id.toLowerAscii()

proc createSession(userId: string, durationHours: int = 24): Session =
  let now = getTime()
  Session(
    id: generateSessionId(), userId: userId,
    data: initTable[string, string](),
    createdAt: now, expiresAt: now + initDuration(hours = durationHours)
  )

proc getSession(id: string): Option[Session] =
  if id notin sessions: return none(Session)
  let session = sessions[id]
  if getTime() > session.expiresAt:
    sessions.del(id)
    return none(Session)
  some(session)

proc deleteSession(id: string) =
  sessions.del(id)
```

## OAuth2 Client

```nim
import asynchttpclient, asyncdispatch, json, uri, strutils

type
  OAuth2Config = object
    clientId: string
    clientSecret: string
    authUrl: string
    tokenUrl: string
    redirectUri: string
    scopes: seq[string]

proc buildAuthUrl(config: OAuth2Config, state: string): string =
  let params = @[
    ("client_id", config.clientId),
    ("redirect_uri", config.redirectUri),
    ("response_type", "code"),
    ("scope", config.scopes.join(" ")),
    ("state", state)
  ]
  config.authUrl & "?" & params.mapIt(
    encodeUrl(it[0]) & "=" & encodeUrl(it[1])
  ).join("&")

proc exchangeCode(config: OAuth2Config, code: string): Future[JsonNode] {.async.} =
  let client = newAsyncHttpClient()
  defer: client.close()
  client.headers = newHttpHeaders({
    "Content-Type": "application/x-www-form-urlencoded",
    "Accept": "application/json"
  })
  let body = [
    ("grant_type", "authorization_code"),
    ("code", code),
    ("redirect_uri", config.redirectUri),
    ("client_id", config.clientId),
    ("client_secret", config.clientSecret)
  ].mapIt(encodeUrl(it[0]) & "=" & encodeUrl(it[1])).join("&")
  let resp = await client.post(config.tokenUrl, body)
  return parseJson(await resp.body)

# GitHub OAuth2 example config
let githubOAuth = OAuth2Config(
  clientId: "your-client-id",
  clientSecret: "your-client-secret",
  authUrl: "https://github.com/login/oauth/authorize",
  tokenUrl: "https://github.com/login/oauth/access_token",
  redirectUri: "http://localhost:8080/auth/callback",
  scopes: @["user:email", "read:user"]
)

let authUrl = buildAuthUrl(githubOAuth, "random-state-123")
echo "Redirect user to: ", authUrl
```

## Auth Middleware สำหรับ Jester

```nim
import jester, asyncdispatch, json, strutils, tables, times

const JWT_SECRET = "change-this-in-production"

type
  AuthUser = object
    id: string
    role: string

var activeTokens = initTable[string, bool]()

proc extractToken(request: Request): string =
  let auth = request.headers.getOrDefault("Authorization", "")
  if auth.startsWith("Bearer "):
    auth[7..^1]
  elif "token" in request.params:
    request.params["token"]
  else:
    ""

proc getCurrentUser(request: Request): Option[AuthUser] =
  let token = extractToken(request)
  if token.len == 0: return none(AuthUser)
  if token in activeTokens and not activeTokens[token]:
    return none(AuthUser)
  let payload = verifyJWT(JWT_SECRET, token)
  if payload.isNone: return none(AuthUser)
  let p = payload.get()
  some(AuthUser(id: p["sub"].getStr(), role: p.getOrDefault("role", %"user").getStr()))

template requireAuth(request: Request, body: untyped) =
  let userOpt = getCurrentUser(request)
  if userOpt.isNone:
    resp(Http401, $(%*{"error": "Unauthorized"}), "application/json")
  else:
    let currentUser {.inject.} = userOpt.get()
    body

template requireRole(request: Request, role: string, body: untyped) =
  requireAuth(request):
    if currentUser.role != role and currentUser.role != "admin":
      resp(Http403, $(%*{"error": "Forbidden"}), "application/json")
    else:
      body

routes:
  post "/auth/login":
    let data = parseJson(request.body)
    let username = data["username"].getStr()
    let password = data["password"].getStr()
    if username == "admin" and password == "secret":
      let token = createJWT(JWT_SECRET, JWTPayload(
        sub: "admin-001", iss: "myapp",
        exp: (epochTime() + 3600).int64,
        iat: epochTime().int64,
        data: %*{"role": "admin", "username": username}
      ))
      resp $(%*{"token": token, "expires_in": 3600}), "application/json"
    else:
      resp(Http401, $(%*{"error": "Invalid credentials"}), "application/json")

  get "/api/me":
    requireAuth(request):
      resp $(%*{"id": currentUser.id, "role": currentUser.role}), "application/json"

runForever()
```

## สรุป Part 29

ในบทนี้เราได้เรียนรู้:
- ✅ Password hashing (bcrypt, PBKDF2)
- ✅ JWT creation และ verification
- ✅ Session management
- ✅ OAuth2 client flow
- ✅ Auth middleware สำหรับ Jester
- ✅ Role-based access control

---

**Previous**: [Part 28 - Database](part28_database.md)
**Next**: [Part 30 - Windows Internals](../security/part30_windows_internals.md)
