# Backend Architecture & Contribution Guidelines (Go)

Welcome to the backend repository! This document outlines our folder structure, architectural patterns, and coding standards. By following these guidelines, you help maintain a clean, scalable, and readable codebase.

---

## 📁 Folder Structure

We follow a **Feature-Based (Modular)** architecture. Files are grouped by their **business domain**, not their technical role.

```text
backend/
├── cmd/
│   └── api/
│       └── main.go            # Entry point — starts the HTTP server on a port
├── internal/
│   ├── config/                 # Environment variables and app-wide constants
│   │   └── env.go              # Parses and validates all .env variables (envconfig/viper)
│   ├── lib/                    # Third-party client initializations (wrappers)
│   │   ├── db.go               # Initializes the shared *sql.DB / gorm.DB instance
│   │   └── redis.go            # Initializes the Redis (go-redis) client
│   ├── middleware/              # Global HTTP middlewares
│   │   ├── auth.go             # JWT authentication guard
│   │   ├── error_handler.go    # Centralized error response formatter (recover middleware)
│   │   ├── logger.go           # Request logging middleware
│   │   ├── rate_limiter.go     # Rate limiting rules
│   │   └── security.go         # CORS, security headers configuration
│   ├── modules/                 # Feature modules (Domain-driven)
│   │   └── featurename/         # One package per business domain (e.g., auth, user, group)
│   │       ├── featurename.go        # All request/response structs & validation tags for this module
│   │       ├── handler/
│   │       │   └── featurename_handler.go   # Handles HTTP request/response
│   │       ├── service/
│   │       │   └── featurename_service.go   # Core business logic & database queries
│   │       └── router/
│   │           └── featurename_router.go    # Route definitions for this module
│   └── utils/                   # Shared utilities and helpers
│       ├── app_error.go        # Custom error type for structured error throwing
│       ├── jwt.go              # JWT sign/verify service
│       ├── mailer.go           # Email sending logic
│       ├── redis_helper.go     # Redis helper functions (e.g., cache, permission)
│       └── tokenizer.go        # Token generation utilities
├── .env                        # Local environment variables (never commit this)
├── .env.example                # Template for required environment variables
├── go.mod                      # Module definition & dependency versions
└── go.sum
```

### `config/` vs `lib/` — What's the difference?

| Folder | Purpose | Example |
|---|---|---|
| `config/` | Reads, validates, and exports environment values | `env.go` parsing `.env` into a typed struct |
| `lib/` | Initializes third-party client instances using those values | `redis.go` creating a `go-redis` client |

> **Rule:** `lib/` files import from `config/`, never the other way around.

---

## 🏗️ Architectural Concepts

### 1. Handlers (`handler/`)
- **Responsibility:** Extract data from the HTTP request (params, body, query), call the service, and write the HTTP response.
- **Rule:** Handlers must be **thin**. No business logic, no database calls. All logic lives in the service.

### 2. Services (`service/`)
- **Responsibility:** All business logic and database interactions live here.
- **Rule:** Services must **not** know about `http.Request` or `http.ResponseWriter` (or the framework's context type beyond `context.Context`). They receive plain structs and return plain structs/errors. This makes them independently testable.

### 3. Routers (`router/`)
- **Responsibility:** Map HTTP methods and paths to handler functions. Apply module-specific middlewares (e.g., `middleware.Authenticate()`, `middleware.RateLimiter()`).

### 4. Schema Files (`featurename.go` at module root)
- **Responsibility:** Declare all **request/response structs** and their **validation tags** (using a library like `go-playground/validator`) for a given module.
- These are imported by both the service (for input validation) and anywhere a typed response is needed.

---

## ✍️ Coding Conventions & Naming Rules

### File Naming
Use **snake_case**, with a suffix describing the file's role.

| Type | Convention | Example |
|---|---|---|
| Handler | `name_handler.go` | `auth_handler.go` |
| Service | `name_service.go` | `group_service.go` |
| Router | `name_router.go` | `user_router.go` |
| Schema file | `name.go` (at module root) | `auth.go` |

### Package Naming
- Package names are **short, lowercase, single-word** — no underscores or camelCase (e.g., `auth`, `group`, `user`).
- The package name should match the module folder name.

### Variable & Function Naming
- Use **camelCase** for unexported variables and functions (`userID`, `getUserByID`).
- Use **PascalCase** for anything exported outside the package (`GetUserByID`, `CreateGroup`).
- Functions should use descriptive action verbs.

### Structs & Interfaces
- Use **PascalCase** for exported struct and interface names.
- Interface names describing a single behavior should end in `-er` where natural (`Notifier`, `TokenGenerator`).

### Struct Naming Convention (Zod-equivalent)
This project uses a strict naming convention for all request/response structs and their validation.

| Item | Suffix | Example |
|---|---|---|
| Request struct | `Params` | `LoginParams` |
| Response struct | `Response` | `LoginResponse` |

```go
// Always use this pattern
type LoginParams struct {
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8"`
}

type LoginResponse struct {
    Token string `json:"token"`
}
```

### Validation Tag Ordering (1 → 3)
Always follow this order when writing `validate` tags, to keep them consistent and readable:

1. **Presence** — `required`
2. **Type / Format** — `email`, `url`, `numeric`
3. **Constraints** — `min=`, `max=`, `oneof=`

```go
// Example — correct tag ordering
type CreateUserParams struct {
    Name  string `json:"name" validate:"required,min=1,max=50"`
    Email string `json:"email" validate:"required,email"`
    Age   int    `json:"age" validate:"omitempty,min=18"`
}
```

> **Important:** `required` fires when the field is the zero value (empty string, `0`, `nil`). For strings where `0`-length is meaningful but distinguishable from "not sent", prefer a pointer type (`*string`) combined with `omitempty`.

---

## 📡 HTTP Response Format

All API responses must follow a consistent JSON structure.

### Success Response
```json
{
  "success": true,
  "data": {}
}
```

### Success Response (with message)
```json
{
  "success": true,
  "message": "Operation completed successfully"
}
```

### Error Response
```json
{
  "success": false,
  "message": "A human-readable error description"
}
```

> The `success` boolean field is **always required** in every response.

---

## ⚠️ Error Handling

This project uses a custom `AppError` type for all predictable errors, combined with a global error-handling middleware.

### Defining `AppError`
```go
// utils/app_error.go
package utils

type AppError struct {
    StatusCode int
    Message    string
}

func (e *AppError) Error() string {
    return e.Message
}

func NewAppError(statusCode int, message string) *AppError {
    return &AppError{StatusCode: statusCode, Message: message}
}
```

### Throwing Errors in Services
```go
import "backend/internal/utils"

// In service
if group == nil {
    return nil, utils.NewAppError(404, "Group not found")
}
if !authorized {
    return nil, utils.NewAppError(403, "You are not authorized to perform this action")
}
```

### Handler Pattern — Always Return the Error, Never Write the Response Inline
All handlers **must** return the error up the chain (or pass it to the shared error-writer helper). Never format the JSON error response inline inside the handler.

```go
// ❌ Wrong — bypasses the global error handler, duplicates code everywhere
if err != nil {
    w.WriteHeader(http.StatusInternalServerError)
    json.NewEncoder(w).Encode(map[string]any{"success": false, "message": err.Error()})
    return
}

// ✅ Correct — delegates to the shared error-response writer
if err != nil {
    utils.WriteError(w, err)
    return
}
```

### The Recover/Error Middleware Must Wrap All Routes
Register the panic-recovery and error-formatting middleware **before** any route-specific middleware, so it wraps every handler in the router chain.

```go
// main.go
r := chi.NewRouter()
r.Use(middleware.Recoverer)     // ✅ error/panic handler FIRST in the chain
r.Use(middleware.Logger)

r.Mount("/auth", authRouter)    // routes after
r.Mount("/user", userRouter)
```

### Rules
- Always return an `*AppError` (or wrap it with `fmt.Errorf("...: %w", err)`) for predictable errors (not found, unauthorized, forbidden).
- **Never** swallow an error silently (`if err != nil { return }` with no logging/propagation).
- **Never** ignore errors with `_ = err`. Handle it or return it.
- Always propagate errors upward with `return nil, err` (or `return err`). Never write the HTTP response directly from inside a service.

---

## 🚀 How to Add a New Feature Module

1. Create a new package: `internal/modules/yourfeature/`
2. Create the schema file: `yourfeature.go` — define all request/response structs and validation tags.
3. Create `service/yourfeature_service.go` — write all business logic as a struct with methods, constructed via a constructor function (`NewYourFeatureService(...)`).
4. Create `handler/yourfeature_handler.go` — thin handler that calls the service.
5. Create `router/yourfeature_router.go` — register HTTP routes and attach middlewares.
6. Mount the router in `cmd/api/main.go`.

---

## 🗄️ Database Conventions

- All database schema changes are defined as **versioned SQL migration files** in `migrations/`.
- Never open more than one database connection pool. Use a single shared `*sql.DB` / `gorm.DB` instance created in `lib/db.go`.
- Use an explicit transaction (`db.Begin()` / `db.Transaction(func(tx *gorm.DB) error {...})`) for operations that must be **atomic** (all succeed or all fail).
- Always select only the columns you need (`Select(...)` in GORM, or explicit column lists in raw SQL). Avoid `SELECT *`.

---

*Thank you for contributing! Following these guidelines ensures our codebase remains robust, scalable, and a joy to work in.*
