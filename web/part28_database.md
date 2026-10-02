# Part 28: Database - SQLite และ PostgreSQL

## SQLite

```nim
import db_sqlite, json, strutils

# เปิด connection
let db = open("myapp.db", "", "", "")
defer: db.close()

# สร้าง tables
db.exec(sql"""
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    updated_at TEXT DEFAULT CURRENT_TIMESTAMP
  )
""")

db.exec(sql"""
  CREATE TABLE IF NOT EXISTS posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    content TEXT,
    published INTEGER DEFAULT 0,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
  )
""")

# สร้าง indexes
db.exec(sql"CREATE INDEX IF NOT EXISTS idx_posts_user ON posts(user_id)")
db.exec(sql"CREATE INDEX IF NOT EXISTS idx_posts_published ON posts(published)")

# INSERT
db.exec(sql"""
  INSERT INTO users (username, email, password_hash) 
  VALUES (?, ?, ?)
""", "alice", "alice@example.com", "hash123")

# Get last insert ID
let lastId = db.getLastId()
echo "Last ID: ", lastId

# SELECT
let row = db.getRow(sql"SELECT * FROM users WHERE id = ?", "1")
echo "User: ", row.join(", ")

# SELECT all
let rows = db.getAllRows(sql"SELECT id, username, email FROM users")
for row in rows:
  echo row[0], ": ", row[1], " (", row[2], ")"

# UPDATE
db.exec(sql"""
  UPDATE users 
  SET email = ?, updated_at = CURRENT_TIMESTAMP 
  WHERE id = ?
""", "newalice@example.com", "1")

# Transactions
db.exec(sql"BEGIN TRANSACTION")
try:
  db.exec(sql"INSERT INTO users VALUES (NULL, 'carol', 'carol@mail.com', 'hash789', datetime('now'), datetime('now'))")
  db.exec(sql"INSERT INTO posts VALUES (NULL, 3, 'My Post', 'Content...', 1, datetime('now'))")
  db.exec(sql"COMMIT")
except:
  db.exec(sql"ROLLBACK")
  raise
```

## ORM-like Pattern

```nim
import db_sqlite, options, strutils, sequtils, tables

type
  User = object
    id: int
    username: string
    email: string
    createdAt: string

  Post = object
    id: int
    userId: int
    title: string
    content: string
    published: bool

let db = open(":memory:", "", "", "")

db.exec(sql"""CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT, email TEXT, created_at TEXT DEFAULT CURRENT_TIMESTAMP)""")

# Model methods
proc create(db: DbConn, u: User): User =
  db.exec(sql"INSERT INTO users (username, email) VALUES (?, ?)", 
    u.username, u.email)
  let row = db.getRow(sql"SELECT * FROM users WHERE id = ?", $db.getLastId())
  User(id: parseInt(row[0]), username: row[1], email: row[2], createdAt: row[3])

proc findById(db: DbConn, T: typedesc[User], id: int): Option[User] =
  let row = db.getRow(sql"SELECT * FROM users WHERE id = ?", $id)
  if row[0].len == 0: return none(User)
  some(User(id: parseInt(row[0]), username: row[1], email: row[2], createdAt: row[3]))

proc findAll(db: DbConn, T: typedesc[User]): seq[User] =
  let rows = db.getAllRows(sql"SELECT * FROM users")
  rows.mapIt(User(id: parseInt(it[0]), username: it[1], email: it[2], createdAt: it[3]))

proc update(db: DbConn, u: User) =
  db.exec(sql"UPDATE users SET username = ?, email = ? WHERE id = ?",
    u.username, u.email, $u.id)

proc delete(db: DbConn, T: typedesc[User], id: int) =
  db.exec(sql"DELETE FROM users WHERE id = ?", $id)

# Usage
var u = db.create(User(username: "alice", email: "alice@mail.com"))
echo "Created: ", u.id, " ", u.username

let found = db.findById(User, u.id)
if found.isSome:
  var user = found.get()
  user.email = "newemail@mail.com"
  db.update(user)

let allUsers = db.findAll(User)
echo "All users: ", allUsers.len
```

## PostgreSQL

```nim
import db_postgres

# Connect
let db = open("host=localhost port=5432 dbname=myapp", "postgres", "password", "")
defer: db.close()

# Create tables
db.exec(sql"""
  CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
  )
""")

# INSERT with returning
let row = db.getRow(sql"""
  INSERT INTO users (username, email) 
  VALUES ($1, $2)
  RETURNING id, username, email, created_at
""", "alice", "alice@example.com")

echo "Created user: ", row

# Complex queries
let results = db.getAllRows(sql"""
  SELECT 
    u.id,
    u.username,
    COUNT(o.id) as order_count,
    SUM(o.total) as total_spent
  FROM users u
  LEFT JOIN orders o ON u.id = o.user_id
  WHERE u.created_at > NOW() - INTERVAL '30 days'
  GROUP BY u.id, u.username
  HAVING COUNT(o.id) > 0
  ORDER BY total_spent DESC
  LIMIT 10
""")

for row in results:
  echo row[1], ": ", row[2], " orders, $", row[3]

# Prepared statements
let stmt = db.prepare("stmt1", 
  "SELECT * FROM products WHERE category = $1 AND price < $2",
  2)

let products = db.execPrepared("stmt1", ["Electronics", "100"])
for p in products:
  echo p[1], ": $", p[2]

# JSONB support
db.exec(sql"""
  CREATE TABLE IF NOT EXISTS events (
    id SERIAL PRIMARY KEY,
    type TEXT,
    data JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
  )
""")

db.exec(sql"""
  INSERT INTO events (type, data) 
  VALUES ($1, $2::jsonb)
""", "user_signup", """{"username": "alice", "ip": "127.0.0.1"}""")
```

## Connection Pool

```nim
import db_postgres, locks, sequtils

type
  PooledConnection = object
    db: DbConn
    inUse: bool

  ConnectionPool = object
    connections: seq[PooledConnection]
    lock: Lock
    connStr, user, password, database: string

proc newConnectionPool(connStr, user, password, database: string, size: int): ConnectionPool =
  result = ConnectionPool(
    connections: newSeq[PooledConnection](size),
    connStr: connStr, user: user, password: password, database: database
  )
  initLock(result.lock)
  for i in 0..<size:
    let db = open(connStr, user, password, database)
    result.connections[i] = PooledConnection(db: db, inUse: false)

proc acquire(pool: var ConnectionPool): ptr PooledConnection =
  acquire(pool.lock)
  defer: release(pool.lock)
  for conn in pool.connections.mitems:
    if not conn.inUse:
      conn.inUse = true
      return addr conn
  raise newException(ResourceExhaustedError, "Connection pool exhausted")

template withDb(pool: var ConnectionPool, body: untyped): untyped =
  let conn = pool.acquire()
  let db {.inject.} = conn.db
  try:
    body
  finally:
    pool.release(conn)
```

## Migration System

```nim
import db_sqlite, os, algorithm, strutils

type Migration = object
  version: int
  name: string
  up: string
  down: string

proc initMigrationsTable(db: DbConn) =
  db.exec(sql"""
    CREATE TABLE IF NOT EXISTS migrations (
      version INTEGER PRIMARY KEY,
      name TEXT,
      applied_at TEXT DEFAULT CURRENT_TIMESTAMP
    )
  """)

proc getAppliedVersion(db: DbConn): int =
  let row = db.getRow(sql"SELECT MAX(version) FROM migrations")
  if row[0].len == 0: 0
  else: parseInt(row[0])

proc applyMigration(db: DbConn, m: Migration) =
  echo "Applying migration ", m.version, ": ", m.name
  db.exec(sql(m.up))
  db.exec(sql"INSERT INTO migrations (version, name) VALUES (?, ?)",
    $m.version, m.name)

let migrations = @[
  Migration(
    version: 1, name: "create_users",
    up: "CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)",
    down: "DROP TABLE users"
  ),
  Migration(
    version: 2, name: "add_email_to_users",
    up: "ALTER TABLE users ADD COLUMN email TEXT",
    down: "SELECT 1"
  ),
  Migration(
    version: 3, name: "create_posts",
    up: """CREATE TABLE posts (
      id INTEGER PRIMARY KEY,
      user_id INTEGER REFERENCES users(id),
      title TEXT, content TEXT)""",
    down: "DROP TABLE posts"
  )
]

proc migrate(db: DbConn, target: int = -1) =
  db.initMigrationsTable()
  let current = db.getAppliedVersion()
  let finalTarget = if target < 0: migrations[^1].version else: target
  if current == finalTarget:
    echo "Already at version ", current
    return
  let sorted = migrations.sortedByIt(it.version)
  if finalTarget > current:
    for m in sorted:
      if m.version > current and m.version <= finalTarget:
        db.applyMigration(m)
  echo "Migration complete. Version: ", db.getAppliedVersion()

let db2 = open(":memory:", "", "", "")
db2.migrate()
```

## สรุป Part 28

ในบทนี้เราได้เรียนรู้:
- ✅ SQLite: CRUD operations, transactions, indexes
- ✅ ORM-like pattern ใน Nim
- ✅ PostgreSQL: advanced queries, JSONB, prepared statements
- ✅ Connection pooling
- ✅ Database migration system

---

**Previous**: [Part 27 - Jester](part27_jester.md)
**Next**: [Part 29 - Authentication](part29_auth.md)
