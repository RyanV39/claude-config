Commit changes, create a branch, and open a PR with a filled-in template.

Follow these steps in order. Do not skip any step.

## Step 0: Check GitHub authentication

Run `gh auth status` to verify the user is authenticated.

- If it succeeds, proceed.
- If it fails for any reason (not found or not authenticated), stop and tell the user: "`gh` is not set up yet — please run `gh auth login` first, then re-run this command." Do not continue with the remaining steps.

## Step 0b: Detect base branch

Determine the base branch for the PR by running:

```bash
git remote show origin | grep 'HEAD branch' | awk '{print $NF}'
```

If that fails or returns empty, fall back to checking whether `main` or `develop` exists on remote:

```bash
git branch -r | grep -E 'origin/(main|develop)' | head -5
```

Use the result (prefer `develop` if both exist, otherwise `main`) as `BASE_BRANCH` for the rest of the steps.

## Step 1: Commit changes

Run `git status` to check for uncommitted changes (staged or unstaged).

If there are changes:
- Run `git diff` and `git diff --staged` to understand what changed.
- Generate a commit message following the 50/72 rule: subject line ≤50 chars, body lines ≤72 chars. Use conventional commit format (`feat:`, `fix:`, `chore:`, etc.) based on the nature of the changes.
- Stage all modified/tracked files and commit with the generated message.
- Do NOT use `git add -A` — prefer `git add` on specific files.
- Do NOT include a `Co-Authored-By` trailer or any author attribution in the commit message.

If there are no uncommitted changes, run `git fetch origin` and then `git log origin/<BASE_BRANCH>..HEAD --oneline` to check for commits ahead of remote.

- If there are commits ahead of remote, proceed to Step 2 using those commits to inform the branch name and PR content.
- If there are no uncommitted changes AND no commits ahead of remote, tell the user "Nothing to commit or push. Exiting." and stop — do not continue with any further steps.

## Step 2: Branch handling

Check the current branch with `git branch --show-current`.

- If the current branch is `<BASE_BRANCH>`:
  - Run `git log <BASE_BRANCH>..HEAD --oneline` to read only the commits newly added on this branch.
  - Derive a branch name from the most relevant commit(s): use the conventional prefix (`feat/`, `fix/`, `chore/`, etc.) from the commit type, then a short kebab-case slug of the subject. Example: `feat: add login screen` → `feat/add-login-screen`.
  - Use AskUserQuestion to show the suggested name and ask the user to confirm or provide a different name (offer the suggestion as the first option).
  - Create and checkout the new branch: `git checkout -b <branch-name>`
- If the current branch is anything other than `<BASE_BRANCH>`, stay on that branch and proceed.

## Step 3: Choose PR template

Check whether a `.github/PULL_REQUEST_TEMPLATE/` directory exists:

```bash
ls .github/PULL_REQUEST_TEMPLATE/ 2>/dev/null
```

**If templates are found**, use AskUserQuestion to ask the user which PR template to use. Offer options based on the files present — common ones are:
- `Feature` (default) → `.github/PULL_REQUEST_TEMPLATE/pull_request_template.md`
- `Bug` → `.github/PULL_REQUEST_TEMPLATE/bugfix_pull_request_template.md`
- `Tech` → `.github/PULL_REQUEST_TEMPLATE/tech_pull_request_template.md`

Only offer options for files that actually exist. Mark `Feature` (or the first available option) as the recommended/default.

**If no templates are found**, also check for a single template at `.github/pull_request_template.md`. If that exists, use it without asking.

**If no templates exist at all**, write a concise PR description from scratch with these sections: Summary, Changes, and Testing.

Read the chosen template file (if any), then fill in its sections based on the actual changes: use `git diff <BASE_BRANCH>..HEAD` and the commit messages to understand what was changed, and write concrete content for each section. Do not leave placeholder text — replace every section with real content derived from the diff and commits.

## Step 4: Push the branch

Push the current branch to remote:

```bash
git push -u origin HEAD
```

## Step 5: Create or update the PR

First check whether a PR already exists for the current branch:

```bash
gh pr view --json url,title 2>/dev/null
```

**If no PR exists**, create one targeting `<BASE_BRANCH>` using `gh pr create`:
- `--base <BASE_BRANCH>`
- `--title` based on the branch name or latest commit message
- `--body` set to the filled-in template content

```bash
gh pr create --base <BASE_BRANCH> --title "<title>" --body "$(cat <<'EOF'
<template content here>
EOF
)"
```

**If a PR already exists**, first check whether it is already merged:

```bash
gh pr view --json state --jq '.state'
```

- If the state is `MERGED`, treat it as if no PR exists and create a new one with `gh pr create` as above.
- If the state is `OPEN`, update its description to reflect all commits now on the branch (not just the new ones). Re-run `git diff <BASE_BRANCH>..HEAD` and `git log <BASE_BRANCH>..HEAD` to get the full picture, regenerate the filled-in template body, then update:

```bash
gh pr edit --body "$(cat <<'EOF'
<updated template content here>
EOF
)"
```

Return the PR URL to the user when done.
