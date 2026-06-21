# Data Model

> **Source of truth for the schema.** JPA `@Entity` classes must match this document exactly.
> Update this doc *before* writing any entity, migration, or repository.

## Overview

<!-- One paragraph: what does this service own? What are the main aggregates? -->

## E-R Diagram

```mermaid
erDiagram
    %% Add entities here. Entity names must match @Entity class names (PascalCase).
    %% Example:
    %% CUSTOMER ||--o{ ORDER : places
    %% CUSTOMER {
    %%     uuid id PK
    %%     string email UK "NOT NULL"
    %%     timestamptz created_at "NOT NULL"
    %% }
```

## Entity Catalog

<!-- One section per entity -->

### EntityName

| Field | DB Type | Java Type | Constraints | Notes |
|---|---|---|---|---|
| id | uuid | UUID | PK | Generated |

**Relationships:**
- (list JPA associations and their fetch type)

## Enums

| Enum | Values | Persisted as |
|---|---|---|
| | | STRING |

## Change Log

| Ticket | Date | Change |
|---|---|---|
| INIT | <!-- date --> | Initial model |
