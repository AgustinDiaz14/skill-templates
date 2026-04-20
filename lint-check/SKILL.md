---
name: lint-check
description: Run ESLint + TypeScript checks on changed files, auto-fix what's possible, report the rest.
---

Run code quality checks on all files changed on the current branch. Auto-fix what ESLint can fix, then report remaining issues.

## Step 1: Detect the base branch

```bash
git fetch origin
CANDIDATES="main $(git branch -r --list 'origin/release*' --sort=-committerdate | sed 's|origin/||' | head -5)"
for branch in $CANDIDATES; do
  echo "$(git rev-list --count $(git merge-base HEAD origin/$branch)..HEAD) $branch"
done | sort -n | head -1 | awk '{print $2}'
```

Store result as `BASE_BRANCH`.

## Step 2: Get changed files

```bash
git diff --name-only origin/$BASE_BRANCH...HEAD
```

Split into two groups:
- **Client files**: `.ts` / `.tsx` files under `client/`
- **Server files**: `.ts` files under `server/src/` or `server/test/`

Ignore deleted files (check they still exist before linting).

## Step 3: Run ESLint with auto-fix

For each group, run from the appropriate directory:

**Client:**
```bash
# From root
scripts/nvm-run.sh npx eslint --fix client/<file1> client/<file2> ...
```

**Server:**
```bash
# From root
scripts/nvm-run.sh npx eslint --fix server/<file1> server/<file2> ...
```

If `--fix` modifies files, report what was auto-fixed.

## Step 4: Run ESLint (report remaining)

Run ESLint again without `--fix` to capture remaining issues:

```bash
scripts/nvm-run.sh npx eslint <file1> <file2> ...
```

## Step 5: Run TypeScript type checking

From the root:

**Client:**
```bash
scripts/nvm-run.sh npx tsc --noEmit -p client/tsconfig.json
```

**Server:**
```bash
scripts/nvm-run.sh npx tsc --noEmit -p server/tsconfig.json
```

## Step 6: Report results

Present a summary:
- Files checked (count per group)
- Auto-fixed issues (if any)
- Remaining ESLint errors/warnings grouped by file
- TypeScript errors grouped by file
- If all clean, say "All checks passed."

## Rules

- Never modify files beyond what `eslint --fix` does automatically.
- If no files changed on the branch, tell the user and stop.
- If only one group has changes (e.g., only client), skip the other group entirely.
