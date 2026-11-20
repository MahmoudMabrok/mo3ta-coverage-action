# mo3ta-coverage GitHub Action

Run only the tests that cover changed files in your PR and enforce a
changed-line coverage threshold. Fails the build when coverage on your
changed lines is below the limit.

```yaml
permissions:
  contents: read
  pull-requests: write   # needed for the default PR report comment

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
the job "Summary" automatically, and (by default) posts the same report as
a single updatable comment on the PR — re-runs replace the previous report
instead of stacking new comments. No extra steps required.

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `base` | Base branch/revision to diff against. Empty = auto-resolve: `pull_request` → the PR's exact base commit, `push` → previous commit (`HEAD~1`, fallback `origin/main`), other events → `origin/main` | *(auto)* |
| `mode` | `fast` (naming), `smart` (naming + reverse imports, default), `full` (`jest --findRelatedTests`) | `smart` |
| `depth` | Max reverse-import depth for `smart` mode (1 = direct importers, 2 = also importers of importers) | `2` |
| `limit` | Minimum changed-line coverage percentage | `80` |
| `ci` | Clean console output + markdown table file (recommended for CI) | `true` |
| `ci-report-path` | Where to write the markdown report, relative to `working-directory` | `coverage/mo3ta-coverage-report.md` |
| `show-covered` | Also print covered changed lines in the report | `false` |
| `files` | Explicit list of files to scan (space- or comma-separated). Skips auto-detection. | — |
| `files-from` | Path to a newline-separated file list to scan. Skips auto-detection. | — |
| `report-to-summary` | Append the markdown table to `$GITHUB_STEP_SUMMARY` automatically | `true` |
| `comment-on-pr` | Post the report as a **single, updatable comment** on the PR (previous report comment is updated in place on re-runs). Requires `pull-requests: write` | `true` |
| `comment-header` | Heading the action looks for when finding the previous comment to update | `mo3ta-coverage report` |
| `working-directory` | Working directory for the CLI (for monorepos / sub-project checkouts) | — |
| `cli-repository` | GitHub repo (owner/repo) that hosts the mo3ta-coverage CLI — fetched from GitHub at run time, not npm | `MahmoudMabrok/mo3ta-coverage` |
| `cli-ref` | Branch, tag, or commit of the CLI repo to use. Empty = default branch = always latest | *(latest)* |

## How it works

1. Runs on **Node 20**.
2. Fetches the mo3ta-coverage CLI **from GitHub, not npm**: a shallow
   clone of `cli-repository` at `cli-ref` (default: the repo's default
   branch, i.e. always the latest version), then `npm ci` of its
   committed `package-lock.json` into a runner temp dir — the checkout's
   own `package.json`/lock files are never touched.
3. Invokes `node <cli>/index.js` with the inputs mapped to the CLI's
   flags.
4. The CLI finds files changed against `base` (all authors, via
   `merge-base`), maps each changed file to its related Jest test files
   using naming conventions and a depth-limited reverse-import walk,
   runs Jest with `--coverage`, parses `coverage/lcov.info`, and computes
   coverage on the changed lines only.
5. Exits non-zero when tests fail **or** when changed-line coverage is
   below `limit`.
6. When `report-to-summary: true` and `GITHUB_STEP_SUMMARY` is set, the
   action appends the markdown table to the job summary.
7. When `comment-on-pr: true` (default) and the workflow ran on a
   `pull_request` with a token that has `pull-requests: write`, the action
   posts the report as a PR comment. It searches the PR's existing comments
   for one containing the `comment-header` text: if found, that comment is
   updated in place; otherwise a new comment is created. On non-PR events
   (push, schedule, …) the step skips with a notice.

## Auto-resolved base

By default the action figures out what to diff against, based on the
triggering event:

| Event | Resolved base | Notes |
|-------|---------------|-------|
| `pull_request` | the PR's exact base commit | from `github.event.pull_request.base.sha` — stable even if the base branch moves while the check runs. Like any diff-against-base check it needs the history present in the checkout; use `fetch-depth: 0` in the checkout step (the example below does) |
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

## Pin to a specific CLI version

By default the action always tracks the **latest** CLI: it fetches the
default branch of `MahmoudMabrok/mo3ta-coverage` on every run.

To pin a specific CLI version, point `cli-ref` at a tag, branch, or
commit of the [CLI repository](https://github.com/MahmoudMabrok/mo3ta-coverage):

```yaml
- uses: MahmoudMabrok/mo3ta-coverage-action@main
  with:
    cli-ref: v1.1.0   # any tag / branch / commit of the CLI repo
```

## Related

- [mo3ta-coverage](https://github.com/MahmoudMabrok/mo3ta-coverage) —
  the underlying CLI (fetched from this GitHub repo at run time).
- [Full CLI docs](https://github.com/MahmoudMabrok/mo3ta-coverage#cli-options)
  in the main repo.

## License

MIT — see [LICENSE](LICENSE).
