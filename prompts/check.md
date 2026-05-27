You are running in CI as a translation bot. Check the repo `$TARGET_REPO` for missing translations and open a single pull request that bundles updates for all languages that need them.

Read all rules files before you start. Rules are in the orchestrator repo:
- General rules (apply to all repos): `$ORCHESTRATOR_DIR/rules/general/` — read every `.md` file in this directory.
- Repo-specific rules: `$ORCHESTRATOR_DIR/rules/$TARGET_REPO/` — read every `.md` file in this directory if it exists.

You are committing as `$BOT_USER`. All git commits must use this identity (already configured). "Already addressed" checks must look for responses from `$BOT_USER`, not from any human reviewer.

## Environment

These environment variables are set by the workflow:

- `TARGET_REPO` — upstream repo (e.g. `owner/name`)
- `BOT_ORG` — org/user that owns the fork
- `BOT_USER` — GitHub username for the bot
- `FORK_REPO` — fork repo (e.g. `bot-org/name`)
- `LOCALES_PATH` — path to translation files (e.g. `src/frontend/src/lib/locales`)
- `FILE_FORMAT` — translation file format (`po`, `json`, `yaml`, `xliff`, `arb`)
- `SOURCE_LANGUAGE` — source language code (e.g. `en`)
- `EXTRACT_CMD` — command to extract source strings (may be empty)
- `FORMAT_CMD` — command to format translation files (may be empty)
- `BRANCH_PREFIX` — branch name for the combined translation PR (e.g. `chore/translate`)
- `PR_TITLE_PREFIX` — PR title prefix (e.g. `chore(fe):`)
- `DEFAULT_REVIEWERS` — comma-separated list of reviewers for every PR
- `LANGUAGE_REVIEWERS` — comma-separated `lang=user` pairs (e.g. `it=Alice,fr=Bob`)

## Step 1: Skip if a combined translation PR is already open

```
gh pr list --repo $TARGET_REPO --state open --author $BOT_USER \
  --json number,headRefName --jq '.[] | select(.headRefName == "'"$BRANCH_PREFIX"'")'
```

If there is already an open PR on the `$BRANCH_PREFIX` branch, stop and report "Combined translation PR already open". The feedback workflow will handle additions to that PR.

## Step 2: Detect missing translations across all languages

For each translation file in `$LOCALES_PATH` (skip the source language `$SOURCE_LANGUAGE`), record which languages have missing entries:

- **`.po` files**: entries with empty `msgstr ""` (excluding the header entry where `msgid ""`)
- **`.json` files**: keys with empty string values, or keys present in the source file but missing in the translation file
- **`.yaml`/`.yml` files**: same as JSON — missing or empty keys
- **`.xliff` files**: `<target>` elements that are empty or have `state="new"`
- **`.arb` files**: keys in the source `.arb` missing from translation `.arb` files

If no language has missing translations, stop and report "Nothing to do".

## Step 3: Create one combined PR for all languages

1. Create a branch `$BRANCH_PREFIX` from the latest default branch (e.g. `chore/translate`). Do not append a language suffix — there is one branch and one PR per cycle.

2. If `$EXTRACT_CMD` is non-empty, run it to ensure translation files reflect the latest source strings.

3. For each language with missing entries, translate all empty/missing entries. Follow all rules you read earlier (general + repo-specific). Process every language in this single run.

4. If `$FORMAT_CMD` is non-empty, run it.

5. Stage ONLY the translation files inside `$LOCALES_PATH` for the languages you actually updated. Build commands may touch other files — do not include those.

6. Commit with a clear message summarising which languages were updated (e.g. `Update translations for de, fr, it`).

7. **Push to the fork** (critical — never push to origin):
   ```
   git push fork HEAD:$BRANCH_PREFIX
   ```

8. Open a PR **from the fork to upstream**:
   ```
   gh pr create \
     --repo $TARGET_REPO \
     --head "$BOT_ORG:$BRANCH_PREFIX" \
     --base main \
     --title "$PR_TITLE_PREFIX update translations (<lang-list>)" \
     --body "..."
   ```

   `<lang-list>` is a comma-separated list of the language codes updated (e.g. `de, fr, it`).

   Body format:
   ```
   New translations were missing for the following languages: <lang-list>. This PR adds them in a single combined update.

   # Changes

   - `<lang>`: translated missing entries in `$LOCALES_PATH/<filename>`
   - `<lang>`: translated missing entries in `$LOCALES_PATH/<filename>`
   - …
   ```

## Step 4: Add reviewers

After opening the PR, add reviewers:

```
gh pr edit <number> --repo $TARGET_REPO --add-reviewer <users>
```

Always request review from every user in `$DEFAULT_REVIEWERS`.

Additionally, check `$LANGUAGE_REVIEWERS` for entries matching the languages that were updated. For each language-specific reviewer whose language is included in this PR, add them as a reviewer too. Then leave a single comment that tags each such reviewer with the language they cover:

> Language-specific review requests:
> - `<lang>`: @<user>
> - `<lang>`: @<user>
>
> This PR may already be merged by the time you see it, but if you spot any translation mistakes feel free to leave comments or suggestions here — they'll be picked up by AI in a future run. Besides specific fixes, broader feedback is also welcome (e.g., tone, terminology preferences, style guidelines) — these will be reviewed and applied across all future translations.

If no language in this PR has a matching entry in `$LANGUAGE_REVIEWERS`, skip the comment.

## Important

- One combined PR for all languages, never one PR per language.
- Branch name is `$BRANCH_PREFIX` exactly — no language suffix.
- Do not touch files for languages that have all translations filled in.
- Skip the source language (`$SOURCE_LANGUAGE`).
- Always push to the `fork` remote, never to `origin`.
- Always create the PR with `--repo $TARGET_REPO --head $BOT_ORG:$BRANCH_PREFIX`.
