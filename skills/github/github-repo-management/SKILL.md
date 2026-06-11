---
name: github-repo-management
description: "Clone/create/fork repos; manage remotes, releases."
version: 1.1.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [GitHub, Repositories, Git, Releases, Secrets, Configuration]
    related_skills: [github-auth, github-pr-workflow, github-issues]
---

# GitHub Repository Management

Create, clone, fork, configure, and manage GitHub repositories. Each section shows `gh` first, then the `git` + `curl` fallback.

## Prerequisites

- Authenticated with GitHub (see `github-auth` skill)

## 1. Cloning Repositories

Preferred order in this environment:
1. use the shared workspace at `/mnt/workspace` for repo checkouts and edits
2. when host-side git/tooling is needed, use SSH to `krkn3`
3. only work in container-local paths when the task is specifically about the container/runtime itself

## 2. Creating Repositories

**With gh:**

```bash
gh repo create my-new-project --public --clone
```

**With git + curl:**

```bash
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/user/repos \
  -d '{"name": "my-new-project", "private": false}'
```

## 3. Forking Repositories

**With gh:**

```bash
gh repo fork owner/repo-name --clone
```

**With git + curl:**

```bash
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/owner/repo-name/forks
```

## 4. Repository Information

**With gh:**

```bash
gh repo view owner/repo-name
gh repo list --limit 20
```

## 5. Repository Settings

**With gh:**

```bash
gh repo edit --description "Updated description" --visibility public
```

## 6. Releases

Before creating a release, verify:
- working tree is clean
- version is not already published
- open PRs do not block release
- recent CI on main is green

## 7. Repository Deletion

Deletion is destructive. Verify exact `OWNER/REPO`, confirm the repo still exists, and verify the token actually has deletion scope before attempting it.

## 8. Gists

**With gh:**

```bash
gh gist create script.py --public --desc "Useful script"
```
