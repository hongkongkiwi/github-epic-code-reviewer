# GitHub Epic Code Reviewer

A small, self-hostable AI pull request reviewer for GitHub Actions.

It posts a sticky PR summary and inline comments on changed lines. The reviewer is strict by design: it asks the model for evidence-backed defects, filters out comments that do not point to changed lines, and caps the number of inline findings.

The action now runs as a review pipeline:

- It reads repo rules from `AGENTS.md` and `.github/epic-code-reviewer.md`.
- It reads review-only policy from `REVIEW.md`, including nested files under changed paths.
- It reads path-specific rules from `.github/epic-code-reviewer-rules/*.instructions.md`.
- It collects nearby source lines around changed hunks.
- It adds simple symbol matches, CODEOWNERS, CI logs, and scanner logs to the review context.
- It adds related test/spec files when their names match changed files.
- It can read CI and scanner logs before asking the model to review.
- It runs narrow review passes for bugs, security, tests, API compatibility, and deploy/config risk.
- It has extra lanes for LLM/agent code, tool permission boundaries, and stale review claims.
- It runs a judge pass that drops weak findings.
- It avoids reposting the same inline finding on later pushes.
- It writes a neutral check run with machine-readable severity counts.
- It scores PR risk from path and size signals, then uses that risk in the review prompt.
- It writes compact markdown task memory for artifacts/debugging.
- It splits stable reviewer rules from per-PR context with a cache boundary marker for providers that support prompt caching.

## Use It In Another Repo

Copy these files into the target repository:

- `templates/ai-pr-review-with-scanners.yml` to `.github/workflows/ai-pr-review.yml`
- `templates/epic-code-reviewer.config.json` to `epic-code-reviewer.config.json`
- `templates/epic-code-reviewer.schema.json` to `epic-code-reviewer.schema.json`
- `templates/REVIEW.md` to `REVIEW.md`
- `templates/epic-code-reviewer-memory.json` to `.github/epic-code-reviewer-memory.json`
- `templates/epic-code-reviewer-task-memory.md` to `.github/epic-code-reviewer-task-memory.md` if you want a checked-in seed file
- `templates/epic-code-reviewer-rules/*.instructions.md` to `.github/epic-code-reviewer-rules/`
- optional `.github/epic-code-reviewer.md` for repo-specific review rules

For OpenAI, add one repository secret:

- `REVIEWER_OPENAI_API_KEY`

Use this action input:

```yaml
with:
  provider: openai
  model: gpt-4.1-mini
```

For OpenAI-compatible gateways, also add:

- `REVIEWER_OPENAI_BASE_URL`

Use `provider: openai-compatible`. For Ollama or another local endpoint, use `templates/ai-pr-review-local.yml`.

For OpenRouter, add:

- `REVIEWER_OPENROUTER_API_KEY`

Use this action input:

```yaml
with:
  provider: openrouter
  model: anthropic/claude-sonnet-4.5
```

Optional OpenRouter metadata:

- `REVIEWER_OPENROUTER_SITE_URL`
- `REVIEWER_OPENROUTER_APP_NAME`
- `REVIEWER_OPENROUTER_BASE_URL`

The default workflow skips pull requests from forks. That keeps repository secrets away from outside code. If you need fork support, run this on a locked-down self-hosted runner and review the permissions first.

For a lighter setup, use `templates/ai-pr-review.yml`. It skips the test and Semgrep steps.

For token-controlled reviews, use `templates/ai-pr-review-on-demand.yml`. It runs only after a repository owner, member, or collaborator comments with `@epic-reviewer`.

For production pinning after the first release, use `templates/ai-pr-review-pinned.yml`. It points at `hongkongkiwi/github-epic-code-reviewer@v1` instead of `@main`.

This repository also has `.github/workflows/self-review.yml` for on-demand self-review. It checks the same trust rules and skips the model call when the selected provider has no matching secret. Set repo variables `REVIEWER_PROVIDER` and `REVIEWER_MODEL` to dogfood OpenRouter or Anthropic.

If you want OpenRouter as the default, start with `templates/ai-pr-review-openrouter.yml`.

## Anthropic

Use `templates/ai-pr-review-anthropic.yml` and add:

- `REVIEWER_ANTHROPIC_API_KEY`

Use `provider: anthropic` for Anthropic's Messages API, or `provider: openrouter` for Anthropic models routed through OpenRouter.

## Local Models

Use `templates/ai-pr-review-local.yml` on a self-hosted runner with an OpenAI-compatible endpoint, such as Ollama on `http://127.0.0.1:11434/v1`.

## Config

`epic-code-reviewer.config.json` controls noise:

- `min_confidence`: drops weak findings.
- `max_inline_comments`: keeps the review short.
- `ignore_paths`: skips generated files and lockfiles.
- `rules_files`: reads local instructions such as `AGENTS.md`.
- `review_rule_files`: review-only policy files, usually `REVIEW.md`.
- `path_rule_dirs`: directories with `*.instructions.md` path rules.
- `fail_on_block`: fails the workflow when a blocking finding survives filtering.
- `specialist_passes`: controls which narrow review passes run.
- `risk_tier_passes`: changes model passes by low, medium, or high PR risk.
- `skip_judge_on_low_risk`: avoids the judge pass on small low-risk reviews.
- `auto_review_enabled`: when false, only trusted comment commands run.
- `fallback_provider` and `fallback_model`: retry a failed primary model request elsewhere.
- `ci_log_paths` and `scanner_log_paths`: files copied into model context. SARIF and Semgrep JSON are summarized before raw logs.
- `judge_enabled`: runs the reject-weak-findings pass.
- `dry_run`: writes JSON and the job summary without posting comments.
- `check_run_enabled`: writes a neutral GitHub check run.
- `memory_path`: suppresses dismissed findings by fingerprint.
- `task_memory_path`: where CI writes a compact markdown record of risk, safeguards, and findings.
- `audit_log_path`: appends trusted comment-command records as JSON lines.
- `include_related_files`: copies matching test/spec files into context.

The config file declares `epic-code-reviewer.schema.json`, so editors with JSON Schema support can catch misspelled fields.

## Commands

On a pull request, comment:

```text
@epic-reviewer retry
```

That reruns the review.

Other commands:

```text
@epic-reviewer ask why is this risky?
@epic-reviewer describe
@epic-reviewer fix
@epic-reviewer quick
@epic-reviewer deep
@epic-reviewer security
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

Optional local hooks:

```bash
lefthook install
lefthook run pre-commit
```

Release tags use `.github/workflows/release.yml`. Push `v1.2.3` and the workflow publishes a GitHub release, then moves the matching major tag such as `v1`.

## Notes

This action sends PR diffs to the configured model provider. For private code that must stay on your own network, run it on a self-hosted runner with an OpenAI-compatible local endpoint and set `REVIEWER_OPENAI_BASE_URL`.
