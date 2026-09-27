# Go Fiber Tutorial — With Real-Life Examples

[Fiber](https://gofiber.io/) is a Go web framework inspired by Express.js, built on top of `fasthttp` — one of the fastest HTTP engines for Go. This tutorial walks through Fiber's core features using practical, real-world examples.

---

## 1. Installation & Setup

```bash
mkdir fiber-app && cd fiber-app
go mod init fiber-app
go get github.com/gofiber/fiber/v2
```

### Hello World

```go
package main

import "github.com/gofiber/fiber/v2"

func main() {
    app := fiber.New()

    app.Get("/", func(c *fiber.Ctx) error {
        return c.SendString("Hello, Fiber!")
    })

    app.Listen(":3000")
}
```

```bash
go run main.go
curl localhost:3000
```

---

## 2. Basic Routing

**Real-life example: a bookstore API skeleton**

```go
func main() {
    app := fiber.New()

    app.Get("/books", getBooks)
    app.Get("/books/:id", getBook)
    app.Post("/books", createBook)
    app.Put("/books/:id", updateBook)
    app.Delete("/books/:id", deleteBook)

    app.Listen(":3000")
}

func getBooks(c *fiber.Ctx) error {
    return c.SendString("List of all books")
}

func getBook(c *fiber.Ctx) error {
    id := c.Params("id")
    return c.SendString("Book with ID: " + id)
}

func createBook(c *fiber.Ctx) error {
    return c.SendString("Book created")
}

func updateBook(c *fiber.Ctx) error {
    return c.SendString("Book updated")
}

func deleteBook(c *fiber.Ctx) error {
    return c.SendString("Book deleted")
}
```

- `c.Params("id")` reads a URL path parameter (`/books/42` → `id = "42"`).
- Fiber's `Ctx` (`c`) is your gateway to everything about the request/response.

---

## 3. Route Parameters & Query Strings

**Real-life example: product search & filtering**

```go
// /products/:category?  -> optional param
app.Get("/products/:category?", func(c *fiber.Ctx) error {
    category := c.Params("category", "all") // default value if missing
    return c.SendString("Category: " + category)
})

// /search?q=laptop&minPrice=500&maxPrice=1500
app.Get("/search", func(c *fiber.Ctx) error {
    query := c.Query("q")
    minPrice := c.Query("minPrice", "0")     // default "0"
    maxPrice := c.Query("maxPrice", "999999")

    return c.JSON(fiber.Map{
        "query":    query,
        "minPrice": minPrice,
        "maxPrice": maxPrice,
    })
})
```

```bash
curl "localhost:3000/search?q=laptop&minPrice=500"
```

---

## 4. Handling JSON: Request Bodies & Responses

**Real-life example: user registration**

```go
type User struct {
    Name     string `json:"name"`
    Email    string `json:"email"`
    Password string `json:"password"`
}

app.Post("/register", func(c *fiber.Ctx) error {
    var user User

    if err := c.BodyParser(&user); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid request body",
        })
    }

    // (in real life you'd hash the password, save to DB, etc.)
    return c.Status(fiber.StatusCreated).JSON(fiber.Map{
        "message": "User registered successfully",
        "user":    user.Name,
    })
})
```

```bash
curl -X POST localhost:3000/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com","password":"secret123"}'
```

- `c.BodyParser(&user)` works for JSON, form data, and XML — Fiber auto-detects based on `Content-Type`.
- `fiber.Map` is a shorthand for `map[string]interface{}`, handy for quick JSON responses.

---

## 5. Input Validation

**Real-life example: validating a signup form**

```bash
go get github.com/go-playground/validator/v10
```

```go
import "github.com/go-playground/validator/v10"

type SignupRequest struct {
    Name     string `json:"name" validate:"required,min=2"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8"`
}

var validate = validator.New()

app.Post("/signup", func(c *fiber.Ctx) error {
    var req SignupRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "invalid body"})
    }

    if err := validate.Struct(req); err != nil {
        return c.Status(422).JSON(fiber.Map{"error": err.Error()})
    }

    return c.JSON(fiber.Map{"message": "Signup valid, proceeding..."})
})
```

---

## 6. Middleware

Middleware runs before/after your handlers — great for logging, auth, CORS, rate limiting.

**Real-life example: logging, CORS, and recovering from panics**

```bash
go get github.com/gofiber/fiber/v2/middleware/logger
go get github.com/gofiber/fiber/v2/middleware/cors
go get github.com/gofiber/fiber/v2/middleware/recover
```

```go
import (
    "github.com/gofiber/fiber/v2/middleware/cors"
    "github.com/gofiber/fiber/v2/middleware/logger"
    "github.com/gofiber/fiber/v2/middleware/recover"
)

func main() {
    app := fiber.New()

    app.Use(logger.New())   // logs every request: method, path, status, latency
    app.Use(recover.New())  // recovers from panics so the server doesn't crash
    app.Use(cors.New(cors.Config{
        AllowOrigins: "https://myfrontend.com",
        AllowMethods: "GET,POST,PUT,DELETE",
    }))

    app.Get("/", func(c *fiber.Ctx) error {
        return c.SendString("Protected by middleware!")
    })

    app.Listen(":3000")
}
```

### Custom Middleware Example: Request Timer

```go
app.Use(func(c *fiber.Ctx) error {
    start := time.Now()
    err := c.Next() // pass control to the next handler
    log.Printf("%s %s took %v", c.Method(), c.Path(), time.Since(start))
    return err
})
```

---

## 7. Real-Life Example: JWT Authentication

A common real-world need — protecting routes so only logged-in users can access them.

```bash
go get github.com/golang-jwt/jwt/v5
```

```go
import (
    "time"
    "github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("your-secret-key")

// --- Login: issue a token ---
app.Post("/login", func(c *fiber.Ctx) error {
    type LoginRequest struct {
        Email    string `json:"email"`
        Password string `json:"password"`
    }
    var req LoginRequest
    c.BodyParser(&req)

    // (pretend this is a real DB check)
    if req.Email != "alice@example.com" || req.Password != "secret123" {
        return c.Status(401).JSON(fiber.Map{"error": "invalid credentials"})
    }

    claims := jwt.MapClaims{
        "email": req.Email,
        "exp":   time.Now().Add(time.Hour * 24).Unix(),
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, _ := token.SignedString(jwtSecret)

    return c.JSON(fiber.Map{"token": signed})
})

// --- Middleware: verify the token ---
func authRequired(c *fiber.Ctx) error {
    authHeader := c.Get("Authorization") // e.g. "Bearer eyJhbGci..."
    if authHeader == "" {
        return c.Status(401).JSON(fiber.Map{"error": "missing token"})
    }

    tokenStr := strings.TrimPrefix(authHeader, "Bearer ")
    token, err := jwt.Parse(tokenStr, func(t *jwt.Token) (interface{}, error) {
        return jwtSecret, nil
    })
    if err != nil || !token.Valid {
        return c.Status(401).JSON(fiber.Map{"error": "invalid or expired token"})
    }

    claims := token.Claims.(jwt.MapClaims)
    c.Locals("email", claims["email"]) // pass data to the next handler
    return c.Next()
}

// --- Protected route ---
app.Get("/profile", authRequired, func(c *fiber.Ctx) error {
    email := c.Locals("email")
    return c.JSON(fiber.Map{"message": "Welcome!", "email": email})
})
```

```bash
# 1. Login and get a token
curl -X POST localhost:3000/login -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"secret123"}'

# 2. Use the token on a protected route
curl localhost:3000/profile -H "Authorization: Bearer <token-here>"
```

---

## 8. Route Groups & API Versioning

**Real-life example: versioned e-commerce API**

```go
func main() {
    app := fiber.New()

    api := app.Group("/api")

    v1 := api.Group("/v1")
    v1.Get("/products", getProductsV1)
    v1.Get("/orders", getOrdersV1)

    v2 := api.Group("/v2")
    v2.Get("/products", getProductsV2) // improved response format

    // group with its own middleware (e.g. admin-only routes)
    admin := api.Group("/admin", authRequired, adminOnly)
    admin.Get("/dashboard", getDashboard)
    admin.Delete("/users/:id", deleteUser)

    app.Listen(":3000")
}
```

- `/api/v1/products`, `/api/v2/products`, `/api/admin/dashboard` — all cleanly namespaced.
- Groups can chain their own middleware, applied only to routes within that group.

---

## 9. File Uploads

**Real-life example: profile picture upload**

```go
app.Post("/upload", func(c *fiber.Ctx) error {
    file, err := c.FormFile("avatar")
    if err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "no file uploaded"})
    }

    // basic validation
    if file.Size > 5*1024*1024 { // 5MB limit
        return c.Status(400).JSON(fiber.Map{"error": "file too large"})
    }

    savePath := "./uploads/" + file.Filename
    if err := c.SaveFile(file, savePath); err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "failed to save file"})
    }

    return c.JSON(fiber.Map{"message": "uploaded", "filename": file.Filename})
})
```

```bash
curl -X POST localhost:3000/upload -F "avatar=@photo.jpg"
```

### Serving Uploaded Files (Static Files)

```go
app.Static("/uploads", "./uploads") // serves files at /uploads/photo.jpg
```

---

## 10. Real-Life Example: Todo App with a Database (GORM + SQLite)

A complete CRUD example — the kind of thing you'd actually ship.

```bash
go get github.com/gofiber/fiber/v2
go get gorm.io/gorm
go get gorm.io/driver/sqlite
```

```go
package main

import (
    "github.com/gofiber/fiber/v2"
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type Todo struct {
    ID    uint   `json:"id" gorm:"primaryKey"`
    Title string `json:"title"`
    Done  bool   `json:"done"`
}

var db *gorm.DB

func main() {
    var err error
    db, err = gorm.Open(sqlite.Open("todos.db"), &gorm.Config{})
    if err != nil {
        panic("failed to connect database")
    }
    db.AutoMigrate(&Todo{})

    app := fiber.New()

    app.Get("/todos", getTodos)
    app.Post("/todos", createTodo)
    app.Put("/todos/:id", updateTodo)
    app.Delete("/todos/:id", deleteTodo)

    app.Listen(":3000")
}

func getTodos(c *fiber.Ctx) error {
    var todos []Todo
    db.Find(&todos)
    return c.JSON(todos)
}

func createTodo(c *fiber.Ctx) error {
    var todo Todo
    if err := c.BodyParser(&todo); err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "invalid body"})
    }
    db.Create(&todo)
    return c.Status(201).JSON(todo)
}

func updateTodo(c *fiber.Ctx) error {
    id := c.Params("id")
    var todo Todo
    if err := db.First(&todo, id).Error; err != nil {
        return c.Status(404).JSON(fiber.Map{"error": "todo not found"})
    }

    var updates Todo
    c.BodyParser(&updates)
    todo.Title = updates.Title
    todo.Done = updates.Done
    db.Save(&todo)

    return c.JSON(todo)
}

func deleteTodo(c *fiber.Ctx) error {
    id := c.Params("id")
    if err := db.Delete(&Todo{}, id).Error; err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "could not delete"})
    }
    return c.JSON(fiber.Map{"message": "deleted"})
}
```

```bash
curl -X POST localhost:3000/todos -H "Content-Type: application/json" -d '{"title":"Buy milk"}'
curl localhost:3000/todos
curl -X PUT localhost:3000/todos/1 -d '{"title":"Buy milk","done":true}'
curl -X DELETE localhost:3000/todos/1
```

---

## 11. Error Handling

**Real-life example: centralized error responses**

```go
app := fiber.New(fiber.Config{
    ErrorHandler: func(c *fiber.Ctx, err error) error {
        code := fiber.StatusInternalServerError
        if e, ok := err.(*fiber.Error); ok {
            code = e.Code
        }
        return c.Status(code).JSON(fiber.Map{
            "error": err.Error(),
        })
    },
})

app.Get("/risky", func(c *fiber.Ctx) error {
    return fiber.NewError(fiber.StatusNotFound, "resource not found")
})
```

Any handler that returns `fiber.NewError(...)` (or any error) is caught here and turned into a consistent JSON error response — no repeated boilerplate in every handler.

---

## 12. Real-Life Example: Rate Limiting a Public API

Protect an endpoint people might abuse (e.g. a password-reset or OTP endpoint).

```bash
go get github.com/gofiber/fiber/v2/middleware/limiter
```

```go
import "github.com/gofiber/fiber/v2/middleware/limiter"

app.Post("/forgot-password", limiter.New(limiter.Config{
    Max:        3,               // 3 requests
    Expiration: 1 * time.Minute, // per minute
    LimitReached: func(c *fiber.Ctx) error {
        return c.Status(429).JSON(fiber.Map{
            "error": "too many requests, try again later",
        })
    },
}), func(c *fiber.Ctx) error {
    return c.JSON(fiber.Map{"message": "reset link sent"})
})
```

---

## 13. Real-Life Example: Real-Time Chat with WebSockets

```bash
go get github.com/gofiber/websocket/v2
```

```go
import "github.com/gofiber/websocket/v2"

app.Get("/ws/:room", websocket.New(func(c *websocket.Conn) {
    room := c.Params("room")

    for {
        mt, msg, err := c.ReadMessage()
        if err != nil {
            break // client disconnected
        }
        log.Printf("Room %s received: %s", room, msg)

        // echo back to the same client (in real life, broadcast to all clients in the room)
        if err := c.WriteMessage(mt, msg); err != nil {
            break
        }
    }
}))
```

Connect from a browser console: `new WebSocket("ws://localhost:3000/ws/general")`.

---

## 14. Environment Config & Graceful Shutdown

**Real-life example: production-ready startup**

```bash
go get github.com/joho/godotenv
```

`.env` file:
```
PORT=3000
DB_URL=postgres://user:pass@localhost:5432/mydb
```

```go
import (
    "os"
    "os/signal"
    "syscall"
    "github.com/joho/godotenv"
)

func main() {
    godotenv.Load()
    port := os.Getenv("PORT")

    app := fiber.New()
    app.Get("/", func(c *fiber.Ctx) error { return c.SendString("OK") })

    go func() {
        if err := app.Listen(":" + port); err != nil {
            log.Fatal(err)
        }
    }()

    // graceful shutdown on Ctrl+C / SIGTERM
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit

    log.Println("shutting down...")
    app.Shutdown()
}
```

---

## 15. Testing Fiber Handlers

```go
package main

import (
    "net/http/httptest"
    "testing"

    "github.com/gofiber/fiber/v2"
)

func TestHelloRoute(t *testing.T) {
    app := fiber.New()
    app.Get("/", func(c *fiber.Ctx) error {
        return c.SendString("Hello, Fiber!")
    })

    req := httptest.NewRequest("GET", "/", nil)
    resp, _ := app.Test(req)

    if resp.StatusCode != 200 {
        t.Errorf("expected 200, got %d", resp.StatusCode)
    }
}
```

```bash
go test ./...
```

---

## Quick Reference Cheat Sheet

| Task                      | Code                                              |
|---------------------------|----------------------------------------------------|
| Create app                | `app := fiber.New()`                               |
| Route param               | `c.Params("id")`                                   |
| Query param               | `c.Query("q", "default")`                          |
| Parse JSON body           | `c.BodyParser(&struct)`                            |
| Send JSON                 | `c.JSON(fiber.Map{...})`                           |
| Set status + JSON         | `c.Status(201).JSON(...)`                          |
| Middleware                | `app.Use(middleware.New())`                        |
| Route group               | `app.Group("/api")`                                |
| Protected route           | `app.Get("/x", authMiddleware, handler)`           |
| File upload               | `c.FormFile("field")` + `c.SaveFile(file, path)`   |
| Static files              | `app.Static("/path", "./dir")`                     |
| Custom error              | `fiber.NewError(code, "message")`                  |
| Read header               | `c.Get("Authorization")`                           |
| Store per-request data    | `c.Locals("key", value)`                           |
| Graceful shutdown         | `app.Shutdown()`                                   |

---

## Common Gotchas

- **`c.BodyParser` needs a pointer** — `&user`, not `user`, or fields won't populate.
- **Route order matters** — `/books/:id` before a catch-all `/books/*` can shadow more specific routes if declared after them.
- **`fiber.Ctx` is reused across requests** — never store it in a goroutine or a struct for later use; copy the data you need out of it first.
- **Middleware order matters** — `app.Use(recover.New())` should generally be registered early so it can catch panics from everything after it.
- **CORS must be registered before routes** it protects — middleware only applies to routes registered after it (or globally if used at the top with `app.Use`).

---

**Next steps:** explore Fiber's [official docs](https://docs.gofiber.io/), the `fiber/middleware` package for more built-ins (compression, caching, CSRF), and pair it with `sqlx` or `GORM` for real database-backed apps.
