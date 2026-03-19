---
description: "Generate Drift table, DAO, and migration for an entity"
argument-hint: "<table description with columns>"
---

# Generate Drift Table + DAO + Migration

## Instructions

You are generating Drift persistence components for an entity.

Parse `$ARGUMENTS` for the table description and column details.

Use the Agent tool with `subagent_type: "flutter-crossplatform-development:flutter-feature-developer"`:

```
Create Drift persistence for: $ARGUMENTS

## Context
Read the current project to find:
- Existing Drift database class and its current `schemaVersion`
- Existing table definitions and DAO patterns
- Existing entity that this table persists (if any)
- Directory structure for data layer persistence

## Deliverables
Write or update the following files directly in the project:

1. **Table definition** — Drift table class:
   - Column types matching the entity fields
   - `IntColumn` with `autoIncrement()` for primary key, or `TextColumn` for UUID-based IDs
   - Indexes on frequently queried columns
   - Type converters for complex types (enums, JSON columns)
   - `@DataClassName` annotation if the generated class name should differ

2. **DAO** — `@DriftAccessor(tables: [...])` class:
   - CRUD methods: `insert`, `update`, `delete`, `getById`, `getAll`
   - Reactive queries: `watchAll()`, `watchById()` returning `Stream<List<T>>`
   - Batch operations for bulk inserts/updates
   - Custom queries with `select`, `join`, `orderBy`, `limit` as needed
   - Pagination support if the table can grow large

3. **Database update** — Add the new table to `@DriftDatabase(tables: [...])` and increment `schemaVersion`.

4. **Migration** — Add migration step in `MigrationStrategy`:
   - `onCreate` for fresh installs
   - `onUpgrade` with `stepByStep` for version N → N+1
   - Create table SQL for the new version

5. **Database DTO + Mapper** — If the Drift companion/data class differs from the domain entity:
   - Mapper between Drift data class and domain entity
   - Extension method or mapper class

Place files according to the project's existing directory structure.

After writing, run `dart run build_runner build --delete-conflicting-outputs` to generate Drift code.
```

After the agent completes, show the user the files created, the new schema version, and remind them to run build_runner.
