---
name: address-review
description: Address review comments on a pull request
---

Address code review comments on the current branch's PR. Fetches unresolved review threads, plans how to address each one (including the exact thread replies), implements the changes after approval, and posts replies to each thread.

## Step 1: Identify the PR and repo

```bash
gh pr view --json number,url,title -q '{number: .number, url: .url, title: .title}'
gh repo view --json owner,name -q '{ owner: .owner.login, repo: .name }'
```

If no PR exists for the current branch, tell the user and stop.

Store the PR number, owner, and repo name as separate values — you will need them throughout.

## Step 2: Fetch unresolved review threads

Use the GitHub GraphQL API to fetch review threads with their comment history. The query is capped at 100 threads and 50 comments per thread — sufficient for virtually all PRs. If a PR exceeds these limits, paginate using `pageInfo { hasNextPage endCursor }` and `after:`.

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $number) {
        reviewThreads(first: 100) {
          nodes {
            id
            isResolved
            isOutdated
            path
            line
            startLine
            comments(first: 50) {
              nodes {
                body
                author { login }
                createdAt
              }
            }
          }
        }
      }
    }
  }
' -f owner=OWNER -f repo=REPO -F number=PR_NUMBER
```

Replace `OWNER`, `REPO`, and `PR_NUMBER` with the values from Step 1.

**Filter to unresolved threads only** (`isResolved: false`). Exclude threads that are outdated (`isOutdated: true`) — mention them separately as "outdated and skipped" so the user is aware.

If all threads are resolved (or there are none), tell the user there are no review comments to address and stop.

## Step 3: Read the relevant code

For each unresolved thread, read the file at the path indicated by the thread. Read enough surrounding context to understand the code — not just the specific line. If multiple threads reference the same file, read the file once.

**Critical**: Read **all** comments in each thread, not just the first one. Reviewers often clarify or expand on their feedback in follow-up messages. The last comment in the thread is often the most relevant.

## Step 4: Present the plan

Present a structured plan to the user. For **each** unresolved thread, include:

1. **Thread location**: The file path and line number
2. **Review comment**: Quote the reviewer's comment (and any follow-ups in the thread)
3. **Action**: One of:
   - **Address**: Describe the specific code change you will make
   - **Dismiss**: Explain why the comment doesn't apply or why no change is needed
4. **Thread reply**: The **exact** message you will post in the PR thread after changes are committed. This message must:
   - For addressed comments: Briefly explain what was changed and how the feedback was incorporated
   - For dismissed comments: Provide clear, respectful reasoning for why no change was made
   - End with: `— 🤖 Claude Code`

Format example:

```
### Thread 1 · `server/src/controllers/UserController.ts` L42

> "This should validate the input before passing to the use case" — @reviewer

**Action**: Address — Add Zod validation for the request body before calling the use case.

**Thread reply**:
> Added input validation using a Zod schema before the use case call. The schema validates `name` (string, non-empty) and `email` (valid email format). Invalid requests now return 400 with structured error details.
>
> — 🤖 Claude Code

---

### Thread 2 · `client/components/Modal.tsx` L18

> "Can we use a portal here to avoid z-index issues?" — @reviewer
> "Actually the z-index is fine in this case, but it's still a good pattern to follow" — @reviewer

**Action**: Dismiss — The component already renders via `ReactDOM.createPortal` to `document.body` (see L5). The reviewer may have missed this since the portal call is at the top of the file.

**Thread reply**:
> This component already uses `ReactDOM.createPortal` to render to `document.body` (line 5), so z-index stacking context isn't an issue here. No changes needed.
>
> — 🤖 Claude Code
```

After presenting the full plan, **ask the user for feedback**. Do not proceed until they respond.

## Step 5: Iterate on the plan

If the user requests changes:

- Adjust the proposed actions, code changes, or thread reply messages as requested
- Present the updated sections (only the changed parts, not the entire plan again)
- Ask for confirmation

Repeat until the user explicitly approves the plan.

## Step 6: Implement the changes

Once approved, implement all code changes from the plan. For each change:

1. Read the file (if not already in context)
2. Make the edit
3. Run linting and type-checking to verify correctness:

Change to the appropriate subdirectory (`cd client/` or `cd server/`) first, then run each command separately:

```bash
cd client/  # or cd server/
npx eslint "<file>"
```

```bash
npx tsc --noEmit
```

If linting or type-checking fails, fix the issues before proceeding.

## Step 7: Ask for confirmation

Present a summary of all changes made (files modified, what changed in each). Ask the user to review and confirm they are happy with the implementation.

If the user wants adjustments, make them and ask again. Do not proceed to committing until the user explicitly confirms.

## Step 8: Commit, push, and reply

Only after explicit user confirmation:

### 8a. Branch safety check

Before committing, verify the current branch is not protected:

```bash
git branch --show-current
```

If the branch is `main` or starts with `release`, **stop immediately** — do not commit. Notify the user and ask them to switch to the correct feature branch.

### 8b. Commit and push

Check recent `git log --oneline -5` to match the repo's commit message style, then:

```bash
git add "<file1>" "<file2>" ...
git commit -m "$(cat <<'EOF'
<descriptive commit message>

Co-Authored-By: Claude Code <noreply@anthropic.com>
EOF
)"
git push
```

### 8c. Reply to each thread

For each thread in the approved plan, post the approved reply message using the GraphQL API:

```bash
gh api graphql -f query='
  mutation($threadId: ID!, $body: String!) {
    addPullRequestReviewThreadReply(input: {
      pullRequestReviewThreadId: $threadId,
      body: $body
    }) {
      comment {
        url
      }
    }
  }
' -f threadId='THREAD_NODE_ID' -f body="$(cat <<'EOF'
APPROVED_REPLY_MESSAGE
EOF
)"
```

Replace `THREAD_NODE_ID` with the thread `id` from Step 2, and `APPROVED_REPLY_MESSAGE` with the exact reply text approved in the plan. Use the heredoc pattern shown above to safely handle multi-line replies and quotes.

**Post the replies exactly as approved.** Do not modify them at this stage.

## Rules

- **Never resolve threads** — that is the reviewer's responsibility.
- **Never force-push** — always use regular `git push`.
- **Preserve thread context** — always read ALL comments in a thread before planning a response. A thread may have back-and-forth discussion; the latest comment is often the most important.
- **Be honest in replies** — if a comment was dismissed, explain why clearly. Never claim changes were made when they were not.
- **Group related changes** — if multiple threads affect the same file, make all changes to that file together for a clean diff.
- **Plan completeness** — the plan must include the exact thread reply text. No replies should be composed after the plan is approved.
