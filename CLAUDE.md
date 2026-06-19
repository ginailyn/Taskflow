# Taskflow — Project Conventions

## Project Overview

REST API for task management built with TypeScript, NestJS, and PostgreSQL, following Clean Architecture principles.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20+ |
| Framework | NestJS 10+ |
| Language | TypeScript 5+ (strict mode) |
| Database | PostgreSQL 16 |
| ORM | TypeORM |
| Validation | class-validator + class-transformer |
| Auth | JWT (Passport.js) |
| Testing | Jest + Supertest |
| Linting | ESLint + Prettier |
| Containerization | Docker + Docker Compose |

---

## Commands

```bash
# Development
npm run start:dev          # start with hot reload
npm run build              # compile to dist/

# Database
npm run migration:generate # generate migration from entity changes
npm run migration:run      # apply pending migrations
npm run migration:revert   # revert last migration

# Testing
npm run test               # unit tests
npm run test:e2e           # end-to-end tests
npm run test:cov           # coverage report

# Linting
npm run lint               # ESLint check
npm run format             # Prettier format
```

---

## Architecture — Clean Architecture

The project is organized into four concentric layers. Dependencies point inward only: outer layers depend on inner layers, never the reverse.

```
src/
├── domain/          # Innermost — pure business logic, no framework dependencies
├── application/     # Use cases, orchestrates domain objects
├── infrastructure/  # Framework, DB, external services (NestJS, TypeORM)
└── presentation/    # HTTP layer — controllers, DTOs, guards
```

### Layer Rules

- `domain/` must not import from any other layer or from NestJS/TypeORM.
- `application/` may only import from `domain/`.
- `infrastructure/` implements interfaces defined in `domain/` and `application/`.
- `presentation/` delegates to `application/` use cases; it never calls repositories directly.

---

## Directory Structure

```
src/
├── domain/
│   ├── entities/          # Business entities (plain classes, no decorators)
│   ├── value-objects/     # Immutable value types (e.g., TaskStatus, Priority)
│   ├── repositories/      # Repository interfaces (abstract classes)
│   └── exceptions/        # Domain-specific exceptions
│
├── application/
│   ├── use-cases/         # One class per use case
│   ├── dtos/              # Internal data transfer objects (not HTTP)
│   └── ports/             # Interface definitions for external services
│
├── infrastructure/
│   ├── database/
│   │   ├── entities/      # TypeORM entities (@Entity decorators live here)
│   │   ├── repositories/  # TypeORM implementations of domain repository interfaces
│   │   └── migrations/    # TypeORM migration files
│   ├── config/            # Environment configuration (ConfigService)
│   └── services/          # External service adapters (email, storage, etc.)
│
├── presentation/
│   ├── controllers/       # NestJS @Controller classes
│   ├── dtos/              # HTTP request/response DTOs (class-validator decorators)
│   ├── guards/            # Auth guards (JWT, roles)
│   ├── interceptors/      # Response transformation, logging
│   ├── filters/           # Exception filters
│   └── pipes/             # Validation pipes
│
├── shared/
│   ├── constants/         # App-wide constants
│   └── types/             # Shared TypeScript types and interfaces
│
└── modules/               # NestJS feature modules (one per domain aggregate)
    ├── tasks/
    ├── users/
    └── auth/
```

Each feature module (`tasks/`, `users/`, etc.) mirrors the four-layer structure internally when it grows large enough to warrant it.

---

## Naming Conventions

### Files

| Artifact | Pattern | Example |
|---|---|---|
| Entity (domain) | `{name}.entity.ts` | `task.entity.ts` |
| Entity (TypeORM) | `{name}.orm-entity.ts` | `task.orm-entity.ts` |
| Repository interface | `{name}.repository.ts` | `task.repository.ts` |
| Repository impl | `{name}.repository.impl.ts` | `task.repository.impl.ts` |
| Use case | `{verb}-{noun}.use-case.ts` | `create-task.use-case.ts` |
| Controller | `{name}.controller.ts` | `tasks.controller.ts` |
| DTO (request) | `{verb}-{noun}.dto.ts` | `create-task.dto.ts` |
| Module | `{name}.module.ts` | `tasks.module.ts` |
| Migration | TypeORM auto-generated timestamp prefix | `1718700000000-CreateTasksTable.ts` |
| Test (unit) | `{name}.spec.ts` | `create-task.use-case.spec.ts` |
| Test (e2e) | `{name}.e2e-spec.ts` | `tasks.e2e-spec.ts` |

### Classes and Interfaces

```typescript
// Domain entity — plain class, PascalCase
class Task { ... }

// Repository interface — prefix with I
interface ITaskRepository { ... }

// Use case — verb + noun + suffix
class CreateTaskUseCase { ... }

// DTO — verb + noun + suffix
class CreateTaskDto { ... }

// TypeORM entity — suffix with OrmEntity to distinguish from domain entity
class TaskOrmEntity { ... }

// Value object — descriptive noun
class TaskStatus { ... }
```

### Variables and Functions

- `camelCase` for variables, parameters, and methods.
- `SCREAMING_SNAKE_CASE` for module-level constants.
- `PascalCase` for classes, interfaces, enums, and type aliases.
- Enum values: `SCREAMING_SNAKE_CASE`.

```typescript
const MAX_TASK_TITLE_LENGTH = 255;

enum TaskStatus {
  PENDING = 'PENDING',
  IN_PROGRESS = 'IN_PROGRESS',
  DONE = 'DONE',
}
```

---

## TypeScript Conventions

- `strict: true` is always on. No `any` without an explicit comment explaining why.
- Prefer `interface` for object shapes that may be implemented or extended; use `type` for unions, intersections, and aliases.
- Always annotate return types on public methods.
- Use `readonly` on properties that must not change after construction.
- Use optional chaining (`?.`) and nullish coalescing (`??`) instead of manual null checks.
- Avoid enums for simple string unions; use `as const` objects when the enum won't be used as a type in class decorators.

---

## Domain Layer Conventions

- Domain entities are plain TypeScript classes with no framework decorators.
- Entities own their own invariants; throw a domain exception if an invariant is violated.
- Value objects are immutable: all properties `readonly`, no setters, equality by value.
- Repository interfaces use abstract classes (not TypeScript interfaces) so NestJS DI can inject them by token.

```typescript
// src/domain/repositories/task.repository.ts
export abstract class TaskRepository {
  abstract findById(id: string): Promise<Task | null>;
  abstract save(task: Task): Promise<Task>;
  abstract delete(id: string): Promise<void>;
}
```

---

## Application Layer Conventions

- One use case per file. Each use case has a single public `execute()` method.
- Use cases receive and return plain objects or domain entities — never HTTP request/response objects.
- Use cases are decorated with `@Injectable()` so NestJS can inject them, but they must not import from `@nestjs/common` beyond that.

```typescript
@Injectable()
export class CreateTaskUseCase {
  constructor(private readonly taskRepository: TaskRepository) {}

  async execute(input: CreateTaskInput): Promise<Task> {
    // ...
  }
}
```

---

## Infrastructure Layer Conventions

- TypeORM entities live in `infrastructure/database/entities/` and are named `*OrmEntity`.
- Mappers convert between domain entities and ORM entities. One mapper per aggregate.
- Repository implementations extend the abstract repository from `domain/`.
- Never expose `EntityManager` or raw queries outside the repository implementation.

---

## Presentation Layer Conventions

- Controllers are thin: validate input (via pipes), call one use case, return the result.
- All public endpoints have Swagger decorators (`@ApiOperation`, `@ApiResponse`).
- Request DTOs use `class-validator` decorators. Every field is explicitly typed.
- Responses are shaped by a response DTO or a serialization interceptor — never return ORM entities directly.

```typescript
@Controller('tasks')
export class TasksController {
  constructor(private readonly createTask: CreateTaskUseCase) {}

  @Post()
  async create(@Body() dto: CreateTaskDto): Promise<TaskResponseDto> {
    const task = await this.createTask.execute(dto);
    return TaskResponseDto.fromDomain(task);
  }
}
```

---

## Database Conventions

- Table names: `snake_case`, plural (`tasks`, `users`, `task_assignments`).
- Column names: `snake_case`.
- Primary keys: UUID (`uuid_generate_v4()`), stored as `uuid` column type.
- All tables have `created_at` and `updated_at` timestamps, managed by TypeORM `@CreateDateColumn` / `@UpdateDateColumn`.
- Soft deletes: use `@DeleteDateColumn` (`deleted_at`) instead of hard deletes where the domain requires an audit trail.
- All schema changes go through TypeORM migrations — never modify the DB schema by hand or use `synchronize: true` in production.
- Migrations are checked into source control and run in CI before deployment.

---

## Environment Variables

Variables are loaded via NestJS `ConfigModule` with validation schema. Never access `process.env` directly outside `config/`.

```
# .env.example (commit this; never commit .env)
NODE_ENV=development
PORT=3000

DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USER=taskflow
DATABASE_PASSWORD=
DATABASE_NAME=taskflow_db

JWT_SECRET=
JWT_EXPIRES_IN=1d
```

---

## Error Handling

- Domain exceptions extend a base `DomainException` class and are thrown from the domain layer.
- Application use cases let domain exceptions bubble up; they do not catch and swallow errors.
- A global `HttpExceptionFilter` maps domain exceptions to appropriate HTTP status codes.
- Never return `500` for business rule violations — map them to `400`, `404`, `409`, etc.

---

## Testing Conventions

- Unit tests cover use cases and domain entities. Mock all external dependencies.
- Integration tests cover repository implementations against a real test database.
- E2E tests cover HTTP endpoints via Supertest against a running NestJS app with a test DB.
- Test files live next to the code they test (`*.spec.ts`) except for E2E tests which live in `test/`.
- Use `describe` / `it` blocks, not `test()`. One `describe` per class, one `it` per behavior.
- Each test is independent: seed its own data, clean up after itself.
- Coverage threshold: 80% lines for unit tests.

```typescript
describe('CreateTaskUseCase', () => {
  it('creates a task with PENDING status by default', async () => {
    // ...
  });

  it('throws when title is empty', async () => {
    // ...
  });
});
```

---

## Git Conventions

### Branch Names

```
feat/{short-description}       # new feature
fix/{short-description}        # bug fix
chore/{short-description}      # tooling, deps, config
refactor/{short-description}   # refactoring without behavior change
test/{short-description}       # adding or fixing tests
```

### Commit Messages (Conventional Commits)

```
<type>(<scope>): <short description in imperative mood>

feat(tasks): add task priority field
fix(auth): handle expired JWT correctly
chore(deps): upgrade TypeORM to 0.3.20
test(tasks): add unit tests for CreateTaskUseCase
```

Types: `feat`, `fix`, `chore`, `refactor`, `test`, `docs`, `perf`, `ci`.

---

## Module Boundaries

- Cross-module communication goes through NestJS module exports and use cases — never direct repository access across module boundaries.
- A module may only import public APIs (use cases, DTOs) from another module, never its internal repository or entity.

---

## What Not to Do

- Do not put business logic in controllers or DTOs.
- Do not import NestJS or TypeORM decorators into `domain/`.
- Do not use `synchronize: true` outside local development.
- Do not commit `.env` files.
- Do not use `any` without a comment.
- Do not return ORM entities from controllers.
- Do not write raw SQL outside repository implementations.
