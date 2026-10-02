# Part 49: GraphQL ใน Nim

## GraphQL คืออะไร

```
REST API:
  GET /users/1
  GET /users/1/posts
  GET /users/1/posts/5/comments
  (multiple requests, over/under fetching)

GraphQL:
  Single endpoint POST /graphql
  query {
    user(id: 1) {
      name
      posts {
        title
        comments { body }
      }
    }
  }
  (one request, exactly what you need)

Nim package: nimble install graphql
```

## GraphQL Schema

```nim
# graphql_demo.nim
import graphql, asyncdispatch, json, asynchttpserver, strutils

# Define schema
const schema = """
type User {
  id: Int!
  username: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: Int!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
}

type Comment {
  id: Int!
  body: String!
  author: User!
}

type Query {
  user(id: Int!): User
  users: [User!]!
  post(id: Int!): Post
  posts(userId: Int): [Post!]!
  search(query: String!): [Post!]!
}

type Mutation {
  createUser(username: String!, email: String!): User!
  createPost(title: String!, content: String!, authorId: Int!): Post!
  addComment(postId: Int!, body: String!, authorId: Int!): Comment!
}

type Subscription {
  newPost: Post!
  newComment(postId: Int!): Comment!
}
"""

# In-memory data
type
  User = object
    id: int
    username: string
    email: string
  
  Post = object
    id: int
    title, content: string
    authorId: int
  
  Comment = object
    id: int
    body: string
    postId, authorId: int

var users = @[
  User(id: 1, username: "alice", email: "alice@mail.com"),
  User(id: 2, username: "bob",   email: "bob@mail.com")
]
var posts = @[
  Post(id: 1, title: "Hello World", content: "First post!", authorId: 1),
  Post(id: 2, title: "Nim is Great", content: "Love Nim!", authorId: 2)
]
var comments = @[
  Comment(id: 1, body: "Nice post!", postId: 1, authorId: 2)
]
var nextUserId = 3
var nextPostId = 3
var nextCommentId = 2

# Resolvers
proc resolveUser(context: GraphQLContext, id: int): JsonNode =
  for u in users:
    if u.id == id:
      return %*{"id": u.id, "username": u.username, "email": u.email}
  newJNull()

proc resolveUsers(context: GraphQLContext): JsonNode =
  result = newJArray()
  for u in users:
    result.add(%*{"id": u.id, "username": u.username, "email": u.email})

proc resolveUserPosts(context: GraphQLContext, userId: int): JsonNode =
  result = newJArray()
  for p in posts:
    if p.authorId == userId:
      result.add(%*{"id": p.id, "title": p.title, "content": p.content})

proc resolvePostComments(context: GraphQLContext, postId: int): JsonNode =
  result = newJArray()
  for c in comments:
    if c.postId == postId:
      result.add(%*{"id": c.id, "body": c.body})

# Setup GraphQL server
let gql = newGraphQL(schema)

gql.addResolver("Query", "user") do (ctx: GraphQLContext, args: JsonNode) -> JsonNode:
  resolveUser(ctx, args["id"].getInt())

gql.addResolver("Query", "users") do (ctx: GraphQLContext, args: JsonNode) -> JsonNode:
  resolveUsers(ctx)

gql.addResolver("Query", "post") do (ctx: GraphQLContext, args: JsonNode) -> JsonNode:
  let id = args["id"].getInt()
  for p in posts:
    if p.id == id:
      return %*{"id": p.id, "title": p.title, "content": p.content, "authorId": p.authorId}
  newJNull()

gql.addResolver("User", "posts") do (ctx: GraphQLContext, args: JsonNode) -> JsonNode:
  let userId = ctx.parent["id"].getInt()
  resolveUserPosts(ctx, userId)

gql.addResolver("Mutation", "createUser") do (ctx: GraphQLContext, args: JsonNode) -> JsonNode:
  let user = User(
    id: nextUserId,
    username: args["username"].getStr(),
    email: args["email"].getStr()
  )
  inc nextUserId
  users.add(user)
  %*{"id": user.id, "username": user.username, "email": user.email}

gql.addResolver("Mutation", "createPost") do (ctx: GraphQLContext, args: JsonNode) -> JsonNode:
  let post = Post(
    id: nextPostId,
    title: args["title"].getStr(),
    content: args["content"].getStr(),
    authorId: args["authorId"].getInt()
  )
  inc nextPostId
  posts.add(post)
  %*{"id": post.id, "title": post.title, "content": post.content}

# HTTP handler
proc handler(req: Request) {.async.} =
  if req.url.path == "/graphql":
    case req.reqMethod:
    of HttpPost:
      let body = parseJson(req.body)
      let query = body["query"].getStr()
      let variables = body.getOrDefault("variables", newJNull())
      
      let result = await gql.execute(query, variables)
      await req.respond(Http200, $result,
        newHttpHeaders({"Content-Type": "application/json"}))
    
    of HttpGet:
      # GraphQL playground
      let html = """
<!DOCTYPE html><html><body>
<h1>GraphQL Playground</h1>
<p>POST /graphql with JSON: {"query": "{ users { id username } }"}</p>
</body></html>"""
      await req.respond(Http200, html, newHttpHeaders({"Content-Type": "text/html"}))
    
    else:
      await req.respond(Http405, "Method Not Allowed")
  else:
    await req.respond(Http404, "Not Found")

echo "GraphQL server on :4000"
let server = newAsyncHttpServer()
waitFor server.serve(Port(4000), handler)
```

## GraphQL ตัวอย่าง Query

```graphql
# query_examples.graphql

# Basic query
query GetUser {
  user(id: 1) {
    id
    username
    posts {
      title
    }
  }
}

# ระบุแค่เฉพาะ field ที่ต้องการ
query GetUserPosts {
  user(id: 1) {
    username
    posts {
      id
      title
      content
      comments {
        body
        author {
          username
        }
      }
    }
  }
}

# Mutation
mutation CreatePost {
  createPost(
    title: "New Post"
    content: "Content here"
    authorId: 1
  ) {
    id
    title
  }
}

# Variables
query GetUserById($id: Int!) {
  user(id: $id) {
    id
    username
    email
  }
}
# Variables: {"id": 1}

# Fragments (สำหรับ reuse)
fragment UserFields on User {
  id
  username
  email
}

query GetUsers {
  users {
    ...UserFields
  }
}

# Introspection (ดูโครงสร้าง schema)
query IntrospectSchema {
  __schema {
    types {
      name
      kind
    }
  }
}
```

## สรุป Part 49

- ␅ GraphQL เทียบ REST
- ␅ Schema definition language
- ␅ Resolvers ใน Nim
- ␅ Query, Mutation, Subscription
- ␅ Variables และ Fragments

---
**Next**: [Part 50 - Windows Persistence](../security/part50_persistence.md)
