---
name: refactor-check
description: Scan for Clean Architecture violations, component scope violations, convention issues, and missing test coverage.
---

Validate architectural boundaries and code conventions across the codebase.

## Step 1: Ask scope

Ask the user:
- **Scope**: Full codebase scan or only changed files on current branch?
- If branch, detect changed files using same logic as `/lint-check`.

## Step 2: Run checks

### Check 1: Clean Architecture violations (server)

**Controllers importing repository implementations:**
```bash
# Controllers should only import use cases, not repositories
grep -rn "import.*RepositoryImpl" server/src/controllers/
```
Violation: Controllers must go through use cases, never access repositories directly.

**Use cases importing Sequelize models:**
```bash
# Use cases should only import repository interfaces and domain models
grep -rn "import.*from.*repository/models" server/src/domain/usecases/
grep -rn "import.*from.*repository/implementations" server/src/domain/usecases/
```
Violation: Use cases must depend on repository interfaces, not implementations.

**Repository implementations not using RequestContext:**
```bash
# Repo impls should use RequestContext.getModel(), not direct Sequelize
grep -rn "import.*SequelizeProvider" server/src/repository/implementations/
grep -rn "import.*MultiTenantSequelizeProvider" server/src/repository/implementations/
```
Violation: Repository implementations must use `RequestContext.getModel(ModelClass)`.

### Check 2: Component architecture violations (client)

**Cross-feature imports:**
```bash
# Features must NOT import from other features
grep -rn "from.*components/features/" client/components/features/ | grep -v "from.*components/features/$(dirname)"
```
Check each match to see if a feature imports from a DIFFERENT feature directory.

**UI/Shared importing from features:**
```bash
grep -rn "from.*components/features/" client/components/ui/
grep -rn "from.*components/features/" client/components/shared/
```
Violation: `ui/` and `shared/` must never depend on `features/`.

### Check 3: Convention violations

**Server actions missing "use server":**
```bash
# All files in apiRequests/ (except ApiFetch.ts, ErrorsHandler.ts, ZodSchemas/) must start with "use server"
```
Read each file in `client/utils/apiRequests/` and check first line is `"use server"`. Exclude utility files.

**Missing Zod validation in server actions:**
Check that every `get()`, `post()`, `put()`, `remove()` call passes a Zod schema as the second argument.

**Missing JSDoc on exported functions:**
```bash
# Check for exported functions/classes without preceding JSDoc
grep -B1 "^export " server/src/domain/usecases/*.ts | grep -v "^\*"
```
Skip test files (`*.test.ts`).

**Em-dash usage:**
```bash
grep -rn "—" server/src/ client/
```
Violation: Use hyphens (-) instead of em-dashes.

### Check 4: Documentation Hygiene (Doc Sync)

**Missing docs or links:**
- Verify `docs/architecture/` contains `system-overview.md`, `server-clean-architecture.md`, `client-architecture.md`, `server-multi-tenant.md`.
- Check if `AGENTS.md` Section 10 links to all these files.
- Check if `AGENTS.md` Section 9 lists all skills found in `.claude/skills/`.

**Doc Drift:**
- Detect significant changes on current branch (e.g., new `domain/models`, new `features/` folders, major refactors in `providers/`).
- If significant changes found, check if `docs/architecture/*.md` files were also modified.
- If code changed but docs didn't, flag as "Stale Documentation Warning".

### Check 5: Missing test coverage

**Use cases without unit tests:**
Compare files in `server/src/domain/usecases/` against `server/test/useCases/`:
```bash
ls server/src/domain/usecases/*.ts | sed 's|.*/||; s|\.ts||'
ls server/test/useCases/*.test.ts | sed 's|.*/||; s|\.test\.ts||'
```
Flag use cases that have no corresponding test file.

**Routes without integration tests:**
Compare route files against test files:
```bash
ls server/src/routes/*.ts
ls server/test/*Api.test.ts
```
Flag routes that have no integration test.

**Note:** Only flag missing tests as warnings. Do NOT auto-create tests unless the user explicitly asks.

## Step 3: Report results

Present findings grouped by category:

```
## Clean Architecture Violations
- ❌ server/src/controllers/.../X.ts imports RepositoryImpl directly (line N)

## Component Architecture Violations
- ❌ client/components/features/A imports from features/B (line N)

## Convention Violations
- ⚠️ client/utils/apiRequests/X.ts missing "use server" directive
- ⚠️ server/src/domain/usecases/X.ts:L42 — exported function missing JSDoc

## Missing Test Coverage
- ⚠️ server/src/domain/usecases/XUseCases.ts — no unit test found
- ⚠️ server/src/routes/x.ts — no integration test found
```

Use ❌ for violations and ⚠️ for warnings.

## Step 4: Ask about fixing

After presenting the report, ask the user:

> "Want me fix the violations? I can:
> 1. Fix all auto-fixable issues
> 2. Fix specific items (tell me which)
> 3. Leave as-is (report only)"

If the user chooses to fix:
- Fix code violations (move imports, add "use server", add JSDoc, etc.)
- Run `npx eslint --fix` on modified files
- Run `npx tsc --noEmit` to verify
- Do NOT create missing tests unless explicitly asked

## Rules

- This is a read-then-ask skill. Never auto-fix without asking first.
- Missing tests are warnings, not errors.
- Be specific about line numbers and file paths in reports.
- For large codebases, prefer scanning changed files over full scan.
