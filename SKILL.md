---
name: repo-metrics
description: >-
  Collect due-diligence metrics across a list of GitHub / GitLab / local git
  repos: logical LOC, non-authored LOC, jscpd duplication ratio, non-merge
  commit count, and build/test verification. Use when asked to measure a repo
  list, produce a due-diligence scorecard, or gather LOC/duplication/commit
  evidence across many repositories.
disable-model-invocation: true
---

# Repo Metrics (repo list → due-diligence scorecard)

Produces `repo_metrics.csv`, `repo_metrics.jsonl`, `summary.json`, and a
per-repo JSON artifact for every repository in a list.

## Metrics

| # | Column | How it is measured |
|---|--------|--------------------|
| 1 | `logical_loc` | `scc --format json`, summed `Code` column, whole repo |
| 2 | `non_authored_loc` | `logical_loc − authored_loc`, where `authored_loc` re-runs scc with `--exclude-dir` for each vendored directory plus `--no-gen --no-min-gen` |
| 3 | `duplication_ratio` | jscpd `statistics.total.percentage / 100` |
| 4 | `non_merge_commits` | `git log <branch> --no-merges`, subjects matching `\brevert\b` dropped |
| 5 | `build_test_status` | `Builds + test suite passes` / `Builds only` / `Not verified`, plus `build_command` and `runtime_version` |

Metric 2 also emits its intermediate figures — `vendored_loc` and
`generated_minified_loc` — so the subtraction can be audited rather than taken
on trust.

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| `git` | required |
| `scc` | found via `SCC_BIN`, `PATH`, or `.tools/bin/scc` searched upward from cwd |
| `jscpd` | needs Node. Without it the run falls back to a built-in detector and labels every row `duplication_engine=builtin-fallback` |
| Tokens | `key=value` file found via `DATALABS_TOKENS`, or a `tokens` file searched upward from cwd. Env `GH_TOKEN` / `GITHUB_TOKEN` / `GITLAB_TOKEN` also work |
| Docker | only for `--build-check docker`; the run downgrades to `detect` if the daemon is unreachable |

Installing Node and jscpd without touching the system:

```bash
curl -fsSL https://nodejs.org/dist/v22.11.0/node-v22.11.0-darwin-arm64.tar.gz -o /tmp/node.tar.gz
tar -xzf /tmp/node.tar.gz -C .tools
export PATH="$PWD/.tools/node-v22.11.0-darwin-arm64/bin:$PATH"
npm install -g jscpd
```

## Usage

```bash
python3 scripts/repo_metrics.py --repos repos.txt --out outputs/run1 --workers 8
```

GitLab list where lines are bare `group/repo` paths:

```bash
python3 scripts/repo_metrics.py --repos gitlab_repos.txt --default-platform gitlab \
  --out outputs/gitlab --workers 8
```

Self-hosted GitLab: add `--gitlab-host gitlab.mycorp.com`.

### Input formats

One entry per line; blank lines and `#` comments ignored. Trailing annotations
like `group/repo  [private]` are stripped, so listing exports work unmodified.

```
owner/repo                          # GitHub (or GitLab with --default-platform gitlab)
https://github.com/owner/repo
https://gitlab.com/group/sub/repo
git@github.com:owner/repo.git
/abs/path/to/local/clone
```

`.csv` (columns `full_name` / `url` / `repo_url` / `path`), `.json`, and
`.jsonl` inputs are also accepted.

### Useful flags

| Flag | Effect |
|------|--------|
| `--build-check none\|detect\|docker` | `detect` (default) identifies build system, command and runtime version statically; `docker` attempts a real build and test run |
| `--branch-mode latest` | measure the branch holding the newest commit instead of the default branch |
| `--exclude-dirs a,b,c` | override the metric-2 exclusion list (default `vendor,node_modules,dist,build,generated,migrations`) |
| `--wide-exclude` | also exclude `third_party`, `Pods`, `target`, `.next`, `coverage`, … |
| `--delete-clones` | remove freshly cloned repos after measuring, to bound disk |
| `--force` | re-measure repos that already have an artifact |
| `--no-jscpd` | skip jscpd and use the built-in detector deliberately |

Runs are **resumable**: every repo writes `artifacts/<name>.json` and is skipped
on the next run unless `--force` is passed. Failures are recorded per repo and
never abort the batch.

## Known measurement hazards

These are real findings from a production run, not hypotheticals.

**Data files inflate LOC.** `scc` counts XML, JSON and SQL as code. On a
CodeIgniter corpus, `sitemaps/*.xml` and `log/capistrano.log` produced 23M
"LOC" for a repo holding ~115K lines of PHP — a ~200× overstatement, with
XML at 99.2% of the total. Always read `primary_language` and
`language_breakdown` before trusting `logical_loc`; if the primary language is
XML or JSON, extend `--exclude-dirs`.

**The duplication fallback is not jscpd.** On the same repo jscpd reported
2.58% and the built-in detector reported 0.0%. The fallback exists so a run
completes without Node, not so the numbers are interchangeable. Filter on
`duplication_engine` before reporting.

**jscpd 5 is a Rust rewrite.** Its CLI differs from the documented v3/v4:
`--silent` is a reporter rather than a flag, and `.gitignore` is honoured by
default with `--no-gitignore` to disable. The script targets v5.

**Runtime version is frequently absent.** Metric 5 requires a runtime version,
but legacy repos often pin none — no `.nvmrc`, no `engines.node`, no
`require.php`, no `.python-version`. When `runtime_version` is empty the field
has to come from vendor attestation; no tool can extract what is not there.

## Output columns

`repo`, `platform`, `analysis_branch`, `logical_loc`, `non_authored_loc`,
`non_authored_pct`, `authored_loc`, `vendored_loc`, `generated_minified_loc`,
`duplication_ratio`, `duplication_pct`, `duplication_engine`,
`non_merge_commits`, `revert_commits_excluded`, `build_test_status`,
`build_system`, `build_command`, `test_command`, `runtime_version`,
`runtime_version_source`, `ci_runs_tests`, `dockerfile_present`,
`primary_language`, `language_breakdown`, `contributors`, `first_commit`,
`last_commit`, `measurement_seconds`, `status`, `error`.
