---
description: "Generate entity, DTO, and mapper set for a domain model"
argument-hint: "<entity description with fields>"
---

# Generate Entity + DTO + Mapper

## Instructions

You are generating a complete entity/DTO/mapper set from a description.

Parse `$ARGUMENTS` for the entity name and field descriptions.

Use the Agent tool with `subagent_type: "flutter-crossplatform-development:flutter-feature-developer"`:

```
Create the entity, DTO, and mapper for: $ARGUMENTS

## Context
Read the current project structure to understand existing patterns, naming conventions, and directory layout.

## Deliverables
Write the following files directly into the project:

1. **Domain Entity** — Pure Dart class with:
   - `@immutable` annotation
   - Equatable with `props` override
   - `copyWith` method
   - No JSON annotations, no Flutter imports
   - Value objects for typed primitives if appropriate (e.g., `UserId`, `Email`)

2. **DTO** — Data transfer object with:
   - `@JsonSerializable(fieldRename: FieldRename.snake)`
   - `fromJson` factory and `toJson` method
   - Field types matching the API/database schema

3. **Mapper** — Extension method on DTO for simple one-way conversion (`toEntity()`).
   If bidirectional or complex, use a dedicated mapper class with `toEntity()` and `toDto()`.

Place files according to the project's existing directory structure. If no convention exists, use:
- Entity: `lib/features/<feature>/domain/entities/`
- DTO: `lib/features/<feature>/data/dto/`
- Mapper: `lib/features/<feature>/data/mappers/`
```

After the agent completes, show the user the files created and the field mapping.
