# bedrock-eval

**The [bedrock](https://github.com/KrishanKVerma/bedrock) browser-agent
reliability study, reimplemented as an [Inspect AI](https://inspect.aisi.org.uk/)
evaluation.**

bedrock's finding is that a perceive-plan-act browser agent can reach a task's
success state, take one more action that undoes it, and report success anyway.
This eval reproduces that finding through an independently built harness.

- **A custom solver, not `basic_agent`.** The mechanism depends on the
  ref-numbering bedrock's perception step assigns to interactive elements on
  each page read. Tool-calling would replace that with the provider's own
  function-calling format, so the original planner contract is ported directly.
- **A four-outcome scorer.** Every run resolves to `pass`, `silent_failure`,
  `honest_failure` or `planner_error`, with a rate metric for each. The harness
  never asks the agent whether it succeeded; it checks the page.
- **Pages pinned inside Docker.** Four tasks served from static copies baked
  into the sandbox image rather than fetched live, so a run today exercises the
  page shape a run next year will. The copies are checked against the live
  sources, but they are copies, so numbers here are not directly comparable to
  the published study.

## The result

Eight runs across two models. All eight reported "done". Three were wrong.
None failed honestly.

One run per task per model, so this establishes that the failure reproduces
through a second, independently built harness. It does not establish how often
it occurs, and no rate should be read off eight runs.

## What the tests caught

Inspect's default epoch reducer (`mean`) runs unconditionally, even at
`epochs=1`, and it converts every `Score.value` to a float first. That silently
turns `"pass"` and `"silent_failure"` into `0.0`, so every rate metric would
have read 0.0 regardless of what the agent actually did. The task sets
`epochs=Epochs(1, reducer=[])` to disable the conversion, which is also the
right semantics for `--epochs N`: bedrock treats N repeated runs as N
independent data points, not N samples to collapse into one.

A metric that cannot report anything but zero looks exactly like a model that
never fails. 43 tests cover the solver, the scorer and this case.

## Run it from a clean clone

Requires [uv](https://docs.astral.sh/uv/) and a running Docker daemon.

```bash
git clone https://github.com/KrishanKVerma/bedrock-eval && cd bedrock-eval
uv sync
uv run inspect eval bedrock_reliability/bedrock_reliability --model openai/gpt-5-nano
```

The first run builds the sandbox image, which takes a few minutes. View logs
with `uv run inspect view`.

```bash
uv run pytest -k "not agentic"                                  # tests
uv run ruff check && uv run ruff format --check && uv run mypy src tests
make check                                                      # all CI checks
```

## Status

In discussion with AISI maintainers about listing on the Inspect register:
[inspect_evals issue #2467](https://github.com/UKGovernmentBEIS/inspect_evals/issues/2467).

## Credits

Forked from
[Generality-Labs/inspect-evals-template](https://github.com/Generality-Labs/inspect-evals-template),
which supplies the CI, the managed-file sync and the Claude Code skills
described below.

The parent study, with its traces, retractions and the original harness, is at
[KrishanKVerma/bedrock](https://github.com/KrishanKVerma/bedrock). Eval-specific
detail, including every deviation from the original methodology, is in
[src/bedrock_reliability/README.md](src/bedrock_reliability/README.md).

---

## Repository layout

```text
src/
  bedrock_reliability/   # the eval: task, solver, scorer, sandbox, static pages
  examples/              # template reference evals, not registered
tests/
  bedrock_reliability/   # 43 tests
  examples/
```

Each evaluation lives in its own directory under `src/`, registered via entry
points in `pyproject.toml`. The `src/examples/` directory holds the template's
reference implementations; they are managed files, do not ship in the wheel,
and should stay in place so the sync keeps working.

## CI workflows

- **Checks** (`checks.yml`) — ruff, mypy, POSIX code check, unlisted-eval
  check, package build, autolint, generated-docs check and large-file scan, on
  every push and PR.
- **Markdown Lint** (`markdown-lint.yml`) — markdown style on PRs.
- **PR Template Check** (`pr-template-check.yml`) — PR body carries the
  required checklist.
- **Sync Template** / **Sync Upstream** — weekly syncs of managed files from
  the template and from
  [inspect_evals](https://github.com/UKGovernmentBEIS/inspect_evals).
- **Claude Code Review** (`claude-review.yaml`) — reviews PRs against
  `EVALUATION_CHECKLIST.md` and `BEST_PRACTICES.md`. Skipped unless
  `ANTHROPIC_API_KEY` or `ANTHROPIC_ROLE_ARN` is set; on-demand via `/review`
  unless `AUTO_REVIEW_ALL_COMMITS=true`.

The sync workflows open PRs, which needs **Settings > Actions > General >
Workflow permissions** set to read and write with PR creation allowed, plus a
fine-grained PAT stored as the `SYNC_PAT` secret (Contents, Pull requests and
Workflows all read-and-write). Without it the sync fails on push, because the
default `GITHUB_TOKEN` cannot update files under `.github/workflows/`.

## Checks and enforcement

Each check has an `ENFORCE_<NAME>` toggle in
[`tools/enforcement.config`](tools/enforcement.config). `true` blocks merge on
failure; `false` reports the failure as advisory. `make check` honours the same
file, and an environment variable overrides it for one run:

```bash
ENFORCE_AUTOLINT=true bash tools/run_checks.sh
```

Blocking by default: `ENFORCE_RUFF`, `ENFORCE_MYPY`, `ENFORCE_POSIX_CHECK`,
`ENFORCE_UV_LOCK`, `ENFORCE_UNLISTED_EVALS`, `ENFORCE_PACKAGE`. Advisory by
default: `ENFORCE_AUTOLINT`, `ENFORCE_GENERATED_DOCS`,
`ENFORCE_MARKDOWN_LINT`, `ENFORCE_LARGE_FILES`, `ENFORCE_PR_TEMPLATE`. Setting
all of them to `true` is the registry-ready configuration.

## Claude Code skills

`.claude/skills/` ships template skills, invoked as *"Please run the
/SKILL_NAME skill on EVAL_NAME."* Authoring: `create-eval`,
`investigate-dataset`, `ensure-test-coverage`. Reviewing:
`eval-quality-workflow`, `eval-validity-review`, `review-pr-workflow`. Running
and analysing: `eval-report-workflow`, `read-eval-logs`,
`check-trajectories-workflow`. Submission: `prepare-submission-workflow`.

## Documentation

- [CONTRIBUTING.md](CONTRIBUTING.md) — development setup, testing and
  submission guidelines.
- [BEST_PRACTICES.md](BEST_PRACTICES.md) — evaluation design practices, synced
  from `inspect_evals`.
- [EVALUATION_CHECKLIST.md](EVALUATION_CHECKLIST.md) — the review checklist,
  synced from `inspect_evals`.
- [AUTOMATED_CHECKS.md](AUTOMATED_CHECKS.md) — what `tools/run_autolint.py`
  checks and how to suppress a rule.
- [TASK_VERSIONING.md](TASK_VERSIONING.md) — when to bump an eval's version.
- [AGENTS.md](AGENTS.md) — repo-wide tips for coding agents.
- [MANAGED_FILES.md](MANAGED_FILES.md) — which files are template-managed and
  which are yours.
- [CLAUDE.md](CLAUDE.md) — project instructions Claude Code reads here.
