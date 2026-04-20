---
name: write-tests
description: Generate integration or unit tests for existing code, following project patterns with proper mocking and test helpers.
---

Generate tests for existing server or client code following the project's established testing patterns.

## Step 1: Identify target

Ask the user for:
1. **File path** to test (or auto-detect from changed files on branch via `git diff --name-only`)
2. **Test type** preference:
   - **Integration** (default for server) — Supertest + express + full route
   - **Unit** (for use cases) — mocked repository interfaces
   - **Component** (for client) — RTL render + interaction

## Step 2: Read source and existing tests

1. Read the source file completely
2. Find existing tests in the same domain to match conventions:
   - Server integration: `server/test/*.test.ts`
   - Server unit: `server/test/useCases/*.test.ts`
   - Client: `client/__tests__/` or `client/tests/`
3. Read at least one reference test file in the same category

## Step 3: Generate server integration tests

For route/controller files, generate Supertest tests:

```typescript
import request from "supertest";
import express from "express";
// Import the route builder

// Full provider mock chain
jest.mock("../src/providers/MultiTenantSequelizeProvider", () => ({
    __esModule: true,
    default: { getSequelize: jest.fn().mockResolvedValue({}) },
}));
jest.mock("../src/providers/RequestContext");
jest.mock("../src/providers/CacheProvider", () => ({
    __esModule: true,
    default: {
        getInstance: jest.fn().mockReturnValue({
            tenantMetadataCache: {},
            featureFlagsCache: {},
        }),
    },
}));
jest.mock("../src/providers/MetaDatabaseProvider", () => ({
    __esModule: true,
    default: { getDb: jest.fn().mockReturnValue({}) },
}));
jest.mock("../src/providers/Locals", () => {
    const originalModule = jest.requireActual("../src/providers/Locals");
    return {
        __esModule: true,
        ...originalModule,
        default: {
            getConfig: jest.fn().mockReturnValue({ environment: "test" }),
        },
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
```

**OR** for tests that need real DB operations, use `TestHelper`:

```typescript
import { TestHelper } from "./utils/TestHelper";

beforeAll(async () => {
    TestHelper.setupDatabases();
    await TestHelper.patchRequestContext();
    await TestHelper.seedTestData();
});

afterAll(async () => {
    await TestHelper.cleanup();
});

beforeEach(async () => {
    await TestHelper.truncateAllTables();
});
```

## Step 4: Generate server unit tests

For use case files, mock the repository interface:

```typescript
import { {Entity}UseCases } from "../../src/domain/usecases/{Entity}UseCases";
import { I{Entity}Repository } from "../../src/repository/interfaces/I{Entity}Repository";

const mockRepository: jest.Mocked<I{Entity}Repository> = {
    getAll: jest.fn(),
    getById: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
    // ... all interface methods
};

describe("{Entity}UseCases", () => {
    let useCases: {Entity}UseCases;

    beforeEach(() => {
        jest.clearAllMocks();
        useCases = new {Entity}UseCases(mockRepository);
    });

    // Test business logic, validation, error cases
});
```

Pattern: see `server/test/useCases/StationUseCases.test.ts`

## Step 5: Generate client tests

For client components, use RTL:

```tsx
import { render, screen, fireEvent, waitFor } from "@testing-library/react";

// Mock server actions
jest.mock("@/utils/apiRequests/{Entity}Requests", () => ({
    get{Entities}: jest.fn(),
    // ...
}));

// Mock hooks as needed
jest.mock("@/hooks/useSessionUser", () => ({
    useSessionUser: () => ({
        id: 1, name: "Test", surname: "User", isAdmin: true,
    }),
}));
```

## Step 6: Test coverage

For each source file method/handler, generate:
- **Happy path** test
- **Error cases**: validation errors (400), not found (404), permission denied (403)
- **Edge cases**: empty lists, null values, boundary values

## Step 7: Run and verify

```bash
# Run from root
scripts/nvm-run.sh npm test path/to/file.test.ts
# or directly in subproject
scripts/nvm-run.sh npm test path/to/file.test.tsx
```

If tests fail, analyze the failure, fix, and re-run until green.

Then lint:
```bash
npx eslint path/to/file.test.ts
```

## Rules

- Do NOT add JSDoc to `it`/`test` blocks — the test description IS the documentation.
- Match the mocking patterns of existing tests in the same directory.
- Server integration tests are preferred over unit tests (as per project guidelines).
- Always test both success and failure paths.
- Use `TestHelper` for tests that need real DB, mocks for tests that don't.
