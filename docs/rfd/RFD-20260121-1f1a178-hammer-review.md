# RFD: Hammer Review 1f1a178

**Commit:** 1f1a178a0a8859d7195b0332f1e8c361a02ee197

**Date:** 2026-01-21

## Summary

This commit reformats the heredoc blocks in `.github/workflows/hammer-review.yml` by adding indentation to align the Python and shell script content with the surrounding YAML structure. The commit message states "fix: align hammer review heredocs".

**Files changed:** `.github/workflows/hammer-review.yml`

## Findings

### 1. CRITICAL: Python heredoc indentation causes IndentationError

**Location:** `.github/workflows/hammer-review.yml:78-111`

**Description:** The Python code inside the heredoc starting at line 78 is now indented with leading spaces. Python is whitespace-sensitive and expects the first statement in a module to have zero indentation. When this workflow runs, the Python interpreter will fail with:

```
IndentationError: unexpected indent
```

**Evidence from diff:**
```diff
-import base64
-import os
+          import base64
+          import os
```

The heredoc delimiter `'PY'` is quoted, meaning content is passed literally to Python with all leading whitespace preserved.

**Impact:** The workflow will fail on every execution, preventing all hammer reviews from running.

### 2. CRITICAL: Shell heredoc terminator is indented

**Location:** `.github/workflows/hammer-review.yml:117-135`

**Description:** The heredoc uses `<<EOF` syntax (not `<<-EOF`), which requires the terminator `EOF` to appear at the start of a line with no leading whitespace. In the modified code, `EOF` is indented:

```diff
-EOF
-)
+          EOF
+          )
```

This will cause the shell to not recognize the heredoc terminator, resulting in a parse error or the heredoc consuming subsequent lines until EOF or an error.

**Impact:** The workflow will fail with a shell parse error.

### 3. CRITICAL: Python heredoc terminator is indented

**Location:** `.github/workflows/hammer-review.yml:110-111`

**Description:** Similar to finding #2, the `PY` heredoc terminator is now indented:

```diff
-PY
-)"
+          PY
+          )"
```

With `<<'PY'` syntax, the terminator must be on a line by itself with no leading whitespace.

**Impact:** The heredoc will not terminate correctly, causing shell parse errors.

## Root Cause

The change appears to be an attempt to improve code formatting/readability by aligning heredoc content with YAML indentation. However, heredocs in bash have strict requirements:

1. Content is taken literally (including leading whitespace)
2. The terminator must appear at column 0 (unless `<<-` is used with tabs only)
3. Python specifically requires the first line of code to have zero indentation

## Recommended Actions

1. **Revert this commit immediately** - This commit breaks the hammer review workflow entirely and should be reverted to restore functionality.

2. **Alternative formatting approaches** (if indentation is desired):
   - Use `<<-EOF` with **tabs** (not spaces) for shell heredocs, which strips leading tabs
   - For Python code, either:
     - Keep the heredoc content unindented (current working approach)
     - Use `textwrap.dedent()` inside Python to strip common leading whitespace
     - Move the Python script to a separate file

3. **Add workflow testing** - Consider adding a CI job that validates workflow YAML syntax and runs a dry-run or lint check before merging changes to workflow files.

## Testing Verification

The bugs were verified locally:

```bash
# Python heredoc test - fails with IndentationError
$ python - <<'PY'
          import base64
          print("test")
          PY
  File "<stdin>", line 1
    import base64
IndentationError: unexpected indent

# Shell heredoc terminator test - fails to find delimiter
$ TEST=$(cat <<EOF
          echo test
          EOF
)
bash: warning: here-document delimited by end-of-file (wanted `EOF')
```
