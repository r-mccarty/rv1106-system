# RFD: Hammer Review 7bf9158

**Commit:** 7bf9158dc61521e52e8cb82197ddebc8d92f7bd2

**Date:** 2026-01-21

## Summary

This commit fixes a shell variable expansion bug in the anvil-review workflow. The `$PROMPT` variable was being expanded prematurely during heredoc creation on the GitHub runner, where it was undefined, rather than on the remote sprite where it is assigned. The fix escapes the variable as `\$PROMPT` so expansion occurs at the correct time.

**Files changed:** `.github/workflows/anvil-review.yml` (1 line modified)

## Changes Analysis

### The Bug

In a heredoc passed to `sprite exec`, variables need to be carefully escaped to control when they expand:
- **Unescaped** (`$VAR`): Expands when the heredoc is created (GitHub runner)
- **Escaped** (`\$VAR`): Expands when the script runs (remote sprite)

The problematic code sequence:
```bash
PROMPT_B64="${PROMPT_B64}"                    # $PROMPT_B64 expands on GitHub runner (correct)
PROMPT="\$(printf '%s' \"\$PROMPT_B64\" | base64 -d -i)"  # \$ - expands on sprite (correct)
printf '%s' "$PROMPT" | codex exec ...        # $PROMPT expands on GitHub runner (BUG!)
```

The `PROMPT` variable is assigned on line 103 *during remote execution*, but line 104 was trying to use it via `$PROMPT` which would expand on the GitHub runner—where `PROMPT` is undefined, resulting in an empty string being piped to codex.

### The Fix

```diff
-          printf '%s' "$PROMPT" | codex exec --dangerously-bypass-approvals-and-sandbox -
+          printf '%s' "\$PROMPT" | codex exec --dangerously-bypass-approvals-and-sandbox -
```

By escaping as `\$PROMPT`, the variable now correctly expands on the remote sprite after the base64 decode assigns its value.

## Findings

### 1. LOW: Fix is correct and complete

**Location:** `.github/workflows/anvil-review.yml:104`

**Description:** The escaping fix is correct. The change ensures `$PROMPT` is preserved in the heredoc and expanded at runtime on the sprite, matching the pattern used for `$PROMPT_B64` on line 103.

**Verification:** The surrounding context shows consistent escaping:
- Line 102: `PROMPT_B64="${PROMPT_B64}"` - Intentionally expands on runner to embed the value
- Line 103: Uses `\$` to defer expansion to sprite
- Line 104 (fixed): Now also uses `\$` to defer expansion to sprite

**Impact:** Without this fix, the anvil review workflow would pipe an empty string to codex, causing the review to fail or produce no meaningful output.

### 2. INFO: Testing recommended

**Description:** This was likely a copy-paste error from the hammer-review workflow or a missed escaping during initial implementation. The fix appears straightforward, but integration testing should confirm the prompt is now correctly passed to codex.

## Recommended Actions

1. **Verify in CI** - Monitor the next anvil-review run to confirm the prompt is successfully passed to codex and the review completes.

2. **Consider shellcheck** - Adding shellcheck to the workflow validation could catch similar escaping issues in the future, though heredoc escaping in embedded scripts is notoriously tricky to lint.

## Conclusion

This is a correct and necessary bug fix. The single-character change (`$PROMPT` → `\$PROMPT`) fixes a critical variable expansion timing issue that would have caused the anvil review workflow to fail. The fix follows the established escaping pattern already used elsewhere in the same heredoc. No concerns with this change.
