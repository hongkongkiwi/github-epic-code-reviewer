# GitHub Reviewer PR

A small, self-hostable AI pull request reviewer for GitHub Actions.

It posts a sticky PR summary and inline comments on changed lines. The reviewer is strict by design: it asks the model for evidence-backed defects, filters out comments that do not point to changed lines, and caps the number of inline findings.

The action now runs as a review pipeline:

- It reads repo rules from `AGENTS.md` and `.github/reviewer-pr.md`.
- It reads review-only policy from `REVIEW.md`, including nested files under changed paths.
- It reads path-specific rules from `.github/reviewer-rules/*.instructions.md`.
- It collects nearby source lines around changed hunks.
- It adds simple symbol matches, CODEOWNERS, CI logs, and scanner logs to the review context.
- It adds related test/spec files when their names match changed files.
- It can read CI and scanner logs before asking the model to review.
- It runs narrow review passes for bugs, security, tests, API compatibility, and deploy/config risk.
- It runs a judge pass that drops weak findings.
- It avoids reposting the same inline finding on later pushes.
- It writes a neutral check run with machine-readable severity counts.
- It scores PR risk from path and size signals, then uses that risk in the review prompt.
- It writes compact markdown task memory for artifacts/debugging.
- It splits stable reviewer rules from per-PR context with a cache boundary marker for providers that support prompt caching.

## Use It In Another Repo

Copy these files into the target repository:

- `templates/ai-pr-review-with-scanners.yml` to `.github/workflows/ai-pr-review.yml`
- `templates/reviewer-pr.config.json` to `reviewer-pr.config.json`
- `templates/REVIEW.md` to `REVIEW.md`
- `templates/reviewer-memory.json` to `.github/reviewer-memory.json`
- `templates/reviewer-task-memory.md` to `.github/reviewer-task-memory.md` if you want a checked-in seed file
- `templates/reviewer-rules/*.instructions.md` to `.github/reviewer-rules/`
- optional `.github/reviewer-pr.md` for repo-specific review rules

Add one repository secret:

- `REVIEWER_OPENAI_API_KEY`

For OpenAI-compatible gateways, also add:

- `REVIEWER_OPENAI_BASE_URL`

The default workflow skips pull requests from forks. That keeps repository secrets away from outside code. If you need fork support, run this on a locked-down self-hosted runner and review the permissions first.

For a lighter setup, use `templates/ai-pr-review.yml`. It skips the test and Semgrep steps.

## Anthropic

Use `templates/ai-pr-review-anthropic.yml` and add:

- `REVIEWER_ANTHROPIC_API_KEY`

## Local Models

Use `templates/ai-pr-review-local.yml` on a self-hosted runner with an OpenAI-compatible endpoint, such as Ollama on `http://127.0.0.1:11434/v1`.

## Config

`reviewer-pr.config.json` controls noise:

- `min_confidence`: drops weak findings.
- `max_inline_comments`: keeps the review short.
- `ignore_paths`: skips generated files and lockfiles.
- `rules_files`: reads local instructions such as `AGENTS.md`.
- `review_rule_files`: review-only policy files, usually `REVIEW.md`.
- `path_rule_dirs`: directories with `*.instructions.md` path rules.
- `fail_on_block`: fails the workflow when a blocking finding survives filtering.
- `specialist_passes`: controls which narrow review passes run.
- `ci_log_paths` and `scanner_log_paths`: files copied into model context.
- `judge_enabled`: runs the reject-weak-findings pass.
- `dry_run`: writes JSON and the job summary without posting comments.
- `check_run_enabled`: writes a neutral GitHub check run.
- `memory_path`: suppresses dismissed findings by fingerprint.
- `task_memory_path`: where CI writes a compact markdown record of risk, safeguards, and findings.
- `include_related_files`: copies matching test/spec files into context.

## Commands

On a pull request, comment:

```text
@reviewer retry
```

That reruns the review. `/ai-review retry` works too.

Other commands:

```text
@reviewer ask why is this risky?
@reviewer describe
@reviewer fix
@reviewer quick
@reviewer deep
@reviewer security
```

`ask` posts an answer as a PR comment. `describe` updates the PR title/body. `fix` writes a suggested patch artifact; it does not push commits.

`quick`, `deep`, and `security` change the review profile for that run.

## Model Output Contract

The model must return JSON:

```json
{
  "summary": "short PR review summary",
  "risk_level": "low",
  "findings": [
    {
      "path": "src/app.ts",
      "line": 42,
      "severity": "warn",
      "confidence": 0.83,
      "title": "Missing null check",
      "body": "This can throw when the API returns no user. Guard the value before reading id."
    }
  ]
}
```

Findings that do not land on changed lines are discarded before posting.

## Local Development

The script has no third-party Python dependencies.

```bash
python3 -m py_compile scripts/review_pr.py
python3 -m unittest discover -s tests
```

## Notes

This action sends PR diffs to the configured model provider. For private code that must stay on your own network, run it on a self-hosted runner with an OpenAI-compatible local endpoint and set `REVIEWER_OPENAI_BASE_URL`.
