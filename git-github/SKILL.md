---
name: git-github
description: >
  Expert guidance for git and the GitHub CLI (gh) — daily workflows, branching, committing,
  merging, rebasing, pull requests, issues, and repo management following GitHub Flow.
  Use this skill whenever the user mentions git, gh, GitHub, commit, branch, PR, pull request,
  merge, rebase, push, clone, fork, issue, release, stash, conflict, or anything that sounds
  like version control — even if they just say "how do I undo this?" or "something went wrong
  with my branch."
---

# Git & GitHub CLI

## GitHub Flow — the daily loop

```
main (always deployable)
  └── feature/my-thing   ← short-lived branch
        │  commit, commit, commit
        └─→ PR → review → merge → delete branch
```

1. Pull latest main
2. Create a feature branch
3. Commit small, focused changes
4. Push and open a PR
5. Address review, merge, delete branch

---

## 1. Setup

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase false        # merge on pull (safer default)
git config --global core.autocrlf input      # Linux/macOS

# Authenticate GitHub CLI
gh auth login
```

---

## 2. Daily branch workflow

```bash
# Start work
git switch main
git pull
git switch -c feature/my-thing

# Work, then commit
git add -p                     # stage hunks interactively (preferred over add .)
git commit -m "feat: short description"

# Keep branch up to date with main
git fetch origin
git rebase origin/main         # or: git merge origin/main

# Push
git push -u origin feature/my-thing
```

### Commit message conventions

```
<type>: <short summary>

type: feat | fix | docs | refactor | test | chore | ci
```

Good: `fix: prevent login redirect loop on expired token`
Bad: `fix stuff`, `WIP`, `changes`

---

## 3. Pull requests with gh

```bash
# Open PR (interactive prompts)
gh pr create

# One-liner with title and body
gh pr create --title "feat: add dark mode" --body "Closes #42"

# Open PR from current branch to a specific base
gh pr create --base develop

# Review PRs assigned to you
gh pr list --assignee @me

# Check out a PR locally
gh pr checkout 123

# View PR status / CI checks
gh pr view 123
gh pr checks 123

# Merge when ready
gh pr merge 123 --squash --delete-branch

# Create PR and immediately open in browser
gh pr create --web
```

---

## 4. Issues

```bash
gh issue list                          # open issues
gh issue list --label "bug" --state open
gh issue create --title "Bug: ..." --body "Steps to reproduce..."
gh issue view 42
gh issue close 42
gh issue comment 42 --body "Fixed in #55"
```

---

## 5. Undoing things (most common need)

| Situation | Command |
|---|---|
| Unstage a file | `git restore --staged <file>` |
| Discard working-tree changes | `git restore <file>` |
| Undo last commit (keep changes staged) | `git reset --soft HEAD~1` |
| Undo last commit (keep changes unstaged) | `git reset HEAD~1` |
| Undo last commit (discard changes) | `git reset --hard HEAD~1` ⚠️ |
| Revert a commit already pushed | `git revert <hash>` (creates new commit) |
| Fix the last commit message | `git commit --amend -m "new message"` |
| Add forgotten file to last commit | `git add file && git commit --amend --no-edit` |
| Find a lost commit | `git reflog` |

> **Rule of thumb:** `revert` for pushed commits (safe, non-destructive), `reset` for local-only commits.

---

## 6. Merging vs rebasing

### Merge (preserves history, creates merge commit)
```bash
git switch feature/my-thing
git merge main
```
Use when: merging a completed feature back to main via PR. Let GitHub do this.

### Rebase (linear history, rewrites commits)
```bash
git switch feature/my-thing
git rebase main
```
Use when: keeping a feature branch up-to-date with main during development.

> Never rebase commits that have already been pushed and shared with others.

### Resolve conflicts
```bash
# After a merge or rebase conflict:
git status                    # see conflicted files
# Edit files — look for <<<<<<, =======, >>>>>>>
git add <resolved-file>
git rebase --continue         # or: git merge --continue
git rebase --abort            # bail out entirely
```

---

## 7. Stash

```bash
git stash                     # stash dirty working tree
git stash push -m "wip: auth refactor"   # with a name
git stash list
git stash pop                 # apply latest and remove from stash
git stash apply stash@{1}     # apply specific stash (keep it)
git stash drop stash@{1}      # delete a stash entry
```

---

## 8. Inspecting history

```bash
git log --oneline --graph --all       # visual branch history
git log --oneline -10                 # last 10 commits
git diff main...feature/my-thing      # what changed on the branch
git diff HEAD~1                       # since last commit
git show <hash>                       # one commit in full
git blame <file>                      # who changed which line
git log --follow -p -- <file>         # full history of a file
```

---

## 9. Repos and remotes

```bash
# Clone
gh repo clone owner/repo
git clone <url>

# Fork + clone in one step
gh repo fork owner/repo --clone

# View remotes
git remote -v

# Add upstream after forking
git remote add upstream https://github.com/original/repo.git
git fetch upstream
git merge upstream/main

# Create a new repo on GitHub
gh repo create my-project --public --source=. --remote=origin --push
```

---

## 10. GitHub CLI extras

```bash
# Releases
gh release create v1.0.0 --title "v1.0.0" --notes "First release"
gh release list

# Repo info
gh repo view
gh repo view --web          # open in browser

# Run/view CI (Actions)
gh run list
gh run view <run-id>
gh run watch <run-id>       # stream live

# Gists
gh gist create file.txt --public
gh gist list

# Switch between accounts / orgs
gh auth status
gh auth switch
```

---

## 11. Common gotchas

- **`git push` rejected (non-fast-forward):** someone else pushed to main. Do `git pull --rebase` then push again.
- **Accidentally committed to main:** `git reset HEAD~1` (local), create a branch, push to branch instead.
- **Pushed a secret:** rotate the secret immediately. Use `git filter-repo` or BFG to scrub history, then force-push — but treat the secret as compromised regardless.
- **Detached HEAD:** you're not on a branch. Run `git switch -c new-branch` to create one at this point, or `git switch main` to return.
- **Wrong base branch on PR:** close the PR and re-open with the correct base, or use `gh pr edit 123 --base correct-branch`.
- **Need to cherry-pick one commit:** `git cherry-pick <hash>` applies just that commit to the current branch.
