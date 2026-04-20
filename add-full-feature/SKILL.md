---
name: add-full-feature
description: End-to-end feature scaffolding - combines add-server-feature and add-client-feature into a single flow.
---

Scaffold a complete end-to-end feature across both server and client.

## Step 1: Gather all requirements upfront

Ask the user for everything needed by both server and client skills in a single batch:

**Server questions:**
1. Entity name (PascalCase)
2. Fields with types
3. Relationships to existing entities (1:N or N:M)
4. Permission area
5. Feature flag (yes/no + name)
6. Soft delete (isActive pattern)
7. Timestamps

**Client questions:**
8. Page scope: dedicated page or settings panel
9. Nav group: which submenu or new top-level entry
10. PostHog event names

Propose smart defaults for each based on the entity name and fields.

## Step 2: Execute server-side scaffolding

Follow all steps from the `/add-server-feature` skill:

1. Domain model
2. Repository interface
3. Sequelize model (+ junction model if N:M)
4. Register model in SequelizeProvider
5. Repository implementation
6. Use cases
7. Controller
8. Route file
9. Wire route
10. Migration
11. Integration test scaffold

## Step 3: Execute client-side scaffolding

Follow all steps from the `/add-client-feature` skill:

1. API types
2. Zod schemas
3. Server actions
4. Feature component
5. App Router page
6. Nav entry
7. Client test scaffold

## Step 4: Verify everything

Run from both directories:

```bash
cd server/ && npx tsc --noEmit
cd server/ && npx eslint <server files>
cd client/ && npx tsc --noEmit
cd client/ && npx eslint <client files>
```

Fix all issues before finishing.

## Step 5: Summary

Present a summary of all files created, grouped by layer:
- Domain layer
- Repository layer
- Application layer (use cases, controllers, routes)
- Infrastructure (migrations, SequelizeProvider registration)
- Client (types, schemas, server actions, components, pages, nav)
- Tests (server integration, client RTL)

## Rules

- All rules from both `/add-server-feature` and `/add-client-feature` apply.
- Ask ALL questions upfront - don't go back and forth between server and client phases.
- If anything fails verification, fix it before moving to the next phase.
