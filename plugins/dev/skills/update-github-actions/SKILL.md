---
name: update-github-actions
description: >
  Update GitHub Actions versions referenced via `uses:` in GitHub Actions workflow files
  (anything under .github/workflows, plus composite actions in action.yml/action.yaml, at any depth)
  to the latest release compatible with each action's current pinning style —
  major-version tag (e.g. actions/checkout@v7), full semver tag, or a commit SHA with a version comment.
  Use when the user asks to bump, update, upgrade, refresh, or check GitHub Actions versions,
  or asks "are our workflow actions up-to-date" for this repo.
---

# Update GitHub Actions versions in workflow files

A `uses:` line references a versioned action three ways in practice:

```yaml
- uses: actions/checkout@v6
- uses: actions/checkout@v6.0.1
- uses: actions/checkout@8f4b7f84864484a7bf31766abe9204da3cbe65b3 # v6.0.1
```

The third form (pin to a full commit SHA, with the human-readable tag as a trailing comment)
is a common supply-chain-security convention (e.g. Dependabot, StepSecurity `pin-github-action`) —
when you see it, preserve it: resolve the new tag to its commit SHA and update both the SHA and the comment.
Don't "helpfully" convert a SHA-pinned action to a floating tag, or vice versa — match what's already there.

## 1. Find the target(s)

  - If invoked with an argument that looks like a path or glob, scope to that file (or files) only.
  - Otherwise, recursively find every workflow and composite-action file under the current directory (any depth):

    ```
    find . -type f \( -path '*/.github/workflows/*.yml' -o -path '*/.github/workflows/*.yaml' \)
    find . -type f \( -name 'action.yml' -o -name 'action.yaml' \)
    ```

    Skip directories like `.git`, `node_modules`, and `vendor`.
    Confirm the list with the user before touching more than one file.
  - If an argument contains `--dry-run`, do the full analysis and report but skip step 4 (no file edits).

## 2. Parse each `uses:` line

For every `uses:` value in every job step (and, in composite actions, every `runs.steps` entry):

  - Skip local/relative actions (`uses: ./.github/actions/foo`) — nothing to version.
  - Skip Docker-image actions (`uses: docker://<image>:<tag>`) — that's a container image tag,
    not a GitHub Actions release; point the user at a Docker-image-bump workflow for those instead.
  - Skip refs that aren't versions at all — a branch name (`@main`, `@master`) or an unqualified ref
    that doesn't look like a tag or a 40-character hex SHA.
    These are intentionally floating; leave them alone and don't report them as outdated.
  - Otherwise the reference is `owner/repo@ref` or,
    for a reusable workflow, `owner/repo/.github/workflows/<file>@ref`
    (bump the `@ref` the same way; the path in between is irrelevant to versioning).
  - Classify the current pin:
    1. **Major-version tag** — `v` followed by a single integer (`v4`, `v6`),
       the convention most actions publish for auto-updating within a major.
       This is what most `uses:` lines look like.
    2. **Full tag** — `v` (or no prefix) followed by two or three numeric segments (`v6.0.1`, `4.2.0`).
    3. **Commit SHA** — a 40-character hex string, almost always followed by `# <tag>`
       naming the version it corresponds to (read the comment to know which tag it currently tracks;
       if there's no comment, treat the current version as unknown and just diff SHAs after step 3).

## 3. Find the latest matching ref

Use `curl` via Bash against the GitHub API.
If `GITHUB_TOKEN` is set in the environment, pass it as `-H "Authorization: Bearer $GITHUB_TOKEN"`
on every call to avoid the low unauthenticated rate limit.

  - Get the latest release/tag:

    ```
    curl -s "https://api.github.com/repos/<owner>/<repo>/releases/latest" | jq -r '.tag_name'
    ```

    Some actions don't cut GitHub Releases and only push tags —
    if `releases/latest` 404s, list tags instead and pick the highest by semver:

    ```
    curl -s "https://api.github.com/repos/<owner>/<repo>/tags?per_page=100" | jq -r '.[].name'
    ```

  - Filter the tag list to the current pin's shape from step 2
    (same prefix convention, no `-rc`/`-beta`/`-alpha` suffix unless the current tag already has one)
    and take the highest by semantic-version comparison, not string sort.
    - If the current pin is a **major-version tag** (`v6`),
      the "latest matching" tag is just the highest major tag that exists (`v7` if it exists, else stay on `v6`) —
      major-tag actions intentionally don't publish every patch as its own major tag,
      so compare majors directly rather than expecting a `v6.x.x` series.
    - If the current pin is a **full tag** (`v6.0.1`), find the highest full tag overall,
      same as the container-image skill's semver comparison.
  - If the current pin is a **commit SHA**, resolve the target tag (from the previous bullet) to its commit SHA:

    ```
    curl -s "https://api.github.com/repos/<owner>/<repo>/git/refs/tags/<tag>" | jq -r '.object'
    ```

    If `.object.type` is `"tag"` (an annotated tag), dereference one more hop to get the real commit:

    ```
    curl -s "https://api.github.com/repos/<owner>/<repo>/git/tags/<sha>" | jq -r '.object.sha'
    ```

  - If nothing survives the filter, or the API call fails (private repo, rate-limited, network issue),
    don't guess — leave that line untouched and note it in the report as needing manual review.

## 4. Apply the change

Edit only the version part of the `uses:` line, in place, preserving indentation, quoting style, and step name/comments.
For a SHA-pinned action, update both the SHA and its trailing `# <tag>` comment together — never update one without the other,
since a stale comment next to a fresh SHA (or vice versa) is misleading and defeats the point of the pin.
Don't reorder steps, don't touch unrelated `with:`/`env:` blocks, don't "clean up" unrelated lines.

If the new major version differs from the current major version, still apply it (that's genuinely the latest),
but flag it clearly in the report — actions frequently change required Node/runner versions
or `with:` inputs across a major bump, and it's worth the user reading the release notes.

## 5. Report

Finish with a table like:

| File | Step | Old ref | New ref | Notes |
|----|----|----|----|----|
| .github/workflows/claude.yml | actions/checkout | v6 | v7 | |
| .github/workflows/ci.yml | actions/setup-node | v4 | v4 | already latest |
| .github/workflows/ci.yml | actions/upload-artifact | 8f4b7f8...cbe65b3 # v5.0.0 | 6c1b6e5...a1f2d90 # v6.0.0 | **major bump** — check release notes |
| .github/workflows/deploy.yml | some-org/private-action | main | — | branch ref, not a version pin — skipped |

Then run `git diff` in the affected repo so the changes are visible, and stop.
Don't commit, push, or open a PR unless the user explicitly asks for that next.
