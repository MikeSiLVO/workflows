# workflows

Reusable GitHub Actions workflows and issue templates shared across a set of Kodi add-ons and
skins. The CI is written here once. Each add-on or skin repo carries only small stub workflows
that call these. It is set up for one specific group of repos, but the layout is a general
pattern.

- **Add-ons** (`script.*`, `plugin.*`) use the full set.
- **Skins** (`skin.*`) use a smaller set. A skin has no Python and its bugs are visual rather
  than log-based, so it skips the checks and triage.

---

## Reusable workflows

Called from a repo with `uses: MikeSiLVO/workflows/.github/workflows/<name>.yml@v1`.

| Workflow | What it does | Inputs and secrets |
|---|---|---|
| `checks.yml` | `ruff check` and `pyright` | none |
| `addon-checker.yml` | Runs `kodi-addon-checker`. Copies the add-on into a folder named its id first, so the id check passes | `kodi_branch`, the Kodi branch to check against (default `piers`) |
| `issue-triage.yml` | Labels a bug report `needs-info` until a log link is posted in the body or a human comment, then clears it | `issue_number`. Job needs `issues: write` |
| `stale-issues.yml` | Closes `needs-info` issues that go quiet | Job needs `issues: write` |
| `make-release.yml` | Zips the add-on with dev files removed and makes a GitHub release from the `addon.xml` version and `<news>` | Job needs `contents: write` |
| `deploy-to-repo.yml` | Packages the add-on into `<repo_target>/<channel>/<id>/` and rebuilds `addons.xml` | `channel` (default `piers`), `repo_target` (default `repository.silvo`). Secret `REPO_SILVO_PAT` |
| `close-translation-prs.yml` | Closes PRs that only change `strings.po` from non-Weblate authors, except `en_gb` | `pr_number`. Job needs `issues: write` and `pull-requests: write` |
| `sync-addon-metadata-translations.yml` | Runs xbmc's `sync_addon_metadata_translations` and opens a PR with the result | Job needs `contents: write` and `pull-requests: write` |

Not reusable:

- `sync-templates.yml` renders the issue templates and pushes them to every repo in
  `targets.json`. It runs when `templates/**` or `targets.json` change, or on manual dispatch
  with an optional `only=<repo>`. See **Issue templates**.
- `.github/dependabot.yml` does weekly grouped action updates.

---

## Using it (stubs)

Each repo has small stub workflows in `.github/workflows/` that set the triggers, guard on the
repo, and call the reusable at `@v1`.

Simplest:

```yaml
name: Code checks
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
jobs:
  checks:
    if: github.repository == 'MikeSiLVO/<repo>'
    uses: MikeSiLVO/workflows/.github/workflows/checks.yml@v1
```

With inputs and secrets:

```yaml
name: Deploy
on:
  push: { tags: ['v*'] }
  workflow_dispatch:
jobs:
  deploy:
    if: github.repository == 'MikeSiLVO/<repo>'
    uses: MikeSiLVO/workflows/.github/workflows/deploy-to-repo.yml@v1
    with:
      channel: piers
    secrets: inherit
```

- Pin `@v1`, not `@main`. An unfinished edit to `main` would otherwise change live CI.
- The `github.repository` guard stops forks from running it.
- `secrets: inherit` passes the repo's own secrets, like `REPO_SILVO_PAT`, through.
- Any needed permissions are set on the calling job in the stub.
- The `checks` and `addon-checker` stubs add `paths-ignore: ['**/*.po']` so translation-only changes (Weblate updates, metadata sync) do not trigger them.

### Which stubs by kind

- **Add-on (8):** `checks`, `addon-checker`, `issue-triage`, `stale-issues`, `make-release`,
  `deploy-to-repo`, `close-translation-prs`, `sync-addon-metadata-translations`.
- **Skin (5):** the same minus `checks`, `issue-triage`, and `stale-issues`.

---

## Issue templates

GitHub issue forms cannot be shared across repos, so each repo needs its own copy. The source
is here. `sync-templates.yml` renders it and pushes a copy to each repo's `.github/ISSUE_TEMPLATE/`.

```
templates/
  common/        used by every repo
    config.yml   contact link, blank issues off
    feature.yml  feature request
  bug/
    addon.yml    bug report for add-ons, log required
    skin.yml     bug report for skins, visual, "where in the skin", log optional
```

For each repo the sync checks it out, picks add-on or skin by looking for `point="xbmc.gui.skin"`
in `addon.xml`, renders `common/*` plus the matching `bug/<kind>.yml`, and pushes only if it
changed. Each placeholder is filled from the `targets.json` row.

- `{{ADDON_NAME}}` from `name`
- `{{FORUM_TID}}` from `tid`
- `{{VERSION_EG}}` from `version_eg`

`{{ADDON_NAME}}` is quoted everywhere it is used, because skin names can contain a colon, like
`Aeon Nox: SiLVO`, that would break the YAML.

---

## Adding a repo

1. Add the stub set for its kind to `.github/workflows/`, guarded to that repo.
2. Add a row to `targets.json`:
   ```json
   { "repo": "script.example", "name": "Example", "tid": "123456", "version_eg": "1.0.0" }
   ```
3. Include the repo in the `SYNC_TOKEN` PAT with Contents write.
4. Once per repo, add a `REPO_SILVO_PAT` secret for deploy, and turn on Settings > Actions >
   General > "Allow GitHub Actions to create and approve pull requests" for the translation sync.

Skins are detected from `addon.xml`, so there is no `kind` field.

---

## Secrets

- `SYNC_TOKEN` (this repo). Fine-grained PAT with Contents write, scoped to the target repos.
  Used only by `sync-templates`.
- `REPO_SILVO_PAT` (each repo). Used by `deploy-to-repo`.
- `GITHUB_TOKEN` is automatic. Stubs grant what each job needs.

Secrets live in GitHub Actions settings, never in the repo.

---

## Versioning

Stubs pin `@v1`. After editing a reusable, move the tag onto the new commit.

```
git tag -f v1 && git push -f origin v1
```

This repo is public, so any repo can call the reusables. A reusable in a private repo can only
be called by my other private repos, which is why this one is public.

---

## Adapting it

This is set up for my repos. To reuse it, change `MikeSiLVO` to your own account in every stub's
`uses:` line, point `deploy-to-repo` at your own target repo with a matching deploy secret, and
create your own `SYNC_TOKEN`. The rest works unchanged.
