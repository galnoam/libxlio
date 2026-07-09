---
name: xlio-create-version
description: >-
  Prepare or cut a new XLIO release branch through an optional fork push: update vNext, create a
  vX.Y.Z branch, derive de-duplicated CHANGES entries from commits since the prior release using
  Redmine context, bump configure.ac and the RPM spec changelog, create the signed-off version
  commit, present it for review, and optionally push to the gnoam fork only after confirmation.
  Use when asked to create, cut, or prepare an XLIO version or release such as 3.71.0 or v3.71.0.
---

# Create an XLIO version

Prepare an XLIO release branch from the repository root. Perform the workflow in order and use the current repository history as the formatting source of truth.

## Guardrails

- Keep all existing user work intact. Before changing branches or pulling, inspect `git -c core.fileMode=false status --short`. Stop on any staged or tracked change because `pull --rebase` requires a clean worktree. Allow untracked paths only after confirming they cannot collide with checkout or release-file edits; otherwise stop and ask. Never stash, discard, or overwrite work without approval.
- Change and commit only `CHANGES`, `configure.ac`, and `contrib/scripts/libxlio.spec.in`.
- Never reset or overwrite a branch. The only force option allowed is the empty-expectation, create-only lease in step 8; it must reject rather than update an existing remote ref.
- Do not create a tag, update `vNext` to the release commit, or push anything unless the user explicitly requests it.
- Treat pull requests, integration into upstream `vNext`, and release tags as separate workflows outside this skill.
- Stay on the release branch when finished.

## 1. Resolve the target version

Accept either `X.Y.Z` or `vX.Y.Z`, but require exactly three numeric components. Ask for the version only when it is missing or invalid. Derive:

- Version triplet: `X.Y.Z`
- Branch: `vX.Y.Z`
- `configure.ac`: major `X`, minor `Y`, revision `Z`
- `CHANGES` and RPM release: `X.Y.Z-1`
- Commit subject: `version: X.Y.Z`

Use the current local date when editing the release files.

## 2. Update `vNext` and create the release branch

Before the first mutation, confirm the shell is at the repository root, inspect status, check whether the target branch already exists, and inspect whether local `vNext` has commits not present in its tracking branch:

```bash
git rev-parse --show-toplevel
git -c core.fileMode=false status --short
git show-ref --verify --quiet refs/heads/vX.Y.Z
git rev-list --left-right --count 'vNext...vNext@{upstream}'
```

Interpret `git show-ref` status `0` as "branch exists" and `1` as "branch absent"; any other status is an error, not proof of absence. If the release branch exists locally, stop before pulling and ask whether to reuse it or choose another version. If the user wants to recreate it, first show its tip and commits not reachable from `vNext`; delete it only after explicit confirmation, and never force-delete branch-only commits without clearly identifying what would be lost.

When reusing a branch, inspect its complete divergence from `vNext`. If it already contains a version commit, treat the release as already prepared: do not rebuild notes or create another version commit. If it is partial, identify its base and ask how to resume. Never include an existing version commit itself as a candidate release-note commit.

For `git rev-list --left-right --count`, the left count is local-only `vNext` history. If it is nonzero, or if the histories diverge, stop before rebasing and ask whether those commits belong in the release. If `vNext` has no upstream, compare it with `origin/vNext` and later use `git pull --rebase origin vNext`.

When the worktree and target branch checks are clear, run:

```bash
git switch vNext
git pull --rebase
git rev-list --left-right --count 'HEAD...@{upstream}'
git switch -c vX.Y.Z
```

Skip `git switch vNext` when already on it. If `vNext` has no configured upstream, use `git pull --rebase origin vNext` and compare `HEAD...origin/vNext` afterward. Require the post-pull counts to be `0 0` before branching; otherwise stop and explain the local-only or divergent commits. If a race causes branch creation to report that the branch now exists, stop and apply the same branch review above. Never delete or reset it on your own.

Do all remaining work on `vX.Y.Z`.

## 3. Learn the current release format

Inspect history until you have identified at least the five most recent actual release commits. A release commit must prepend a `Version ...` block; do not assume every commit touching `CHANGES` is a release:

```bash
git --no-pager log -n 20 --format="%H %s" -- CHANGES
git --no-pager show <commit> -- CHANGES configure.ac contrib/scripts/libxlio.spec.in
```

Expand the log beyond 20 entries if needed to find five genuine release commits. Open their actual diffs and match their file set, section order, indentation, date format, blank lines, entry ordering, author line, and commit subject. Let the observed history override the fallback baseline below.

The validated baseline is:

- A release commit changes only the three version files and preserves mode `100644`.
- `CHANGES` receives a new leading `Version X.Y.Z-1:` block with `Date + Time YYYY-MM-DD`, an underline, and non-empty `Added:` and/or `Fixed:` sections. Entry indentation is history-dependent, so copy the newest block exactly rather than assuming tabs or spaces.
- `configure.ac` updates `prj_ver_major`, `prj_ver_minor`, and `prj_ver_revision` in SECTION 1.
- `contrib/scripts/libxlio.spec.in` replaces the newest `%changelog` entry using RPM date formatting.
- The signed-off commit subject is `version: X.Y.Z`, without `-1`.

## 4. Build the release issue set

Identify the newest actual release commit from the inspected history by confirming that its diff prepends a `Version ...` block. Use it as the prior-release boundary; never use a non-release `CHANGES` edit. List every commit after it through the current `HEAD`, including bodies:

```bash
git --no-pager log <previous-release-commit>..HEAD --format="%H%x09%s%x09%b"
```

Process the range as follows:

1. Exclude commits whose subject starts with `[CI]`.
2. Exclude commits associated with `HPCINFRA-*`; these are infrastructure work, not product release-note entries.
3. Exclude numeric Redmine issues that are CI-only, including known issue `4642229` for valgrind suppressions/test infrastructure. When uncertain, inspect the ticket and diff before excluding it.
4. Extract issue IDs from the subject and body using the conventions visible in the repository, such as `RM #1234567` and `issue: 1234567`.
5. Group all commits with the same issue ID into exactly one release-note entry. Never emit the same issue ID twice.
6. Surface every remaining non-CI commit that has no identifiable issue ID. Do not silently omit it; inspect its diff and ask the user if its treatment remains ambiguous.

Do not add an issue that is absent from the commit range.

Resolve every remaining inclusion or exclusion ambiguity before editing or committing. If repository, diff, and ticket evidence still do not establish whether work is CI-only or user-facing, present the evidence and ask the user to decide.

## 5. Choose one title and section per issue

For every distinct issue:

1. Review all associated commit subjects, bodies, and relevant diffs.
2. Look up the issue through the configured Redmine integration and inspect its subject, tracker/type, and context. Never expose credentials or tokens.
3. Synthesize one concise, accurate, user-facing title that represents the complete change. Remove redundant prefixes and keep it to one line.
4. Classify bugs and defects as `Fixed`; classify features, feature requests, tasks, and enhancements as `Added`. Prefer the Redmine tracker when available, then repository evidence.

If Redmine is unavailable, use only the commit and code evidence and never fabricate ticket details. Resolve uncertain titles or classifications with the user before editing or committing; also flag the limitation in the final review.

Follow the ordering convention of the newest release block. If no ordering convention is evident, keep issues in order of first appearance in the commit range.

## 6. Edit the version files

### `CHANGES`

Prepend a block above the current first version block. Match the newest block exactly and omit empty sections:

```text
Version X.Y.Z-1:
Date + Time YYYY-MM-DD
=============================================================
Added:
	- RM #<id> <best title>

Fixed:
	- RM #<id> <best title>

```

Leave one blank line before the preceding `Version ...` block.

### `configure.ac`

In SECTION 1, set:

```m4
define([prj_ver_major], X)
define([prj_ver_minor], Y)
define([prj_ver_revision], Z)
```

### `contrib/scripts/libxlio.spec.in`

Replace the newest `%changelog` entry. Match the observed author line and RPM weekday/month/day spacing:

```text
* <Weekday> <Mon> <DD> <YYYY> NVIDIA CORPORATION <networking-support@nvidia.com> X.Y.Z-1
- Bump version to X.Y.Z
- Please refer to CHANGES for full changelog.
```

Use the RPM convention with two spaces before a single-digit day when current history does so.

## 7. Validate and commit

Inspect the complete patch before staging:

```bash
git -c core.fileMode=false status --short
git --no-pager diff --check
git --no-pager diff --summary -- CHANGES configure.ac contrib/scripts/libxlio.spec.in
git --no-pager diff -- CHANGES configure.ac contrib/scripts/libxlio.spec.in
```

Confirm that:

- Only the three intended paths changed.
- No mode changed from `100644` to `100755`.
- Every included issue appears exactly once.
- Every eligible commit in the range is accounted for, including explicitly explained exclusions.
- The version and date agree across all three files.

Stage exactly the intended files and verify the staged path set before committing:

```bash
git add -- CHANGES configure.ac contrib/scripts/libxlio.spec.in
git --no-pager diff --cached --name-only
git commit -s -m "version: X.Y.Z"
git --no-pager log -1 --format=%B
```

Require the staged path list to contain exactly the three version files. Match legitimate trailers required by current repository history or hooks, but never add assistant-generated attribution. If such attribution appears, amend with the same explicit subject while preserving required legitimate trailers.

## 8. Present the review and ask before pushing

Show the user:

- The new `CHANGES` block.
- The version changes in `configure.ac` and the RPM `%changelog`.
- A final table of issue ID, section, chosen title, contributing commits, and any exclusion or uncertainty.
- The commit hash, branch name, and clean/dirty status.

Then ask whether to push the release branch to the `gnoam` remote. Do not push before an explicit yes. Verify the remote first:

```bash
git remote get-url gnoam
git ls-remote --heads gnoam refs/heads/vX.Y.Z
git push --force-with-lease=refs/heads/vX.Y.Z: gnoam HEAD:refs/heads/vX.Y.Z
```

If `gnoam` is absent, ask for the correct remote. If `ls-remote` fails, stop and report the transport/authentication error. If it succeeds and returns a matching ref, the remote branch already exists: stop and ask the user rather than updating it. Only an empty successful result establishes that the branch is new. The empty expected value in `--force-with-lease=refs/heads/vX.Y.Z:` makes the push atomically create-only and rejects a branch that appears after the check; never weaken or retry that rejection with a normal or unconditional force push. Remain on `vX.Y.Z`.
