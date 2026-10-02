# Part 63 - Full Stack Application in Nim

## บทนำ

ในส่วนนี้เราจะสร้าง full-stack web application ใน Nim ตั้งแต่ต้นจนจบ โดยใช้ Jester + PostgreSQL + Redis + React frontend

---

## Project Structure

```
nimtodo/
├── nim_todo.nimble
├── src/
│   ├── main.nim              # Entry point
│   ├── config.nim            # Configuration
│   ├── db/
│   │   ├── connection.nim       # DB connection pool
│   │   ├── migrations.nim       # Schema migrations
│   │   └── repository.nim       # Data access layer
│   ├── models/
│   │   ├── user.nim             # User model
│   │   └── todo.nim             # Todo model
│   ├── services/
│   │   ├── auth_service.nim     # Authentication
│   │   └── todo_service.nim     # Business logic
│   ├── api/
│   │   ├── middleware.nim        # JWT, rate limit
│   │   ├── auth_routes.nim      # /api/auth/*
│   │   └── todo_routes.nim      # /api/todos/*
│   └── utils/
│       ├── crypto.nim           # Hashing, JWT
│       └── validation.nim       # Input validation
├── frontend/
│   ├── index.html
│   ├── app.js               # React app
│   └── style.css
└── tests/
    ├── test_auth.nim
    └── test_todos.nim
```

---

## 1. Configuration

```nim
# src/config.nim
import std/[os, strutils, strformat]

type
  Config* = object
    # Server
    host*: string
    port*: int
    
    # Database
    dbHost*: string
    dbPort*: int
    dbName*: string
    dbUser*: string
    dbPass*: string
    dbPoolSize*: int
    
    # Redis
    redisHost*: string
    redisPort*: int
    
    # Auth
    jwtSecret*: string
    jwtExpirySecs*: int
    bcryptCost*: int
    
    # App
    debug*: bool
    corsOrigins*: seq[string]

proc loadConfig*(): Config =
  result = Config(
    host: getEnv("HOST", "0.0.0.0"),
    port: parseInt(getEnv("PORT", "8080")),
    
    dbHost: getEnv("DB_HOST", "localhost"),
    dbPort: parseInt(getEnv("DB_PORT", "5432")),
    dbName: getEnv("DB_NAME", "nimtodo"),
    dbUser: getEnv("DB_USER", "postgres"),
    dbPass: getEnv("DB_PASS", "password"),
    dbPoolSize: parseInt(getEnv("DB_POOL_SIZE", "10")),
    
    redisHost: getEnv("REDIS_HOST", "localhost"),
    redisPort: parseInt(getEnv("REDIS_PORT", "6379")),
    
    jwtSecret: getEnv("JWT_SECRET", "change-me-in-production"),
    jwtExpirySecs: parseInt(getEnv("JWT_EXPIRY_SECS", "86400")),
    bcryptCost: parseInt(getEnv("BCRYPT_COST", "12")),
    
    debug: getEnv("DEBUG", "false") == "true",
    corsOrigins: getEnv("CORS_ORIGINS", "http://localhost:3000").split(",")
  )
  
  # Validate required in production
  when not defined(debug):
    if result.jwtSecret == "change-me-in-production":
      quit("JWT_SECRET must be set in production!")
```

---

## 2. Models

```nim
# src/models/user.nim
import std/[times, json, options]

type
  UserRole* = enum
    urUser = "user"
    urAdmin = "admin"
  
  User* = object
    id*: int
    email*: string
    username*: string
    passwordHash*: string
    role*: UserRole
    isActive*: bool
    createdAt*: DateTime
    updatedAt*: DateTime
  
  UserDTO* = object  # Safe for API responses (no password)
    id*: int
    email*: string
    username*: string
    role*: string
    createdAt*: string

proc toDTO*(u: User): UserDTO =
  UserDTO(
    id: u.id,
    email: u.email,
    username: u.username,
    role: $u.role,
    createdAt: u.createdAt.format("yyyy-MM-dd'T'HH:mm:ss")
  )

proc toJson*(dto: UserDTO): JsonNode =
  %*{
    "id": dto.id,
    "email": dto.email,
    "username": dto.username,
    "role": dto.role,
    "createdAt": dto.createdAt
  }
```

```nim
# src/models/todo.nim
import std/[times, json, options]

type
  TodoStatus* = enum
    tsPending = "pending"
    tsInProgress = "in_progress"
    tsDone = "done"
  
  Todo* = object
    id*: int
    userId*: int
    title*: string
    description*: string
    status*: TodoStatus
    priority*: int  # 1-5
    tags*: seq[string]
    dueDate*: Option[DateTime]
    createdAt*: DateTime
    updatedAt*: DateTime
  
  CreateTodoInput* = object
    title*: string
    description*: string
    priority*: int
    tags*: seq[string]
    dueDate*: Option[string]
  
  UpdateTodoInput* = object
    title*: Option[string]
    description*: Option[string]
    status*: Option[string]
    priority*: Option[int]

proc toJson*(todo: Todo): JsonNode =
  result = %*{
    "id": todo.id,
    "userId": todo.userId,
    "title": todo.title,
    "description": todo.description,
    "status": $todo.status,
    "priority": todo.priority,
    "tags": todo.tags,
    "createdAt": todo.createdAt.format("yyyy-MM-dd'T'HH:mm:ss")
  }
  if todo.dueDate.isSome:
    result["dueDate"] = %todo.dueDate.get().format("yyyy-MM-dd")
```

---

## 3. Database Layer

```nim
# src/db/repository.nim
import std/[db_postgres, options, strformat, sequtils, json, times]
import ../models/[user, todo]

type
  Repository* = object
    db*: DbConn

proc newRepository*(db: DbConn): Repository =
  Repository(db: db)

# --- Users ---

proc createUser*(repo: Repository, email, username, passwordHash: string,
                 role = urUser): User =
  let row = repo.db.getRow(
    sql"INSERT INTO users (email, username, password_hash, role) VALUES (?,?,?,?) RETURNING *",
    email, username, passwordHash, $role
  )
  User(
    id: parseInt(row[0]),
    email: row[1],
    username: row[2],
    passwordHash: row[3],
    role: parseEnum[UserRole](row[4]),
    isActive: row[5] == "t",
    createdAt: parse(row[6], "yyyy-MM-dd HH:mm:ss"),
    updatedAt: parse(row[7], "yyyy-MM-dd HH:mm:ss")
  )

proc findUserByEmail*(repo: Repository, email: string): Option[User] =
  let row = repo.db.getRow(sql"SELECT * FROM users WHERE email = ?", email)
  if row[0] == "": return none(User)
  some(User(
    id: parseInt(row[0]),
    email: row[1],
    username: row[2],
    passwordHash: row[3],
    role: parseEnum[UserRole](row[4]),
    isActive: row[5] == "t"
  ))

proc findUserById*(repo: Repository, id: int): Option[User] =
  let row = repo.db.getRow(sql"SELECT * FROM users WHERE id = ?", id)
  if row[0] == "": return none(User)
  some(User(
    id: parseInt(row[0]),
    email: row[1],
    username: row[2],
    passwordHash: row[3],
    role: parseEnum[UserRole](row[4]),
    isActive: row[5] == "t"
  ))

# --- Todos ---

proc createTodo*(repo: Repository, userId: int, input: CreateTodoInput): Todo =
  let tagsJson = $(%input.tags)
  let row = repo.db.getRow(
    sql"""INSERT INTO todos (user_id, title, description, priority, tags)
           VALUES (?,?,?,?,?) RETURNING *""",
    userId, input.title, input.description, input.priority, tagsJson
  )
  Todo(
    id: parseInt(row[0]),
    userId: parseInt(row[1]),
    title: row[2],
    description: row[3],
    status: tsPending,
    priority: parseInt(row[5]),
    tags: parseJson(row[6]).mapIt(it.getStr()),
    createdAt: parse(row[8], "yyyy-MM-dd HH:mm:ss")
  )

proc listTodos*(repo: Repository, userId: int,
                status: Option[string] = none(string),
                page = 1, pageSize = 20): seq[Todo] =
  result = @[]
  let offset = (page - 1) * pageSize
  
  var query = "SELECT * FROM todos WHERE user_id = ?"
  var params = @[$userId]
  
  if status.isSome:
    query.add(" AND status = ?")
    params.add(status.get())
  
  query.add(" ORDER BY created_at DESC LIMIT ? OFFSET ?")
  params.add($pageSize)
  params.add($offset)
  
  for row in repo.db.rows(sql(query), params):
    result.add(Todo(
      id: parseInt(row[0]),
      userId: parseInt(row[1]),
      title: row[2],
      description: row[3],
      status: parseEnum[TodoStatus](row[4]),
      priority: parseInt(row[5]),
      tags: parseJson(row[6]).mapIt(it.getStr()),
      createdAt: parse(row[8], "yyyy-MM-dd HH:mm:ss")
    ))

proc updateTodo*(repo: Repository, id, userId: int,
                 input: UpdateTodoInput): Option[Todo] =
  var setClauses: seq[string] = @[]
  var params: seq[string] = @[]
  
  if input.title.isSome:
    setClauses.add("title = ?")
    params.add(input.title.get())
  if input.status.isSome:
    setClauses.add("status = ?")
    params.add(input.status.get())
  if input.priority.isSome:
    setClauses.add("priority = ?")
    params.add($input.priority.get())
  
  if setClauses.len == 0: return none(Todo)
  
  setClauses.add("updated_at = NOW()")
  params.add($id)
  params.add($userId)
  
  let query = &"UPDATE todos SET {setClauses.join(", ")} WHERE id = ? AND user_id = ? RETURNING *"
  let row = repo.db.getRow(sql(query), params)
  
  if row[0] == "": return none(Todo)
  some(Todo(
    id: parseInt(row[0]),
    userId: parseInt(row[1]),
    title: row[2],
    status: parseEnum[TodoStatus](row[4])
  ))

proc deleteTodo*(repo: Repository, id, userId: int): bool =
  repo.db.exec(sql"DELETE FROM todos WHERE id = ? AND user_id = ?", id, userId)
  repo.db.changes() > 0
```

---

## 4. API Routes

```nim
# src/api/todo_routes.nim
import jester, json, options, strutils, strformat
import ../models/todo
import ../services/todo_service
import ./middleware

proc todoRoutes*(router: var Router, todoSvc: TodoService) =
  router.get "/api/todos":
    let userId = requireAuth(request)
    let page = parseInt(request.params.getOrDefault("page", "1"))
    let pageSize = parseInt(request.params.getOrDefault("pageSize", "20"))
    let statusFilter = request.params.getOrDefault("status", "")
    
    let filter = if statusFilter != "": some(statusFilter) else: none(string)
    let todos = todoSvc.listTodos(userId, filter, page, pageSize)
    
    resp Http200, $(todos.mapIt(it.toJson()).`%*`)
  
  router.post "/api/todos":
    let userId = requireAuth(request)
    
    var input: CreateTodoInput
    try:
      let body = parseJson(request.body)
      input = CreateTodoInput(
        title: body["title"].getStr(),
        description: body.getOrDefault("description").getStr(""),
        priority: body.getOrDefault("priority").getInt(3),
        tags: body.getOrDefault("tags").mapIt(it.getStr())
      )
    except:
      resp Http400, "{\"error\": \"Invalid request body\"}"
      return
    
    if input.title.len == 0:
      resp Http400, "{\"error\": \"Title is required\"}"
      return
    
    let todo = todoSvc.createTodo(userId, input)
    resp Http201, $todo.toJson()
  
  router.put "/api/todos/@id":
    let userId = requireAuth(request)
    let todoId = parseInt(request.params["id"])
    
    var input: UpdateTodoInput
    let body = parseJson(request.body)
    if body.hasKey("title"):    input.title = some(body["title"].getStr())
    if body.hasKey("status"):   input.status = some(body["status"].getStr())
    if body.hasKey("priority"): input.priority = some(body["priority"].getInt())
    
    let todo = todoSvc.updateTodo(todoId, userId, input)
    if todo.isNone:
      resp Http404, "{\"error\": \"Todo not found\"}"
    else:
      resp Http200, $todo.get().toJson()
  
  router.delete "/api/todos/@id":
    let userId = requireAuth(request)
    let todoId = parseInt(request.params["id"])
    
    if todoSvc.deleteTodo(todoId, userId):
      resp Http204, ""
    else:
      resp Http404, "{\"error\": \"Todo not found\"}"

# src/api/middleware.nim
proc requireAuth*(req: Request): int =
  ## Extract and validate JWT, return userId
  let authHeader = req.headers.getOrDefault("Authorization", "")
  if not authHeader.startsWith("Bearer "):
    raise newException(HttpError, "Missing or invalid Authorization header")
  
  let token = authHeader[7..^1]
  result = validateJWT(token)  # Returns userId

proc corsMiddleware*(resp: var Response, allowOrigins: seq[string]) =
  resp.headers["Access-Control-Allow-Origin"] = allowOrigins.join(",")
  resp.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE, OPTIONS"
  resp.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization"
```

---

## 5. Frontend (React)

```html
<!-- frontend/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Nim Todo App</title>
  <link rel="stylesheet" href="style.css">
  <script src="https://unpkg.com/react@18/umd/react.development.js"></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
</head>
<body>
  <div id="root"></div>
  <script type="text/babel" src="app.js"></script>
</body>
</html>
```

```javascript
// frontend/app.js
// Nim Todo App - React Frontend

const { useState, useEffect, useCallback } = React;

const API_BASE = '/api';

// API client
const api = {
  token: localStorage.getItem('authToken'),
  
  async request(method, path, body = null) {
    const headers = { 'Content-Type': 'application/json' };
    if (this.token) headers['Authorization'] = `Bearer ${this.token}`;
    
    const resp = await fetch(`${API_BASE}${path}`, {
      method,
      headers,
      body: body ? JSON.stringify(body) : null
    });
    
    if (resp.status === 401) {
      this.token = null;
      localStorage.removeItem('authToken');
      window.location.reload();
    }
    
    if (!resp.ok) {
      const err = await resp.json();
      throw new Error(err.error || 'Request failed');
    }
    
    return resp.status === 204 ? null : resp.json();
  },
  
  login: (email, password) => api.request('POST', '/auth/login', { email, password }),
  register: (email, username, password) => api.request('POST', '/auth/register', { email, username, password }),
  
  todos: {
    list: (params = {}) => api.request('GET', `/todos?${new URLSearchParams(params)}`),
    create: (data) => api.request('POST', '/todos', data),
    update: (id, data) => api.request('PUT', `/todos/${id}`, data),
    delete: (id) => api.request('DELETE', `/todos/${id}`),
  }
};

// --- Login Component ---
function LoginForm({ onLogin }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    try {
      const { token } = await api.login(email, password);
      api.token = token;
      localStorage.setItem('authToken', token);
      onLogin();
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };
  
  return (
    <div className="auth-container">
      <h2>Login to Nim Todo</h2>
      {error && <div className="error">{error}</div>}
      <form onSubmit={handleSubmit}>
        <input type="email" placeholder="Email" value={email}
               onChange={e => setEmail(e.target.value)} required />
        <input type="password" placeholder="Password" value={password}
               onChange={e => setPassword(e.target.value)} required />
        <button type="submit" disabled={loading}>
          {loading ? 'Logging in...' : 'Login'}
        </button>
      </form>
    </div>
  );
}

// --- Todo Item Component ---
function TodoItem({ todo, onUpdate, onDelete }) {
  const statusColors = { pending: '#ffc107', in_progress: '#007bff', done: '#28a745' };
  
  return (
    <div className="todo-item" style={{ borderLeft: `4px solid ${statusColors[todo.status]}` }}>
      <div className="todo-header">
        <h3 className={todo.status === 'done' ? 'completed' : ''}>{todo.title}</h3>
        <span className={`priority p${todo.priority}`}>P{todo.priority}</span>
      </div>
      {todo.description && <p>{todo.description}</p>}
      <div className="todo-tags">
        {todo.tags?.map(tag => <span key={tag} className="tag">{tag}</span>)}
      </div>
      <div className="todo-actions">
        <select value={todo.status}
                onChange={e => onUpdate(todo.id, { status: e.target.value })}>
          <option value="pending">Pending</option>
          <option value="in_progress">In Progress</option>
          <option value="done">Done</option>
        </select>
        <button className="btn-delete" onClick={() => onDelete(todo.id)}>Delete</button>
      </div>
    </div>
  );
}

// --- Main App ---
function App() {
  const [todos, setTodos] = useState([]);
  const [isLoggedIn, setIsLoggedIn] = useState(!!api.token);
  const [loading, setLoading] = useState(false);
  const [newTodo, setNewTodo] = useState({ title: '', priority: 3 });
  const [filter, setFilter] = useState('');
  
  const loadTodos = useCallback(async () => {
    setLoading(true);
    try {
      const params = filter ? { status: filter } : {};
      const data = await api.todos.list(params);
      setTodos(data);
    } finally {
      setLoading(false);
    }
  }, [filter]);
  
  useEffect(() => {
    if (isLoggedIn) loadTodos();
  }, [isLoggedIn, loadTodos]);
  
  const createTodo = async (e) => {
    e.preventDefault();
    const created = await api.todos.create(newTodo);
    setTodos(prev => [created, ...prev]);
    setNewTodo({ title: '', priority: 3 });
  };
  
  const updateTodo = async (id, changes) => {
    const updated = await api.todos.update(id, changes);
    setTodos(prev => prev.map(t => t.id === id ? updated : t));
  };
  
  const deleteTodo = async (id) => {
    await api.todos.delete(id);
    setTodos(prev => prev.filter(t => t.id !== id));
  };
  
  if (!isLoggedIn) return <LoginForm onLogin={() => setIsLoggedIn(true)} />;
  
  return (
    <div className="app">
      <header>
        <h1>Nim Todo App</h1>
        <button onClick={() => { api.token = null; localStorage.clear(); setIsLoggedIn(false); }}>
          Logout
        </button>
      </header>
      
      <form className="create-form" onSubmit={createTodo}>
        <input placeholder="New todo title..." value={newTodo.title}
               onChange={e => setNewTodo({...newTodo, title: e.target.value})} required />
        <select value={newTodo.priority}
                onChange={e => setNewTodo({...newTodo, priority: parseInt(e.target.value)})}>
          {[1,2,3,4,5].map(p => <option key={p} value={p}>Priority {p}</option>)}
        </select>
        <button type="submit">Add Todo</button>
      </form>
      
      <div className="filters">
        <button className={filter === '' ? 'active' : ''} onClick={() => setFilter('')}>All</button>
        <button className={filter === 'pending' ? 'active' : ''} onClick={() => setFilter('pending')}>Pending</button>
        <button className={filter === 'in_progress' ? 'active' : ''} onClick={() => setFilter('in_progress')}>In Progress</button>
        <button className={filter === 'done' ? 'active' : ''} onClick={() => setFilter('done')}>Done</button>
      </div>
      
      {loading ? <p>Loading...</p> : (
        <div className="todo-list">
          {todos.length === 0 ? <p className="empty">No todos yet!</p> :
           todos.map(todo => <TodoItem key={todo.id} todo={todo}
                                       onUpdate={updateTodo} onDelete={deleteTodo} />)}
        </div>
      )}
    </div>
  );
}

ReactDOM.render(<App />, document.getElementById('root'));
```

---

## 6. Database Migrations

```nim
# src/db/migrations.nim
import std/[db_postgres, strformat, logging]

proc runMigrations*(db: DbConn) =
  info "Running database migrations..."
  
  # Create migrations table
  db.exec sql"""
    CREATE TABLE IF NOT EXISTS migrations (
      id SERIAL PRIMARY KEY,
      name VARCHAR(255) UNIQUE NOT NULL,
      applied_at TIMESTAMP DEFAULT NOW()
    )
  """
  
  let migrations = [
    ("001_users", """
      CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        email VARCHAR(255) UNIQUE NOT NULL,
        username VARCHAR(100) UNIQUE NOT NULL,
        password_hash VARCHAR(255) NOT NULL,
        role VARCHAR(20) DEFAULT 'user',
        is_active BOOLEAN DEFAULT TRUE,
        created_at TIMESTAMP DEFAULT NOW(),
        updated_at TIMESTAMP DEFAULT NOW()
      );
      CREATE INDEX idx_users_email ON users(email);
    """),
    ("002_todos", """
      CREATE TABLE IF NOT EXISTS todos (
        id SERIAL PRIMARY KEY,
        user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
        title VARCHAR(500) NOT NULL,
        description TEXT DEFAULT '',
        status VARCHAR(20) DEFAULT 'pending',
        priority INTEGER DEFAULT 3 CHECK (priority BETWEEN 1 AND 5),
        tags JSONB DEFAULT '[]',
        due_date DATE,
        created_at TIMESTAMP DEFAULT NOW(),
        updated_at TIMESTAMP DEFAULT NOW()
      );
      CREATE INDEX idx_todos_user_id ON todos(user_id);
      CREATE INDEX idx_todos_status ON todos(status);
      CREATE INDEX idx_todos_tags ON todos USING gin(tags);
    """),
    ("003_refresh_tokens", """
      CREATE TABLE IF NOT EXISTS refresh_tokens (
        id SERIAL PRIMARY KEY,
        user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
        token VARCHAR(500) UNIQUE NOT NULL,
        expires_at TIMESTAMP NOT NULL,
        created_at TIMESTAMP DEFAULT NOW()
      );
      CREATE INDEX idx_refresh_tokens_token ON refresh_tokens(token);
    """)
  ]
  
  for (name, sql) in migrations:
    let applied = db.getValue(sql"SELECT COUNT(*) FROM migrations WHERE name = ?", name)
    if applied == "0":
      db.exec(sql(sql))
      db.exec(sql"INSERT INTO migrations (name) VALUES (?)", name)
      info &"Applied migration: {name}"
    else:
      debug &"Skipping migration {name} (already applied)"
  
  info "Migrations complete!"
```

---

## 7. Main Entry Point

```nim
# src/main.nim
import jester, asyncdispatch, db_postgres, logging, strformat
import ./config
import ./db/[connection, migrations]
import ./api/[auth_routes, todo_routes, middleware]
import ./services/[auth_service, todo_service]

when isMainModule:
  # Setup logging
  addHandler(newConsoleLogger())
  addHandler(newFileLogger("nimtodo.log", fmtStr = "[$datetime] [$levelid] "))
  
  let cfg = loadConfig()
  info &"Starting Nim Todo App on {cfg.host}:{cfg.port}"
  
  # Connect to database
  let db = open(
    &"{cfg.dbHost}:{cfg.dbPort}/{cfg.dbName}",
    cfg.dbUser, cfg.dbPass, ""
  )
  defer: db.close()
  
  runMigrations(db)
  
  # Initialize services
  let authSvc = newAuthService(db, cfg)
  let todoSvc = newTodoService(db)
  
  # Setup routes
  router mainRouter:
    # CORS preflight
    options "/**":
      resp Http200, ""
    
    # Health check
    get "/health":
      resp Http200, "{\"status\": \"ok\"}"
    
    # Static files
    get "/":
      resp readFile("frontend/index.html")
    
    get "/app.js":
      resp readFile("frontend/app.js")
  
  mainRouter.authRoutes(authSvc)
  mainRouter.todoRoutes(todoSvc)
  
  let settings = newSettings(
    port = Port(cfg.port),
    bindAddr = cfg.host
  )
  
  info &"Server ready at http://{cfg.host}:{cfg.port}"
  runForever()
```

---

## สรุป Part 63

| Layer | Technology |
|-------|------------|
| Backend | Nim + Jester |
| Database | PostgreSQL + migrations |
| Cache | Redis (auth sessions) |
| Frontend | React (vanilla JSX) |
| Auth | JWT + bcrypt |
| Dev | Docker Compose |

**Next**: [Part 64 - Testing Strategies in Nim](../advanced/part64_testing.md)
