# GitHub Epic Code Reviewer

AI pull request review that runs inside your own GitHub Actions account.

It posts a sticky PR summary, optional inline review comments, a neutral check run, and artifacts for patches, task memory, and command audit logs. You bring the model: OpenAI, OpenRouter, Anthropic, an OpenAI-compatible gateway, or a local Ollama endpoint.

Use it when you want CodeRabbit-style PR feedback without tying every repo to one hosted reviewer.

## What It Posts

- A PR summary that stays in one sticky comment.
- Inline comments only on changed lines.
- A neutral check run with counts by severity.
- A suggested patch artifact when a trusted user asks for `@epic-reviewer fix`.
- A compact task-memory artifact with risk notes, safeguards, and findings.
- A JSONL command audit for trusted comment commands.

The bot is intentionally picky. Weak findings get filtered before GitHub sees them.

## How The Review Works

For each allowed PR run, the action builds context, asks the model for defects, then checks the model's answer before posting.

- Reads repo rules from `AGENTS.md` and `.github/epic-code-reviewer.md`.
- Reads review policy from `REVIEW.md`, including nested policy files under changed paths.
- Reads path rules from `.github/epic-code-reviewer-rules/*.instructions.md`.
- Adds nearby source lines around each changed hunk.
- Adds CODEOWNERS, simple symbol matches, matching tests/specs, CI logs, and scanner output when present.
- Scores PR risk from path and size signals.
- Runs focused passes for bugs, security, tests, API compatibility, deploy/config changes, LLM/agent code, tool permissions, and stale claims.
- Runs a judge pass that can reject weak findings.
- Drops findings that do not land on changed lines.
- Dedupes comments already posted by the action.

That last part matters. Review bots lose trust fast when they comment on untouched code, repeat themselves, or nitpick Markdown.

## Quick Start

Use the pinned template for normal repos.

1. Copy `templates/ai-pr-review-pinned.yml` to `.github/workflows/ai-pr-review.yml`.
2. Copy `templates/epic-code-reviewer.config.json` to `epic-code-reviewer.config.json`.
3. Copy `templates/epic-code-reviewer.schema.json` to `epic-code-reviewer.schema.json`.
4. Copy `templates/REVIEW.md` to `REVIEW.md`.
5. Add one provider secret, such as `REVIEWER_OPENAI_API_KEY`.
6. Open a pull request.

The pinned workflow uses:

```yaml
uses: hongkongkiwi/github-epic-code-reviewer@v1
```

Start with `@v1` so patch releases arrive without editing each repo. Pin to a full tag if your release policy requires exact action versions.

## Workflow Templates

| Template | Use it for |
| --- | --- |
| `templates/ai-pr-review-pinned.yml` | Standard install using `@v1`. |
| `templates/ai-pr-review-with-scanners.yml` | Review plus tests and Semgrep logs. |
| `templates/ai-pr-review.yml` | Smaller install without scanner steps. |
| `templates/ai-pr-review-on-demand.yml` | Only runs after a trusted `@epic-reviewer` comment. |
| `templates/ai-pr-review-openrouter.yml` | OpenRouter-first setup. |
| `templates/ai-pr-review-anthropic.yml` | Direct Anthropic setup. |
| `templates/ai-pr-review-local.yml` | Self-hosted runner with a local OpenAI-compatible endpoint. |

This repo also includes `.github/workflows/self-review.yml` for dogfooding. It skips the model call when the chosen provider secret is missing.

## Provider Setup

OpenAI:

```yaml
env:
  REVIEWER_OPENAI_API_KEY: ${{ secrets.REVIEWER_OPENAI_API_KEY }}
with:
  provider: openai
  model: gpt-4.1-mini
```

OpenRouter:

```yaml
env:
  REVIEWER_OPENROUTER_API_KEY: ${{ secrets.REVIEWER_OPENROUTER_API_KEY }}
  REVIEWER_OPENROUTER_SITE_URL: ${{ vars.REVIEWER_OPENROUTER_SITE_URL }}
  REVIEWER_OPENROUTER_APP_NAME: GitHub Epic Code Reviewer
with:
  provider: openrouter
  model: anthropic/claude-sonnet-4.5
```

Anthropic direct:

```yaml
env:
  REVIEWER_ANTHROPIC_API_KEY: ${{ secrets.REVIEWER_ANTHROPIC_API_KEY }}
with:
  provider: anthropic
  model: claude-sonnet-4-5-20250929
```

OpenAI-compatible gateway:

```yaml
env:
  REVIEWER_OPENAI_API_KEY: ${{ secrets.REVIEWER_OPENAI_API_KEY }}
  REVIEWER_OPENAI_BASE_URL: ${{ secrets.REVIEWER_OPENAI_BASE_URL }}
with:
  provider: openai-compatible
  model: your-model-name
```

Ollama on a self-hosted runner:

```yaml
env:
  REVIEWER_OPENAI_BASE_URL: http://127.0.0.1:11434/v1
with:
  provider: ollama
  model: qwen2.5-coder:32b
```

You can also set `fallback-provider` and `fallback-model` so one failed model request retries elsewhere.

## Trust Model

The default templates are conservative because model calls need secrets.

- PR runs are skipped for forks.
- Draft PRs are skipped.
- `issue_comment` commands run only for `OWNER`, `MEMBER`, or `COLLABORATOR`.
- The workflow checks out `refs/pull/<number>/merge` after preflight passes.
- Write permissions are limited to PR review comments, issue comments, and check runs.
- Config, memory, audit, and patch paths must stay inside the workspace.
- The Python reviewer has no third-party package dependency.

If you need fork support, use a locked-down self-hosted runner and review the workflow permissions first. Don't hand repository secrets to untrusted PR code.

## Commands

Comment on a pull request:

```text
@epic-reviewer retry
```

Other commands:

```text
@epic-reviewer ask why is this risky?
@epic-reviewer describe
@epic-reviewer fix
@epic-reviewer quick
@epic-reviewer deep
@epic-reviewer security
```

`ask` posts an answer as a PR comment. `describe` updates the PR title and body. `fix` writes a patch artifact but does not push commits.

`quick`, `deep`, and `security` change the review profile for that run.

## Config

`epic-code-reviewer.config.json` controls review noise and context. It declares `epic-code-reviewer.schema.json`, so editors with JSON Schema support can catch misspelled fields.

Common settings:

| Setting | Meaning |
| --- | --- |
| `min_confidence` | Drops weak inline findings. |
| `max_inline_comments` | Caps PR noise. |
| `ignore_paths` | Skips generated files, lockfiles, vendored code, or repo-specific paths. |
| `rules_files` | Repo instructions such as `AGENTS.md`. |
| `review_rule_files` | Review-only policy files, usually `REVIEW.md`. |
| `path_rule_dirs` | Directories with `*.instructions.md` files. |
| `fail_on_block` | Fails the workflow when a blocking finding survives filtering. |
| `specialist_passes` | Chooses focused model passes. |
| `risk_tier_passes` | Changes passes by low, medium, or high PR risk. |
| `skip_judge_on_low_risk` | Saves tokens on small low-risk PRs. |
| `auto_review_enabled` | Disables automatic PR review while keeping trusted commands. |
| `fallback_provider` | Retries a failed model request with another provider. |
| `ci_log_paths` | Adds CI logs to model context. |
| `scanner_log_paths` | Adds scanner output; SARIF and Semgrep JSON are summarized. |
| `judge_enabled` | Runs the reject-weak-findings pass. |
| `dry_run` | Writes JSON and the job summary without posting comments. |
| `check_run_enabled` | Writes the neutral check run. |
| `memory_path` | Suppresses dismissed findings by fingerprint. |
| `task_memory_path` | Writes the compact markdown task record. |
| `audit_log_path` | Appends trusted command records as JSON lines. |
| `include_related_files` | Adds matching test/spec files to context. |

## Model Output Contract

The model must return JSON:

```json
{
  "summary": "Short PR review summary.",
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

Valid severities are `info`, `warn`, and `block`. Findings outside changed lines are discarded.

## Local Development

The reviewer is a composite GitHub Action around one Python script.

```bash
python3 -m py_compile scripts/review_pr.py
python3 -m unittest discover -s tests
```

Optional local hooks:

```bash
lefthook install
lefthook run pre-commit
```

Workflow lint:

```bash
actionlint .github/workflows/*.yml templates/ai-pr-review*.yml
```

## Release

Push a semver tag such as `v1.2.3`. `.github/workflows/release.yml` publishes a GitHub release, then moves the matching major tag such as `v1`.

## Data Sent To Models

The action sends selected PR diff, nearby source context, repo review rules, and configured logs to the chosen model provider. For private code that must stay on your own network, use `templates/ai-pr-review-local.yml` on a self-hosted runner with a local endpoint.
