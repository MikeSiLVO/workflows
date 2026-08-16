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
| `checks.yml` | `ruff check` and `pyright` | `deps` for extra pip packages, `kodistubs` version (default `21.0.0`) |
| `addon-checker.yml` | Runs `kodi-addon-checker`. Copies the add-on into a folder named its id first, so the id check passes | `kodi_branch`, the Kodi branch to check against (default `piers`) |
| `issue-triage.yml` | Labels a bug report `needs-info` until a log link is posted in the body or a human comment, then clears it | `issue_number`. Job needs `issues: write` |
| `needs-info.yml` | Comments a nudge when the `needs-info` label is added | `message` (required), `label` (default `needs-info`). Job needs `issues: write` |
| `stale-issues.yml` | Closes `needs-info` issues that go quiet | `only_labels`, `days_before_stale`, `days_before_close`, `stale_message`, `close_message`. Job needs `issues: write` |
| `make-release.yml` | Zips the add-on with dev files removed and makes a GitHub release from the `addon.xml` version and `<news>` | Job needs `contents: write` |
| `deploy-to-repo.yml` | Packages the add-on into `<repo_target>/<channel>/<id>/` and rebuilds `addons.xml`. For skins it packs `media/` into a bundle first, see **Skin texture packing** | `channel` (default `piers`), `repo_target` (default `repository.silvo`). Secret `DEPLOY_TOKEN`. The target repo must contain `_repo_generator.py` |
| `close-translation-prs.yml` | Closes PRs that only change `strings.po` from non-Weblate authors, except `en_gb` | `pr_number`. Job needs `issues: write` and `pull-requests: write` |
| `sync-addon-metadata-translations.yml` | Runs xbmc's `sync_addon_metadata_translations` and opens a PR with the result | Job needs `contents: write` and `pull-requests: write` |

Not reusable:

- `sync-templates.yml` renders the issue templates and pushes them to every repo in
  `targets.json`. It runs when `templates/**` or `targets.json` change, or on manual dispatch
  with an optional `only=<repo>`. See **Issue templates**.
- `lint-canary.yml` runs every consumer against unpinned ruff and pyright each week. When a
  newer release is out it opens an issue saying whether the bump is safe; the pins in
  `checks.yml` are edited by hand, since no token can write `.github/workflows` without
  becoming a CI-rewrite credential for every repo.
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
- `secrets: inherit` passes the repo's own secrets, like `DEPLOY_TOKEN`, through.
- Any needed permissions are set on the calling job in the stub.
- The `checks` and `addon-checker` stubs add `paths-ignore: ['**/*.po']` so translation-only changes (Weblate updates, metadata sync) do not trigger them.
- A reusable cannot declare its own `on:` triggers. They belong to the stub.
- `workflow_dispatch` inputs do not reach the reusable on their own. Forward them with `with:`.
- `GITHUB_TOKEN` cannot write to another repo, which is why `deploy-to-repo` takes a PAT.
- `checks` runs without the repo's local environment. An add-on importing pip packages must list
  them in `deps`, or pyright reports missing imports that pass locally.

Ready-to-copy sets live in `examples/addon/` and `examples/skin/`. Copy one into a repo's
`.github/workflows/` and replace `OWNER/REPO`.

### Which stubs by kind

- **Add-on (8):** `checks`, `addon-checker`, `issue-triage`, `stale-issues`, `make-release`,
  `deploy-to-repo`, `close-translation-prs`, `sync-addon-metadata-translations`.
- **Skin (6):** `needs-info`, `stale-issues`, `make-release`, `deploy-to-repo`,
  `close-translation-prs`, `sync-addon-metadata-translations`.

A skin has no Python, so it drops `checks`. Its bugs are visual rather than log-based, so it uses
`needs-info` in place of `issue-triage`, which exists to chase a log link. `addon-checker` validates
skins too and can be added, but a skin depending on a beta of another add-on will fail it on the
version check until that dependency ships.

---

## Skin texture packing

`deploy-to-repo.yml` packs a skin's `media/` into `media/Textures.xbt` and deletes the loose image
files, so the zip ships the bundle and nothing else. A `themes/<name>/` folder becomes
`media/<name>.xbt`. Add-ons are untouched; skins are detected from `addon.xml`.

This is what `xbmc/repository-generator` does when it builds the official Kodi repo, so a zip from
here matches a released one. Kodi reads the bundle before loose files, so there is no reason to
ship both.

The packer is built from xbmc's `Omega` branch and cached between runs. Keep it there. Newer
branches write a format Kodi before v22 cannot open, and the skin then has no textures at all.

Two things to expect:

- The zip does not get smaller. PNGs are already compressed and the bundle is not. The gain is at
  runtime, where Kodi unpacks one bundle instead of decoding every file.
- `background="true"` no longer defers a bundled texture, since the bundle is checked before the
  background loader.

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
4. Once per repo, add a `DEPLOY_TOKEN` secret for deploy, and turn on Settings > Actions >
   General > "Allow GitHub Actions to create and approve pull requests" for the translation sync.

Skins are detected from `addon.xml`, so there is no `kind` field.

---

## Secrets

Two personal access tokens, plus the automatic one.

| Secret | Lives in | Grants | Used by |
|---|---|---|---|
| `SYNC_TOKEN` | this repo | Contents write on every repo in `targets.json` | `sync-templates` to push rendered templates, `lint-canary` to read consumer files |
| `DEPLOY_TOKEN` | each add-on repo | Contents write on the deploy target repo | `deploy-to-repo` to commit the packaged zip |
| `GITHUB_TOKEN` | automatic | whatever the stub's `permissions:` block grants | everything else |

Secrets live in GitHub Actions settings, never in the repo.

### Creating one

Both are fine-grained tokens. Settings > Developer settings > Personal access tokens >
Fine-grained tokens > Generate new token.

| Field | `SYNC_TOKEN` | `DEPLOY_TOKEN` |
|---|---|---|
| Resource owner | the account owning the repos | the same account |
| Repository access | Only select repositories | Only select repositories |
| Select | every repo in `targets.json` | the deploy target repo only |
| Permissions > Contents | Read and write | Read and write |

Contents is the only permission either needs. Metadata read is added automatically and cannot be
removed. Do not grant Actions or Workflows. Nothing here writes to `.github/workflows`, and
withholding it means a leaked token cannot rewrite CI.

The token's own name is a label and is referenced nowhere. The secret name is the one that has to
be exact.

A token that lapses breaks the deploy that day, and the failure reads `fatal: Authentication
failed`, which does not look like an expiry. Either set no expiration or record the renewal date
somewhere visible.

### Storing one

Repo > Settings > Secrets and variables > Actions > New repository secret.

Stubs pass secrets down with `secrets: inherit`, so a reusable sees whatever the calling repo
holds. A missing or expired token in one repo does not affect the others.

Values cannot be read back, but names and dates can:

```
gh api repos/OWNER/REPO/actions/secrets --jq '.secrets[] | "\(.name)  \(.updated_at[:10])"'
```

---

## Versioning

Stubs pin `@v1`. After editing a reusable, move that tag to the new commit and add a point tag.

```
git tag v1.2.0
git tag -f v1
git push origin v1.2.0 && git push -f origin v1
```

`v1` floats so no stub needs editing. The point tag does not move, so a repo can be pinned to a
known good version if a change breaks its deploy.

`v2` is for a change consumers have to adapt to. Anything else stays on `v1`, since a new major
means editing the `uses:` line in every stub in every repo.

This repo has to be public, because the consumers are.

A private repo's reusables reach only the owner's **private** repos. Setting access to
"Accessible from repositories owned by the user" does not extend them to the owner's *public*
repos. A public consumer calling a private reusable fails at zero seconds with a startup failure
and no error naming the cause, so it reads like a broken workflow rather than a permissions rule.

Being public is safe here. The repo holds only CI configuration; secrets are Actions secrets and
never live in files.

---

## Adapting it

This is set up for one specific account. To run the same pattern on another:

1. Fork this repo and keep it public, or copy it privately and accept that only repos under the
   same account will be able to call it.
2. Create the deploy target repo, one directory per channel, containing `_repo_generator.py`.
   `deploy-to-repo` runs that script inside the target to rebuild `addons.xml`, and fails at the
   last step without it.
3. Create `SYNC_TOKEN` here, scoped to the repos listed in `targets.json`.
4. For each add-on repo, copy a stub set from `examples/`, add a `targets.json` row, and add
   `DEPLOY_TOKEN`.
5. Tag `v1` so the stubs resolve.

The reusables take the account from `github.repository_owner`, so nothing here needs renaming. The
account appears only in each stub's `if: github.repository ==` guard, which is what stops a fork
from running against the original repos. Set it before enabling Actions.
