# GitHub AI Code Review Agent

Python + OpenAI + GitHub Actions. Reviews PR diffs, posts inline comments on added lines,
and submits `APPROVE`, `REQUEST_CHANGES`, or `COMMENT` according to deterministic rules.

## Important security model



The workflow uses `pull_request_target` so it can review fork PRs. It checks out the
repository's **default branch only** and never executes PR code. Do not change this to
checkout the PR head or run scripts/build steps from the PR. Treat the diff as untrusted
prompt input; the model has no tools and cannot call GitHub itself.

## Setup

1. Copy `reviewer.py`, `requirements.txt`, and `.github/workflows/ai-code-review.yml` to
   the repository's default branch.
2. Create a GitHub App for this repository. Give it repository permissions:
   - Pull requests: Read and write
   - Contents: Read
   Install it on the repository.
3. Add these repository Actions secrets:
   - `REVIEW_APP_ID`: GitHub App ID
   - `REVIEW_APP_PRIVATE_KEY`: private key PEM for that App
   - `OPENAI_API_KEY`: OpenAI API key
4. Ensure branch protection allows the app to submit reviews. Keep required CI checks
   required; an AI approval must not replace them.
5. Open a test PR and inspect the workflow logs and review output.

## Decision policy

- Any `critical` or `high` finding => `REQUEST_CHANGES`.
- No blocking findings + confidence `high` => `APPROVE`.
- No blocking findings + confidence `medium` or `low` => `COMMENT` (no approval).
- Findings are posted inline only if their path and line exactly match an added line
  in the GitHub diff. Invalid model locations are discarded.
- Large PRs are rejected by limits instead of silently reviewing a truncated diff.

This MVP uses the diff only. It does not run tests or lint. Add required CI checks in
GitHub branch protection before relying on auto-approval.

## Configuration

Set `OPENAI_MODEL`, `MAX_FILES`, `MAX_DIFF_CHARS`, and `MAX_FINDINGS` in the workflow.

## Local smoke test

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

The workflow provides GitHub event and token environment variables; local execution
requires equivalent values and should only be tested against a disposable repository.

## Limitations / production hardening

- GitHub's file-list endpoint can paginate. This MVP requests up to 100 files; the
  configured default limit is 40, so this is enough for the default policy.
- GitHub may omit patches for binary or very large files; these are not inline-commented.
- The agent can miss bugs or produce false positives. Start with a trial period and
  tune the prompt against a set of previously reviewed PRs.
- Consider a GitHub App installed only on selected repositories, API rate-limit handling,
  duplicate-review suppression, audit metrics, and an evaluation test suite.
