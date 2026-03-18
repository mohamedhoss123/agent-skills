---
name: expressjs
description: Provides best practices and a reusable workflow for building Express.js APIs with modular monolith architecture, zod validation in routing, controller/service separation, and automatic Swagger/OpenAPI documentation for each endpoint.
---

# Express.js Modular API Skill

This skill helps you implement and maintain Express.js APIs in a **modular monolith** style where:

- 🔹 **Routing is responsible for request validation** using **Zod**.
- 🔹 **Controllers are classes that only contain orchestration/business logic** (no DB/raw persistence logic).
- 🔹 **Services are classes that contain database or persistence logic**.
- 🔹 **Every endpoint is documented with Swagger/OpenAPI**.

## When to Use This Skill

Use this skill when you are building or extending an Express.js backend and you want to ensure:

- Consistent **request validation** at the routing layer (via Zod).
- Strict **separation of concerns** (routing -> controller -> service).
- **Swagger/OpenAPI docs** that are generated/maintained per endpoint.
- A **modular monolith** service structure where each feature owns its routes, controllers, and services.

## Core Principles

### 1) Modular Monolith Architecture

- Each feature lives in its own folder (e.g., `modules/hall`, `modules/user`).
- Each module exports its router and any helper utilities.
- The main app imports module routers and mounts them on path prefixes.

### 2) Routing Layer: Validation + Documentation

- Routing files live in `src/modules/<feature>/routes` or `src/modules/<feature>/router`.
- Use **Zod schemas** to validate `req.body`, `req.query`, `req.params`, and `req.headers`.
- Do not put business logic or database calls in route handlers.

### 3) Controllers: Orchestration Only (OOP-Friendly)

- Controllers live in `src/modules/<feature>/controllers`.
- Prefer **classes** for controllers to enable dependency injection, easy testing, and clearer encapsulation.
- Controllers should:
  - Be instantiated with service dependencies (e.g., `new UserController(userService)`).
  - Call services to perform work.
  - Transform input/output shapes if needed.
  - Return proper HTTP responses.
- Controllers should not import database-layer code directly.

### 4) Services: DB / Persistence Logic (OOP-Friendly)

- Services live in `src/modules/<feature>/services`.
- Prefer **classes** for services so you can encapsulate related persistence methods and state.
- Each service encapsulates persistence logic (ORM/DB queries, caches, external APIs).
- Services should expose clear public methods (e.g., `createUser()`, `getById()`, `updateStatus()`).
- Controllers instantiate or receive these service instances (prefer constructor injection) and call their methods.

### 5) Swagger / OpenAPI per Endpoint

- Use `swagger-jsdoc` to generate OpenAPI docs from JSDoc comments in your route definitions.
- Serve the docs via `swagger-ui-express` (e.g., `app.use('/api/docs', swaggerUi.serve, swaggerUi.setup(specs))`).
- Ensure every route declares its OpenAPI metadata in JSDoc so documentation stays in sync with the code.

## Checklist for New Endpoints

1. ✅ Add a Zod schema for request validation in the routing folder.
2. ✅ Add an Express route that uses the schema and calls a controller.
3. ✅ Implement the controller (no DB logic) in the controller folder.
4. ✅ Implement the persistence logic in the service folder.
5. ✅ Add/verify Swagger/OpenAPI metadata for the route.

## Quick Tips

- Keep Zod schemas close to the route they validate (in `routes/schemas.ts` or similar).
- Keep controllers thin: they should mainly `await service.*` and return results.
- Avoid importing database helpers in controllers; only import the service layer.
- Prefer shared utilities for common response patterns (e.g., `sendSuccess(res, data)`).

## Standard Response Envelope

To keep API responses consistent, return JSON in a single envelope shape such as:

```json
{ "status": "SUCCESS|ERROR|UNAUTHORIZED|USER_NOT_FOUND|...", "data": { ... } }
```

- Use `SUCCESS` for 2xx responses.
- Use `ERROR` for general failures, and provide details in `data`.
- Use specific statuses like `UNAUTHORIZED`, `USER_NOT_FOUND`, etc. when they map to known error cases.
- Ensure the controller always sends this envelope (e.g., `res.json({ status: 'SUCCESS', data: result })`).

## When Not to Use This Skill

- When working on pure frontend code (React, Vite, etc.).
- When you are writing ad-hoc scripts or CLI tools without an Express app.
- When a project is strictly microservices and already has a different architecture requirement.
