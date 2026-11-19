# mo3ta-coverage GitHub Action

Run only the tests that cover changed files in your PR and enforce a
changed-line coverage threshold. Fails the build when coverage on your
changed lines is below the limit.

```yaml
permissions:
  contents: read

jobs:
  changed-line-coverage:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0   # full history needed for merge-base diff

      - name: Install dependencies
        run: npm ci

      - name: Check changed-line coverage
        uses: MahmoudMabrok/mo3ta-coverage-action@v1
        with:
          mode: smart     # fast | smart (default) | full
          limit: 80
```

When the job runs on GitHub the action appends a markdown coverage table to
the PR's "Summary" automatically — no extra steps required.

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `base` | Base branch/revision to diff against. Empty = auto-resolve: `pull_request` → `origin/<PR base branch>`, `push` → previous commit (`HEAD~1`, fallback `origin/main`), other events → `origin/main` | *(auto)* |
| `mode` | `fast` (naming), `smart` (naming + reverse imports, default), `full` (`jest --findRelatedTests`) | `smart` |
| `depth` | Max reverse-import depth for `smart` mode (1 = direct importers, 2 = also importers of importers) | `2` |
| `limit` | Minimum changed-line coverage percentage | `80` |
| `ci` | Clean console output + markdown table file (recommended for CI) | `true` |
| `ci-report-path` | Where to write the markdown report, relative to `working-directory` | `coverage/mo3ta-coverage-report.md` |
| `show-covered` | Also print covered changed lines in the report | `false` |
| `files` | Explicit list of files to scan (space- or comma-separated). Skips auto-detection. | — |
| `files-from` | Path to a newline-separated file list to scan. Skips auto-detection. | — |
| `report-to-summary` | Append the markdown table to `$GITHUB_STEP_SUMMARY` automatically | `true` |
| `working-directory` | Working directory for the CLI (for monorepos / sub-project checkouts) | — |

## How it works

1. Runs on **Node 20**.
2. Invokes `node <action>/dist/index.js` with the inputs mapped to the
   published CLI's flags.
3. The CLI finds files changed against `base` (all authors, via
   `merge-base`), maps each changed file to its related Jest test files
   using naming conventions and a depth-limited reverse-import walk,
   runs Jest with `--coverage`, parses `coverage/lcov.info`, and computes
   coverage on the changed lines only.
4. Exits non-zero when tests fail **or** when changed-line coverage is
   below `limit`.
5. When `report-to-summary: true` and `GITHUB_STEP_SUMMARY` is set, the
   action appends the markdown table to the job summary.

## Auto-resolved base

By default the action figures out what to diff against, based on the
triggering event:

| Event | Resolved base | Notes |
|-------|---------------|-------|
| `pull_request` | `origin/<PR base branch>` | uses `github.base_ref`; needs the branch fetched (default checkout does this, or use `fetch-depth: 0`) |
| `push` | `HEAD~1` (fallback `origin/main`) | changed lines of the push, regardless of branch |
| anything else | `origin/main` | schedule, dispatch, … |

To override, pass `base` explicitly — any git ref that the checkout
resolves works (e.g. `base: HEAD~1`, `base: origin/develop`, `base: v1.0.0`).

```yaml
- uses: MahmoudMabrok/mo3ta-coverage-action@v1
  with:
    base: origin/develop
    limit: 70
```

## Pin to a specific version

The npm package version is the source of truth. Currently the action repo
tracks the CLI at `v1` = `1.0.24`. Pin to a semver, tag, or commit — see
[the tag list](https://github.com/MahmoudMabrok/mo3ta-coverage-action/tags)
once tags are cut.

```yaml
uses: MahmoudMabrok/mo3ta-coverage-action@1.0.24
```

## Related

- [mo3ta-coverage (npm)](https://www.npmjs.com/package/mo3ta-coverage) —
  the underlying CLI.
- [Full CLI docs](https://github.com/MahmoudMabrok/mo3ta-coverage#cli-options)
  in the main repo.

## License

MIT — see [LICENSE](LICENSE).
