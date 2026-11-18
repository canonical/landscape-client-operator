---
# Weekly changelog PR bot (GitHub Agentic Workflow).
#
# This is the gh-aw SOURCE file. After editing, regenerate the compiled
# GitHub Actions workflow with:
#
#     gh aw compile changelog-bot
#
# and commit the generated `.github/workflows/changelog-bot.lock.yml` alongside it.
#
# The shared plumbing (engine, checkout, runner setup, gh/node/compose installs,
# threat detection, docs push handler) comes from the shared component imported
# below. What stays here is only what differs for this repo: the trigger, the
# runners, the prompt body, and the changelog PR's allowed-files.

on:
  # Fire every Saturday at around 12:00 UTC.
  schedule: weekly on saturday around 12:00 UTC
  # Allow maintainers to trigger a run on demand.
  workflow_dispatch:

permissions:
  contents: read
  # The agent must list open pull requests to enforce "never open a second
  # changelog PR". Without this the token is Contents:read only and every
  # list_pull_requests call returns 403 "Resource not accessible by integration".
  pull-requests: read

# Run on the Canonical self-hosted runner fleet. The agent and
# threat-detection jobs need Docker/AWF and target the noble/amd64 labels;
# the activation, conclusion and safe_outputs jobs use the slim self-hosted
# label. These cannot live in a shared component (gh-aw only allows them in a
# main workflow).
runs-on: [self-hosted, noble, amd64]
runs-on-slim: self-hosted

imports:
  - canonical/landscape-internal-docs/.github/workflows/shared/changelog-bot-infra.md@d627562cfa37544175061045d201f4304154a62e

# The agent never pushes directly; its file edits are turned into pull requests.
# The shared component supplies the safe-outputs token, runners, threat
# detection, and the docs push / add-comment handlers. What stays here is the
# own-repo changelog PR, whose allowed-files is this repo's alone.
safe-outputs:
  create-pull-request:
    # The own-repo changelog PR is created with the same PAT as the rest of the
    # safe_outputs job, not the default GITHUB_TOKEN: gh-aw binds that job to a
    # single credential (github/gh-aw#77580) and its checkout/push use
    # LANDSCAPE_CHANGELOG_TOKEN. Naming it here keeps the handler from falling
    # back to the job-wide GITHUB_TOKEN env if the component ever stops setting
    # the blanket token. A GITHUB_TOKEN-authored PR would also not trigger this
    # repo's lint/cla checks.
    github-token: ${{ secrets.LANDSCAPE_CHANGELOG_TOKEN }}
    title-prefix: "build: "
    labels: [changelog, automation]
    team-reviewers: [canonical/landscape]
    draft: false
    if-no-changes: ignore
    # Push with a plain git push rather than the createCommitOnBranch GraphQL
    # path: that path produces commits GitHub does not mark verified for this
    # bot token, and pushSignedCommits then refuses to publish the branch.
    signed-commits: false
    max: 1
    preserve-branch-name: true
    recreate-ref: true
    # The bot only ever edits CHANGELOG.md in this repo; the handler rejects
    # any other path outright. (gh-aw allowed-files filters by path only, not
    # diff hunk, so "insert one new dated section, never touch existing
    # sections" stays prompt-enforced.)
    allowed-files: ["CHANGELOG.md"]
---

# Weekly changelog update

You maintain the changelog for the Landscape Client charm (the subordinate Juju charm that registers machines with Landscape). Your job is to draft a new dated changelog section summarizing recent user-facing changes and open a pull request with it.

## Hard prohibitions

- NEVER create, edit, or delete any file other than `CHANGELOG.md` in this repository. This is mechanically enforced by the pull-request handler: it rejects any patch touching another path.
- NEVER edit or delete an existing `CHANGELOG.md` section or entry, even one with problems. Flag problems in the `Changelog entries to review` section instead.

## Gather the commits

1. The changelog lives at `CHANGELOG.md` in the repository root. It may not exist yet; this bot creates it on the first run.
2. The scan window is time-based: list the commits from the last four weeks against the repository's default branch, not the ref this run was dispatched on: `git log --since="4 weeks ago" --no-merges --pretty=%H%x09%s origin/main`. Always use `origin/main` for the range — a `workflow_dispatch` run may check out a bot or feature branch whose own commits are automation churn, and those must never be classified as product changes. If `origin/main` is not present locally, use the checked-out HEAD and say so in the run output. Do NOT anchor on release tags or on `CHANGELOG.md` commits: every push to `main` is published to Charmhub `latest/edge` and CI tags the commit once per platform as `<base>-<arch>-rev<N>` (older releases used legacy `rev<N>` tags), so the most recent tagged commit is almost always the tip of `main` and a tag anchor would leave an empty window. Four weeks against a weekly schedule gives three weeks of overlap with the previous run, so omissions are still caught; the deduplication rules below prevent double-filing.
3. Skip any commit whose subject has no trailing pull request reference ` (#1234)`. Such commits cannot carry the `(#<N>)` marker the deduplication relies on, so they can never be filed safely; if one looks user-facing, flag it in the review section below instead of adding a changelog entry for it.
4. Deduplicate the rest by pull request number. For each commit, extract its PR number `<N>` from the trailing ` (#1234)` reference in the subject. Skip the commit if `CHANGELOG.md` exists and already contains the exact reference `(#<N>)` anywhere (match the full number — `(#123` must not match `(#1234)`). Every entry this bot writes ends with its ` (#1234)` reference, so an existing mention means a previous run already covered that PR.

## Select user-facing commits only

Do not decide from memory or by skimming. Work from the exact `git log` output you collected and classify every commit, one line at a time.

First, write out a classification table with one row per commit, in this exact format:

```
| # | type | user-facing? | subject |
|---|------|--------------|---------|
| 1 | feat | yes | add hash-id component to snap |
```

Rules for the `user-facing?` column — include only commits that a Landscape administrator would notice. Commit subjects follow conventional commits by convention but this is not enforced, so tolerate free-form subjects and judge those by their content:

- **yes**: `feat:`, `fix:`, `perf:`, and any breaking change (a `!` before the colon, e.g. `feat!:` / `fix!:`).
- **maybe**: `refactor:` commits, only if they are user-facing.
- **no**: `chore:`, `ci:`, `test:`, `build:`, `docs:`, and `style:` commits, plus anything that only touches tooling, tests, or CI, and any dependency bumps.

After the table, write the count: `Included: N of M`. `N` is the number of rows marked `yes` (count a `maybe` as included only if you marked it yes). If the table has fewer rows than the number of lines in your `git log` output, you dropped commits — redo the table until every commit appears.

**Only if `N` is 0** may you skip drafting a new changelog section and opening a pull request. In that case do not stay silent: find the open changelog pull request in this repository (label `changelog`) and post a comment on it containing the full classification table and the line `Included: 0 of M`, so a human can see exactly which commits were considered and why nothing was drafted. If no open changelog PR exists, report the table in your run summary instead.

If `N` is 1 or more, you MUST continue and open a pull request. Never stop just because most commits are chores; a single `feat:` or `fix:` means proceed.

Breaking changes are not always flagged in the subject. For every selected commit, also inspect the full commit message with `git show -s --format=%B <sha>` and look for `BREAKING CHANGE:` or `BREAKING-CHANGE:` footers. A commit carrying such a footer is a breaking change even when its subject has no `!` — classify it as breaking and note the breaking nature in the docs island bullet.

Add each selected commit's subject line, including its type prefix and any `(LP: #NNNN)` Launchpad references, but excluding any `(scope)` after the type prefix. Proofread and correct any obvious spelling or grammar errors. Remove Jira issue references such as `LNDENG-5678` wherever they occur in the subject. Do **not** strip `(LP: #NNNN)` references, and do **not** strip the trailing ` (#1234)` pull request reference — it is the coverage marker later runs deduplicate against.

## Sort the commits

Group the commits by their type. Adhere to the following order.

- Breaking changes (a `!` in the subject or a `BREAKING CHANGE:`/`BREAKING-CHANGE:` footer)
- `feat`
- `fix`
- `perf`
- `refactor`

Within each group, maintain the relative ordering of the commits.

## Compose the new section

Charm releases are Charmhub revisions assigned at publish time, not versions declared in this repository, so changelog sections are dated rather than versioned.

If `CHANGELOG.md` does not exist yet, create it with exactly this header:

```
# Changelog

All notable user-facing changes to this charm are documented here, newest first.
```

Insert a new dated section directly under that header, before any existing sections, so the file stays newest-first:

```
## YYYY-MM-DD

### Features

- <subject> (#1234)
```

- Use `date +%F` for the section date (e.g. `2026-11-10`).
- Group entries under headings in this order: `### Breaking changes`, `### Features`, `### Fixes`, `### Performance`, `### Refactors`. Omit headings that have no entries.
- Each entry is a bullet beginning with `- ` and ending with the ` (#1234)` pull request reference described above.

## Review existing changelog entries

If `CHANGELOG.md` exists, review the entries in its newest sections (those added within the window) and the commits in the window for mismatches. Do NOT edit or delete anything. List every finding under a `Changelog entries to review` heading in the pull request body, one bullet per entry or commit with the reason and evidence; if no pull request is opened, report them in your run summary instead. Flagged items are excluded from the docs release-notes island.

- **User-facing commit with no pull request reference**: a commit skipped by the ` (#1234)` rule above that matches "Select user-facing commits". Propose the entry it would have gotten.
- **Existing changelog entry with no user impact**: find the commit behind the entry's `(#<N>)` reference (`git log --no-merges --grep="(#<N>)" -1 --format=%H`) and check `git show --name-only --format= <commit>`. Flag it if every touched path is a test (`tests/`), CI/tooling (`.github/`, `Makefile`, lint or build configuration), or docs file.
- **Fix for an unreleased regression**: for a selected `fix:` commit F, take the hunks F modified (`git show -U0 F`) and run `git blame --porcelain -L <start>,<end> F^ -- <file>` to find the commits that introduced those lines. If they all fall inside the window, the bug was never in a stable channel — do not add a changelog entry for it and flag the commit instead. A PR title or body that references another PR merged in the same window is supporting evidence. If blame results are mixed, still write the entry but flag it as "possible", not certain.

## Open the pull request

Before making any edit, create your working branch from the default branch, not from the ref this run was dispatched on: `git checkout -b <branch> origin/main`. The dispatch ref may be a bot or feature branch carrying automation commits; if you commit on top of it, the resulting PR contains all of those commits. The PR must contain ONLY the changelog commit on top of `origin/main`.

First check for an existing open changelog pull request in this repository (label `changelog`). Exactly one should ever be open at a time.

- If one is already open, do NOT open a second. Check whether its newest section already covers the commits you selected (the ` (#1234)` references). If it does, make no changes and note that the existing PR is already current. If it is missing any of your selected commits, post the missing entries as a comment on that PR with `add_comment` and stop; there is no handler that pushes to an existing own-repo pull request branch, so do not claim you updated it and do not open a second PR.
- Only if no changelog PR is open, open one with the change. Use a title like `changelog update for <date>` (the `build: ` prefix is added automatically) and a body that lists the included commits, notes that only user-facing changes were selected, and includes the `Changelog entries to review` heading when anything was flagged.

Never open a second changelog PR while one is open.

## Docs release notes (shared PR)

You also maintain one island of the user-facing release notes for the in-development Landscape release, drafted in ONE SHARED pull request against `canonical/landscape-documentation`. You own exactly one marked section (island) in that PR and NEVER open separate per-repo docs PRs beyond the shared one.

The shared docs PR (title `build: draft release notes for Landscape YY.MM`, labels `changelog` + `automation`, based on `develop`, the release-notes page pre-seeded with all eight islands) is created weekly — Fridays around 12:00 UTC — by the `release-notes-draft` workflow in `canonical/landscape-documentation`. Bots NEVER create it; you only ever push to its head branch.

The docs work happens in the separate `./landscape-documentation` checkout. The hard prohibitions above apply to this repository (landscape-client-charm) only; editing the release-notes page in the docs checkout is expected and allowed.

Your island content derives from ALL user-facing commits selected in the window above — not only the ones that produced new changelog entries: a PR whose entry already existed still belongs in the release notes. Rewrite those selected commits into docs-appropriate bullets, excluding any entries flagged in `Changelog entries to review`.

1. Determine the in-development Landscape release `YY.MM`: Landscape releases are `YY.04` and `YY.10`. From today's date, the in-development release is the next boundary strictly in the future (before April -> `YY.04`; before October -> `YY.10`; otherwise `(YY+1).04`). As of November 2026 this is **27.04** (betas are underway), so the target page is `docs/reference/release-notes/27.04-release-notes.md`.
2. Find the shared docs PR: use the GitHub tools to list open PRs in `canonical/landscape-documentation` with labels `changelog` and `automation` (title starts with `build: `). If one exists, this is THE docs PR — use it regardless of which `YY.MM` page it targets.
3. If a docs PR exists:
   - In the `./landscape-documentation` checkout, check out that PR's head branch (branches were fetched via the wildcard fetch at setup: `git checkout <branch>`). If the branch is not among the fetched refs, skip the docs work for this run — the agent has no credentials to fetch it.
   - Other Landscape bots push to this same shared branch, so the checkout (fetched at setup time) may be stale and another bot may push while you work. The agent environment has NO git credentials — `git fetch` fails here, so NEVER attempt to fetch or rebase. Work from the PR branch ref as fetched at setup; keep every edit strictly inside your own island markers so it applies cleanly over the other bots' island updates. `push_to_pull_request_branch` applies your patch to the branch's live tip in a credentialed job; if a concurrent push made it conflict, the delivery fails safely — do not retry, the next run redoes the work. NEVER force-push over another bot's commits.
   - Update ONLY your own island (see below). If the release-notes page for the in-development version does not exist on that branch, create it from the template below and add it to the toctree in `docs/reference/release-notes/index.md` (newest non-LTS page goes directly under the LTS entries, matching existing ordering).
   - Commit in the docs checkout and deliver via `push_to_pull_request_branch` with `pull_request_number` set to the docs PR number.
4. If no open docs PR with labels `changelog` + `automation` exists when you run, skip the docs work for this run entirely: make no changes in the `./landscape-documentation` checkout and NEVER create the shared PR — it is created centrally by the `release-notes-draft` workflow (see above), so a missing PR means the next weekly creation will cover the gap. (Your in-repo changelog work above is unaffected.)
5. If you have no user-facing changes this cycle, leave your island untouched; do not create or update the docs PR. (Your own-repo changelog PR logic is unaffected.)

### Your island

Island markers are HTML comments, invisible in rendered docs. Your slug is `landscape-client-charm` and your display name is `Landscape Client charm`. The full island set, in this fixed order:

1. `landscape-server` — Landscape Server
2. `landscape-client` — Landscape Client
3. `landscape-ui` — Web portal
4. `landscape-go` — Snaps (debarchive, outbox, task-handler services)
5. `landscape-server-operator` — Landscape Server charm
6. `landscape-client-charm` — Landscape Client charm
7. `landscape-debarchive-operator` — Debarchive charm
8. `landscape-task-handler-operator` — Task handler charm

Island format inside `## Additional updates`:

```markdown
<!-- changelog-bot:landscape-client-charm:start -->
### Landscape Client charm

#### Features
- <docs-appropriate bullet>

#### Fixes
- <docs-appropriate bullet>
<!-- changelog-bot:landscape-client-charm:end -->
```

Rules:

- BEFORE writing, read `.github/instructions/release-notes.instructions.md` and `.github/instructions/documentation.instructions.md` in the docs checkout and follow them (applicability: state self-hosted vs SaaS and the component; factual, non-promotional; do not invent benefits, migration steps, or breaking-change claims).
- Rewrite your selected commits into user-facing docs bullets: no conventional-commit prefixes, no `(scope)`, no `(#1234)`, no Jira keys. Keep Launchpad references but format them as `[LP: #NNNN](https://bugs.launchpad.net/landscape/+bug/NNNN)`. Write in the style of existing release notes (see any `docs/reference/release-notes/*.md` page): complete sentences, no trailing period inconsistency, admin-focused.
- Omit `#### Features` / `#### Fixes` when empty. If your island has no entries yet, put `(pending)` as the island body.
- NEVER edit outside your own island markers. Other sections belong to other bots.
- When creating the page, include ALL eight islands in the fixed order above (yours filled, the rest with `(pending)`), so later bot runs find their markers.

### New-page template

File `docs/reference/release-notes/YY.MM-release-notes.md` (dots in the filename, matching `26.10-release-notes.md`; anchor uses dashes):

```markdown
---
myst:
  html_meta:
    description: "Release notes for Landscape YY.MM. Draft; updated automatically until release."
---

(reference-release-notes-YY-MM)=
# YY.MM release notes

> Landscape Server YY.MM runs on Ubuntu 26.04 LTS Resolute Raccoon, 24.04 LTS Noble Numbat, or 22.04 LTS Jammy Jellyfish.

```{note}
This page is a draft maintained automatically until release.
```

## Highlights

(pending - written at release time)

## Additional updates

(eight islands here)

## Upgrade

(pending - written at release time)

## Bug fixes

(pending - written at release time)
```
