# OpenCodeReview PR Review Action

A reusable GitHub Action that runs [OpenCodeReview](https://github.com/alibaba/open-code-review) on a pull request and posts inline review comments with a sticky summary. Other repositories can adopt AI-powered PR review with a single `uses:` step.

```yaml
- uses: alibaba/open-code-review/action@v1
  with:
    llm_url: ${{ secrets.OCR_LLM_URL }}
    llm_auth_token: ${{ secrets.OCR_LLM_AUTH_TOKEN }}
    llm_model: ${{ vars.OCR_LLM_MODEL }}
    llm_use_anthropic: ${{ vars.OCR_LLM_USE_ANTHROPIC }}
```

## How it works

1. Resolves the PR base ref and head SHA (or accepts overrides for comment triggers).
2. Checks out the head commit (full history) and computes `git merge-base` so the review covers the full base-to-head diff, not just the last push.
3. Installs OCR via npm and configures the LLM endpoint through environment variables (highest priority) plus `ocr config set llm.extra_body`.
4. Runs `ocr review --from <merge-base> --to <head> --format json`.
5. Uploads the raw JSON result and stderr as workflow artifacts.
6. Posts inline review comments and a summary via `actions/github-script`, using the shared module at `action/scripts/post-review-comments.js`.

## Usage

### Minimal hardened config

```yaml
name: OCR Review
on:
  pull_request_target:
    types: [opened, synchronize, reopened]
concurrency:
  group: ocr-${{ github.event.pull_request.number }}
  cancel-in-progress: true
permissions:
  contents: read
  pull-requests: write
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: alibaba/open-code-review/action@v1
        with:
          llm_url: ${{ secrets.OCR_LLM_URL }}
          llm_auth_token: ${{ secrets.OCR_LLM_AUTH_TOKEN }}
          llm_model: ${{ vars.OCR_LLM_MODEL }}
          llm_use_anthropic: ${{ vars.OCR_LLM_USE_ANTHROPIC }}
```

### Trigger from a comment

For `issue_comment` triggers, resolve the PR context first and pass it in:

```yaml
- uses: alibaba/open-code-review/action@v1
  with:
    llm_url: ${{ secrets.OCR_LLM_URL }}
    llm_auth_token: ${{ secrets.OCR_LLM_AUTH_TOKEN }}
    llm_model: ${{ vars.OCR_LLM_MODEL }}
    llm_use_anthropic: ${{ vars.OCR_LLM_USE_ANTHROPIC }}
    base_ref: ${{ steps.pr-context.outputs.base_ref }}
    head_sha: ${{ steps.pr-context.outputs.head_sha }}
```

## Inputs

| input | required | default | description |
|---|---|---|---|
| `llm_url` | yes | — | LLM API endpoint (env `OCR_LLM_URL`) |
| `llm_auth_token` | yes | — | LLM auth token (env `OCR_LLM_TOKEN`) |
| `llm_model` | yes | — | model name (env `OCR_LLM_MODEL`) |
| `llm_use_anthropic` | yes | — | `true` = Anthropic, `false` = OpenAI-compatible (env `OCR_USE_ANTHROPIC`) |
| `llm_auth_header` | no | — | custom auth header name (env `OCR_LLM_AUTH_HEADER`) |
| `llm_extra_headers` | no | — | extra headers `K=V,K=V` (env `OCR_LLM_EXTRA_HEADERS`) |
| `llm_extra_body` | no | `{"thinking": {"type": "disabled"}}` | extra_body JSON (written via `ocr config set`) |
| `llm_timeout` | no | — | LLM request timeout, seconds (env `OCR_LLM_TIMEOUT`) |
| `github_token` | no | `${{ github.token }}` | token for posting comments |
| `ocr_version` | no | `latest` | npm version spec |
| `review_concurrency` | no | — | `ocr review --concurrency` |
| `background` | no | — | `ocr review --background` |
| `rule` | no | — | `ocr review --rule` file path |
| `upload_artifacts` | no | `true` | upload JSON + stderr artifacts |
| `sticky_summary` | no | `true` | summary dimension: update existing summary in place |
| `incremental` | no | `false` | incremental dimension: only append comments not overlapping history |
| `base_ref` | no | — | override base ref (comment triggers) |
| `head_sha` | no | — | override head SHA (comment triggers) |
| `node_version` | no | `24` | Node.js version |

## Outputs

| output | description |
|---|---|
| `comments_total` | total comments produced by OCR |
| `comments_inline` | inline comments successfully posted |
| `comments_skipped` | inline comments skipped by incremental mode |
| `comments_failed` | inline comments that failed to post |
| `summary_comment_url` | URL of the posted/updated summary comment |

## Required secrets and variables

Add under **Settings → Secrets and variables → Actions**:

| kind | name | required | notes |
|---|---|---|---|
| secret | `OCR_LLM_URL` | yes | LLM endpoint |
| secret | `OCR_LLM_AUTH_TOKEN` | yes | mapped to env `OCR_LLM_TOKEN` internally |
| variable | `OCR_LLM_MODEL` | yes | model name |
| variable | `OCR_LLM_USE_ANTHROPIC` | yes | `true`/`false` |

> Note: the secret name `OCR_LLM_AUTH_TOKEN` differs from the env var
> `OCR_LLM_TOKEN` that OCR reads. The action bridges this mapping for you.

## Minimum permissions

```yaml
permissions:
  contents: read
  pull-requests: write
```

## Comment posting modes

Two boolean dimensions select one of four posting modes. The default is **sticky full** (mode 3).

| mode | sticky_summary | incremental | inline | summary |
|---|---|---|---|---|
| 1 full append (legacy) | false | false | new review with all comments | new comment each run |
| 2 incremental append | false | true | only non-overlapping comments | new comment each run |
| 3 sticky full (default) | true | false | new review with all comments | updated in place |
| 4 sticky incremental | true | true | only non-overlapping comments | updated in place |

- **Full**: every run posts all current inline comments (reviews may repeat on the same line across runs).
- **Incremental** (non-destructive): before posting, the action lists existing bot review comments and skips any current comment whose `(path, line range)` overlaps an existing one. History is never deleted; comments only accumulate on new lines.
- **Sticky**: the summary is posted as an issue comment tagged with `<!-- ocr-summary -->` and updated in place on reruns instead of spamming.

When inline comments cannot be posted (e.g. a line cannot be resolved against the diff), they fall back into the summary so feedback is never lost.

## Retry and rate-limit tuning

When posting comments individually (fallback after a batch failure), the action honors GitHub rate-limit headers (`retry-after`, `x-ratelimit-*`) with exponential backoff. Tune via repository **variables**:

| variable | default | description |
|---|---|---|
| `OCR_RETRY_BASE_DELAY` | `60000` | base backoff (ms) for secondary limits with no header |
| `OCR_RETRY_MAX_DELAY` | `300000` | universal cap (ms) on any computed wait |
| `OCR_MAX_RETRIES` | `3` | max retries per comment |
| `OCR_SUCCESS_DELAY` | `2000` | delay after a successful post |
| `OCR_FAILURE_DELAY` | `1000` | delay after a non-retryable failure |
| `OCR_LOW_REMAINING_THRESHOLD` | `3` | proactively slow down at/below this remaining quota |
| `OCR_LOW_REMAINING_SPACING` | `10000` | request spacing (ms) when quota is low |

See GitHub's [Rate limits for the REST API](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api).

## Security: fork PRs and `pull_request_target`

- This action uses `pull_request_target` semantics: secrets are available even for PRs from forks, because OCR only **reads the diff** and never executes code from the PR.
- Do **not** pass a `rule` file sourced from the PR branch when secrets are in scope; only use trusted rules from your own repository.
- Do **not** combine `pull_request` (non-target) with a checkout of PR-supplied code while secrets are in scope — that exposes secrets to untrusted code.
- The authenticated token used for posting is scoped to the `pull-requests: write` and `contents: read` permissions only.

## Customizing the comment author

By default comments are posted as `github-actions[bot]`. To post as a GitHub App, mint a token with `actions/create-github-app-token@v1` and pass it via `github_token`.

## Versioning

Pin to a major version to auto-receive fixes within v1, or pin to a commit SHA for the strictest reproducibility:

```yaml
- uses: alibaba/open-code-review/action@v1
- uses: alibaba/open-code-review/action@<commit-sha>
```
