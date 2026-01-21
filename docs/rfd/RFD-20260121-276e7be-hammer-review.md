# RFD: Hammer Review 276e7be

**Commit:** 276e7be0a080813edc0388ae39844fc5f50f9cbb

**Date:** 2026-01-21

## Summary

This commit restructures the "Run hammer review" step in `.github/workflows/hammer-review.yml` to fix the broken heredoc issues identified in RFD-20260121-1f1a178. The previous commit introduced indented heredocs that caused shell parse errors. This commit replaces the inline approach with a properly structured remote script execution pattern using `sprite exec`.

**Files changed:** `.github/workflows/hammer-review.yml`

## Changes Analysis

### Key Changes

1. **Removed broken inline execution** - The previous approach tried to run `claude` directly in the runner with a malformed heredoc
2. **Added sprite CLI authentication** - New call to `sprite auth setup --token`
3. **Added remote script execution** - Introduced a `REMOTE_SCRIPT` heredoc that:
   - Clones the repository if not present
   - Fetches and checks out the target commit
   - Decodes and executes the prompt via `claude`
4. **Fixed auth header format** - Changed from `"Authorization: Bearer ${TOKEN}"` to `"Authorization: token ${TOKEN}"`

## Findings

### 1. LOW: Potential issue with `base64 -d -i` flag on some systems

**Location:** `.github/workflows/hammer-review.yml:101`

**Description:** The command uses `base64 -d -i`:
```bash
PROMPT="\$(printf '%s' \"\$PROMPT_B64\" | base64 -d -i)"
```

The `-i` flag (ignore garbage) is a GNU coreutils extension and may not be available on all systems. However, since this runs inside a sprite VM which likely uses a Linux environment with GNU coreutils, this is a low-risk issue.

**Impact:** Minimal - the sprite environment likely supports this flag.

### 2. INFO: Hardcoded workspace path

**Location:** `.github/workflows/hammer-review.yml:88-92`

**Description:** The script uses a hardcoded path `/home/sprite/workspace`:
```bash
cd /home/sprite/workspace
if [ ! -d "/home/sprite/workspace/${REPO_NAME}/.git" ]; then
  git clone https://github.com/${REPO_FULL}.git "${REPO_NAME}"
fi
```

This is appropriate for the sprite VM environment and matches the expected workspace structure.

**Impact:** None - this is the correct pattern for sprite VMs.

### 3. INFO: Git checkout strategy handles both main and non-main repos

**Location:** `.github/workflows/hammer-review.yml:94-98`

**Description:** The script checks if `refs/heads/main` exists before trying to checkout main:
```bash
if git show-ref --verify --quiet "refs/heads/main"; then
  git checkout main
fi
git checkout ${SHA}
git reset --hard ${SHA}
```

This is a defensive pattern that handles repositories where `main` may not exist (e.g., repos using `master` or other branch names). The subsequent checkout and reset to `${SHA}` ensures the correct commit is checked out regardless.

**Impact:** None - this is good defensive coding.

### 4. LOW: Missing error handling for curl failure

**Location:** `.github/workflows/hammer-review.yml:80`

**Description:** While curl uses `-fsSL` (fail silently on HTTP errors), if the network request fails or the Python script has an error, `PROMPT_B64` could be empty or contain error output:
```bash
PROMPT_B64="$(curl -fsSL -H "$AUTH_HEADER" https://raw.githubusercontent.com/r-mccarty/agent-harness/main/scripts/hammer/build_prompt.py | python | tr -d '\n')"
```

The script could benefit from validating that `PROMPT_B64` is non-empty before proceeding.

**Impact:** Low - the `-f` flag should cause curl to fail fast on HTTP errors, and `set -euo pipefail` in the remote script would catch downstream failures.

## Positive Changes

1. **Fixes critical bugs from previous commit** - The heredoc is now properly structured with the `EOF` delimiter at the correct position
2. **Proper sprite workflow** - Uses `sprite exec` to run the review inside the sprite VM, which is the intended execution model
3. **Auth header format correction** - GitHub raw content API prefers `Authorization: token` over `Authorization: Bearer` for PAT tokens

## Recommended Actions

1. **No blocking issues** - This commit can proceed as it fixes critical bugs from the previous commit.

2. **Optional: Add PROMPT_B64 validation** - Consider adding a check:
   ```bash
   if [ -z "$PROMPT_B64" ]; then
     echo "Failed to fetch or build prompt"
     exit 1
   fi
   ```

3. **Monitor workflow execution** - Verify the workflow succeeds on the next push without `[skip-hammer]`.

## Conclusion

This commit successfully addresses the critical heredoc issues from commit 1f1a178. The restructured approach using `sprite exec` with a properly formatted shell heredoc is correct and follows the expected sprite VM execution pattern. No critical or high-severity issues were found.
