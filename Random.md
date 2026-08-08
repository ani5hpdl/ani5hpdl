# NestJS Fundamentals — Reference Project

A small, fully-working NestJS app built purely as a **reference you can re-read**
instead of re-asking an AI every time you forget how something works. Every
concept is implemented AND commented in the code itself.

## What's inside

| Concept | Where to look |
|---|---|
| Modules | `src/*.module.ts` (every module) |
| Controllers | `src/users/users.controller.ts`, `src/auth/auth.controller.ts` |
| Providers (services) | `src/users/users.service.ts`, `src/auth/auth.service.ts` |
| Provider patterns (`useFactory`) | `src/auth/auth.module.ts` (JwtModule.registerAsync) |
| DTOs + validation (`class-validator`) | `src/users/dto/`, `src/auth/dto/` |
| Global Pipe (`ValidationPipe`) | `src/main.ts` |
| Middleware (Express-style) | `src/common/middleware/logger.middleware.ts`, applied in `src/app.module.ts` |
| Guards — Authentication | `src/auth/guards/jwt-auth.guard.ts` |
| Guards — Authorization (roles) | `src/common/guards/roles.guard.ts` |
| Passport Strategy (JWT via **cookie**, not header) | `src/auth/strategies/jwt.strategy.ts` |
| Custom decorators | `src/common/decorators/roles.decorator.ts`, `current-user.decorator.ts` |
| Interceptors (response transform + logging) | `src/common/interceptors/` |
| Exception Filters (standardized errors) | `src/common/filters/http-exception.filter.ts` |
| Built-in exceptions in use | `NotFoundException`, `ConflictException`, `UnauthorizedException`, `ForbiddenException` — search these across `src/` |

## Why cookies instead of the `Authorization` header

This project signs a JWT on login/register and stores it in an **httpOnly
cookie** (`access_token`) instead of returning it in the response body —
mirroring the Express `jwt` + `cookie-parser` pattern, but Nest-style: the
cookie is read inside `JwtStrategy`'s custom extractor
(`ExtractJwt.fromExtractors`) rather than in raw middleware.

## Setup

```bash
npm install
cp .env.example .env
npm run start:dev
```

Server runs on `http://localhost:3000`.

## Try it (curl)

```bash
# Register (first user becomes admin automatically, for demo purposes)
curl -c cookies.txt -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","email":"alice@example.com","password":"password123"}'

# Login (reuses the saved cookie jar)
curl -c cookies.txt -b cookies.txt -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"password123"}'

# Protected route - who am I?
curl -b cookies.txt http://localhost:3000/auth/profile

# Admin-only route - list all users
curl -b cookies.txt http://localhost:3000/users

# Get a single user by id
curl -b cookies.txt http://localhost:3000/users/1

# Logout
curl -b cookies.txt -X POST http://localhost:3000/auth/logout
```

## The request lifecycle, end to end

For a request like `GET /users` (admin-only):

```
Request
  → Middleware        (LoggerMiddleware - logs the raw request)
  → Guards             (JwtAuthGuard verifies the cookie JWT → RolesGuard checks admin role)
  → Interceptors (pre)  (LoggingInterceptor logs "-->")
  → Pipes               (ValidationPipe - N/A here, no body/DTO on this route)
  → Route Handler       (UsersController.findAll calls UsersService)
  → Interceptors (post) (TransformInterceptor wraps the result in { success, statusCode, data })
                         (LoggingInterceptor logs "<--" with timing)
  → Exception Filter    (only runs if something threw - formats the error consistently)
Response
```

If at any point a Guard, Pipe, or the handler itself throws (e.g.
`ForbiddenException`, `NotFoundException`), execution jumps straight to the
Exception Filters — Interceptors after that point are skipped.

## Quick concept refreshers

**Middleware vs Guards vs Interceptors vs Pipes vs Filters**

| Layer | Runs | Knows the route/handler? | Typical job |
|---|---|---|---|
| Middleware | Before everything | No (raw req only) | Logging, CORS, raw request tasks |
| Guard | After middleware, before handler | Yes (`ExecutionContext`) | Auth (401) & Authz (403) |
| Pipe | Just before handler | Yes | Validate/transform input |
| Interceptor | Wraps handler (before + after) | Yes | Logging, caching, response shaping |
| Exception Filter | On thrown errors | Yes | Standardized error responses |

**401 vs 403**
- `UnauthorizedException` → "I don't know who you are" (missing/invalid token)
- `ForbiddenException` → "I know who you are, but you can't do this" (wrong role)

**Provider types** (`src/auth/auth.module.ts` has a live `useFactory` example)
- `useClass` — swap implementations
- `useValue` — constants/mocks
- `useFactory` — build dynamically, can inject other providers
- `useExisting` — alias an existing provider

## Extending this project

- Swap the in-memory array in `UsersService` for a real database
  (TypeORM/Prisma/Mongoose) — the rest of the app doesn't need to change.
- Add a `RefreshToken` flow alongside the access token cookie.
- Add more roles or a permissions system on top of `RolesGuard`.
