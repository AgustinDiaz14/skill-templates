---
name: add-posthog
description: Add PostHog analytics tracking to a feature component following the project's naming conventions and patterns.
---

Add PostHog event tracking to a feature component.

## Step 1: Identify target

Ask the user for:
1. **Component file path** to instrument
2. **Entity name** for event naming (e.g., "station", "activity", "vehicle")

## Step 2: Read the component

Read the component file and identify:
- CRUD operations (create, update, delete)
- Key user interactions (toggle, filter, search, export)
- Whether it uses `useFormHelper`
- Whether it already has PostHog tracking

## Step 3: Determine tracking strategy

### Components using `useFormHelper`

If the component uses `useFormHelper`, add tracking via `handleFormResult`:

```typescript
handleFormResult({
    isSuccess: true,
    successMsg: "Operación exitosa",
    trackingEventName: "entity_action",
    trackingProperties: {
        entity_id: entity.id,
        entity_name: entity.name,
        // ... contextual info
    },
});
```

The `useFormHelper` hook automatically:
- Captures `entity_action` on success
- Captures `entity_action_failed` with `error_status` on failure

### Components with custom flows

For operations not using `useFormHelper`, add `posthog.capture` directly:

```typescript
import posthog from "posthog-js";

// On success
posthog.capture("entity_action", {
    entity_id: entity.id,
    entity_name: entity.name,
});

// On failure
posthog.capture("entity_action_failed", {
    entity_id: entity.id,
    entity_name: entity.name,
    error_status: response.status || 500,
});
```

## Step 4: Apply naming conventions

**Event names** — `snake_case`, format: `entity_action`
- `station_created`
- `station_updated`
- `station_deleted`
- `activity_disabled`
- `vehicle_checklist_submitted`

**Failure events** — append `_failed`:
- `station_created_failed`
- `vehicle_checklist_submitted_failed`

**Properties** — always include:
- `entity_id` — primary identifier
- `entity_name` — human-readable name
- Contextual info: `is_child`, `parent_id`, `is_admin`, etc.

## Step 5: Verify

1. Ensure `import posthog from "posthog-js"` is present (only in `"use client"` components)
2. Ensure tracking is on success AND failure paths
3. Run `npx eslint <file>` and `npx tsc --noEmit`

## Reference patterns

- `StationManagementSettings.tsx` — CRUD tracking via `useFormHelper`
- `ActivityRow.tsx` — toggle/action tracking

## Rules

- Only track in `"use client"` components
- Never track in server actions or server components
- Use `useFormHelper.trackingEventName` when available (avoids duplicate tracking)
- Always include `entity_id` in properties
- Always track both success and failure
- Never track sensitive data (passwords, tokens)
