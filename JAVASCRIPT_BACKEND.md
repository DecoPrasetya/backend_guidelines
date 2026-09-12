# Backend Architecture & Contribution Guidelines (Java Script)

Welcome to the backend repository! This document outlines our folder structure, architectural patterns, and coding standards. By following these guidelines, you help maintain a clean, scalable, and readable codebase.

---

## 📁 Folder Structure

We follow a **Feature-Based (Modular)** architecture. Files are grouped by their **business domain**, not their technical role.

```text
backend/
├── prisma/                    # Prisma ORM — schema and database migrations
│   ├── schema.prisma          # Single source of truth for all database models
│   ├── seed.ts                # Database seeding script
│   └── migrations/            # Auto-generated migration history
├── src/
│   ├── config/                # Environment variables and app-wide constants
│   │   └── env.ts             # Parses and validates all .env variables using Zod
│   ├── lib/                   # Third-party client initializations (wrappers)
│   │   └── redis.ts           # Initializes the Redis/ioredis client
│   ├── middleware/            # Global Express middlewares
│   │   ├── auth.ts            # JWT authentication guard
│   │   ├── errorHandler.ts    # Centralized error response formatter
│   │   ├── parser.ts          # Body parser, cookie parser setup
│   │   ├── rateLimiter.ts     # Rate limiting rules
│   │   └── security.ts        # Helmet, CORS configuration
│   ├── modules/               # Feature modules (Domain-driven)
│   │   └── feature-name/      # One folder per business domain (e.g., auth, user, group)
│   │       ├── feature-name.ts      # All Zod schemas & TypeScript types for this module
│   │       ├── controller/          # Handles HTTP request/response
│   │       │   └── feature-name.controller.ts
│   │       ├── service/             # Core business logic & database queries
│   │       │   └── feature-name.service.ts
│   │       └── router/              # Route definitions for this module
│   │           └── feature-name.router.ts
│   ├── utils/                 # Shared utilities and helpers
│   │   ├── AppError.ts        # Custom error class for structured error throwing
│   │   ├── jwt.ts             # JWT sign/verify service class
│   │   ├── mailer.ts          # Email sending logic
│   │   ├── redis.ts           # Redis helper functions (e.g., cache, permission)
│   │   └── tokenizer.ts       # Token generation utilities
│   ├── app.ts                 # Express app initialization, middleware & route registration
│   └── server.ts              # Entry point — starts the HTTP server on a port
├── .env                       # Local environment variables (never commit this)
├── .env.example               # Template for required environment variables
├── tsconfig.json              # TypeScript configuration
└── package.json
```

### `config/` vs `lib/` — What's the difference?

| Folder | Purpose | Example |
|---|---|---|
| `config/` | Reads, validates, and exports environment values | `env.ts` parsing `.env` with Zod |
| `lib/` | Initializes third-party client instances using those values | `redis.ts` creating an `ioredis` client |

> **Rule:** `lib/` files import from `config/`, never the other way around.

---

## 🏗️ Architectural Concepts

### 1. Controllers (`controller/`)
- **Responsibility:** Extract data from `req` (params, body, query), call the service, and return the HTTP response.
- **Rule:** Controllers must be **thin**. No business logic, no database calls. All logic lives in the service.

### 2. Services (`service/`)
- **Responsibility:** All business logic and database interactions live here.
- **Rule:** Services must **not** know about `req` or `res`. They receive plain objects and return plain objects. This makes them independently testable.

### 3. Routers (`router/`)
- **Responsibility:** Map HTTP methods and paths to controller functions. Apply module-specific middlewares (e.g., `authenticate()`, `rateLimiter`).

### 4. Schema Files (`feature-name.ts` at module root)
- **Responsibility:** Declare all **Zod validation schemas** and **TypeScript types** for a given module.
- These are imported by both the service (for input validation) and anywhere a typed response is needed.

---

## ✍️ Coding Conventions & Naming Rules

### File Naming
Use **kebab-case** with descriptive suffixes.

| Type | Convention | Example |
|---|---|---|
| Controller | `name.controller.ts` | `auth.controller.ts` |
| Service | `name.service.ts` | `group.service.ts` |
| Router | `name.router.ts` | `user.router.ts` |
| Schema file | `name.ts` (at module root) | `auth.ts` |

### Variable & Function Naming
- Use **camelCase** for variables and function names.
- Functions should use descriptive action verbs: `getUserById`, `createGroup`, `sendMessage`.

### Classes & Types/Interfaces
- Use **PascalCase** for class names, interfaces, and types.

### Zod Schema Naming Convention
This project uses a strict naming convention for all Zod schemas and their inferred TypeScript types.

| Item | Suffix | Example |
|---|---|---|
| Zod Schema | `Schema` | `LoginParamsSchema` |
| Inferred TypeScript type | `Payload` | `LoginParamsPayload` |

```typescript
// Always use this pattern
export const LoginParamsSchema = z.object({ ... })
export type LoginParamsPayload = z.infer<typeof LoginParamsSchema>
```

### Zod Chaining Order (1 → 5)
Always follow this order when chaining Zod methods:

1. **Initialization** — `z.string()`, `z.coerce.number()`
2. **Transformation** — `.trim()`, `.toLowerCase()`
3. **Validation** — `.min()`, `.max()`, `.email()`, `.url()`
4. **Custom Validation** — `.refine()`, `.superRefine()`
5. **Output Modifier** — `.optional()`, `.default()`, `.nullable()`

```typescript
// Example — correct chaining order
export const CreateUserParamsSchema = z.object({
    name: z.string("Name must be a string").trim().min(1, "Name is required").max(50, "Name must be at most 50 characters long"),
    email: z.string("Email must be a string").trim().toLowerCase().email("Invalid email address").min(1, "Email is required"),
    age: z.coerce.number().int("Age must be an integer").min(18, "Must be at least 18").optional()
})
```

> **Important:** `z.string("message")` fires when the value is **not a string** (wrong type). Use `.min(1, "message")` to handle **empty strings**, since `""` is still technically a valid string.

---

## 📡 HTTP Response Format

All API responses must follow a consistent JSON structure.

### Success Response
```json
{
  "success": true,
  "data": { }
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

This project uses a custom `AppError` class for all predictable errors, combined with a global error handler middleware.

### Throwing Errors in Services
```typescript
import { AppError } from '../../../utils/AppError';

// In service
if (!group) throw new AppError(404, "Group not found");
if (!authorized) throw new AppError(403, "You are not authorized to perform this action");
```

### Controller Catch Block — Always use `next(error)`
All `catch` blocks in controllers and middlewares **must** forward the error to `next()`. Never handle errors inline inside a catch block.

```typescript
// ❌ Wrong — bypasses the global errorHandler, duplicates code everywhere
} catch (error: any) {
    res.status(error.statusCode || 500).json({ success: false, message: error.message })
}

// ✅ Correct — delegates to the global errorHandler in app.ts
} catch (error) {
    next(error)
}
```

### `errorHandler` Must Be Registered Last in `app.ts`
Express only recognizes a middleware as an error handler if it has **exactly 4 parameters** `(err, req, res, next)`. It **must** be registered after all routes and other middlewares.

```typescript
// app.ts
app.use('/auth', authRouter)    // routes first
app.use('/user', userRouter)

app.use(errorHandler)           // ✅ error handler LAST
```

### Rules
- Always throw `AppError` for predictable errors (not found, unauthorized, forbidden).
- **Never** leave an empty `catch` block.
- **Never** use `catch (error: any)` — use `catch (error)` (unknown type is safer).
- Always use `next(error)` inside catch blocks. Never call `res.status()` inside a catch block directly.

---

## 🚀 How to Add a New Feature Module

1. Create a new folder: `src/modules/your-feature/`
2. Create the schema file: `your-feature.ts` — define all Zod schemas and TypeScript types.
3. Create `service/your-feature.service.ts` — write all business logic as a class exported as a singleton.
4. Create `controller/your-feature.controller.ts` — thin handler that calls the service.
5. Create `router/your-feature.router.ts` — register HTTP routes and attach middlewares.
6. Register the router in `src/app.ts`.

---

## 🗄️ Prisma Conventions

- All database models are defined in a **single file**: `prisma/schema.prisma`.
- Never instantiate `new PrismaClient()` multiple times. Use a shared singleton instance.
- Use `prisma.$transaction([])` for operations that must be **atomic** (all succeed or all fail).
- Always use `select` in Prisma queries to explicitly choose the fields you need. Avoid fetching unnecessary columns.

---

*Thank you for contributing! Following these guidelines ensures our codebase remains robust, scalable, and a joy to work in.*

<br/>

---
&copy; 2026 [decoprasetya.my.id](https://decoprasetya.my.id). All rights reserved.
