Cut a release: create a `release/vX` branch off the develop branch, tag it `vX`, push both to remote, then publish the GitHub Release.

Here `X` is the app's version name (e.g. version name `1.2.0` → branch `release/v1.2.0`, tag `v1.2.0`).

Follow these steps in order. Do not skip any step.

## Step 0: Check GitHub authentication

Run `gh auth status` to verify the user is authenticated.

- If it succeeds, proceed.
- If it fails for any reason (not found or not authenticated), stop and tell the user: "`gh` is not set up yet — please run `gh auth login` first, then re-run this command." Do not continue with the remaining steps.

## Step 1: Detect the develop base branch

Find the develop/dev branch to cut the release from:

```bash
git branch -r | grep -E 'origin/(develop|dev)$'
```

Use `develop` if it exists, otherwise `dev`. Store it as `DEV_BRANCH`. If neither exists, stop and ask the user which branch to base the release on.

## Step 2: Resolve the version, branch name, and tag name

Read the app's `versionName` from the Android app module's build file:

```bash
rg 'versionName\s*=' androidApp/build.gradle.kts
```

If `androidApp/build.gradle.kts` doesn't exist, locate the app module's build file (`rg -l 'versionName' --glob '*.gradle.kts'`) and read `versionName` from there.

Extract the version:

- Take the raw `versionName` value (e.g. `1.3.0-KMP`).
- `X` is **only** the leading semver portion matching `[0-9]+\.[0-9]+\.[0-9]+` (e.g. `1.3.0`). Always strip any trailing suffix such as `-KMP`, `-UAT`, `-STG` — the branch and tag use the clean semver only.

Then set:
- `BRANCH = release/vX` (e.g. `release/v1.3.0`)
- `TAG = vX` (e.g. `v1.3.0`)

## Step 3: Fetch and check what already exists

```bash
git fetch origin --tags
```

Check whether the release branch already exists (local or remote):

```bash
git branch -a --list "$BRANCH" "origin/$BRANCH" "remotes/origin/$BRANCH"
```

Check whether the tag already exists (local or remote):

```bash
git tag -l "$TAG"
git ls-remote --tags origin "refs/tags/$TAG"
```

Remember both results — they drive the branching logic below.

## Step 4a: Release branch does NOT exist — create it

If `BRANCH` does not exist on remote:

- Check out the release branch from the tip of the develop branch:

```bash
git checkout -b "$BRANCH" "origin/$DEV_BRANCH"
```

- Push it to remote:

```bash
git push -u origin "$BRANCH"
```

Then continue to Step 5 (tag handling).

## Step 4b: Release branch ALREADY exists — open a PR into it and merge before tagging

If `BRANCH` already exists on remote, do **not** recreate or force it. Instead, bring the latest develop changes into the release branch via a PR, and that PR must be merged before the tag is (force-)updated.

- Gather the commits on `DEV_BRANCH` that are not yet on the release branch:

```bash
git log --oneline "origin/$BRANCH..origin/$DEV_BRANCH"
```

- If there are no such commits, tell the user the release branch is already up to date with `DEV_BRANCH`, skip PR creation, and continue to Step 5 (tag handling).
- If there are commits, open a PR merging `DEV_BRANCH` into `BRANCH` by invoking the `/pr` command flow with base = `BRANCH`. Create it directly with `gh`:

```bash
gh pr create --base "$BRANCH" --head "$DEV_BRANCH" \
  --title "Release $TAG: merge $DEV_BRANCH into $BRANCH" \
  --body "<summary of the commits gathered above>"
```

  Fill the body with a concise summary derived from the gathered commits. If a PR from `DEV_BRANCH` into `BRANCH` already exists and is `OPEN`, update its body instead of creating a duplicate (mirror the `/pr` command's create-or-update logic).

- The release branch must be merged **before** the tag is updated. Report the PR URL and, using AskUserQuestion, ask the user whether to merge it now:
  - Question: "The release branch needs the latest `$DEV_BRANCH` changes merged before the tag can be updated. Merge PR `<url>` now?"
  - Options: "Merge it now" / "I'll merge it myself — stop here (Recommended)"
- If the user chooses to merge now, merge the PR (`gh pr merge <url> --merge`), then `git fetch origin` so the local ref for `BRANCH` reflects the merge, and continue to Step 5.
- If the user chooses to merge it themselves, **stop here** — tell them to merge the PR and re-run `/release` afterward. Do not touch the tag until the release branch is merged.

## Step 5: Handle the tag

Only reached once the release branch is ready and merged (Step 4a, or Step 4b's up-to-date / just-merged path).

Make sure you are on `BRANCH` with the merged tip (`git branch --show-current`; check it out and `git pull` if needed) so the tag points at the correct commit.

**If `TAG` does not exist** (local or remote):

```bash
git tag -a "$TAG" -m "$TAG"
git push origin "$TAG"
```

**If `TAG` already exists**, use AskUserQuestion to ask whether to force-push the tag:

- Question: "Tag `$TAG` already exists on remote. Force-push it to point at the current (merged) release commit?"
- Options: "No, keep existing tag (Recommended)" / "Yes, force-push the tag"

If the user chooses **No**, keep the existing tag and continue to Step 6 (the release can still be published from the existing tag).

If the user chooses **Yes**, first confirm the release branch is merged and up to date (Step 4b must have completed its merge — never force-update the tag against an unmerged release branch), then re-point and force-push:

```bash
git tag -f -a "$TAG" -m "$TAG"
git push origin "$TAG" --force
```

## Step 6: Publish the GitHub Release

Once the branch and tag are pushed to remote successfully, invoke the `release-tag` skill to publish the GitHub Release for `TAG`.

The tag already exists at this point, so tell the skill the tag `$TAG` on branch `$BRANCH` has already been created and pushed and it only needs to publish the GitHub Release from it (it will detect the existing tag and skip re-creating it).

## Step 7: Report back

Report to the user:
- The release branch name and that it was pushed.
- The tag name and that it was pushed.
- The GitHub Release URL returned by the `release-tag` skill.
- If Step 4b ran, the PR URL instead of a tag/release.
