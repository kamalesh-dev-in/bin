# git_compare — Branch Comparison Tool

Compare two git branches and see what's different: files created/updated/deleted, divergent commits, and commands that affect each branch.

## Usage

```bash
git_compare                  # current branch vs master
git_compare <branch>         # current branch vs <branch>
git_compare <a> <b>          # branch <a> vs branch <b>
git_compare --summary        # summary counts only
git_compare --stat           # git diff --stat style
git_compare --files          # detailed file listing (default)
git_compare --diff           # show actual content diff for all changed files
git_compare --diff <file>    # show diff for a specific file only
git_compare --help           # show help
```

## Examples

```bash
# On branch feature/auth, compare against master
git_compare

# Compare current branch against develop
git_compare develop

# Compare two specific branches
git_compare feature/auth master

# Quick summary without file listing
git_compare --summary

# See line-level diff stats
git_compare --stat

# Show actual content changes for all files
git_compare --diff

# Show diff for one specific file
git_compare --diff backend/src/services/agent_lifecycle.ts
```

## Output Sections

### Status

Shows ahead/behind commit counts and file change summary:
- `+ Created` — files that exist in branch B but not in branch A
- `~ Updated` — files modified between the two branches
- `- Deleted` — files that exist in branch A but not in branch B
- `→ Renamed` — files renamed between branches

### Commits

Lists commits unique to each branch (divergent commits). This tells you what work exists on one side but not the other.

### Content Diff (`--diff` mode)

Shows the actual line-by-line content changes for each file:
- **Created files** — shows first 50 lines of new content with `+` prefix
- **Updated files** — shows full `git diff` output with additions/deletions highlighted
- **Deleted files** — shows first 50 lines of removed content with `-` prefix
- Each file shows additions/deletions count and change type label
- Files are numbered (e.g., `2/12`) for easy tracking

### Files (default mode)

Lists every file changed between the branches, grouped by type (created, updated, deleted).

### Safe Commands

Read-only commands that don't modify either branch:
- `git diff branchA..branchB` — view full diff
- `git log branchA..branchB --oneline` — view commits
- `git diff branchA..branchB -- <file>` — diff a specific file

### Branch-Affecting Commands

Commands that modify one or both branches, with risk indicators:
- `git checkout` — switch branches
- `git merge` — merge one branch into another
- `git push` — push to remote
- `git push --force` — force push (marked destructive)

### Quick Actions

Suggested commands based on the divergence state:
- **Fast-forward** — when one branch is ahead with no divergence
- **Merge/rebase** — when branches have diverged

## How It Works

1. Resolves branch names (defaults to current branch vs master)
2. Finds the merge-base (common ancestor commit)
3. Counts commits ahead/behind using `git rev-list`
4. Classifies file changes using `git diff --name-status`
5. Identifies divergent commits using `git log A..B` and `git log B..A`
6. Detects if branches have a remote tracking branch
7. Suggests merge or rebase based on divergence state

## Location

```
workspace/bin/git_compare      # the script
workspace/bin/git_compare.md   # this file
```
