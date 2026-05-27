You are running in CI, triggered by a polling check that found recent activity on PR `#$PR_NUMBER` in the `$TARGET_REPO` repository. Address any unaddressed reviewer feedback.

Read all rules files before applying any fixes. Rules are in the orchestrator repo:
- General rules (apply to all repos): `$ORCHESTRATOR_DIR/rules/general/` — read every `.md` file in this directory.
- Repo-specific rules: `$ORCHESTRATOR_DIR/rules/$TARGET_REPO/` — read every `.md` file in this directory if it exists.

You are committing as `$BOT_USER`. All git commits must use this identity (already configured). "Already replied" checks must look for responses from `$BOT_USER`.

## Environment

- `TARGET_REPO` — upstream repo (e.g. `owner/name`)
- `BOT_ORG` — org/user that owns the fork
- `BOT_USER` — GitHub username for the bot
- `FORK_REPO` — fork repo (e.g. `bot-org/name`)
- `PR_NUMBER` — the PR to check for feedback
- `PR_HEAD_REF` — the PR's head branch name
- `PR_MERGED` — `true` if the PR was merged, `false` if open
- `LOCALES_PATH` — path to translation files
- `FILE_FORMAT` — translation file format
- `EXTRACT_CMD` / `FORMAT_CMD` — build commands (may be empty)
- `BRANCH_PREFIX` — branch name prefix
- `PR_TITLE_PREFIX` — PR title prefix
- `DEFAULT_REVIEWERS` — comma-separated reviewer list
- `LANGUAGE_REVIEWERS` — comma-separated `lang=user` pairs
- `FEEDBACK_ALLOWLIST` — comma-separated list of allowed commenters

## Step 0: Determine PR kind

Fetch PR details:
```
gh pr view $PR_NUMBER --repo $TARGET_REPO --json title,state,mergedAt,merged,headRefName
```

Decide:
- **Translation PR (open)** — title contains translation-related keywords and PR is open. Push fixes to the fork branch (Step 2a).
- **Translation PR (merged)** — same title pattern, PR is merged. Open a NEW translation PR with the fixes (Step 2b).
- **Context-audit PR (open)** — title relates to translation context. Edits go to source files; re-run extract/format commands.
- **Context-audit PR (merged)** — reply suggesting a re-dispatch of the context-audit workflow.

If the title matches none of the above, stop and report "Not a translation-related PR".

## Step 1: Fetch feedback

Only process comments from users listed in `$FEEDBACK_ALLOWLIST`. Ignore everyone else, including yourself.

Pull all three comment streams:
- Inline review comments: `gh api repos/$TARGET_REPO/pulls/$PR_NUMBER/comments`
- Issue-style PR comments: `gh api repos/$TARGET_REPO/issues/$PR_NUMBER/comments`
- Review bodies: `gh api repos/$TARGET_REPO/pulls/$PR_NUMBER/reviews` — each review's `body` (when non-empty) is a comment to address.

A comment is **unaddressed** if there is no reply from `$BOT_USER` acknowledging it.

Every comment from allowed users MUST get a reply — even a simple acknowledgement when no code change is needed.

If no comments are unaddressed, stop and report "No unaddressed feedback".

## Step 2a: Apply fixes to an open PR

For each unaddressed comment:

1. Fetch the fork branch and check it out:
   ```
   git fetch fork $PR_HEAD_REF
   git checkout -B $PR_HEAD_REF fork/$PR_HEAD_REF
   ```

2. Apply the requested fix to the relevant translation file.

3. If `$EXTRACT_CMD` is non-empty, run it. If `$FORMAT_CMD` is non-empty, run it.

4. Stage only the relevant file(s). Do not stage unrelated changes from build commands.

5. Commit and push to the **fork**:
   ```
   git push fork HEAD:$PR_HEAD_REF
   ```

6. Reply to the comment confirming the fix.

## Step 2b: Create a new PR for fixes to a merged PR

You cannot push to a merged branch. Instead, route the fix into the combined translation branch:

1. Check if `$BRANCH_PREFIX` is already open as a PR on `$TARGET_REPO`. If yes, add the fix as a new commit on that PR's branch instead of opening another (fetch the branch from `fork`, apply the fix, push).

2. If no combined PR is open, create a branch `$BRANCH_PREFIX` from the latest default branch.

3. Apply the requested fix(es) to the relevant translation file(s). Multiple languages may be fixed together — they all share the same branch.

4. Run extract/format if configured, stage only the relevant translation file(s), commit.

5. Push to the **fork**:
   ```
   git push fork HEAD:$BRANCH_PREFIX
   ```

6. If a combined PR is already open, the push updates it — no new PR needed. Otherwise create a PR **from fork to upstream**:
   ```
   gh pr create --repo $TARGET_REPO --head "$BOT_ORG:$BRANCH_PREFIX" --base main ...
   ```

7. Add reviewers from `$DEFAULT_REVIEWERS` plus the language-specific reviewers from `$LANGUAGE_REVIEWERS` for any language touched by this fix.

8. Reply to each original comment on the merged PR with a link to the open/new combined PR.

## Step 3: Classify each comment

- **Narrow**: a fix to a specific translation. Nothing further needed beyond the fix.
- **Broad**: feedback that implies a pattern or rule for future translations (e.g. "always use formal 'you' in German", "prefer X over Y").

## Step 4: Rule-proposal PR for broad feedback

For each broad comment, create a rule-proposal PR on the **orchestrator** repo (`$GITHUB_REPOSITORY`):

1. The orchestrator repo is checked out at `$ORCHESTRATOR_DIR`. Work there for this step.

2. Create a branch `chore/translation-rule-<short-slug>` from the orchestrator's main branch.

3. Choose the target file:
   - **General rule** (applies to all languages across all repos) → add to or create a file in `rules/general/`.
   - **Repo-specific rule** → add to or create a file in `rules/$TARGET_REPO/`. Use descriptive filenames (e.g. `russian.md`, `terminology.md`).

4. The rule text should be self-contained and prescriptive.

5. Commit and push to the orchestrator's `origin`:
   ```
   cd $ORCHESTRATOR_DIR
   git checkout -b chore/translation-rule-<short-slug>
   git add rules/
   git commit -m "chore: propose translation rule — <summary>"
   git push origin HEAD
   ```

6. Open a PR on the orchestrator repo:
   ```
   gh pr create \
     --repo $GITHUB_REPOSITORY \
     --head "chore/translation-rule-<short-slug>" \
     --base main \
     --title "chore: propose translation rule — <one-line summary>" \
     --body "Rule proposed based on reviewer feedback from $TARGET_REPO#$PR_NUMBER.

   ## Context

   Reviewer: @<username>
   Original comment: <quoted text>"
   ```

7. Reply to the original feedback comment on the target repo PR with a link to the rule-proposal PR.

## Important

- Only fix what the reviewer asked for — do not add new translations or make unrelated changes.
- Preserve all variables (like `{variable}`) and tags (like `<0>...</0>`).
- Always reply to every comment from allowed users.
- Always push translation fixes to the `fork` remote, never to `origin`.
- Push rule proposals to the orchestrator's `origin` (not the fork).
