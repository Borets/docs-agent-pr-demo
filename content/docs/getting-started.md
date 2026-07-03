# Getting Started

This page is intentionally small so documentation-agent PRs are easy to review.

## Prerequisites

Before you begin, make sure you have the following installed:

- **Node.js** 18 or later
- **Git**
- A GitHub account with access to the repository

## Local setup

1. Clone the repository to your local machine.
2. Install dependencies with your preferred package manager.
3. Copy the example environment file and fill in any required values.
4. Start the development server.
5. Open `http://localhost:3000` to view the local documentation site.

## Running checks

Before opening a PR, run the project checks and confirm they pass:

```bash
npm run lint
npm run test
```

Include the check results in your PR description so reviewers can verify the baseline.

## Next steps

- Review [Release Notes](./release-notes.md) for recent changes.
- Open a draft PR when your edits are ready — humans merge.
