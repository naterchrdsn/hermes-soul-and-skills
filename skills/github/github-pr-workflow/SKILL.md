---
name: github-pr-workflow
description: "GitHub PR lifecycle: branch, commit, open, CI, merge."
version: 1.1.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [GitHub, Pull-Requests, CI/CD, Git, Automation, Merge]
    related_skills: [github-auth, github-code-review]
---

# GitHub Pull Request Workflow

Complete guide for managing the PR lifecycle. Each section shows the `gh` way first, then the `git` + `curl` fallback for machines without `gh`.

## Prerequisites

- Authenticated with GitHub (see `github-auth` skill)
- Inside a git repository with a GitHub remote

## 1. Branch Creation

```bash
git fetch origin
git checkout main && git pull origin main
git checkout -b feat/add-user-authentication
```

Branch naming conventions:
- `feat/description`
- `fix/description`
- `refactor/description`
- `docs/description`
- `ci/description`

## 2. Making Commits

Use the agent's file tools (`write_file`, `patch`) to make changes, then commit.

Commit message format:

```text
type(scope): short description

Longer explanation if needed.
```

## 3. Pushing and Creating a PR

```bash
git push -u origin HEAD
```

Verify the push before claiming success:

```bash
BRANCH=$(git branch --show-current)
git fetch origin "$BRANCH"
git rev-parse HEAD
git rev-parse "origin/$BRANCH"
git status -sb
```

## 4. Monitoring CI Status

**With gh:**

```bash
gh pr checks
gh pr checks --watch
```

**With git + curl:**

```bash
SHA=$(git rev-parse HEAD)
curl -s \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/commits/$SHA/status
```

## 5. Auto-Fixing CI Failures

Loop:
1. Check CI status
2. Read failure logs
3. Fix with `read_file` + `patch`/`write_file`
4. Commit and push
5. Re-check CI
6. Repeat up to 3 attempts

## 6. Merging

**With gh:**

```bash
gh pr merge --squash --delete-branch
gh pr merge --auto --squash --delete-branch
```

**With git + curl:**

```bash
curl -s -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/pulls/$PR_NUMBER/merge
```
