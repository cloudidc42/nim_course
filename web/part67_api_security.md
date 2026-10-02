# Part 67 - API Security: OAuth2, JWT & mTLS

## บทนำ

API Security คือหัวใจของ production application ที่ปลอดภัย
เรียนรู้การ implement OAuth2 Authorization Code + PKCE, JWT best practices,
mutual TLS (mTLS) สำหรับ service-to-service authentication

---

## 1. JWT Implementation

```nim
# jwt_impl.nim
# JSON Web Token — HS256, RS256, ES256 signing

import std/[base64, json, times, strutils, strformat, options]
import std/[hmac, sha2, openssl]

type
  Algorithm* = enum
    algHS256 = "HS256"
    algHS384 = "HS384"
    algHS512 = "HS512"
    algRS256 = "RS256"
    algES256 = "ES256"

  JWTHeader* = object
    alg*: string
    typ*: string
    kid*: string  # Key ID (สำหรับ key rotation)

  JWTClaims* = object
    iss*: string    # Issuer
    sub*: string    # Subject (user ID)
    aud*: seq[string]  # Audience
    exp*: int64     # Expiration (Unix timestamp)
    nbf*: int64     # Not Before
    iat*: int64     # Issued At
    jti*: string    # JWT ID (สำหรับ revocation)
    # Custom claims
    roles*: seq[string]
    scope*: string
    email*: string

  JWTError* = enum
    jwtOk
    jwtExpired
    jwtInvalidSignature
    jwtInvalidFormat
    jwtNotYetValid
    jwtAudienceMismatch

proc urlSafeBase64Encode*(data: string): string =
  encode(data).strip(chars = {'='}).replace("+", "-").replace("/", "_")

proc urlSafeBase64Decode*(data: string): string =
  var s = data.replace("-", "+").replace("_", "/")
  while s.len mod 4 != 0:
    s &= "="
  decode(s)

proc signHS256*(data, secret: string): string =
  let mac = hmac_sha256(secret, data)
  urlSafeBase64Encode(mac)

proc createJWT*(claims: JWTClaims, secret: string,
    alg = algHS256, kid = ""): string =
  let header = JWTHeader(alg: $alg, typ: "JWT", kid: kid)
  let headerJson = %*{"alg": header.alg, "typ": header.typ}
  if kid.len > 0:
    headerJson["kid"] = %kid
  
  let claimsJson = %*{
    "iss": claims.iss,
    "sub": claims.sub,
    "aud": claims.aud,
    "exp": claims.exp,
    "iat": claims.iat,
    "jti": claims.jti
  }
  if claims.roles.len > 0:
    claimsJson["roles"] = %claims.roles
  if claims.scope.len > 0:
    claimsJson["scope"] = %claims.scope
  if claims.email.len > 0:
    claimsJson["email"] = %claims.email
  if claims.nbf > 0:
    claimsJson["nbf"] = %claims.nbf
  
  let headerEncoded = urlSafeBase64Encode($headerJson)
  let payloadEncoded = urlSafeBase64Encode($claimsJson)
  let signingInput = &"{headerEncoded}.{payloadEncoded}"
  
  let signature = case alg
    of algHS256: signHS256(signingInput, secret)
    else: signHS256(signingInput, secret)  # Simplified
  
  &"{signingInput}.{signature}"

proc verifyJWT*(token, secret: string,
    expectedAudience = "",
    leewaySeconds = 0): tuple[claims: JWTClaims, err: JWTError] =
  let parts = token.split('.')
  if parts.len != 3:
    return (JWTClaims(), jwtInvalidFormat)
  
  # Verify signature
  let signingInput = &"{parts[0]}.{parts[1]}"
  let expectedSig = signHS256(signingInput, secret)
  if expectedSig != parts[2]:
    return (JWTClaims(), jwtInvalidSignature)
  
  # Decode payload
  let payloadJson = parseJson(urlSafeBase64Decode(parts[1]))
  
  let now = epochTime().int64
  
  # Check expiration
  let exp = payloadJson.getOrDefault("exp").getBiggestInt(0)
  if exp > 0 and now > exp + leewaySeconds:
    return (JWTClaims(), jwtExpired)
  
  # Check not-before
  let nbf = payloadJson.getOrDefault("nbf").getBiggestInt(0)
  if nbf > 0 and now < nbf - leewaySeconds:
    return (JWTClaims(), jwtNotYetValid)
  
  # Check audience
  if expectedAudience.len > 0:
    let aud = payloadJson.getOrDefault("aud")
    var audienceMatch = false
    if aud.kind == JArray:
      for a in aud:
        if a.getStr() == expectedAudience:
          audienceMatch = true
    elif aud.kind == JString:
      audienceMatch = aud.getStr() == expectedAudience
    if not audienceMatch:
      return (JWTClaims(), jwtAudienceMismatch)
  
  let claims = JWTClaims(
    iss: payloadJson.getOrDefault("iss").getStr(),
    sub: payloadJson.getOrDefault("sub").getStr(),
    exp: exp,
    iat: payloadJson.getOrDefault("iat").getBiggestInt(0),
    jti: payloadJson.getOrDefault("jti").getStr(),
    scope: payloadJson.getOrDefault("scope").getStr(),
    email: payloadJson.getOrDefault("email").getStr()
  )
  
  return (claims, jwtOk)

# Token refresh pattern
type
  TokenPair* = object
    accessToken*: string
    refreshToken*: string
    expiresIn*: int
    tokenType*: string

proc issueTokenPair*(userId, email: string, roles: seq[string],
    secret, refreshSecret: string): TokenPair =
  let now = epochTime().int64
  
  let accessClaims = JWTClaims(
    iss: "myapp",
    sub: userId,
    aud: @["myapp-api"],
    exp: now + 900,  # 15 minutes
    iat: now,
    jti: &"access-{userId}-{now}",
    email: email,
    roles: roles,
    scope: "read write"
  )
  
  let refreshClaims = JWTClaims(
    iss: "myapp",
    sub: userId,
    aud: @["myapp-refresh"],
    exp: now + 604800,  # 7 days
    iat: now,
    jti: &"refresh-{userId}-{now}"
  )
  
  TokenPair(
    accessToken: createJWT(accessClaims, secret),
    refreshToken: createJWT(refreshClaims, refreshSecret),
    expiresIn: 900,
    tokenType: "Bearer"
  )
```

---

## 2. OAuth2 Authorization Code + PKCE

```nim
# oauth2_pkce.nim
# OAuth2 Authorization Code Flow with PKCE (Proof Key for Code Exchange)
# สำหรับ public clients (mobile, SPA) ที่ไม่สามารถเก็บ client_secret ได้

import std/[asyncdispatch, asynchttpserver, httpclient, uri, json]
import std/[strformat, strutils, base64, sha2, random, tables, times]
import std/[options, cgi]

type
  PKCEChallenge* = object
    verifier*: string   # สุ่มขึ้นมา client-side
    challenge*: string  # SHA256(verifier) แล้ว base64url encode
    method*: string     # "S256"

  AuthCode* = object
    code*: string
    clientId*: string
    userId*: string
    redirectUri*: string
    scopes*: seq[string]
    codeChallenge*: string
    expiresAt*: Time

  OAuth2Server* = ref object
    authCodes*: Table[string, AuthCode]
    clients*: Table[string, tuple[secret, redirectUri: string]]
    jwtSecret*: string

proc generateVerifier*(length = 64): string =
  ## Generate cryptographically random code verifier
  randomize()
  const chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~"
  result = ""
  for _ in 0..<length:
    result &= chars[rand(chars.len - 1)]

proc generateChallenge*(verifier: string): PKCEChallenge =
  ## S256 challenge: BASE64URL(SHA256(verifier))
  let hash = sha256(verifier)
  let challenge = encode(hash).strip(chars = {'='}).replace("+", "-").replace("/", "_")
  PKCEChallenge(verifier: verifier, challenge: challenge, method: "S256")

proc verifyChallenge*(verifier, challenge: string): bool =
  ## Server verifies the code verifier against stored challenge
  let expectedChallenge = generateChallenge(verifier).challenge
  expectedChallenge == challenge

proc newOAuth2Server*(jwtSecret: string): OAuth2Server =
  result = OAuth2Server(
    authCodes: initTable[string, AuthCode](),
    clients: initTable[string, tuple[secret, redirectUri: string]](),
    jwtSecret: jwtSecret
  )
  # Register a test client
  result.clients["myapp-spa"] = (secret: "", redirectUri: "http://localhost:3000/callback")

proc handleAuthorize*(server: OAuth2Server, req: Request): Future[void] {.async.} =
  ## GET /authorize — redirect ไป login page หรือออก auth code
  let params = req.url.query.decodeQuery()
  let paramsTable = newStringTable()
  for (k, v) in params:
    paramsTable[k] = v
  
  let clientId = paramsTable.getOrDefault("client_id", "")
  let redirectUri = paramsTable.getOrDefault("redirect_uri", "")
  let codeChallenge = paramsTable.getOrDefault("code_challenge", "")
  let scopes = paramsTable.getOrDefault("scope", "").split(' ')
  let state = paramsTable.getOrDefault("state", "")
  
  if not server.clients.hasKey(clientId):
    await req.respond(Http400, """{"error":"invalid_client"}""")
    return
  
  # In real app: redirect to login page, then after login issue code
  # For demo: auto-issue code for user "demo-user"
  let code = &"code-{clientId}-{epochTime().int}"
  server.authCodes[code] = AuthCode(
    code: code,
    clientId: clientId,
    userId: "demo-user",
    redirectUri: redirectUri,
    scopes: scopes,
    codeChallenge: codeChallenge,
    expiresAt: getTime() + initDuration(seconds = 300)  # 5 min
  )
  
  let callbackUrl = &"{redirectUri}?code={encodeUrl(code)}&state={encodeUrl(state)}"
  await req.respond(Http302, "", newHttpHeaders({"Location": callbackUrl}))

proc handleToken*(server: OAuth2Server, req: Request): Future[void] {.async.} =
  ## POST /token — exchange code for tokens
  let body = req.body.decodeQuery()
  var params = newStringTable()
  for (k, v) in body:
    params[k] = v
  
  let grantType = params.getOrDefault("grant_type", "")
  
  if grantType == "authorization_code":
    let code = params.getOrDefault("code", "")
    let codeVerifier = params.getOrDefault("code_verifier", "")
    
    if not server.authCodes.hasKey(code):
      await req.respond(Http400, """{"error":"invalid_grant"}""")
      return
    
    let authCode = server.authCodes[code]
    
    # Verify PKCE
    if authCode.codeChallenge.len > 0:
      if not verifyChallenge(codeVerifier, authCode.codeChallenge):
        await req.respond(Http400, """{"error":"invalid_grant","error_description":"PKCE verification failed"}""")
        return
    
    # Check expiry
    if getTime() > authCode.expiresAt:
      server.authCodes.del(code)
      await req.respond(Http400, """{"error":"invalid_grant","error_description":"Code expired"}""")
      return
    
    # Issue tokens
    server.authCodes.del(code)  # One-time use
    let tokenPair = issueTokenPair(
      authCode.userId, &"{authCode.userId}@example.com",
      @["user"], server.jwtSecret, server.jwtSecret & "-refresh"
    )
    
    let response = %*{
      "access_token": tokenPair.accessToken,
      "refresh_token": tokenPair.refreshToken,
      "token_type": tokenPair.tokenType,
      "expires_in": tokenPair.expiresIn,
      "scope": authCode.scopes.join(" ")
    }
    await req.respond(Http200, $response)
  
  elif grantType == "refresh_token":
    let refreshToken = params.getOrDefault("refresh_token", "")
    let (claims, err) = verifyJWT(
      refreshToken, server.jwtSecret & "-refresh",
      expectedAudience = "myapp-refresh"
    )
    
    if err != jwtOk:
      await req.respond(Http400, &"""{"error":"invalid_grant","error_description":"{err}"}""")
      return
    
    let newPair = issueTokenPair(
      claims.sub, claims.email, claims.roles,
      server.jwtSecret, server.jwtSecret & "-refresh"
    )
    await req.respond(Http200, $(%*{
      "access_token": newPair.accessToken,
      "token_type": "Bearer",
      "expires_in": newPair.expiresIn
    }))

# Client-side PKCE flow helper
proc startOAuthFlow*(clientId, redirectUri, authEndpoint: string,
    scopes: seq[string]): tuple[url: string, verifier: string] =
  let pkce = generateChallenge(generateVerifier())
  let state = &"state-{epochTime().int}"
  
  let params = encodeUrl(clientId) & 
    "&response_type=code" &
    &"&redirect_uri={encodeUrl(redirectUri)}" &
    &"&scope={encodeUrl(scopes.join(' '))}" &
    &"&code_challenge={pkce.challenge}" &
    &"&code_challenge_method=S256" &
    &"&state={state}"
  
  (
    url: &"{authEndpoint}?client_id={params}",
    verifier: pkce.verifier
  )
```

---

## 3. API Key Management

```nim
# api_keys.nim
# API key generation, hashing, rate limiting

import std/[tables, times, strformat, strutils, asyncdispatch]
import std/[sha2, base64, random, options, json]

type
  ApiKeyScope* = enum
    aksRead
    aksWrite
    aksAdmin

  ApiKey* = object
    id*: string           # Public key prefix (not secret)
    hashedKey*: string    # SHA256 hash of actual key
    name*: string         # Human-readable name
    scopes*: set[ApiKeyScope]
    createdAt*: Time
    lastUsedAt*: Time
    expiresAt*: Option[Time]
    isRevoked*: bool
    rateLimit*: int       # requests per minute
    requestCount*: Table[int64, int]  # minute -> count

  ApiKeyStore* = ref object
    keys*: Table[string, ApiKey]  # key ID -> ApiKey

proc hashKey*(key: string): string =
  ## One-way hash of the API key
  let hash = sha256(key)
  encode(hash)

proc generateApiKey*(name: string, scopes: set[ApiKeyScope],
    rateLimit = 1000, expiresInDays = -1): tuple[key: ApiKey, secret: string] =
  ## Returns key metadata + the raw secret (only shown once)
  randomize()
  
  # Generate: "sk_live_" + 32 random bytes base64
  var rawBytes: array[32, byte]
  for i in 0..<32:
    rawBytes[i] = byte(rand(255))
  let secret = "sk_live_" & encode(rawBytes).strip(chars = {'='})
  
  let id = "key_" & secret[8..15]  # Use part of secret as ID prefix
  
  var expiry = none(Time)
  if expiresInDays > 0:
    expiry = some(getTime() + initDuration(days = expiresInDays))
  
  let key = ApiKey(
    id: id,
    hashedKey: hashKey(secret),
    name: name,
    scopes: scopes,
    createdAt: getTime(),
    rateLimit: rateLimit,
    expiresAt: expiry
  )
  
  (key: key, secret: secret)

proc validateApiKey*(store: ApiKeyStore, rawKey: string,
    requiredScope: ApiKeyScope): tuple[valid: bool, keyId: string, reason: string] =
  let hashed = hashKey(rawKey)
  
  for id, key in store.keys:
    if key.hashedKey == hashed:
      if key.isRevoked:
        return (false, id, "API key revoked")
      
      if key.expiresAt.isSome and getTime() > key.expiresAt.get():
        return (false, id, "API key expired")
      
      if requiredScope notin key.scopes:
        return (false, id, &"Insufficient scope (need {requiredScope})")
      
      # Check rate limit
      let minute = epochTime().int64 div 60
      let count = key.requestCount.getOrDefault(minute, 0)
      if count >= key.rateLimit:
        return (false, id, "Rate limit exceeded")
      
      return (true, id, "")
  
  (false, "", "Invalid API key")

proc revokeKey*(store: ApiKeyStore, keyId: string) =
  if store.keys.hasKey(keyId):
    store.keys[keyId].isRevoked = true
    echo &"[API Keys] Revoked key {keyId}"

# Middleware
proc apiKeyMiddleware*(store: ApiKeyStore, rawKey: string,
    requiredScope: ApiKeyScope): bool =
  let (valid, _, reason) = store.validateApiKey(rawKey, requiredScope)
  if not valid:
    echo &"[API Auth] Rejected: {reason}"
  valid
```

---

## 4. mTLS (Mutual TLS)

```nim
# mtls.nim
# Mutual TLS — both client and server verify each other's certificates
# สำหรับ service-to-service authentication

import std/[asyncdispatch, asyncnet, net, ssl, strformat, os]

# Generate test certificates (ใช้ openssl ใน real environment)
# openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.crt -days 365 -nodes
# openssl req -x509 -newkey rsa:4096 -keyout client.key -out client.crt -days 365 -nodes
# For mTLS: both signed by same CA, or exchange certs out-of-band

type
  MTLSConfig* = object
    certFile*: string    # Our certificate
    keyFile*: string     # Our private key
    caFile*: string      # CA certificate to verify peer
    serverName*: string  # Expected server hostname (client-side)

proc createServerContext*(config: MTLSConfig): SslContext =
  ## Create SSL context for mTLS server
  let ctx = newContext(
    verifyMode = SslCVerifyPeer,  # Require client cert
    certFile = config.certFile,
    keyFile = config.keyFile,
    caFile = config.caFile
  )
  return ctx

proc createClientContext*(config: MTLSConfig): SslContext =
  ## Create SSL context for mTLS client
  newContext(
    verifyMode = SslCVerifyPeer,
    certFile = config.certFile,
    keyFile = config.keyFile,
    caFile = config.caFile
  )

proc startMTLSServer*(config: MTLSConfig, port: int,
    handler: proc(client: AsyncSocket)): Future[void] {.async.} =
  let ctx = createServerContext(config)
  let server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(port))
  server.listen()
  
  echo &"[mTLS] Server listening on :{port} (mutual auth required)"
  
  while true:
    let (client, _) = await server.acceptAddr()
    
    # Wrap with TLS — client MUST provide certificate
    wrapConnectedSocket(ctx, client, handshakeAsServer)
    
    asyncCheck handler(client)

proc mtlsRequest*(url: string, config: MTLSConfig): Future[string] {.async.} =
  ## Make mTLS HTTP request
  let ctx = createClientContext(config)
  
  let uri = parseUri(url)
  let client = newAsyncSocket()
  await client.connect(uri.hostname, Port(uri.port.parseInt()))
  
  wrapConnectedSocket(ctx, client, handshakeAsClient, config.serverName)
  
  let req = &"GET {uri.path} HTTP/1.1\r\nHost: {uri.hostname}\r\nConnection: close\r\n\r\n"
  await client.send(req)
  
  var response = ""
  while true:
    let chunk = await client.recv(4096)
    if chunk.len == 0: break
    response &= chunk
  
  client.close()
  return response

# Certificate pinning (extra security layer)
type
  CertPin* = object
    host*: string
    pinnedSHA256*: string  # Expected cert fingerprint

proc checkCertPin*(ctx: SslContext, pin: CertPin): bool =
  ## Verify certificate matches pinned fingerprint
  # In practice: extract cert from context, hash it, compare
  # This is a simplified demonstration
  echo &"[CertPin] Verifying pin for {pin.host}"
  true  # Placeholder — real impl would check DER-encoded cert SHA256
```

---

## 5. Security Headers Middleware

```nim
# security_headers.nim
# HTTP security headers สำหรับ web applications

import std/[asyncdispatch, asynchttpserver, strformat, tables]

type
  SecurityHeadersConfig* = object
    hsts*: bool           # HTTP Strict Transport Security
    hstsMaxAge*: int      # HSTS max-age in seconds
    csp*: string          # Content-Security-Policy
    xFrameOptions*: string  # DENY, SAMEORIGIN
    xContentTypeOptions*: bool
    referrerPolicy*: string
    permissionsPolicy*: string
    cors*: CORSConfig

  CORSConfig* = object
    allowedOrigins*: seq[string]
    allowedMethods*: seq[string]
    allowedHeaders*: seq[string]
    maxAge*: int
    allowCredentials*: bool

proc defaultSecurityConfig*(): SecurityHeadersConfig =
  SecurityHeadersConfig(
    hsts: true,
    hstsMaxAge: 31536000,  # 1 year
    csp: "default-src 'self'; script-src 'self' 'nonce-{nonce}'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self'; frame-ancestors 'none'",
    xFrameOptions: "DENY",
    xContentTypeOptions: true,
    referrerPolicy: "strict-origin-when-cross-origin",
    permissionsPolicy: "camera=(), microphone=(), geolocation=()",
    cors: CORSConfig(
      allowedOrigins: @["https://myapp.com"],
      allowedMethods: @["GET", "POST", "PUT", "DELETE", "OPTIONS"],
      allowedHeaders: @["Content-Type", "Authorization", "X-Request-ID"],
      maxAge: 86400,
      allowCredentials: true
    )
  )

proc addSecurityHeaders*(headers: var HttpHeaders, config: SecurityHeadersConfig,
    origin = "", nonce = "") =
  ## Add security headers to response
  
  if config.hsts:
    headers["Strict-Transport-Security"] = 
      &"max-age={config.hstsMaxAge}; includeSubDomains; preload"
  
  let csp = if nonce.len > 0:
    config.csp.replace("{nonce}", nonce)
  else:
    config.csp.replace("'nonce-{nonce}'", "")
  headers["Content-Security-Policy"] = csp
  
  headers["X-Frame-Options"] = config.xFrameOptions
  
  if config.xContentTypeOptions:
    headers["X-Content-Type-Options"] = "nosniff"
  
  headers["Referrer-Policy"] = config.referrerPolicy
  headers["Permissions-Policy"] = config.permissionsPolicy
  headers["X-XSS-Protection"] = "0"  # Modern: disable (use CSP instead)
  
  # CORS
  let cors = config.cors
  if origin.len > 0 and (origin in cors.allowedOrigins or "*" in cors.allowedOrigins):
    headers["Access-Control-Allow-Origin"] = origin
    headers["Vary"] = "Origin"
    if cors.allowCredentials:
      headers["Access-Control-Allow-Credentials"] = "true"
    headers["Access-Control-Allow-Methods"] = cors.allowedMethods.join(", ")
    headers["Access-Control-Allow-Headers"] = cors.allowedHeaders.join(", ")
    headers["Access-Control-Max-Age"] = $cors.maxAge

# Request ID tracking
proc requestIdMiddleware*(req: Request): string =
  ## Generate or extract request ID for distributed tracing
  let existing = req.headers.getOrDefault("X-Request-ID", "")
  if existing.len > 0:
    return existing
  # Generate new ID
  randomize()
  var bytes: array[16, byte]
  for i in 0..<16:
    bytes[i] = byte(rand(255))
  result = encode(bytes).strip(chars = {'='})

# CSRF protection
type
  CSRFConfig* = object
    secret*: string
    cookieName*: string
    headerName*: string
    expirySeconds*: int

proc generateCSRFToken*(config: CSRFConfig, sessionId: string): string =
  let timestamp = epochTime().int
  let data = &"{sessionId}:{timestamp}"
  let mac = hmac_sha256(config.secret, data)
  urlSafeBase64Encode(mac) & "." & $timestamp

proc verifyCSRFToken*(config: CSRFConfig, token, sessionId: string): bool =
  let parts = token.split('.')
  if parts.len != 2:
    return false
  
  let timestamp = parseInt(parts[1])
  if epochTime().int - timestamp > config.expirySeconds:
    return false  # Token expired
  
  let data = &"{sessionId}:{timestamp}"
  let expectedMac = urlSafeBase64Encode(hmac_sha256(config.secret, data))
  
  # Constant-time comparison to prevent timing attacks
  if expectedMac.len != parts[0].len:
    return false
  
  var diff = 0
  for i in 0..<expectedMac.len:
    diff = diff or (ord(expectedMac[i]) xor ord(parts[0][i]))
  
  diff == 0
```

---

## 6. Secure Password Handling

```nim
# password.nim
# bcrypt, Argon2, PBKDF2 สำหรับ secure password storage

import std/[strformat, random, base64, sha2, strutils]

# Note: ใน production ให้ใช้ library nimcrypto หรือ bcrypt
# ตัวอย่างนี้ใช้ PBKDF2-SHA256 pattern

type
  HashAlgorithm* = enum
    haBcrypt    # ดีสำหรับ passwords (adaptive)
    haArgon2id  # ดีที่สุด (memory-hard)
    haPBKDF2    # Good, widely supported

  PasswordHash* = object
    algorithm*: HashAlgorithm
    hash*: string
    salt*: string
    iterations*: int
    version*: int

proc generateSalt*(length = 32): string =
  randomize()
  var bytes: array[32, byte]
  for i in 0..<32:
    bytes[i] = byte(rand(255))
  encode(bytes)

proc pbkdf2Sha256*(password, salt: string, iterations = 600000,
    dkLen = 32): string =
  ## PBKDF2-SHA256 — NIST recommended 600,000+ iterations (2023)
  var dk = newSeq[byte](dkLen)
  var saltBytes = salt & "\x00\x00\x00\x01"
  
  # Simplified PBKDF2 (real impl should use proper HMAC iteration)
  var u = sha256(password & saltBytes)
  var t = u
  
  for _ in 1..<iterations:
    u = sha256(password & u)
    for j in 0..<t.len:
      t[j] = t[j] xor u[j]
  
  encode(t)

proc hashPassword*(password: string, algorithm = haPBKDF2): PasswordHash =
  let salt = generateSalt()
  let hash = pbkdf2Sha256(password, salt)
  
  PasswordHash(
    algorithm: algorithm,
    hash: hash,
    salt: salt,
    iterations: 600000,
    version: 1
  )

proc verifyPassword*(password: string, stored: PasswordHash): bool =
  let computed = pbkdf2Sha256(password, stored.salt, stored.iterations)
  
  # Constant-time comparison
  if computed.len != stored.hash.len:
    return false
  var diff = 0
  for i in 0..<computed.len:
    diff = diff or (ord(computed[i]) xor ord(stored.hash[i]))
  diff == 0

proc serializeHash*(h: PasswordHash): string =
  ## Store as "$algorithm$iterations$salt$hash"
  &"${h.algorithm}${h.iterations}${h.salt}${h.hash}"

proc parseHash*(s: string): PasswordHash =
  let parts = s.split('$')
  PasswordHash(
    algorithm: parseEnum[HashAlgorithm](parts[1]),
    iterations: parseInt(parts[2]),
    salt: parts[3],
    hash: parts[4]
  )
```

---

## สรุป

| Mechanism | Use Case | Key Points |
|-----------|----------|-----------|
| JWT (HS256) | Stateless auth tokens | เก็บ secret อย่าง safe, ตั้ง exp สั้น |
| JWT (RS256) | เมื่อ verifier ไม่ควรรู้ secret | Public key สำหรับ verify |
| OAuth2 + PKCE | Public clients (SPA, mobile) | ไม่ต้องมี client_secret |
| API Keys | Server-to-server | Hash ก่อนเก็บใน DB |
| mTLS | Service mesh | Both sides ต้องมี cert |
| CSRF | Browser-based forms | Double-submit cookie pattern |
| Password hashing | User passwords | Argon2id > bcrypt > PBKDF2 |

---

**Next**: [Part 68 - Nim for Embedded Systems](../advanced/part68_embedded.md)
