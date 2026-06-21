# Architecture

> Components, layering, package structure, and cross-cutting decisions.
> Update only when structure changes — new module, new boundary, new dependency, or a notable decision.

## Context

<!-- One paragraph: what problem does this service solve? Who calls it? What does it call? -->

## Component Map

```
src/main/java/com/example/service/
├── config/          # Spring configuration, beans, security config
├── controller/      # REST controllers (@RestController)
├── service/         # Business logic (@Service)
├── repository/      # Spring Data JPA repositories
├── domain/          # JPA entities (@Entity)
├── dto/             # Request/response DTOs (records preferred)
├── mapper/          # Entity ↔ DTO mappers
├── exception/       # Custom exceptions + @ControllerAdvice
└── util/            # Stateless helpers
```

## Layering Rules

- Controllers call Services only (never Repositories directly)
- Services call Repositories and other Services
- Entities never reference DTOs
- DTOs never reference Entities

## Cross-Cutting Concerns

| Concern | Approach |
|---|---|
| Exception handling | `@ControllerAdvice` + RFC 9457 Problem Details |
| Validation | Bean Validation (`@Valid`) on DTOs |
| Transactions | `@Transactional` on service methods |
| Security | Spring Security filter chain — see `dataflow.md` |
| Logging | SLF4J + Logback (JSON in prod) |
| Migrations | Flyway `V__` scripts in `resources/db/migration/` |

## Deployment View

<!-- Runtime environment, container, cloud provider, key env vars -->

## Key Decisions

| Date | Decision | Rationale |
|---|---|---|
| | | |
