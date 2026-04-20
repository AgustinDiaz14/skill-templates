---
name: discover
description: Trace a feature's full vertical slice across all layers - domain model to UI, including DB, tests, and nav entry.
---

Explore and document how a feature/entity is implemented across all layers of the codebase.

## Step 1: Identify the entity

Ask the user for the entity or feature name (e.g., "Station", "Tasks", "Vehicles", "Inventory").

Check if the user passed `--save` to persist output to a file.

## Step 2: Trace all layers

For the given entity, search and read files across every layer. Use file search and grep to find relevant files.

### Server layers:

1. **Domain Model** — `server/src/domain/models/`
   - Find the interface file. Document all fields and their types.

2. **Repository Interface** — `server/src/repository/interfaces/I{Entity}Repository.ts`
   - Document all available methods and their signatures.

3. **Sequelize Model** — `server/src/repository/models/`
   - Document columns, data types, decorators.
   - List all associations (`@ForeignKey`, `@BelongsTo`, `@HasMany`, `@BelongsToMany`).
   - Check for junction tables in `repository/models/associations/`.

4. **Repository Implementation** — `server/src/repository/implementations/`
   - Document key queries (complex `findAll` with includes, custom SQL, etc.).

5. **Use Cases** — `server/src/domain/usecases/`
   - Document business rules, validations, error types thrown.

6. **Controller** — `server/src/controllers/implementations/`
   - Document endpoint handlers and their request/response shapes.

7. **Routes** — `server/src/routes/`
   - Document HTTP methods, paths, and middleware chain (auth, permissions, feature flags, validation schemas).

8. **Migration(s)** — `server/migrations/`
   - Search for migrations referencing the entity table name.
   - Document schema evolution (create table, add columns, etc.).

9. **Registration** — `server/src/providers/SequelizeProvider.ts`
   - Verify the model is registered in the `models` array.

### Client layers:

10. **API Types** — `client/utils/apiTypes/`
    - Find the TypeScript interface on the client side.

11. **Server Actions** — `client/utils/apiRequests/`
    - Document API functions, Zod schemas, endpoint mappings.

12. **Feature Components** — `client/components/features/`
    - List all components in the feature directory.
    - Document key UI patterns used (tables, forms, modals).

13. **App Router Pages** — `client/app/fsa/`
    - Find the page file(s) for this feature.

14. **Nav Entry** — `client/components/layout/navBar/NavBar.tsx`
    - Search `rawMenuConfig` for the feature's route.
    - Document permission and feature flag gating.

### Tests:

15. **Server Integration Tests** — `server/test/`
    - Find `*.test.ts` files related to this entity.

16. **Server Unit Tests** — `server/test/useCases/`
    - Find use case test files.

17. **Client Tests** — `client/__tests__/` or `client/tests/`
    - Find related test files.

## Step 3: Present results

Output a structured markdown document with:

- File paths as clickable links
- Key interfaces/types inline
- Data flow diagram (text-based):
  ```
  Client Component → Server Action → API Route → Controller → UseCase → Repository → DB
  ```
- Gaps identified (missing tests, missing Zod validation, etc.)

## Step 4: Save (optional)

If user passed `--save`, write the output to `.specs/discovery-{entity}.md`.

Otherwise, print to chat (ephemeral).

## Rules

- Read files, never modify them.
- If a layer is missing for the entity, flag it as a gap.
- Group related files together (e.g., all vehicle-related models, not just VehicleModel).
- Be thorough — check for related entities (e.g., Station also affects User via `stationId`).
