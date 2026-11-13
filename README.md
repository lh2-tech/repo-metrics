# repo-metrics

Due-diligence metrics for a list of git repositories (GitHub, GitLab, Bitbucket
or local clones), plus an optional sample of each repo's oldest commits.

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

`repo_metrics.csv`, one row per repo:

| Repo | Logical LOC (excludes blanks & comments) | Non-Authored LOC (auto-gen + vendored + deps) | Duplication Ratio (0.00-1.00) | Non-Merge Commit Count | Builds & Tests |
|---|---|---|---|---|---|
| lh2-tech/codebase-profiler | 14221 | 1 | 0.0188 | 69 | Build + Test present |
| lh2-tech/repo-metrics | 2239 | 0 | 0.0075 | 5 | Build + Test present |
| owner/missing-repo | | | | | FAILED: repository not found |

`Builds & Tests` reads `Build + Test present` when the repo has a Dockerfile and `Not verified` otherwise.

Also written: `repo_metrics_detailed.csv` (every measured column), `repo_metrics.jsonl`, `summary.json` and `artifacts/`.
With `run_metrics_and_sample.py` you also get `samples/<repo>/` (optionally `.zip`), `sample_report.csv` and `run_report.json`.
