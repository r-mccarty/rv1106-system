# RFD: Hammer Review 0e44043

**Commit:** 0e4404392b70d324e8fe9da4819b9e291d3c3d7e

**Date:** 2026-01-21

## Summary

This commit adds a safeguard to the anvil-review workflow that removes stale `.git/index.lock` files before attempting git operations. Stale lock files can occur when a previous git process was interrupted (e.g., sprite timeout, SIGKILL, or crash), leaving the lock in place and blocking subsequent git operations like `git fetch`.

**Files changed:** `.github/workflows/anvil-review.yml` (3 lines added)

## Changes Analysis

### The Change

```diff
          cd "/home/sprite/workspace/${REPO_NAME}"
+          if [ -f ".git/index.lock" ]; then
+            rm -f .git/index.lock
+          fi
          git fetch origin
```

The addition checks for the existence of `.git/index.lock` and removes it before running `git fetch origin`. This is a defensive measure to prevent workflow failures caused by leftover lock files.

### Context

Git uses `.git/index.lock` as a mutex to prevent concurrent modifications to the index. Normally, git removes this file when operations complete. However, if a process is killed unexpectedly (common in CI environments with timeouts), the lock file remains, causing subsequent git commands to fail with:

```
fatal: Unable to create '/path/to/repo/.git/index.lock': File exists.
```

### Why This Fix is Needed

The sprite environment persists the workspace between runs. If a previous anvil-review run was interrupted during a git operation, the lock file would remain and cause subsequent runs to fail. This is particularly relevant given:
- The workflow uses `concurrency.cancel-in-progress: false` (line 17), but external factors can still interrupt execution
- Sprite VMs may be reused across runs, preserving workspace state

## Findings

### 1. LOW: Conditional check is redundant but harmless

**Location:** `.github/workflows/anvil-review.yml:95-97`

**Description:** The `if [ -f ... ]` check is technically unnecessary since `rm -f` already handles non-existent files silently. However, the explicit check makes the intent clear and has no functional downside.

**Alternative:** Could simplify to just `rm -f .git/index.lock` without the conditional.

**Impact:** None—the code works correctly either way. The explicit check is arguably more readable.

### 2. INFO: Addresses a real operational issue

**Description:** This is a pragmatic fix for a known git behavior that can cause CI failures. The pattern of removing stale lock files before git operations is common in CI/CD systems where workspace persistence and process interruption are factors.

### 3. INFO: No risk of data loss

**Description:** Removing `index.lock` is safe when no git operation is actively running. In this workflow context:
- The script runs with `set -euo pipefail` (line 89)
- This is the first git operation after entering the directory
- The sprite runs workflows serially (no concurrent git operations on the same repo)

The lock file is purely a mutex mechanism and contains no data that would be lost.

## Recommended Actions

1. **Monitor workflow runs** - Verify that this fix resolves any lock-related failures that may have been occurring.

2. **Consider similar fix for hammer-review** - If the hammer-review workflow (`hammer-review.yml`) has the same workspace persistence pattern, it may benefit from the same safeguard.

## Conclusion

This is a correct and low-risk defensive fix. Removing stale `.git/index.lock` files before git operations is a standard practice in persistent CI environments. The change addresses a real operational issue without introducing new risks or side effects.
