---
name: debug-test
description: Diagnose and fix a failing test - run it, analyze the failure, implement the fix, and verify.
---

Debug a failing test by systematically analyzing the error and fixing the root cause.

## Step 1: Identify the test

Ask the user for:
- **Test file path** or test name
- If not provided, ask which test is failing

## Step 2: Run the test

```bash
# Run from root
scripts/nvm-run.sh npm test path/to/file.test.ts
# or directly in subproject
scripts/nvm-run.sh npm test path/to/file.test.tsx
```

Capture the full output including:
- Which test(s) failed
- Error message and stack trace
- Expected vs. actual values

## Step 3: Read source files

1. Read the **test file** completely
2. Read the **source file** being tested
3. If the test uses mocks, read the mocked modules too
4. If the test uses `TestHelper`, read `server/test/utils/TestHelper.ts`

## Step 4: Analyze the failure

Common failure patterns:

### Assertion mismatch
- Expected value differs from actual
- Check: Is the source code returning the wrong thing, or is the test expectation wrong?
- Check: Are there type coercions (number vs. string from SQLite)?

### Mock issues
- Mock not returning expected value
- Mock not being called
- Mock implementation missing a method
- Check: Is the mock matching the current interface signature?

### Setup issues
- Missing seed data (role, permission, station)
- DB not reset between tests
- Missing provider mocks
- Check: Does `beforeEach` properly reset state?

### Async issues
- Promise not awaited
- `void` vs. `async` handler mismatch
- Race condition in test
- Check: Are all async operations awaited?

### Import/module issues
- Circular dependency
- Mock hoisting order (`jest.mock` must be before imports in some patterns)
- Check: Are mocks declared before the module that uses them is imported?

## Step 5: Propose and implement fix

Explain the root cause clearly, then:
1. Propose the fix
2. Implement it
3. If the fix is in the source code (not the test), make sure it doesn't break other tests

## Step 6: Re-run and verify

```bash
npm test -- path/to/file.test.ts
```

If still failing, repeat Steps 4-6.

Once green, run related tests to make sure nothing else broke:
```bash
npm test -- --testPathPattern="relatedPattern"
```

## Step 7: Lint check

```bash
npx eslint path/to/modified/files
```

## Rules

- Prefer fixing the source code if the test expectation is correct.
- Prefer fixing the test if the source code behavior is correct.
- Never delete or skip a test to "fix" it unless the test is genuinely wrong.
- If a test is flaky (passes sometimes, fails sometimes), identify the race condition.
- Always re-run after fixing to confirm the fix works.
