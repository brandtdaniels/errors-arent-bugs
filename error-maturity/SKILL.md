---
name: error-handling-maturity
description: "Analyze error handling maturity in a codebase using the Error Handling Maturity Model — a UX-centered framework (Levels 0–3) (Silent → Acknowledged → Identifiable → Guided). Use this skill whenever the user asks to audit, assess, analyze, or review error handling in code, a project, a repo, or a directory. Also trigger when the user mentions 'error handling maturity', 'error handling audit', 'how mature is our error handling', 'error handling review', 'assess error handling', or references the Error Handling Maturity Model. Supports Swift, Kotlin, and TypeScript/JavaScript. Accepts a local directory path or a GitHub repo URL."
---

# Error Handling Maturity Analyzer

Analyze a codebase's error handling against the Error Handling Maturity Model — a UX-centered framework (Levels 0–3).

## The Model (quick reference)

| Level | Name | User Experience | What the User Thinks |
|-------|------|-----------------|----------------------|
| 0 | Silent | Nothing happens. The action silently fails. | "Did I tap that? Is this thing frozen? Did it work or not?" |
| 1 | Acknowledged | User knows something broke but can't act on it. | "OK, it broke. Now what? I guess I'll try again and hope for the best." |
| 2 | Identifiable | User gets a code/reference they can use with support or docs. | "I can look this up or tell support exactly what happened." |
| 3 | Guided | App offers a next step. User stays in motion. | "That didn't work, but I know what to do next." |

## Key Framing: Errors Are Not Defects

Poor error handling is almost never caused by bad engineering. It is caused by the **absence of sad path specifications.** When a product team specifies a feature, they describe the happy path in detail. What happens when the download fails? When the device runs out of storage? When connectivity drops mid-flow? These are foreseeable conditions -- but they are rarely specified.

Without a specification, error handling is left to individual engineers' judgment. The result is inconsistency across flows and across platforms.

**This reframing matters when communicating findings:** Filing a "bug" for poor error handling implies something was built wrong. In reality, something was never built at all. Error handling work isn't fixing defects -- it's completing features.

## Before Starting

Read `references/patterns.md` in this skill's directory for the full pattern catalog per language. It contains concrete code examples for each level in Swift, Kotlin, and TypeScript/JavaScript, plus cross-language signals for remote logging, error wrapping, and recovery patterns.

## Step 1: Acquire the Code

Determine the source from the user's request:

### Local directory
```bash
# Verify the path exists and list structure
ls -la <path>
find <path> -type f \( -name "*.swift" -o -name "*.kt" -o -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" \) | head -50
```

### GitHub repo
```bash
# Clone to working directory
git clone --depth 1 <repo-url> /home/claude/repo-analysis
cd /home/claude/repo-analysis
find . -type f \( -name "*.swift" -o -name "*.kt" -o -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" \) | head -50
```

If the repo is large (many hundreds of files), ask the user if they want to focus on a subdirectory or specific module.

## Step 2: Scan for Error Handling Patterns

Use grep/ripgrep to find all error handling sites. Run these scans based on which languages are present:

### Swift
```bash
# Find all error handling sites
grep -rn --include="*.swift" -E "(catch\s*\{|catch\s+let|try\?|try!|\.failure|Result<|throws)" <path>

# Find what happens inside catch blocks (look for patterns after catch)
grep -rn --include="*.swift" -A 5 "catch\s*\{" <path>
grep -rn --include="*.swift" -A 5 "catch\s+let" <path>

# Find error types and enums
grep -rn --include="*.swift" -E "(enum\s+\w+.*Error|: Error\s*\{|: LocalizedError)" <path>

# Find user-facing error handling
grep -rn --include="*.swift" -E "(showAlert|showError|showToast|errorMessage|\.alert)" <path>

# Find logging patterns
grep -rn --include="*.swift" -E "(print\(|debugPrint|os_log|Logger\.|\.error\(|\.warning\(|Crashlytics|Sentry)" <path>

# Find retry/recovery patterns
grep -rn --include="*.swift" -E "(retry|fallback|cached|offline|backoff|recover)" <path>
```

### Kotlin
```bash
grep -rn --include="*.kt" -E "(catch\s*\(|runCatching|\.getOrNull|\.getOrDefault|Result\.|sealed\s+class.*Error)" <path>
grep -rn --include="*.kt" -A 5 "catch\s*\(" <path>
grep -rn --include="*.kt" -E "(Toast\.|Snackbar\.|showError|_errorState|UiState\.Error)" <path>
grep -rn --include="*.kt" -E "(Timber\.|Log\.|e\.printStackTrace|println\(e|Crashlytics|Sentry)" <path>
grep -rn --include="*.kt" -E "(retry|fallback|cache|offline|backoff|WorkManager|BackoffPolicy)" <path>
```

### TypeScript / JavaScript
```bash
grep -rn --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" -E "(catch\s*\(|catch\s*\{|\.catch\()" <path>
grep -rn --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" -A 5 "catch" <path>
grep -rn --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" -E "(extends\s+Error|class\s+\w+Error|ErrorBoundary)" <path>
grep -rn --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" -E "(setError|toast\.error|alert\(|showNotification|showError)" <path>
grep -rn --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" -E "(console\.(log|error|warn|debug)|Sentry\.|datadog|logger\.|captureException)" <path>
grep -rn --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" -E "(retry|fallback|cache|offline|backoff|serviceWorker|sync\.register)" <path>
```

### Cross-Platform Note

If the codebase targets multiple platforms (iOS, Android, web), note which platform each error handling site belongs to. Cross-platform inconsistency is one of the primary failure modes -- the same flow at Level 3 on iOS and Level 0 on Android is a specification gap, not an engineering failure. Tag findings by platform so the report can surface this.

**Important:** Exclude test files, generated code, and third-party dependencies:
```bash
# Add these to your grep commands
--exclude-dir=node_modules --exclude-dir=Pods --exclude-dir=build --exclude-dir=.build \
--exclude-dir=vendor --exclude-dir=__tests__ --exclude-dir=test --exclude-dir=tests \
--exclude-dir=spec --exclude-dir=fixtures --exclude-dir=generated --exclude-dir=dist
```

## Step 3: Read and Classify Each Error Handling Site

For each error handling site found in Step 2, read the surrounding context (usually 10-20 lines) to understand what's happening. Classify it into one of the four levels.

### Classification Rules

**Level 0 — Silent** if ANY of these are true:
- Empty catch block
- Catch block only has `return`, sets nil/null, or returns a default value
- `try?` or `runCatching { }.getOrNull()` with no other handling
- Only local logging (print, console.log, debugPrint, Log.d, println)
- No user-facing feedback of any kind

**Level 1 — Acknowledged** if:
- User sees a message, toast, alert, snackbar, or error state — BUT
- The message is generic ("Something went wrong", "Please try again", "Error occurred") — OR
- The message is raw technical info (exception message, stack trace, error.localizedDescription without mapping)
- Note: remote logging (Sentry, Crashlytics, Timber remote, etc.) may be present — good for engineering, but doesn't change the UX level

**Level 2 — Identifiable** if:
- Error is mapped to a code, category, or reference (e.g., "UP-1042", "AUTH-2001")
- Custom error types/enums with user-facing codes exist
- The error message includes something the user can quote to support
- Error type switching/matching produces specific (not generic) messages per error kind

**Level 3 — Guided** if:
- A recovery action is offered (retry button, alternative workflow, navigation to fix)
- Graceful degradation occurs (fallback to cache, offline mode, stale data with indicator)
- Automatic recovery (retry queue, background sync, exponential backoff with status)
- User state is preserved across the failure (drafts saved, form state kept)
- The user is told what to do next, not just what went wrong

### Edge Cases
- If a catch block has remote logging AND a generic message → Level 1 (the UX is still a dead end)
- If a catch block has a specific message but no code → borderline Level 1/2; classify as Level 1 unless the message is specific enough that support could act on it
- If a catch block has retry logic but no user feedback about the retry → Level 0 (silent retry is still silent from the user's perspective, but note it as "engineering-mature")
- If retry logic is present AND the user sees a status → Level 3
- React ErrorBoundary with only a generic fallback → Level 1; with a retry/reset action → Level 3

## Step 4: Generate the Report

Produce TWO outputs:

### A. Inline Chat Summary

Provide a concise overview in the conversation:

1. **Overall distribution** — What percentage of error handling sites are at each level?
2. **Maturity floor** — What's the lowest level present? Where?
3. **Maturity ceiling** — What's the highest level present? Where?
4. **Top findings** — 3-5 most impactful observations
5. **Quick wins** — Easiest improvements that would move the needle

### B. Markdown Report File

Create a detailed report at `/mnt/user-data/outputs/error-handling-maturity-report.md` with this structure:

```markdown
# Error Handling Maturity Report

**Repository/Directory:** [name]
**Date:** [date]
**Languages analyzed:** [list]
**Files scanned:** [count]
**Error handling sites found:** [count]

## Executive Summary

[2-3 sentences on overall maturity posture]

## Maturity Distribution

| Level | Name | Count | % | Example File |
|-------|------|-------|---|--------------|
| 0 | Silent | X | X% | path/to/file.swift:42 |
| 1 | Acknowledged | X | X% | path/to/file.kt:87 |
| 2 | Identifiable | X | X% | path/to/file.ts:23 |
| 3 | Guided | X | X% | path/to/file.swift:105 |

## Cross-Platform Consistency Matrix

[If the product ships on multiple platforms, fill in this matrix. Uniform low scores = investment needed. Variance between platforms = specification gap (one team figured it out; that knowledge was never shared or required).]

| Flow | iOS | Android | Web |
|------|-----|---------|-----|
| [Flow name] | [0–3] | [0–3] | [0–3] |

## Engineering Observations

### Remote Logging Coverage
[Are errors being sent to an observability platform? Which ones?]

### Error Architecture
[Bubble-up (global handler) or layer-by-layer (contextual wrapping)? Evidence from the code?]

**Maturity ceilings by architecture:**
- **Bubble-up:** Caps at Level 1. By the time the error reaches the top, context about what the user was doing is lost. Generic messages are the best possible outcome.
- **Layer-by-layer:** Enables Level 3. Each layer enriches errors with context, making codes (L2) and context-specific guided recovery (L3) achievable.

### Error Type System
[Are there custom error types? Error enums? Error mapping layers?]

## Detailed Findings by File

### [filename]
**Overall level: [N] — [Name]**

| Line | Pattern | Level | Detail |
|------|---------|-------|--------|
| 42 | empty catch | 1 | `catch { }` — error silently swallowed |
| 87 | generic toast | 2 | Shows "Something went wrong" on network failure |
| ... | ... | ... | ... |

[Repeat for each file with error handling sites]

## Recommendations

### Priority 1: Eliminate Silent Failures
[Specific files and patterns to fix first]

### Priority 2: Add Error Codes
[Where error mapping infrastructure would have the most impact]

### Priority 3: Build Recovery Paths
[Which flows would benefit most from guided recovery]

## Architectural Recommendation

[Based on the current error propagation patterns, recommend bubble-up vs. layer-by-layer
and what it would take to move to the next level of maturity across the codebase]
```

Use `present_files` to share the markdown report with the user after creating it.

## Important Notes

- **Focus on application code, not tests.** Exclude test files from analysis.
- **Don't count logging frameworks as error handling.** Logging is engineering infrastructure. The maturity level is determined by what the USER experiences.
- **Context matters.** A `try?` in a non-critical utility function is less concerning than a `try?` in a user-facing data loading flow. Weight findings by user impact.
- **Be specific in recommendations.** Don't just say "improve error handling." Point to exact files, lines, and patterns, and describe what the improved version would look like.
- **Note positive patterns.** If the codebase has good error wrapping, custom error types, or recovery patterns, call those out. Teams need to know what's working, not just what's broken.
- **Not every flow needs Level 3.** The appropriate target level depends on business impact -- support ticket volume, churn risk, and failure frequency. The diagnosis matrix helps prioritize. A flow that fails rarely and has low user impact may be fine at Level 1.
- **This report is the "Diagnose" step.** The full practice framework is: Diagnose (this report) → Specify (product defines target levels per flow, once, platform-agnostic) → Verify (QA confirms each platform meets spec). Frame your findings and recommendations in terms of what to specify next, not just what to fix.
- **Never recommend static interfaces for error reporting.** Suggesting a static `ErrorReporter.capture()` or similar global singleton reproduces the same design problems as other static loggers — tight coupling, untestability, hidden dependencies. When recommending remote observability, note that the error reporter should be injected as a dependency (protocol/interface), not called as a static method.
