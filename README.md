# Error Handling Maturity Model — Demo

This repo contains three things:

1. **The model** — a UX-centered framework for specifying, building, and verifying how applications handle failure
2. **The skill** — a Claude Code slash command that automates the diagnosis step against any codebase
3. **The report** — an example analysis run against a real iOS package

---

## The Problem in One Sentence

Poor error handling is almost never caused by bad engineering. It's caused by the absence of sad path specifications — the happy path was designed and built, the sad path was never specified.

This reframing matters: filing a "bug" for poor error handling implies something was built wrong. In reality, something was never built at all. Error handling work isn't fixing defects — it's completing features.

---

## The Model

Four levels, framed entirely around what the user experiences.

| Level | Name | What the User Experiences | What the User Thinks |
|-------|------|--------------------------|----------------------|
| **0** | Silent | Nothing happens. The action silently fails. | "Did I tap that? Is this frozen? Did it work?" |
| **1** | Acknowledged | User knows something broke, but can't act on it. | "OK, it broke. Now what?" |
| **2** | Identifiable | User gets a code or reference they can use with support. | "I can look this up or tell support exactly what happened." |
| **3** | Guided | App offers a next step. User stays in motion. | "That didn't work, but I know what to do next." |

### The cross-platform problem

The same product, the same failure, handled differently on every platform — not because of bad engineering, but because no one specified the sad path. The model's diagnosis matrix makes this visible:

| Flow | iOS | Android | Web |
|------|-----|---------|-----|
| Map download | 3 | 1 | 0 |
| User login | 2 | 2 | 1 |
| Waypoint sync | 0 | 0 | 0 |

Uniform low scores = investment needed. Variance across platforms = specification gap (one team figured it out; that knowledge was never shared or required).

### The practice

**Diagnose → Specify → Verify**

- **Diagnose:** Trigger failures across platforms. Fill in the matrix.
- **Specify:** Product defines target maturity per flow, once, platform-agnostic. Error handling becomes a product requirement.
- **Verify:** QA confirms each platform meets the spec. "This flow is Level 1 on Android but the spec says Level 2" is an unambiguous, actionable finding.

---

## The Skill

The skill is a Claude Code slash command that automates the Diagnose step. Point it at a local directory or GitHub repo and it scans the codebase, classifies every error handling site against the model, and produces a full maturity report.

### Installation

Copy `error-maturity/` into `~/.claude/skills/`. The slash command is available immediately in any Claude Code session.

### Usage

```
/error-handling-maturity ~/path/to/your/module
/error-handling-maturity https://github.com/org/repo
```

### What it does

1. Scans all source files for error handling sites (catch blocks, try?, Result types, error delegates)
2. Reads surrounding context to classify each site at Level 0–3
3. Produces a maturity distribution, cross-platform consistency matrix, and detailed file-by-file findings
4. Identifies the highest-priority improvements with specific file references and before/after code examples

Supports Swift, Kotlin, and TypeScript/JavaScript.

---

## The Report — Example Analysis

The following is an excerpt from a real analysis of `ONXOfflineMaps`, an iOS Swift package (108 source files, ~130 catch blocks, ~45 `try?` sites).

### Maturity distribution

| Level | Name | Approx. % | Representative Site |
|-------|------|-----------|---------------------|
| 0 | Silent | ~75% | `OfflineMapStore.swift:72` — logs, returns `[]` |
| 1 | Acknowledged | ~21% | `OfflineMapManager.swift:305` — delegates `.failed` state |
| 2 | Identifiable | ~3% | `UpdateOfflineMapTask.swift:102` — typed `insufficientDiskSpace` delegate |
| 3 | Guided | ~1% | `OfflineMapCreateSizeEstimator.swift:117` — auto-retry with delay |

### Highest-priority finding: zero remote observability

Every catch block in this package logs to `os.Logger` only. Without remote observability, there is no way to know which error paths are actually being hit in production. Engineering has no visibility into failure patterns.

This is the prerequisite for all other improvements. Without it, the practical maturity ceiling is Level 1 regardless of code quality — improvement is reactive (user reports) rather than proactive.

### What good looks like in this codebase

`OfflineMapCreateSizeEstimator.swift:109` is the model for the rest of the package:

```swift
// Level 3 — typed catch + auto-retry
} catch let error as SizeEstimatorError {
    switch error {
    case .definitionExceedsTileLimit:
        delegate?.sizeEstimationFailed(reason: .mapTooLarge)
    case .failedButRetry:
        scheduleRetry(delay: retryDelay)
    default:
        delegate?.sizeEstimationFailed(reason: .sizeEstimateFailed)
    }
}
```

Compare to the dominant pattern in the package:

```swift
// Level 0 — silent
} catch {
    storeLogging.assertion("Failed to read OfflineMaps from storage: \(error)")
}
```

### Top recommendations

1. **Add remote observability.** Every `logging.assertion()` call and every error in the operation layer should be reported to Sentry/Crashlytics/Datadog. This is the foundation everything else builds on.

2. **Propagate typed failure reasons to the public delegate.** `LegacyUpdateOfflineMapOperation` already has a rich internal error taxonomy (`.databaseError`, `.networkError`, `.unrecoverableStreamError`). None of it reaches the public delegate contract. One enum change moves the operation layer from Level 1 to Level 2.

3. **Surface background update failures.** `BackgroundUpdateManager.swift:67` catches errors and logs locally. If a background update fails, the user's offline map is silently stale — a high-impact Level 0 failure.

---

## Full Documents

- [The Error Handling Maturity Model](error-handling-maturity-model.md) — the complete framework
- [Example Report: ONXOfflineMaps](example-report.md) — the full example analysis
