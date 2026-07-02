# About This Project

This repository exists as a **safe, disposable target** for testing and validating
automated documentation-agent pull request creation.

## Why it exists

Writing documentation tooling requires a real Git repository to push branches to,
open draft PRs against, and inspect diffs on. Using a production docs repository
for that kind of testing risks noisy PR queues, accidental merges, and reviewer
fatigue.

This project gives the docs agent a dedicated place to:

- Prove it can create correctly scoped branches (`docs-agent/<slug>`).
- Verify draft PR content, formatting, and metadata before any changes reach real
  user-facing documentation.
- Let engineers review agent behavior and iterate on prompts without touching
  customer-visible pages.

## What lives here

All test content is under `content/docs/`. The files are intentionally minimal so
reviewers can focus on the agent's output rather than the substance of the docs
themselves.

## What does not live here

This repository is **not** a source of truth for any product documentation. Nothing
here is published to a public docs site. Do not link to these pages from
customer-facing materials.
