---
name: add-client-feature
description: Scaffold a new client-side feature - server actions, Zod schemas, feature component, App Router page, nav entry, and tests.
---

Scaffold a new client-side feature page or section, following the project's BFF pattern with Server Actions.

## Step 1: Gather requirements

Ask the user for:

1. **Feature name** (e.g., "notifications", "schedules")
2. **API endpoints** it consumes from the server (e.g., `GET /notifications`, `POST /notifications`)
3. **Scope**: dedicated page (App Router) or settings panel
4. **Permission required**: area + type from `PermissionArea`/`PermissionType`, or public (null)
5. **Feature flag**: whether to gate behind a feature flag name
6. **Nav group**: Show the existing `rawMenuConfig` groups from `NavBar.tsx` and ask which submenu to add under, or create a new top-level entry
7. **PostHog events**: entity name for tracking (e.g., `notification_created`, `notification_deleted`)

Batch related questions. Propose defaults from the diff/context.

## Step 2: Generate API types

Create `client/utils/apiTypes/{Entity}.ts`:

```typescript
/**
 * {Entity} type matching the server domain model.
 */
export interface {Entity} {
    id: number;
    // ... fields matching server domain model
}
```

## Step 3: Generate Zod schemas

Create `client/utils/apiRequests/ZodSchemas/{Entity}Schemas.ts`:

```typescript
import { z } from "zod";

export const {Entity}Schema = z.object({
    id: z.number(),
    // ... fields
});

export const {Entity}ArraySchema = z.array({Entity}Schema);
```

## Step 4: Generate server actions

Create `client/utils/apiRequests/{Entity}Requests.ts`:

```typescript
"use server";

import { ApiResponse, get, post, put, remove } from "@/utils/apiRequests/ApiFetch";
import { {Entity} } from "@/utils/apiTypes/{Entity}";
import { {Entity}Schema, {Entity}ArraySchema } from "@/utils/apiRequests/ZodSchemas/{Entity}Schemas";
import { Success, SuccessSchema } from "@/utils/apiRequests/ZodSchemas/commonSchemas";

/**
 * Get all {entities}.
 */
export async function get{Entities}(): Promise<ApiResponse<{Entity}[]>> {
    return get("/{entities}", {Entity}ArraySchema);
}

/**
 * Create a new {entity}.
 */
export async function create{Entity}(params: Create{Entity}Params): Promise<ApiResponse<Success>> {
    return post("/{entities}", SuccessSchema, params);
}

// ... other CRUD operations
```

Rules:
- File MUST start with `"use server"`
- ALL responses validated with Zod schemas
- Use `get`, `post`, `put`, `remove` from `ApiFetch`
- JSDoc on every exported function
- Pattern: see `client/utils/apiRequests/StationRequests.ts`

## Step 5: Generate feature component

Create `client/components/features/{featureName}/` directory with main component:

`{FeatureName}Page.tsx`:

```tsx
"use client";

import { useEffect, useState } from "react";
import { get{Entities} } from "@/utils/apiRequests/{Entity}Requests";
import { {Entity} } from "@/utils/apiTypes/{Entity}";
import TableComponent from "@/components/shared/TableComponent";
import { useFormHelper } from "@/hooks/useFormHelper";
import AlertComponent from "@/components/alerts/AlertComponent";
// ... import design system components as needed

/**
 * Main page component for {Entity} management.
 */
export default function {FeatureName}Page() {
    // ... component logic using design system components
}
```

Rules:
- Use `"use client"` directive
- Use existing design system primitives: `BasicInput`, `SubmitButton`, `AddButton`, `EditButton`, `DeleteButton`, `TableComponent`, `PaginationComponent`, `AlertComponent`, `Modal`, `ActionMenu`, `DateFormatter`
- Use `useFormHelper` for CRUD operations with PostHog tracking (`trackingEventName`, `trackingProperties`)
- Use `useSessionUser()` for current user info (never call `/me`)
- Use `useUserPermissions()` for permission-gated UI
- Follow component scope rules from `components/README.md`
- Prefer standard Tailwind classes over arbitrary values

## Step 6: Generate App Router page

If a dedicated page, create `client/app/fsa/{featureName}/page.tsx`:

```tsx
import {FeatureName}Page from "@/components/features/{featureName}/{FeatureName}Page";

/**
 * App Router page for {featureName}.
 */
export default function Page() {
    return <{FeatureName}Page />;
}
```

This is a Server Component that renders the client feature component.

## Step 7: Add navigation entry

Edit `client/components/layout/navBar/NavBar.tsx`:

1. Find `rawMenuConfig` array
2. Add new entry to the appropriate submenu group (as specified by user), or as new top-level entry
3. Assign next available `id` (check existing max id)
4. Include proper `permission` and optional `featureFlag`
5. Import the appropriate Heroicon for the menu item

Example entry:
```typescript
{
    id: 16,
    slot: "{Display Name}",
    href: "/fsa/{featureName}",
    icon: <SomeIcon />,
    permission: { area: PermissionArea.{AREA}, type: PermissionType.READ },
    featureFlag: features.{FLAG_NAME}, // if applicable
},
```

## Step 8: Generate client test scaffold

Create test file in `client/__tests__/{featureName}/` or `client/tests/`:

```tsx
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import {FeatureName}Page from "@/components/features/{featureName}/{FeatureName}Page";

// Mock server actions
jest.mock("@/utils/apiRequests/{Entity}Requests", () => ({
    get{Entities}: jest.fn(),
    create{Entity}: jest.fn(),
    // ...
}));

// Mock hooks
jest.mock("@/hooks/useSessionUser", () => ({
    useSessionUser: () => ({ id: 1, name: "Test", surname: "User", isAdmin: true }),
}));

describe("{FeatureName}Page", () => {
    beforeEach(() => {
        jest.clearAllMocks();
    });

    it("renders the page", async () => {
        // ... test rendering
    });

    // ... interaction tests
});
```

## Step 9: Verify

Run from `client/`:

```bash
npx tsc --noEmit
npx eslint <all generated files>
```

Fix any issues before finishing.

## Rules

- Server action files MUST start with `"use server"`
- ALL API responses MUST be validated with Zod schemas
- NEVER create custom buttons, inputs, or tables - use the design system
- NEVER import from other `features/` directories
- Use `useSessionUser()` for current user - never call `/me` from client components
- Navigation entries must include permission and/or feature flag gating
