#!/bin/bash
# git_compare — Compare current branch with another branch (default: master)
# Shows file-level create/update/delete, commit diff, and diverge point.
#
# Usage:
#   git_compare                  — compare current branch vs master
#   git_compare <branch>         — compare current branch vs <branch>
#   git_compare <a> <b>          — compare branch <a> vs branch <b>
#   git_compare --summary        — summary only (no file listing)
#   git_compare --files          — detailed file listing (default)
#   git_compare --diff           — show actual content diff per file
#   git_compare --diff <file>    — show diff for a specific file only
#
# Save at: workspace/bin/git_compare

set -euo pipefail

# ── Colors ──────────────────────────────────────────────────────────
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
BOLD='\033[1m'
DIM='\033[2m'
NC='\033[0m'

# ── Parse args ──────────────────────────────────────────────────────
MODE="files"
BRANCH_A=""
BRANCH_B=""
DIFF_FILE=""

while [ $# -gt 0 ]; do
  arg="$1"
  case "$arg" in
    --summary) MODE="summary" ;;
    --files)   MODE="files" ;;
    --stat)    MODE="stat" ;;
    --diff)
      MODE="diff"
      shift || true
      # If next arg exists and is not an option flag
      if [ $# -gt 0 ] && ! echo "$1" | grep -q '^-' 2>/dev/null; then
        # If it looks like a file path (has / or .) or exists on disk
        if echo "$1" | grep -q '[/.]' 2>/dev/null || [ -f "$1" ] 2>/dev/null; then
          DIFF_FILE="$1"
          shift || true
        else
          # Not a file — treat as branch arg
          if [ -z "$BRANCH_A" ]; then BRANCH_A="$1"; shift || true
          elif [ -z "$BRANCH_B" ]; then BRANCH_B="$1"; shift || true
          fi
        fi
      fi
      continue
      ;;
    --help|-h)
      echo "Usage: git_compare [options] [branch_a] [branch_b]"
      echo ""
      echo "Options:"
      echo "  --summary   Summary counts only"
      echo "  --files     Detailed file listing (default)"
      echo "  --stat      git diff --stat style"
      echo "  --diff      Show actual content diff for all changed files"
      echo "  --diff <f>  Show diff for a specific file only"
      echo ""
      echo "Diff subcommands (used with --diff):"
      echo "  :all        Show all files (default)"
      echo "  :created    Show only created files"
      echo "  :updated    Show only updated files"
      echo "  :deleted    Show only deleted files"
      echo ""
      echo "Examples:"
      echo "  git_compare                          # current branch vs master"
      echo "  git_compare develop                  # current branch vs develop"
      echo "  git_compare feature/x master         # feature/x vs master"
      echo "  git_compare --diff                   # show content diff for all files"
      echo "  git_compare --diff src/file.ts       # diff one specific file"
      exit 0
      ;;
    --*) echo "Unknown option: $arg"; exit 1 ;;
    *)
      if [ -z "$BRANCH_A" ]; then BRANCH_A="$arg"
      elif [ -z "$BRANCH_B" ]; then BRANCH_B="$arg"
      else echo "Too many branch arguments"; exit 1
      fi
      ;;
  esac
  shift
done

# ── Resolve branches ────────────────────────────────────────────────
if [ -z "$BRANCH_A" ] && [ -z "$BRANCH_B" ]; then
  # No args: current branch vs master
  BRANCH_B=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || { echo "Not in a git repo"; exit 1; })
  BRANCH_A="master"
elif [ -n "$BRANCH_A" ] && [ -z "$BRANCH_B" ]; then
  # One arg: current branch vs specified
  BRANCH_B=$(git rev-parse --abbrev-ref HEAD)
fi

# Verify branches exist
for branch in "$BRANCH_A" "$BRANCH_B"; do
  if ! git rev-parse --verify "$branch" >/dev/null 2>&1; then
    echo -e "${RED}Branch not found: $branch${NC}"
    git branch -a | head -20
    exit 1
  fi
done

# ── Fetch basic info ───────────────────────────────────────────────
A_HEAD=$(git rev-parse --short "$BRANCH_A")
B_HEAD=$(git rev-parse --short "$BRANCH_B")
MERGE_BASE=$(git merge-base "$BRANCH_A" "$BRANCH_B" 2>/dev/null || echo "")
MERGE_BASE_SHORT=$(echo "$MERGE_BASE" | cut -c1-7)

# ── Commits in B not in A ──────────────────────────────────────────
COMMITS_B_NOT_A=$(git log --oneline "${BRANCH_A}..${BRANCH_B}" 2>/dev/null || true)
if [ -n "$COMMITS_B_NOT_A" ]; then
  COMMITS_B_COUNT=$(echo "$COMMITS_B_NOT_A" | wc -l | tr -d ' ')
else
  COMMITS_B_COUNT=0
fi

# ── Commits in A not in B ──────────────────────────────────────────
COMMITS_A_NOT_B=$(git log --oneline "${BRANCH_B}..${BRANCH_A}" 2>/dev/null || true)
if [ -n "$COMMITS_A_NOT_B" ]; then
  COMMITS_A_COUNT=$(echo "$COMMITS_A_NOT_B" | wc -l | tr -d ' ')
else
  COMMITS_A_COUNT=0
fi

# ── File diff ──────────────────────────────────────────────────────
DIFF_NAMES=$(git diff --name-status "$BRANCH_A" "$BRANCH_B" 2>/dev/null)

CREATED=$(echo "$DIFF_NAMES" | awk '/^A/ {print $2}')
UPDATED=$(echo "$DIFF_NAMES" | awk '/^M/ {print $2}')
DELETED=$(echo "$DIFF_NAMES" | awk '/^D/ {print $2}')
RENAMED=$(echo "$DIFF_NAMES" | grep '^R' || true)

count_lines() {
  local val="$1"
  if [ -z "$val" ]; then echo 0; return; fi
  echo "$val" | wc -l | tr -d ' '
}
CREATED_COUNT=$(count_lines "$CREATED")
UPDATED_COUNT=$(count_lines "$UPDATED")
DELETED_COUNT=$(count_lines "$DELETED")
RENAMED_COUNT=$(count_lines "$RENAMED")
TOTAL=$((CREATED_COUNT + UPDATED_COUNT + DELETED_COUNT + RENAMED_COUNT))

# ── Ahead/Behind ───────────────────────────────────────────────────
AHEAD=$(git rev-list --count "${BRANCH_A}..${BRANCH_B}" 2>/dev/null | tr -d '[:space:]' || echo 0)
BEHIND=$(git rev-list --count "${BRANCH_B}..${BRANCH_A}" 2>/dev/null | tr -d '[:space:]' || echo 0)
[ -z "$AHEAD" ] && AHEAD=0
[ -z "$BEHIND" ] && BEHIND=0

# ═══════════════════════════════════════════════════════════════════
# OUTPUT
# ═══════════════════════════════════════════════════════════════════

echo ""
echo -e "${BOLD}═══════════════════════════════════════════════════════════${NC}"
echo -e "${BOLD}  BRANCH COMPARISON${NC}"
echo -e "${BOLD}═══════════════════════════════════════════════════════════${NC}"
echo ""
echo -e "  ${CYAN}${BRANCH_A}${NC} (${A_HEAD})  vs  ${CYAN}${BRANCH_B}${NC} (${B_HEAD})"
if [ -n "$MERGE_BASE" ]; then
  echo -e "  ${DIM}merge-base: ${MERGE_BASE_SHORT}${NC}"
fi
echo ""

# ── Status summary ─────────────────────────────────────────────────
echo -e "${BOLD}── Status ──────────────────────────────────────────────${NC}"
echo -e "  ${CYAN}${BRANCH_B}${NC} is ${GREEN}${AHEAD} ahead${NC} and ${RED}${BEHIND} behind${NC} ${CYAN}${BRANCH_A}${NC}"
echo ""

echo -e "  ${GREEN}+ Created:${NC}  ${CREATED_COUNT} files"
echo -e "  ${YELLOW}~ Updated:${NC}  ${UPDATED_COUNT} files"
echo -e "  ${RED}- Deleted:${NC}  ${DELETED_COUNT} files"
if [ "$RENAMED_COUNT" -gt 0 ]; then
  echo -e "  ${BLUE}→ Renamed:${NC}  ${RENAMED_COUNT} files"
fi
echo -e "  ${BOLD}  Total:${NC}    ${TOTAL} files changed"
echo ""

# ── Commits unique to each branch ──────────────────────────────────
echo -e "${BOLD}── Commits ─────────────────────────────────────────────${NC}"

if [ "$COMMITS_B_COUNT" -gt 0 ]; then
  echo ""
  echo -e "  ${GREEN}In ${CYAN}${BRANCH_B}${GREEN} but not in ${CYAN}${BRANCH_A}${GREEN} (${COMMITS_B_COUNT}):${NC}"
  echo "$COMMITS_B_NOT_A" | while read -r line; do
    echo -e "    ${DIM}${line}${NC}"
  done
fi

if [ "$COMMITS_A_COUNT" -gt 0 ]; then
  echo ""
  echo -e "  ${RED}In ${CYAN}${BRANCH_A}${RED} but not in ${CYAN}${BRANCH_B}${RED} (${COMMITS_A_COUNT}):${NC}"
  echo "$COMMITS_A_NOT_B" | while read -r line; do
    echo -e "    ${DIM}${line}${NC}"
  done
fi

if [ "$COMMITS_B_COUNT" -eq 0 ] && [ "$COMMITS_A_COUNT" -eq 0 ]; then
  echo -e "  ${DIM}No divergent commits — branches are in sync${NC}"
fi
echo ""

# ── File details ───────────────────────────────────────────────────
if [ "$MODE" = "files" ] && [ "$TOTAL" -gt 0 ]; then
  echo -e "${BOLD}── Files ───────────────────────────────────────────────${NC}"

  if [ "$CREATED_COUNT" -gt 0 ]; then
    echo ""
    echo -e "  ${GREEN}+ Created:${NC}"
    echo "$CREATED" | while read -r f; do
      echo -e "    ${GREEN}+${NC} $f"
    done
  fi

  if [ "$UPDATED_COUNT" -gt 0 ]; then
    echo ""
    echo -e "  ${YELLOW}~ Updated:${NC}"
    echo "$UPDATED" | while read -r f; do
      echo -e "    ${YELLOW}~${NC} $f"
    done
  fi

  if [ "$DELETED_COUNT" -gt 0 ]; then
    echo ""
    echo -e "  ${RED}- Deleted:${NC}"
    echo "$DELETED" | while read -r f; do
      echo -e "    ${RED}-${NC} $f"
    done
  fi

  if [ "$RENAMED_COUNT" -gt 0 ]; then
    echo ""
    echo -e "  ${BLUE}→ Renamed:${NC}"
    echo "$RENAMED" | while read -r line; do
      old=$(echo "$line" | awk '{print $2}')
      new=$(echo "$line" | awk '{print $3}')
      echo -e "    ${BLUE}→${NC} $old ${DIM}→${NC} $new"
    done
  fi
  echo ""

elif [ "$MODE" = "stat" ] && [ "$TOTAL" -gt 0 ]; then
  echo -e "${BOLD}── Diff stat ───────────────────────────────────────────${NC}"
  echo ""
  git diff --stat "$BRANCH_A" "$BRANCH_B"
  echo ""
fi

# ── Diff mode — show actual content changes ───────────────────────
if [ "$MODE" = "diff" ] && [ "$TOTAL" -gt 0 ]; then
  echo -e "${BOLD}── Content Diff ────────────────────────────────────────${NC}"
  echo ""

  if [ -n "$DIFF_FILE" ]; then
    # Single file mode
    echo -e "  ${BOLD}${DIFF_FILE}${NC}"
    echo -e "  ${DIM}──────────────────────────────────────────────────${NC}"
    git diff --color=always "$BRANCH_A" "$BRANCH_B" -- "$DIFF_FILE" 2>/dev/null || \
      echo -e "  ${DIM}(no diff available for this file)${NC}"
    echo ""
  else
    # All files — show diff for each
    ALL_CHANGED=$(echo "$DIFF_NAMES" | awk '{print $2}')
    FILE_INDEX=0

    # Build array of changed files
    CHANGED_FILES=()
    while IFS= read -r f; do
      [ -n "$f" ] && CHANGED_FILES+=("$f")
    done <<< "$ALL_CHANGED"

    for f in "${CHANGED_FILES[@]}"; do
      FILE_INDEX=$((FILE_INDEX + 1))
      # Get change type
      CHANGE_TYPE=$(echo "$DIFF_NAMES" | grep "$f" | head -1 | awk '{print $1}')
      case "$CHANGE_TYPE" in
        A*) TYPE_LABEL="${GREEN}[created]${NC}" ;;
        M*) TYPE_LABEL="${YELLOW}[updated]${NC}" ;;
        D*) TYPE_LABEL="${RED}[deleted]${NC}" ;;
        R*) TYPE_LABEL="${BLUE}[renamed]${NC}" ;;
        *)  TYPE_LABEL="" ;;
      esac

      # Get insert/delete counts
      STAT_LINE=$(git diff --numstat "$BRANCH_A" "$BRANCH_B" -- "$f" 2>/dev/null | head -1)
      ADDITIONS=$(echo "$STAT_LINE" | awk '{print $1}')
      DELETIONS=$(echo "$STAT_LINE" | awk '{print $2}')

      echo -e "  ${BOLD}${FILE_INDEX}/${#CHANGED_FILES[@]}${NC} $TYPE_LABEL ${CYAN}$f${NC}"
      if [ "$ADDITIONS" != "-" ] && [ "$DELETIONS" != "-" ]; then
        echo -e "  ${DIM}+${ADDITIONS} additions, -${DELETIONS} deletions${NC}"
      fi
      echo -e "  ${DIM}──────────────────────────────────────────────────${NC}"

      # Show the actual diff for this file
      if [ "$CHANGE_TYPE" = "D" ]; then
        # For deleted files, show what was removed
        echo -e "  ${DIM}(file deleted — content was:)${NC}"
        git show "${BRANCH_A}:${f}" 2>/dev/null | head -50 | while IFS= read -r line; do
          echo -e "  ${RED}-${NC} $line"
        done
        REMAINING=$(git show "${BRANCH_A}:${f}" 2>/dev/null | wc -l | tr -d ' ')
        if [ "$REMAINING" -gt 50 ]; then
          echo -e "  ${DIM}... ($((REMAINING - 50)) more lines)${NC}"
        fi
      elif [ "$CHANGE_TYPE" = "A" ]; then
        # For created files, show the new content
        echo -e "  ${DIM}(new file — content:)${NC}"
        git show "${BRANCH_B}:${f}" 2>/dev/null | head -50 | while IFS= read -r line; do
          echo -e "  ${GREEN}+${NC} $line"
        done
        REMAINING=$(git show "${BRANCH_B}:${f}" 2>/dev/null | wc -l | tr -d ' ')
        if [ "$REMAINING" -gt 50 ]; then
          echo -e "  ${DIM}... ($((REMAINING - 50)) more lines)${NC}"
        fi
      else
        # For updated files, show the diff
        git diff --color=always "$BRANCH_A" "$BRANCH_B" -- "$f" 2>/dev/null
      fi
      echo ""
    done

    echo -e "  ${DIM}Tip: Use 'git_compare --diff <filepath>' to see one file only${NC}"
    echo ""
  fi
fi

# ── Branch-affecting commands ──────────────────────────────────────
echo -e "${BOLD}── Safe commands (no branch changes) ──────────────────${NC}"
echo ""
echo -e "  ${DIM}git diff ${BRANCH_A}..${BRANCH_B}              # view full diff${NC}"
echo -e "  ${DIM}git log ${BRANCH_A}..${BRANCH_B} --oneline      # view commits${NC}"
echo -e "  ${DIM}git diff ${BRANCH_A}..${BRANCH_B} -- <file>     # diff a specific file${NC}"
echo ""

if [ "$COMMITS_B_COUNT" -gt 0 ]; then
  echo -e "${BOLD}── Commands that affect ${CYAN}${BRANCH_B}${NC} ────────────────────"
  echo ""
  echo -e "  ${YELLOW}git checkout ${BRANCH_B}                   # switch to this branch${NC}"
  echo -e "  ${YELLOW}git stash                                # stash before switching${NC}"

  if [ "$BEHIND" -gt 0 ]; then
    echo ""
    echo -e "  ${RED}git merge ${BRANCH_A}                        # merge ${BRANCH_A} into ${BRANCH_B}${NC}"
    echo -e "  ${RED}git rebase ${BRANCH_A}                       # rebase ${BRANCH_B} onto ${BRANCH_A}${NC}"
    echo -e "  ${DIM}  ↑ These rewrite ${BRANCH_B} history${NC}"
  fi

  if git rev-parse --verify "origin/${BRANCH_B}" >/dev/null 2>&1; then
    echo ""
    echo -e "  ${RED}git push origin ${BRANCH_B}                  # push ${BRANCH_B} to remote${NC}"
    echo -e "  ${RED}git push --force origin ${BRANCH_B}          # force push ${NC}${RED}(destructive)${NC}"
  else
    echo ""
    echo -e "  ${GREEN}git push -u origin ${BRANCH_B}              # push new branch to remote${NC}"
  fi
  echo ""
fi

if [ "$COMMITS_A_COUNT" -gt 0 ]; then
  echo -e "${BOLD}── Commands that affect ${CYAN}${BRANCH_A}${NC} ────────────────────"
  echo ""
  echo -e "  ${RED}git checkout ${BRANCH_A}                       # switch to ${BRANCH_A}${NC}"
  echo -e "  ${RED}git merge ${BRANCH_B}                          # merge ${BRANCH_B} into ${BRANCH_A}${NC}"
  echo -e "  ${RED}git push origin ${BRANCH_A}                    # push ${BRANCH_A}${NC}"
  echo ""
fi

if [ "$AHEAD" -gt 0 ] && [ "$BEHIND" -eq 0 ]; then
  echo -e "${BOLD}── Quick actions ──────────────────────────────────────${NC}"
  echo ""
  echo -e "  ${GREEN}git checkout ${BRANCH_A} && git merge ${BRANCH_B}  # fast-forward merge${NC}"
  echo ""
fi

if [ "$AHEAD" -gt 0 ] && [ "$BEHIND" -gt 0 ]; then
  echo -e "${BOLD}── Quick actions ──────────────────────────────────────${NC}"
  echo ""
  echo -e "  ${YELLOW}git checkout ${BRANCH_A} && git merge ${BRANCH_B}  # merge (creates merge commit)${NC}"
  echo -e "  ${YELLOW}git checkout ${BRANCH_B} && git rebase ${BRANCH_A} # rebase (linear history)${NC}"
  echo -e "  ${DIM}  ↑ Branches have diverged — merge or rebase needed${NC}"
  echo ""
fi

echo -e "${BOLD}═══════════════════════════════════════════════════════════${NC}"
echo ""
