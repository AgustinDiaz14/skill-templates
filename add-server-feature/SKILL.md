---
name: add-server-feature
description: Scaffold a complete server-side feature following Clean Architecture - domain model, repository, use cases, controller, routes, migration, and tests.
---

Scaffold a new server-side feature as a full vertical slice following the project's Clean Architecture.

## Step 1: Gather requirements

Ask the user for:

1. **Entity name** (PascalCase, e.g., "Notification")
2. **Fields** with types (e.g., `title: string`, `userId: number`, `isRead: boolean`)
3. **Relationships** to existing entities:
   - 1:N — uses `@ForeignKey` + `@BelongsTo` decorators on the child model (e.g., User has many Notifications)
   - N:M — requires a junction table in `repository/models/associations/` (e.g., UserRolesModel)
4. **Permission area** — reuse existing `PermissionArea` or create new one
5. **Feature flag** — whether to gate behind a feature flag
6. **Soft delete** — whether to use the `isActive` pattern
7. **Timestamps** — whether the model needs `createdAt`/`updatedAt`

Batch related questions. Propose defaults based on common patterns.

## Step 2: Generate domain model

Create `server/src/domain/models/{Entity}.ts`:

```typescript
/**
 * {Entity} domain model.
 * {Brief description}.
 */
export interface {Entity} {
    id: number;
    // ... fields from user input
}
```

Rules:
- Pure TypeScript interface, NO Sequelize imports
- JSDoc with entity description
- Use `string | null` for nullable strings, not optional fields
- Pattern: see `server/src/domain/models/Station.ts`

## Step 3: Generate repository interface

Create `server/src/repository/interfaces/I{Entity}Repository.ts`:

```typescript
import { {Entity} } from "../../domain/models/{Entity}";

/**
 * Repository interface for {Entity} operations.
 */
export interface I{Entity}Repository {
    getAll(): Promise<{Entity}[]>;
    getById(id: number): Promise<{Entity} | null>;
    create(data: { /* create fields */ }): Promise<{Entity}>;
    update(id: number, data: Partial<{ /* update fields */ }>): Promise<{Entity}>;
    // softDelete if applicable
}
```

Rules:
- Returns domain model types, not Sequelize models
- JSDoc on every method
- Pattern: see `server/src/repository/interfaces/IStationRepository.ts`

## Step 4: Generate Sequelize model

Create `server/src/repository/models/{EntityModel}.ts`:

```typescript
import { Model, Table, Column, DataType, ForeignKey, BelongsTo } from "sequelize-typescript";
import { CreationOptional, InferAttributes, InferCreationAttributes, NonAttribute } from "sequelize";

@Table({
    timestamps: false, // or true if needed
})
export class {Entity}Model extends Model<
    InferAttributes<{Entity}Model>,
    InferCreationAttributes<{Entity}Model>
> {
    @Column({
        type: DataType.BIGINT,
        primaryKey: true,
        autoIncrement: true,
        allowNull: false,
    })
    declare id: CreationOptional<number>;

    // ... columns with decorators
}
```

Rules:
- Use `DataType.BIGINT` for IDs and foreign keys
- Use `DataType.STRING(N)` with explicit length for strings
- Use `CreationOptional` for auto-generated fields (id, defaults)
- For 1:N relationships: `@ForeignKey(() => ParentModel)` + `@BelongsTo(() => ParentModel)` on child
- For N:M relationships: create junction model in `repository/models/associations/`
- Pattern: see `StationModel.ts` (simple), `UserModel.ts` (with associations), `UserRolesModel.ts` (junction)

## Step 5: Register model in SequelizeProvider

Add import and entry to `server/src/providers/SequelizeProvider.ts`:

1. Add import at top with other model imports
2. Add to `static readonly models: ModelCtor[]` array

**CRITICAL**: If this step is missed, the model won't be registered and queries will fail at runtime.

If a junction model was created (N:M), register that too.

## Step 6: Generate repository implementation

Create `server/src/repository/implementations/{Entity}RepositoryImpl.ts`:

```typescript
import RequestContext from "../../providers/RequestContext";
import { {Entity}Model } from "../models/{Entity}Model";
import { I{Entity}Repository } from "../interfaces/I{Entity}Repository";
import { {Entity} } from "../../domain/models/{Entity}";

/**
 * Sequelize implementation of the {Entity} repository.
 */
export class {Entity}RepositoryImpl implements I{Entity}Repository {
    async getAll(): Promise<{Entity}[]> {
        const results = await RequestContext.getModel({Entity}Model).findAll();
        return results.map(r => r.get({ plain: true }) as {Entity});
    }
    // ... other methods
}
```

Rules:
- ALWAYS use `RequestContext.getModel(ModelClass)` to access models
- NEVER import Sequelize, SequelizeProvider, or MultiTenantSequelizeProvider directly
- Map Sequelize model instances to domain interfaces
- Pattern: see `StationRepositoryImpl.ts`

## Step 7: Generate use cases

Create `server/src/domain/usecases/{Entity}UseCases.ts`:

```typescript
import { I{Entity}Repository } from "../../repository/interfaces/I{Entity}Repository";
import { {Entity} } from "../models/{Entity}";
import { NotFoundError } from "../../errors/DbErrors";
import { BadParamsError } from "../../errors/GenericErrors";

/**
 * Use cases for {Entity} management.
 */
export class {Entity}UseCases {
    constructor(private repository: I{Entity}Repository) {}
    // ... business logic methods
}
```

Rules:
- Constructor receives repository INTERFACE (dependency injection)
- Contains business logic, validation
- Throws `BadParamsError` for validation failures, `NotFoundError` for missing entities
- NEVER imports Sequelize models or repository implementations
- Pattern: see `StationUseCases.ts`

## Step 8: Generate controller

Create `server/src/controllers/implementations/{Entity}Controller.ts`:

```typescript
import { Request, Response } from "express";
import { {Entity}UseCases } from "../../domain/usecases/{Entity}UseCases";
import { ErrorsHandler } from "../../utils/ErrorsHandler";
import Logger from "../../providers/Logger";

/**
 * Controller for {Entity} management endpoints.
 */
export class {Entity}Controller {
    constructor(private useCases: {Entity}UseCases) {}

    async getAll(req: Request, res: Response): Promise<void> {
        try {
            const result = await this.useCases.getAll();
            res.status(200).json(result);
        } catch (error) {
            Logger.error(error);
            const errorHandled = ErrorsHandler.handleError(error);
            res.status(errorHandled.status).json(errorHandled);
        }
    }
    // ... other handlers
}
```

Rules:
- Constructor receives use cases (NOT repositories)
- Every handler: `async method(req, res): Promise<void>`
- Try/catch with `Logger.error(error)` + `ErrorsHandler.handleError(error)`
- Pattern: see `StationController.ts`

## Step 9: Generate route file

Create `server/src/routes/{entity}.ts`:

```typescript
import { Router, Request, Response } from "express";
import { {Entity}Controller } from "../controllers/implementations/{Entity}Controller";
import { {Entity}UseCases } from "../domain/usecases/{Entity}UseCases";
import { {Entity}RepositoryImpl } from "../repository/implementations/{Entity}RepositoryImpl";
import { authMiddleware } from "../middlewares/auth";
import { checkPermissions } from "../middlewares/permission";
import { PermissionArea, PermissionType } from "../domain/models/Permission";
import { validateParams, ValidatorSchema } from "../middlewares/validation";

export const build{Entity}Router = () => {
    const controller = new {Entity}Controller(
        new {Entity}UseCases(new {Entity}RepositoryImpl())
    );

    const router = Router();

    router.get(
        "/",
        authMiddleware,
        (req: Request, res: Response) => void controller.getAll(req, res)
    );

    // ... other routes with appropriate middlewares
    return router;
};
```

Rules:
- Factory function `build{Entity}Router()` wires dependencies
- Handler pattern: `(req: Request, res: Response) => void controller.method(req, res)`
- Apply middlewares: `authMiddleware`, `checkPermissions`, `validateParams`
- Add `checkFeatureFlag(features.FEATURE_NAME)` if feature-flagged
- Pattern: see `routes/station.ts`

## Step 10: Wire route into application

Mount the router in `server/src/index.ts` or the appropriate parent router file. Find where other routes like `buildStationRouter` are mounted and follow the same pattern.

## Step 11: Generate migration

Create `server/migrations/{timestamp}-create-{entity}.js`:

```javascript
"use strict";

/** @type {import('sequelize-cli').Migration} */
module.exports = {
    async up(queryInterface, Sequelize) {
        await queryInterface.sequelize.transaction(async (t) => {
            await queryInterface.createTable(
                "{Entity}Models",
                {
                    id: {
                        type: Sequelize.BIGINT,
                        allowNull: false,
                        autoIncrement: true,
                        primaryKey: true,
                    },
                    // ... columns
                },
                { transaction: t }
            );
        });
    },

    async down(queryInterface) {
        await queryInterface.sequelize.transaction(async (t) => {
            await queryInterface.dropTable("{Entity}Models", { transaction: t });
        });
    },
};
```

Generate the timestamp:
```bash
date +%Y%m%d%H%M%S
```

Rules:
- CommonJS format (`module.exports`)
- Always wrap in transaction
- Include both `up` and `down`
- Table name = model class name (e.g., `{Entity}Models`)
- Use `Sequelize.BIGINT` for IDs, `Sequelize.STRING(N)` for strings
- For N:M junction tables, generate a separate migration
- Pattern: see `server/migrations/20260131000000-create-stations.js`

## Step 12: Generate integration test scaffold

Create `server/test/{entity}Api.test.ts`:

```typescript
import request from "supertest";
import express from "express";
import { build{Entity}Router } from "../src/routes/{entity}";

// Mock dependencies
jest.mock("../src/providers/MultiTenantSequelizeProvider", () => ({
    __esModule: true,
    default: { getSequelize: jest.fn().mockResolvedValue({}) },
}));
jest.mock("../src/providers/RequestContext");
jest.mock("../src/providers/CacheProvider", () => ({
    __esModule: true, default: {
        getInstance: jest.fn().mockReturnValue({
            tenantMetadataCache: {}, featureFlagsCache: {},
        }),
    },
}));
jest.mock("../src/providers/MetaDatabaseProvider", () => ({
    __esModule: true, default: { getDb: jest.fn().mockReturnValue({}) },
}));
jest.mock("../src/providers/Locals", () => {
    const originalModule = jest.requireActual("../src/providers/Locals");
    return {
        __esModule: true, ...originalModule,
        default: { getConfig: jest.fn().mockReturnValue({ environment: "test" }) },
    };
});
jest.mock("../src/domain/usecases/AuthUseCases");
jest.mock("../src/middlewares/auth", () => ({
    authMiddleware: (req, res, next) => {
        req.user = { id: 1, firefighterId: "123", roleId: 1 };
        next();
    },
}));
jest.mock("../src/middlewares/permission", () => ({
    checkPermissions: () => (req, res, next) => next(),
}));

describe("{Entity} API", () => {
    let app;

    beforeEach(() => {
        jest.clearAllMocks();
        app = express();
        app.use(express.json());
        app.use("/api/{entity}", build{Entity}Router());
    });

    // ... test cases for each endpoint
});
```

Include tests for happy path and error cases (400, 404).

## Step 13: Verify

Run from `server/`:

```bash
npx tsc --noEmit
npx eslint <all generated files>
```

Fix any issues before finishing.

## Rules

- Follow Clean Architecture boundaries strictly: Controller -> UseCase -> Repository Interface <- Repository Implementation
- Never skip the SequelizeProvider registration step
- Always generate both `up` and `down` in migrations
- All exported functions must have JSDoc (except test blocks)
