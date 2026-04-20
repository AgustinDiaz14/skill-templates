---
name: update-design-system
description: Update the /design-system showcase page when a UI component is added or modified in components/ui/.
---

Update the design system showcase page to reflect new or modified UI components.

## Step 1: Identify the component

Ask the user which component was added or modified in `client/components/ui/`.

If not specified, check recent git changes:
```bash
git diff --name-only HEAD~1 -- client/components/ui/
```

## Step 2: Read the component

Read the component file to understand:
- Component name and export
- All props (types, defaults, required vs. optional)
- Variants (sizes, colors, states)
- Interactive states (hover, focus, disabled, loading)

## Step 3: Read current design system page

Read `client/app/design-system/page.tsx` to understand the existing structure and how other components are showcased.

## Step 4: Add showcase section

Add a new section to the design system page for the component:

```tsx
{/* {ComponentName} */}
<section className="space-y-4">
    <h2 className="text-xl font-semibold">{ComponentName}</h2>
    <p className="text-fg-secondary text-sm">
        {Brief description of the component and its purpose}
    </p>
    <div className="flex flex-wrap gap-4 items-center">
        {/* Default variant */}
        <{ComponentName} prop1="value1" />

        {/* Other variants */}
        <{ComponentName} prop1="value2" variant="secondary" />

        {/* Disabled state */}
        <{ComponentName} prop1="value3" disabled />

        {/* Loading state if applicable */}
        <{ComponentName} prop1="value4" loading />
    </div>
</section>
```

Show:
- All meaningful variants
- All interactive states (default, disabled, loading)
- Different sizes if the component supports them
- Edge cases (long text, empty state)

## Step 5: Verify

```bash
cd client/
npx tsc --noEmit
npx eslint app/design-system/page.tsx
```

## Rules

- Keep the design system page organized — group related components together
- Show real, meaningful examples (not placeholder text like "Lorem ipsum")
- Include all variants the component supports
- The design system page is only visible in development (`process.env.NODE_ENV === "development"`)
- Don't add external dependencies for the showcase page
