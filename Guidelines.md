# Microservice Contributor Cheat Sheet

## 1. Architecture Rules

- **Architecture:** Clean Architecture + Domain-Driven Design.
- **Business logic:** Always in `domain/`.
- **Synchronous inter-service communication:** REST.
- **Asynchronous processing:** BullMQ.
- **Dependency injection:** Use the configured DI container.
- **Repositories:** All concrete repositories extend `PrismaBaseRepository`.
- **Code quality:** Follow SOLID, reuse code, and prioritize readability.
- **TypeScript:** Strict typing; avoid `any`.

## 2. Standard Folder Structure

```text
src/
├── application/
│   ├── dtos/
│   │   ├── table/
│   │   ├── staff/
│   │   └── menu/
│   ├── errors/
│   ├── mappers/
│   ├── ports/
│   │   ├── services/
│   │   └── use-cases/
│   └── use-cases/
│       ├── table/
│       ├── staff/
│       └── menu/
│
├── config/                  # DI and environment configuration
├── domain/
│   ├── constants/
│   ├── entities/
│   ├── enums/
│   ├── errors/
│   ├── repositories/        # Repository interfaces
│   └── value-objects/
│
├── infrastructure/
│   ├── database/
│   │   ├── mappers/
│   │   └── repositories/
│   ├── observability/
│   ├── queue/
│   │   ├── producers/
│   │   ├── workers/
│   │   └── processors/
│   ├── redis/
│   └── services/
│
├── presentation/
│   └── http/
│       ├── controllers/
│       ├── middleware/
│       ├── routes/
│       │   └── internal/
│       └── validators/
│
└── shared/
    ├── constants/
    ├── errors/
    └── responses/
```

**Folder rule:** Group files by feature within each layer. If a folder becomes too large, split it into meaningful modules.

## 3. Use Case Standards

**Every use case must have its own implementation file and interface file.**

Example:

```text
application/
├── dtos/
│   └── table/
│       └── view-table.dto.ts
├── ports/
│   └── use-cases/
│       └── table/
│           └── view-table.use-case.port.ts
└── use-cases/
    └── table/
        └── view-table.use-case.ts
```

### Naming conventions

| Element | Name |
|---|---|
| Use case class | `ViewTableUseCase` |
| Use case interface | `IViewTableUseCase` |
| Interface filename | `view-table.use-case.port.ts` |
| Implementation filename | `view-table.use-case.ts` |
| DTO | `ViewTableDTO` |
| DTO filename | `view-table.dto.ts` |

- Keep each use case focused on one application responsibility.
- Define use case interfaces under `application/ports/use-cases/`.
- Use descriptive interface names prefixed with `I`.
- Do not call one use case from another use case.
- Register dependencies through the established DI configuration.

## 4. DTO and Mapper Rules

- Use DTOs for data crossing application boundaries, including use case inputs and outputs where appropriate.
- DTO names must use PascalCase and end with `DTO`.
- DTO filenames must use kebab-case and end with `.dto.ts`.
- Group DTOs by feature.
- **Use mappers when response transformation is substantial or complex.** Keep simple transformations inline when extracting a mapper adds no meaningful value.
- Keep response DTOs separate from database models and domain entities.
- Never return raw Prisma models directly from controllers.

Example:

```text
application/
├── dtos/
│   └── table/
│       └── view-table.dto.ts
└── mappers/
    └── table/
        └── view-table.mapper.ts
```

## 5. Repository Standards

**Repository interfaces belong in `domain/repositories/`.**

```text
domain/
└── repositories/
    └── i-table.repository.ts

infrastructure/
└── database/
    ├── mappers/
    │   └── table.mapper.ts
    └── repositories/
        └── prisma-table.repository.ts
```

| Element | Convention |
|---|---|
| Repository interface | `ITableRepository` |
| Interface filename | `i-table.repository.ts` |
| Implementation class | `PrismaTableRepository` |
| Implementation filename | `prisma-table.repository.ts` |

- All concrete repositories must extend `PrismaBaseRepository`.
- Repository interfaces must not depend on Prisma types.
- Map database models into domain entities.
- Keep persistence logic inside repository implementations.
- Keep business rules out of repositories and the base repository.
- Centralize common persistence behavior and error translation in the base repository.

## 6. Internal REST API Standards

Keep internal endpoints separate from public endpoints.

```text
presentation/http/
├── controllers/
│   └── internal/
│       └── view-table.internal.controller.ts
├── routes/
│   └── internal/
│       └── table.internal.routes.ts
└── validators/
    └── table/
```

**Request flow:**

```text
Internal Route
    ↓
Authentication / Authorization
    ↓
Request Validation
    ↓
Internal Controller
    ↓
Dedicated Use Case
    ↓
Domain + Repository
    ↓
DTO / Mapper
    ↓
HTTP Response
```

- Use REST for synchronous service-to-service calls.
- Use a dedicated use case for each distinct internal operation.
- Validate internal requests and secure internal endpoints.
- Keep controllers thin.
- Return explicit DTOs and standardized response structures.
- Handle downstream failures and timeouts appropriately.

## 7. Domain and Layer Responsibilities

| Layer | Responsibility |
|---|---|
| `domain/` | Business rules, entities, invariants, value objects, repository interfaces |
| `application/` | Use cases, DTOs, ports, workflow orchestration, mappers |
| `infrastructure/` | Prisma, repository implementations, Redis, BullMQ, external adapters |
| `presentation/` | Routes, controllers, middleware, request validation |
| `config/` | DI bindings, environment, composition root |
| `shared/` | Genuinely cross-cutting utilities and response structures |

**Dependency rule:** Domain must not depend on infrastructure, presentation, or external frameworks.

## 8. Code Quality Rules

- Follow SOLID principles.
- Move all business rules into domain entities or value objects.
- Avoid deeply nested `if/else` statements; prefer guard clauses and early returns.
- Avoid magic numbers; define meaningful constants close to their responsibility.
- Use enums for domain statuses and classifications.
- Use descriptive variable and method names.
- Avoid generic names such as `data`, `temp`, `obj`, and `result1`.
- Keep interfaces focused and classes small.
- Reuse code when the shared responsibility is clear.
- Avoid premature abstractions and duplicated business logic.
- Use strict TypeScript and avoid `any`.
- Use Zod with `.strict()` where unknown request fields must be rejected.
- Distinguish domain, application, and infrastructure errors.
- Never expose raw database errors or stack traces to clients.

## 9. BullMQ Standards

Keep queue implementations under `infrastructure/queue/`.

```text
infrastructure/queue/
├── producers/
├── workers/
└── processors/
```

- Use typed job payloads and explicit queue/job names.
- Configure retries and backoff intentionally.
- Make processing idempotent when retries can repeat side effects.
- Keep business rules in the domain.
- Use application ports to avoid coupling application code directly to BullMQ.
- Keep workers and processors focused on job execution and delegation.

## 10. PR Checklist

- [ ] Correct folder structure and feature grouping.
- [ ] One implementation file and one `.use-case.port.ts` interface file per use case.
- [ ] Use case interfaces follow the `IViewTableUseCase` naming pattern.
- [ ] DTOs follow the `ViewTableDTO` naming pattern and `.dto.ts` filename convention.
- [ ] Mappers are used for substantial response transformations.
- [ ] Repository interfaces are under `domain/repositories/`.
- [ ] Concrete repositories extend `PrismaBaseRepository`.
- [ ] Business rules reside in the domain.
- [ ] Internal REST endpoints are separated and secured.
- [ ] No magic numbers, unclear names, or deeply nested conditionals.
- [ ] DI bindings and error handling are correct.
- [ ] Relevant tests are added or updated.

**Team rule:** Follow these conventions consistently. Discuss architectural exceptions with the team before introducing a new pattern.
