# semantic-release-preview

Reusable GitHub Actions workflow that predicts what [semantic-release](https://github.com/semantic-release/semantic-release) would publish for a pull request before merge.

It runs [semantic-release](https://github.com/semantic-release/semantic-release) in **dry-run mode** and comments on the pull request with predicted next versions based on the Github merge strategies:

- Merge commit
- Squash and merge
- Rebase and merge

It also handles the no-release case cleanly and updates an existing bot comment instead of spamming new comments.

## What This Solves

- Shows release impact during PR review
- Reduces surprises from different merge strategies
- Keeps one up-to-date PR comment via a hidden marker
- Cancels stale runs to avoid race-condition comment overwrites

## Reusable Workflow

This repository exposes:

- `.github/workflows/semantic-release-preview.yml`

Use it from another repository with `workflow_call`.

## Requirements In Caller Repo

- A `pull_request` triggered workflow that calls this reusable workflow
- `GITHUB_TOKEN` available (default in GitHub Actions)
- Permissions in caller workflow:
	- `contents: write`
	- `issues: write`
	- `pull-requests: write`
- A semantic-release configuration in the caller repo, or semantic-release defaults if you do not use a config file

## Quick Start

Create a workflow in your repository, for example `.github/workflows/release-dry-run-pr.yml`:

```yaml
name: Release Dry Run PR

on:
	pull_request:
		types:
			- opened
			- edited
			- reopened
			- synchronize
			- ready_for_review

concurrency:
	group: release-dry-run-pr-${{ github.event.pull_request.number }}
	cancel-in-progress: true

permissions:
	contents: write
	issues: write
	pull-requests: write

jobs:
	release-dry-run:
		uses: babblebey/semantic-release-preview/.github/workflows/semantic-release-preview.yml@main
		secrets:
			github_token: ${{ secrets.GITHUB_TOKEN }}
		with:
			semantic_release_command: npx semantic-release
```

## Inputs

The reusable workflow supports these `with:` inputs:

- `semantic_release_command` (string, default: `npx semantic-release`)
	- Command used to execute semantic-release (the workflow appends `--dry-run --no-ci`)
- `node_version` (string, default: `lts/*`)
	- Node.js version for `actions/setup-node`
- `comment_marker` (string, default: `<!-- semantic-release-dry-run -->`)
	- Hidden marker used to locate and update an existing bot comment
- `icon_url` (string, default: `https://semantic-release.org/favicon.svg`)
	- Icon shown in the PR comment heading

## Secrets

Pass this in `jobs.<job>.secrets`:

- `github_token` (optional)
	- Token used for API calls and semantic-release auth checks
	- If omitted, the reusable workflow falls back to `github.token`

## Behavior Notes

- Workflow skips draft PRs.
- Workflow skips fork PRs (`head.repo.full_name != github.repository`) for safety.
- Predictions can differ across merge strategies because commit history differs.
- If no release is expected, the PR comment clearly reports that state.

## Custom semantic-release CLI

If your project uses a different semantic-release CLI package, pass it explicitly:

```yaml
with:
	semantic_release_command: npx semantic-release-react@1.0.0-beta.3
```

## Troubleshooting

- No comment posted:
	- Confirm caller workflow has write permissions for `issues` and `pull-requests`.
	- Confirm PR is not from a fork and is not draft.
- Unexpected version prediction:
	- Verify Conventional Commit messages and PR title/body.
	- Verify your semantic-release branch config in the caller repository.
- Auth/push check failures in dry-run:
	- Ensure `github_token` is passed and permissions include `contents: write`.
