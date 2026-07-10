---
name: release-tag
description: Cut a GitHub release from the current repo — creates an annotated git tag matching the project's version, pushes it, and publishes a GitHub Release with a short human-written summary (not a raw one-line-per-PR changelog). Use this whenever the user asks to "release", "cut a release", "tag a release", "publish a release", or "push a new version tag" on GitHub for any repo.
allowed-tools: Read, Bash(git:*), Bash(gh:*)
---

Release a version of the current project by tagging a commit and publishing a GitHub Release from it. This is a visible, hard-to-reverse action (it pushes a tag and creates a public release) — walk through every check below before pushing or publishing anything, and get explicit confirmation from the user first.

This skill is intentionally repo-agnostic: every project names versions and branches differently, so infer conventions from the repo itself rather than assuming any particular file layout or branch name.

## 1. Figure out the tag name

Look for a version string in whichever of these exists in the repo root (or nearest module), in rough order of likelihood:

- `build.gradle.kts` / `build.gradle` — `versionName = "X.Y.Z"` (Android/Gradle)
- `package.json` — `"version": "X.Y.Z"` (Node/JS)
- `Cargo.toml`, `pyproject.toml`, `setup.py` — `version = "X.Y.Z"`
- `VERSION` / `version.txt` — a bare version string
- The most recent `git tag` if none of the above exist and the user just wants to bump it

Then check `git tag --sort=-creatordate | head -5` to see the existing naming convention (`v` prefix or not, suffixes like `-android`, plain semver, etc.) and match it exactly rather than inventing a new scheme. If the version file and the latest tag disagree on format, ask the user which to follow.

If you can't confidently find a version, ask the user directly what tag name they want rather than guessing.

## 2. Verify the commit being tagged is on a release/* branch

The commit being tagged must be on a `release/*` branch (e.g. `release/v1.2.0`) — this is a hard requirement, not a suggestion. Check with `git branch --show-current`:

- If the current branch matches `release/*`, proceed.
- If it doesn't, **stop**. Do not tag, do not offer to tag anyway. Tell the user the current branch (`<branch>`) isn't a `release/*` branch and that they need to check out or create the correct one first. Do not proceed past this step until they're on one.

Once confirmed on a `release/*` branch, run `git status`, then `git fetch origin <current-branch>` and compare HEAD to `origin/<current-branch>`:

- If HEAD is behind the remote, warn the user; they likely want to pull first.
- If HEAD is ahead (local unpushed commits), warn the user — tagging an unpushed commit means the tag points at a commit nobody else can see until it's pushed too.

## 3. Make sure nothing already exists

Check both, since a stale local tag and an existing GitHub release are different failure modes:

```
git tag -l <tag>
gh release view <tag>
```

If either exists, stop and tell the user — do not overwrite or delete an existing tag/release without them explicitly asking for that. If a tag exists locally but was never pushed, or was pushed but never had a release, say so and ask how they want to proceed (e.g. just push it / just publish the release) rather than restarting the whole flow.

## 4. Confirm with the user before doing anything irreversible

Show the user, in one message:
- The tag name
- The target commit's short SHA and subject line (`git log -1 --format='%h %s'`)
- That this will push the tag to `origin` and create a public GitHub Release

Wait for explicit go-ahead. Nothing in steps 5-7 should run before this.

## 5. Create and push the tag

```
git tag -a <tag> -m "<tag>"
git push origin <tag>
```

Skip tag creation if it already exists locally at the right commit (see step 3) — just push it.

## 6. Publish the GitHub Release with a concise summary

Repos with much PR volume make `gh release create --generate-notes` produce a wall of one-line-per-PR entries nobody reads. Build the changelog yourself and condense it:

```
git log --merges --first-parent <previous-tag>..<tag> --format='%s'
```

(fall back to `git log <previous-tag>..<tag> --format='%s'` if the repo doesn't merge via PRs) to get what changed since the last release, then group it into a couple of short sections — typically `## Highlights` for user-facing features/notable changes and `## Fixes` for bug fixes, a few bullets each. Skip pure internal chores/refactors unless they're worth calling out. If this is the very first tag in the repo, just summarize what the project does instead of a diff.

Write the summary to a temp file and publish with:

```
gh release create <tag> --title <tag> --notes-file <path>
```

Append a `**Full Changelog**: https://github.com/<owner>/<repo>/compare/<previous-tag>...<tag>` line at the end so anyone who wants the full detail can still get it.

## 7. Report back

Print the release URL that `gh release create` returns so the user can open it directly.

## Out of scope

This skill only tags and releases whatever version already exists in the repo — it does not bump version numbers in build files. If the user wants a version bump too, treat that as a separate, explicit step, and respect any project-specific approval requirements (e.g. a `CLAUDE.md` rule requiring sign-off before editing build/gradle files) before touching those files.
