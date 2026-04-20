---
name: add-hook
description: Scaffold a React hook following project patterns - data-fetching or UI behavior - with proper lint-safe patterns.
---

Generate a React hook following the project's established patterns and avoiding common lint pitfalls.

## Step 1: Gather requirements

Ask the user for:

1. **Hook name** (e.g., `useNotifications`, `useClickOutside`)
2. **Hook type**:
   - **Data-fetching** — fetches data via server actions, manages loading/error state
   - **UI behavior** — DOM event handling, scroll management, keyboard shortcuts

## Step 2: Determine pattern

### Data-fetching hook

**Template:**

```typescript
"use client";

import { useCallback, useEffect, useMemo, useState } from "react";
import { get{Entities} } from "@/utils/apiRequests/{Entity}Requests";
import { {Entity} } from "@/utils/apiTypes/{Entity}";

/**
 * Hook to fetch and manage {entities} data.
 *
 * @returns {{
 *   {entities}: {Entity}[] | null,
 *   loading: boolean,
 *   error: Error | null,
 *   refresh: () => void
 * }}
 */
export function use{Entities}(): {
    {entities}: {Entity}[] | null;
    loading: boolean;
    error: Error | null;
    refresh: () => void;
} {
    const [data, setData] = useState<{Entity}[] | null>(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState<Error | null>(null);

    /**
     * Fetch {entities} from the API.
     */
    const fetch{Entities} = useCallback(async () => {
        setLoading(true);
        try {
            const response = await get{Entities}();
            if (response.res) {
                setData(response.res);
                setError(null);
            } else {
                setError(new Error(response.error || "Failed to fetch {entities}"));
            }
        } catch (err) {
            setError(err instanceof Error ? err : new Error("Unknown error"));
        } finally {
            setLoading(false);
        }
    }, []);

    useEffect(() => {
        void fetch{Entities}();
    }, [fetch{Entities}]);

    return { {entities}: data, loading, error, refresh: fetch{Entities} };
}
```

### CRITICAL: Avoiding `set-state-in-effect` lint error

The `react-hooks/set-state-in-effect` rule forbids calling `setState` synchronously inside `useEffect`. This causes cascading renders.

**BAD (do NOT do this):**
```typescript
// ❌ Synchronous setState in useEffect body
useEffect(() => {
    const cached = localStorage.getItem("data");
    if (cached) {
        setData(JSON.parse(cached));      // ❌ Synchronous setState in effect
        setLoading(false);                 // ❌ Synchronous setState in effect
    }
}, []);
```

**GOOD patterns:**

1. **Lazy initial state** — for localStorage/cache reads on mount:
```typescript
// ✅ Read cache in useState initializer, not in useEffect
const [data, setData] = useState<Data | null>(() => {
    const cached = localStorage.getItem("data");
    if (cached) {
        try { return JSON.parse(cached) as Data; } catch { return null; }
    }
    return null;
});
const [loading, setLoading] = useState(() => {
    return !localStorage.getItem("data");
});
```

2. **setState inside async callback** — this is fine:
```typescript
// ✅ setState inside a promise callback (not synchronous in effect body)
useEffect(() => {
    fetchData().then(result => {
        setData(result);   // ✅ Inside async callback, not synchronous
        setLoading(false);
    });
}, []);
```

3. **Derived state with useMemo** — instead of sync effect:
```typescript
// ✅ Derive state from existing state, don't copy it
const enabledItems = useMemo(
    () => items?.filter(item => item.isEnabled) ?? null,
    [items]
);
```

**Reference for what NOT to do:** `client/hooks/useActivities.ts` lines 66-80 (has eslint-disable comment for this exact issue).

### UI behavior hook

**Template:**

```typescript
"use client";

import { useEffect } from "react";

/**
 * Hook to {description of behavior}.
 *
 * @param {paramType} paramName - {description}
 */
export function use{HookName}(param: ParamType): void {
    useEffect(() => {
        const handler = (event: Event) => {
            // ... event handling logic
        };

        document.addEventListener("eventName", handler);
        return () => {
            document.removeEventListener("eventName", handler);
        };
    }, [param]);
}
```

Rules for UI hooks:
- Subscribe in `useEffect`, cleanup on unmount (return cleanup function)
- No data fetching, no server actions
- Minimal state, focused on single behavior
- Pattern: see `useBodyScrollLock.ts`, `useEscapeClose.ts`

## Step 3: Generate file

Create `client/hooks/use{HookName}.ts` with the appropriate template filled in.

## Step 4: Verify

```bash
cd client/
npx eslint hooks/use{HookName}.ts
npx tsc --noEmit
```

Fix any issues — especially `set-state-in-effect` violations.

## Rules

- `"use client"` directive required
- JSDoc with `@returns` for data-fetching hooks
- JSDoc with `@param` for UI behavior hooks
- Use server actions (`apiRequests/`) for data fetching, NOT direct `fetch()` calls
- NEVER set state synchronously in `useEffect` body — use lazy initializers or async callbacks
- Use `useMemo` for derived state, not `useEffect` + `setState`
