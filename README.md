# workflows

Reusable GitHub Actions workflows shared across MikeSiLVO's Kodi add-ons.

Call from a consumer repo:

```yaml
jobs:
  triage:
    permissions: { issues: write }
    uses: MikeSiLVO/workflows/.github/workflows/issue-triage.yml@v1
    secrets: inherit
```

Consumers pin the `@v1` tag. The tag moves forward as the workflows change, so a
mid-edit push to `main` never rewrites live CI.

Available:

- `issue-triage.yml` — labels bug reports `needs-info` until a debug log link is
  posted (body or a human comment), and clears it when one arrives.
- `stale-issues.yml` — closes `needs-info` issues that go quiet.
