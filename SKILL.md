---
name: code-reviewer
description: |
  Review code changes against platform-specific rules (Android/iOS) plus shared general rules.
  Supports: uncommitted changes, staged changes, specific commits, commit ranges, and branch diffs.
  Optionally generates a styled HTML report. Use when user mentions: "review", "code review",
  "帮我看看代码", "check my changes", provides a commit hash, or asks to review before committing.
  Auto-detects platform (Android/iOS/General) from project markers.
---

# Code Reviewer

Review code changes and report issues by severity.

## Rule Files

Read from `references/` relative to this skill directory. Always load general + detected platform:
- `references/review-general.md` — always
- `references/review-android.md` — Android (Kotlin/Java)
- `references/review-ios.md` — iOS (ObjC/Swift)

## Severity Definitions (hard rules)

| Level | Criteria | Action |
|-------|----------|--------|
| **P0** | Will cause: crash, data loss/corruption, security vulnerability, deadlock, infinite loop | **Must fix before merge** |
| **P1** | May cause: race condition under specific timing, resource leak under edge case, silent data error, uncovered error path that breaks UX | Should fix |
| **P2** | Code quality: naming, structure, minor redundancy, non-critical style | Nice to have |

When uncertain between two levels, choose the **lower** severity (less alarm).

## Workflow

### 1. Determine review scope

Detect from user message. Priority order:

| User says | Scope | Git command |
|-----------|-------|-------------|
| "review" (no qualifier) | Uncommitted changes (staged + unstaged) | `git diff HEAD` |
| "review staged" / "review 暂存" | Staged only | `git diff --cached` |
| "review \<sha\>" / "cid \<sha\>" | Single commit | `git show <sha>` |
| "review \<sha1\>..\<sha2\>" | Commit range | `git diff <sha1>..<sha2>` |
| "review branch \<name\>" | Branch vs main/master | `git diff main...<name>` |
| "review last N commits" | Recent N commits | `git diff HEAD~N..HEAD` |

If scope is ambiguous, default to **uncommitted changes** — this is the most common use case.

### 2. Resolve repo

Use current working directory. Validate:
```bash
git rev-parse --show-toplevel 2>/dev/null
```
If not a git repo, ask user for path.

### 3. Detect platform

Check repo root for markers (in order):

| Platform | Markers (any match) |
|----------|-------------------|
| iOS | `*.xcodeproj`, `*.xcworkspace`, `Podfile`, `Package.swift` |
| Android | `build.gradle*`, `settings.gradle*`, `AndroidManifest.xml`, `gradlew` |
| General | Neither matches |

### 4. Pre-flight checks

**Diff size**: Run `git diff --stat` first.
- \> 5000 lines changed → warn user, offer to focus on specific paths
- \> 10000 lines → refuse unless user confirms (context will be too large for quality review)

**File filter** — skip from review (show in stats summary):
- Binary files, images, fonts, videos
- Generated: `*.pb.go`, `*.generated.*`, `R.java`, `BuildConfig.java`, `*.g.dart`
- Lock files: `package-lock.json`, `yarn.lock`, `Podfile.lock`, `*.lock`
- Vendor/deps: `vendor/`, `node_modules/`, `Pods/`, `build/`, `.gradle/`
- IDE: `.idea/`, `.vscode/`, `*.xcuserdata`, `*.iml`

### 5. Gather context

For each changed file, beyond the diff itself:
- Read the **full function/method** surrounding each change (not just diff lines)
- If a public API signature changed, search for callers: `git grep "<function_name>"` to assess impact
- Check the commit message for intent — findings should be about **bugs**, not about **disagreeing with the approach**

### 6. Load rules & review

Read `references/review-general.md` + platform-specific file.

Apply rules to each changed file. For every finding, include ALL fields:

| Field | Description |
|-------|-------------|
| severity | `P0` / `P1` / `P2` (follow hard rules above) |
| title | One-line summary |
| file | File path |
| line | Line number or range |
| dimension | Category (e.g. 线程安全, 内存管理, 逻辑正确性) |
| rule_source | `general` / `android` / `ios` |
| problem | What's wrong and why it matters |
| code | **Exact** original lines from diff (non-empty) |
| code_lang | Language identifier |
| fix_suggestion | How to fix (text) |
| fix_code | Concrete fix code (non-empty, compilable) |
| fix_lang | Language of fix |

**Quality rules:**
- Don't report issues in unchanged code (unless the change directly breaks it)
- Don't suggest "might want to consider..." — every finding must be a concrete problem
- If no issues found, say so. Empty review is a valid result.

### 7. Output

**Default: Terminal markdown** — print directly in chat:

```markdown
## Code Review: <repo_name>
**Scope**: <description>  |  **Platform**: Android  |  **Files**: 12  |  **+247 / -89**

### P0 · Must Fix (2)
#### 1. [线程安全] ConcurrentModificationException risk
📄 `app/src/.../ViewModel.kt:45-52`
**Problem**: ...
**Fix**: ...

### P1 · Should Fix (3)
...

### P2 · Nice to Have (1)
...

**Summary**: 2 P0 / 3 P1 / 1 P2 — Fix P0 before merge.
```

**Optional: HTML report** — only when user asks ("生成报告", "generate report", "HTML"):
```bash
TS=$(date +%Y%m%d_%H%M%S)
REPORT_DIR="<repo_path>/.code-reviews"
mkdir -p "$REPORT_DIR"
python3 <skill_dir>/scripts/render_report.py "$JSON" "$REPORT_DIR/review_${TS}.html"
open "$REPORT_DIR/review_${TS}.html"
```

Add `.code-reviews/` to `.gitignore` if not already there.

## Review Modes

### Standard Review (default)
Manual trigger — user says "review" and gets results in chat.

### Security-Focused Review
When user says "security review" or "安全审查", apply stricter lens:
- Focus on OWASP Top 10, injection, auth bypass, secrets exposure
- Ignore style/naming issues entirely
- All security findings are P0 or P1, never P2

### Quick Review
When user says "quick review" or "快速看看":
- Only report P0 issues
- Skip P1/P2 entirely
- Fastest path to "can I merge this?"

## Smart Behaviors

**Repeated patterns**: If the same issue appears 3+ times across files, report it once with
"Found in N files" instead of N separate findings. List all affected files.

**Related changes**: When a function signature changes, automatically check if callers are
updated. Report missing caller updates as P0 (will cause compile error or runtime crash).

**Test coverage hint**: If the changed code has no corresponding test changes and the repo
has a test directory, mention it as P2 (not a finding, just a note at the end).

## Safety

- **Read-only**: Never modify repo code. Only create `.code-reviews/` for reports.
- **No destructive git**: Never reset, clean, force-push, or amend.
- **Conservative severity**: When unsure, choose lower severity. False P0 alarms erode trust.
