# repo-metrics

Due-diligence metrics for a list of git repositories (GitHub, GitLab, Bitbucket
or local clones), plus an optional sample of each repo's oldest commits.

## What it measures

| Column | Meaning |
|--------|---------|
| `logical_loc` | `scc` code lines, whole repo |
| `non_authored_loc` | `logical_loc` minus authored LOC (vendored, generated and minified code removed) |
| `code_loc` / `data_loc` | LOC split between source and data/markup languages |
| `duplication_ratio` | `jscpd` duplication (built-in fallback if `jscpd`/Node is missing; labelled in `duplication_engine`) |
| `non_merge_commits` | `git log --no-merges`, revert commits dropped |
| `build_test_status` | detected build system, command and runtime version; `docker` mode attempts a real build |

Full column list and measurement caveats are in [SKILL.md](SKILL.md).

## Scripts

| Script | Use |
|--------|-----|
| `scripts/repo_metrics.py` | metrics only |
| `scripts/run_metrics_and_sample.py` | metrics **and** a first-N% commit sample per repo (`--pct`) |
| `scripts/make_sample_repo.py` | slice one repo to its oldest N% of commits |
| `scripts/combine_runs.py` | merge several runs into one table |
| `scripts/make_attestation_form.py` | pre-filled vendor attestation form from a run |
| `docker/` | containerised runner, see [docker/README.md](docker/README.md) |

## Requirements

- Python 3.9+ and `git`
- [`scc`](https://github.com/boyter/scc) on `PATH` (or set `SCC_BIN`)
- [`jscpd`](https://github.com/kucherenko/jscpd) with Node, for real duplication numbers
- Optional: a token for private repos (`GH_TOKEN`, `GITLAB_TOKEN`, `BITBUCKET_TOKEN`), in the environment or a `KEY=value` file; Docker for `--build-check docker`

## Usage

```bash
# metrics
python3 scripts/repo_metrics.py --repos repos.txt --out outputs/run1 --workers 8

# metrics + 5% oldest-commit sample (use --full-clone to avoid a second clone)
python3 scripts/run_metrics_and_sample.py --repos repos.txt --out run1 --pct 5 --full-clone
```

`repos.txt` takes one entry per line: `owner/repo`, a GitHub/GitLab URL, an SSH
URL or a local path. `.csv`, `.json` and `.jsonl` lists also work.

Runs are resumable: each repo writes `artifacts/<name>.json` and is skipped
next time unless `--force` is given. A failing repo is recorded, never fatal.

## Output

`repo_metrics.csv`, `repo_metrics.jsonl`, `summary.json`, `artifacts/`. With
`run_metrics_and_sample.py` also `samples/<repo>/` (optionally `.zip`),
`sample_report.csv` and `run_report.json`.

The sample is the oldest N% of commits **by count**, on one branch; other
branches, the reflog and the origin remote are dropped. It is not PII-scrubbed.

## Notes

Some scripts look for a `datalabs_paths.py` in a parent directory to locate
shared env/output/clone folders. Standalone, they fall back to `./clones`,
`--out` and the environment, so that file is not required.
