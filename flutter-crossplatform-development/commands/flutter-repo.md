---
description: "Generate repository interface, implementation, and API client from an endpoint spec"
argument-hint: "<repository description or API endpoint>"
---

# Generate Repository + API Client

## Instructions

You are generating a repository with its API client from a description or endpoint spec.

Parse `$ARGUMENTS` for the repository description, endpoint details, or API spec.

Use the Agent tool with `subagent_type: "flutter-crossplatform-development:flutter-feature-developer"`:

```
Create a repository with API client for: $ARGUMENTS

## Context
Read the current project to find:
- Existing repository interfaces and implementations (match conventions)
- Existing Dio setup, base URL configuration, interceptors
- Existing entity and DTO patterns
- Directory structure for domain and data layers

## Deliverables
Write the following files directly into the project:

1. **Repository interface** — Abstract class in the domain layer:
   - Method signatures returning domain entities (never DTOs)
   - `Stream` return types for real-time data
   - `@useResult` annotation on methods with return values

2. **Repository implementation** — In the data layer:
   - `@immutable` annotation
   - Constructor-injected API client and optional local data source
   - Every method: call API client → catch `DioException` → map DTO to entity → return entity
   - Domain exception mapping at the boundary (`NetworkException`, `ServerException`, `NotFoundException`, `UnauthorizedException`)

3. **API client** — Retrofit abstract class or Dio wrapper:
   - `@RestApi()` with proper annotations (`@GET`, `@POST`, `@PUT`, `@DELETE`)
   - Request/response types using DTOs
   - Path parameters, query parameters, request body as needed

4. **DTOs** — For request and response bodies:
   - `@JsonSerializable(fieldRename: FieldRename.snake)`
   - `fromJson` / `toJson`

5. **Mappers** — DTO-to-entity conversion:
   - Extension method for simple cases
   - Mapper class for complex/bidirectional cases

Place files according to the project's existing directory structure.
```

After the agent completes, show the user the files created, endpoint mapping, and domain exception types.
