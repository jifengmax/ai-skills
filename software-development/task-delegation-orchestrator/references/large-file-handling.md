# Frontend / Large File Handling Strategies

Subagents frequently fail on large frontend HTML files. This reference documents the thresholds and strategies.

## Why subagents fail on large HTML files

- **Timeout** (600s hard limit) — reading, searching, patching a 2000+ line file takes too long
- **Output truncation** — the file content overflow the subagent's output limit, causing `Response truncated due to output length limit` errors
- **Patch conflicts** — multiple subagents patching the same HTML file create race conditions
- **Context window overflow** — the subagent fills its context with the entire file and has no room for reasoning

## Size thresholds

| File size | Lines (approx) | Strategy |
|-----------|---------------|----------|
| < 30 KB   | < 600         | Safe for subagent delegation |
| 30-100 KB | 600-2000      | Risk of timeout. Consider handling yourself. |
| > 100 KB  | > 2000        | **Must** handle yourself. Subagent will almost certainly fail. |

## Strategies

### Strategy A: Handle yourself (preferred for large files)
Use `read_file` + `patch` directly. For large files:
1. Use `search_files` to find the exact insertion anchor (HTML tag, CSS selector, JS function)
2. Read only the surrounding ~20 lines with `read_file(path, offset=N, limit=20)`
3. Write the patch with exact old_string and new_string

### Strategy B: Delegate with precise anchors
If you must delegate, provide the exact anchor points in context:
```
在 dashboard.html 的第 452 行的 <div class="nav-right"> 后面插入角色选择器。
在第 1920 行的 </script> 前插入 JS 函数。
```

### Strategy C: Split into a new file
For major rewrites, create a new standalone HTML file that imports the same CSS and JS patterns. Then link it from the existing page structure.

## Quick reference commands

```bash
# Check file size
wc -c frontend/dashboard.html
wc -l frontend/dashboard.html

# Find file structure
grep -n "<section\|<div class\|<nav\|function " frontend/dashboard.html | head -30

# Check current line count
wc -l frontend/dashboard.html
```
