# Error Handling Maturity Report

**Repository/Directory:** `ONXOfflineMaps` (LocalPackages)
**Date:** 2026-04-16
**Languages analyzed:** Swift
**Files scanned:** 108 (excluding Generated, Tests)
**Error handling sites found:** ~130 catch blocks + ~45 try? sites

---

## Executive Summary

ONXOfflineMaps is a service/data layer package — it has no direct user-facing presentation, so maturity is assessed at two levels: what the package exposes to consumers (delegate callbacks, public APIs), and the quality of internal error handling. The internal layer is predominantly Level 0: errors are caught, logged locally via `os.Logger`, and silently absorbed. The public delegate boundary reaches Level 1–2 in a few key flows. There is no remote observability anywhere in the package, which means engineering has zero visibility into what is failing in production.

---

## Maturity Distribution

| Level | Name | Approx. % | Representative Site |
|-------|------|-----------|---------------------|
| 0 | Silent | ~75% | `OfflineMapStore.swift:72` — logs, returns `[]` |
| 1 | Acknowledged | ~21% | `OfflineMapManager.swift:305` — delegates `.failed` state |
| 2 | Identifiable | ~3% | `UpdateOfflineMapTask.swift:102` — typed `insufficientDiskSpace` delegate |
| 3 | Guided | ~1% | `OfflineMapCreateSizeEstimator.swift:117` — auto-retry with delay |

---

## Cross-Platform Consistency Matrix

ONXOfflineMaps is iOS-only. Cross-platform comparison does not apply within this package. However, the package is consumed by iOS, and if Android/web have equivalent offline map flows handled separately, this is where cross-platform gap analysis should be applied at the app layer.

---

## Engineering Observations

### Remote Logging Coverage

**None.** Every `logging.error()`, `logging.assertion()`, `os.Logger.*` call in this package uses the local `ONXLogging` subsystem (local `os.Logger`). Zero calls to Sentry, Crashlytics, Datadog, or any remote observability platform were found. Engineering has no production visibility into failure patterns.

This is the highest-priority finding. Without remote logging, the diagnosis matrix cannot be filled with real data — everything below is based on code analysis, not observed failure rates.

### Error Architecture

**Mixed: bubble-up in stores, layer-by-layer in operations.**

The operation layer (`UpdateOfflineMapTask`, `LegacyUpdateOfflineMapOperation`) does contextual error wrapping before delegating up — this is the right architecture and enables the typed delegate boundary. The store and sync layers (`OfflineMapStore`, `OfflineMapSyncController`, `OfflineMapTileStore`) mostly absorb errors at the catch site with local logging and continue execution silently.

**Maturity ceilings by layer:**
- **Stores/Sync (bubble-up style):** Caps at Level 0–1. Database and sync errors are locally logged and swallowed. Consumers don't know they failed.
- **Operations (layer-by-layer):** Enables Level 2. `UpdateOfflineMapTask` classifies errors before delegating; `LegacyUpdateOfflineMapOperation` uses typed `failedToUpdate(reason:)` internally. But the typed reasons don't all reach the public delegate contract.

### Error Type System

Strong taxonomy exists — this is a genuine positive:

- **Public:** `OfflineMapMakerError` (`.insufficientDiskSpace`, `.unexpectedError`), `OfflineMapManagerFactoryError`, `OfflineStatic` result enums
- **Internal:** `UpdateOfflineMapTask.TaskError`, `LegacyUpdateOfflineMapOperation.Error` with typed reasons (`.databaseError`, `.mapDetailRequestFailed`, `.checkForUpdateRequestFailed`, `.unrecoverableStreamError`), `GRPCTilestoreSync.StreamError`
- **`TileServiceError: LocalizedError`** with `errorDescription` — well-formed but the error descriptions are technical, not user-facing

The gap: the rich internal taxonomy doesn't fully propagate to the public delegate interface. `UpdateOfflineMapTaskDelegate.failedToUpdate:` receives the failed map but no typed reason. Callers can't distinguish a GRPC failure from a database failure from a network timeout.

---

## Detailed Findings by File

### OfflineMapStore.swift
**Overall level: 0 — Silent**

| Line | Pattern | Level | Detail |
|------|---------|-------|--------|
| 72 | log + return | 0 | `storeLogging.assertion(...)` — returns `[]`, caller gets empty list silently |
| 366 | log only | 0 | `storeLogging.assertion("Database operation failed")` — no upstream signal |
| 392 | log only | 0 | Same pattern — maker started callback fails silently |
| 418 | log only | 0 | Same pattern — maker started (update) callback fails silently |
| 455 | log + continue | 0 | `storeLogging.error(error.localizedDescription)` — continues constructing record |
| 515 | log only | 0 | Database failure on map completed — silent to delegates |
| 676 | typed catch | 2 | Distinguishes `RecordError.recordNotFound` from other DB errors |

**Note:** `storeLogging.assertion` is stronger than `error` in intent, but both are local `os.Logger` — not remote.

---

### OfflineMapTileStore.swift
**Overall level: 0–1 — Silent / Acknowledged at delegate**

| Line | Pattern | Level | Detail |
|------|---------|-------|--------|
| 131 | log only | 0 | Delete operation catch — `logging.assertion(error.localizedDescription)` |
| 396 | log + state | 1 | `logging.assertion(...)` + sets `isUpdating = false` — app gets false state |
| 450–454 | log only | 0 | Verification and state read failures — silent |
| 585–661 | log only | 0 | Multiple DB state update failures — all locally logged |
| 241 | error wrapping | 2 | Wraps DB error into `OfflineMapTileStore.Error.getMapSizeError(reason:)` — good pattern |

---

### OfflineMapSyncController.swift
**Overall level: 0 — Silent**

All 13 catch blocks follow the same pattern: `logging.error(...)` with a descriptive message, then return or continue. The logging messages are actually quite good ("Failed to set didSync for mapUUID", "Failed to insert delete action into cloud push actions table") — but none of this reaches the app layer. Sync failures are invisible to consumers.

| Line | Pattern | Level | Detail |
|------|---------|-------|--------|
| 174 | log only | 0 | Sync state write failure |
| 242 | log only | 0 | Cloud delete action insert failure |
| 274 | typed log | 0 | Switches on `EncodingError` vs `DatabaseError` for message — good, but still local |
| 385 | log + default | 0 | Read failure returns empty array silently |
| 464 | log only | 0 | Cloud add action insert failure |
| 486 | log only | 0 | Create offline map failure |
| 539 | typed log | 0 | Same as 274 |

---

### UpdateOfflineMapTask.swift
**Overall level: 1–2 — Best in package**

| Line | Pattern | Level | Detail |
|------|---------|-------|--------|
| 88–108 | typed delegate | 2 | Switches on `TaskError.insufficientDiskSpace` → specific delegate; everything else → generic `failedToUpdate`. Good classification, but "everything else" is still coarse. |
| 95 | nested catch | 0 | State-write failure inside error handler logs locally only |
| 362 | log + throw | 2 | Logs, then throws typed `TaskError.databaseError` upstream |

**Positive:** The `insufficientDiskSpace` path is the clearest example of Level 2 in the package. The app can distinguish this specific condition and show a specific message to the user.

---

### LegacyUpdateOfflineMapOperation.swift
**Overall level: 1 — Acknowledged at operation boundary**

Has the richest internal error classification in the package via `failedToUpdate(reason:)` with typed reasons: `.databaseError`, `.mapDetailRequestFailed`, `.checkForUpdateRequestFailed`, `.unrecoverableStreamError`. However, these reasons are lost when the operation delegates to `UpdateOfflineMapTaskDelegate.failedToUpdate:`.

| Lines | Pattern | Level | Detail |
|-------|---------|-------|--------|
| 150, 179, 215, 244 | log + nil | 0 | Init failure paths return nil — caller gets nil operation |
| 358, 695, 795, 828 | log + typed delegate | 1–2 | Logs and calls `failedToUpdate(reason: .databaseError)` — reason is internal only |
| 613–621 | reason switch | 2 | Switches on reason to call appropriate delegate — good internal pattern |

---

### OfflineMapCreateSizeEstimator.swift
**Overall level: 3 — Best in package**

| Line | Pattern | Level | Detail |
|------|---------|-------|--------|
| 109–123 | typed catch + auto-retry | 3 | `definitionExceedsTileLimit` → `.mapTooLarge`; `failedButRetry` → retry with delay; default → `.sizeEstimateFailed`. This is the model for the rest of the package. |

---

### BackgroundUpdateManager.swift
**Overall level: 0 — Silent**

| Line | Pattern | Level | Detail |
|------|---------|-------|--------|
| 67 | log only | 0 | `logging.error(error)` — background update fails silently, user never notified |

This is a high-impact silent failure: a background map update fails, the user's map is stale, and nothing surfaces this.

---

### TileSerivce/BundledTileService.swift, LocalTileService.swift
**Overall level: 0–1**

Both tile services catch errors and return `.failure(error)`. The error is propagated as a `Result.failure`, which is good architecture — the consumer receives the failure. However, the wrapped error is unmodified, so the consumer gets the raw underlying error without context.

---

### OfflineMapManager.swift
**Overall level: 1 — Acknowledged at manager boundary**

| Line | Pattern | Level | Detail |
|------|---------|-------|--------|
| 262 | log only | 0 | Event loop shutdown failure — silent |
| 305 | log + return | 1 | `logging.assertion(...)` + returns `.failed` — typed result |
| 336–505 | log only | 0 | Multiple export/backup failures — all local, no consumer signal |

---

### Databases (OfflineMapsDatabase, OfflineMapTilesDatabase, OfflineMapTilesMapStateDatabase)
**Overall level: 0–1**

Migration failures have a recovery pattern (destroy and reinitialize), which is Level 3 for that specific case. All other catch blocks log locally and absorb.

| Site | Pattern | Level | Detail |
|------|---------|-------|--------|
| Migration catch | destroy + reinit | 3 | Recovers from corrupt DB by destroying and rebuilding — correct and automatic |
| OfflineMapTilesDatabase:124 | error code check | 2 | Checks `DatabaseError.errorCode == 2067` (UNIQUE violation) and suppresses only that |
| All others | log only | 0 | |

---

## Recommendations

### Priority 1: Add Remote Observability (Critical Foundation)

**Every catch block in this package logs to `os.Logger` only.** Without remote observability, there is no way to know which of these error paths are actually being hit in production. This is the prerequisite for all other improvements.

Add remote error reporting (Sentry, Crashlytics, or whatever platform the rest of the app uses) to at minimum:
- All `logging.assertion()` calls (these signal "should never happen" conditions)
- All errors in the operation layer (`UpdateOfflineMapTask`, `LegacyUpdateOfflineMapOperation`)
- Background update failures (`BackgroundUpdateManager:67`)
- Sync failures (`OfflineMapSyncController`)

**Example — current:**
```swift
} catch {
    storeLogging.assertion("Failed to read OfflineMaps from storage: \(error)")
}
```

**Example — with remote logging:**
```swift
} catch {
    storeLogging.assertion("Failed to read OfflineMaps from storage: \(error)")
    ErrorReporter.capture(error, context: "OfflineMapStore.readAllOfflineMaps")
}
```

---

### Priority 2: Propagate Typed Failure Reasons to the Public Delegate

`LegacyUpdateOfflineMapOperation` already has a rich internal `failedToUpdate(reason:)` taxonomy. `UpdateOfflineMapTask` already classifies `insufficientDiskSpace`. None of these typed reasons reach `UpdateOfflineMapTaskDelegate`.

Change:
```swift
protocol UpdateOfflineMapTaskDelegate: AnyObject {
    func updateOfflineMapOperation(_: UpdateOfflineMapTask, failedToUpdate: OfflineMap)
    func updateOfflineMapOperation(_: UpdateOfflineMapTask, insufficientDiskSpaceFor: OfflineMap)
    ...
}
```

To:
```swift
enum OfflineMapUpdateFailureReason {
    case insufficientDiskSpace
    case networkError
    case serverError
    case databaseError
    case unexpectedError
}

protocol UpdateOfflineMapTaskDelegate: AnyObject {
    func updateOfflineMapOperation(_: UpdateOfflineMapTask, failedToUpdate: OfflineMap, reason: OfflineMapUpdateFailureReason)
    ...
}
```

This is a pure data layer change -- the consuming app gets the information it needs to show "Download failed: not enough storage" vs "Download failed: check your connection" without the package knowing anything about UI. This moves the public delegate boundary from Level 1 to Level 2.

---

### Priority 3: Surface Background Update Failures

`BackgroundUpdateManager.swift:67` catches errors and logs locally. If a background update fails, the user's offline map is silently stale.

At minimum, fire a notification or update an observable state when background updates fail, so the app can present a user notification or a "map may be out of date" indicator. The infrastructure for user notifications already exists (`OfflineMapUserNotification`).

---

### Priority 4: Don't Silently Absorb Sync Failures

`OfflineMapSyncController` has 13 catch blocks that all follow the same pattern: log and continue. When cloud sync fails to record an add, update, or delete action, the offline map's sync state silently diverges from the cloud.

The minimum improvement is ensuring sync failures produce an observable state update so the app can show "Cloud sync unavailable" rather than silently falling behind.

---

## Architectural Recommendation

The operation layer (`UpdateOfflineMapTask`, `LegacyUpdateOfflineMapOperation`) already uses the right architecture: errors are caught at each operation step, wrapped with context, and classified before delegating up. This is the pattern that enables Level 2 and Level 3. **Extend it.**

The stores and sync controller use bubble-up/swallow: errors are caught at the call site, locally logged, and execution continues. This caps them at Level 0. The stores don't need to reach Level 3, but they should feed observable error states that the app can react to, and they should remote-log.

**Immediate ceiling:** Without remote observability, the practical maturity ceiling is Level 1 regardless of code quality — engineering cannot see what's failing, so improvement is reactive (user reports) rather than proactive.

**With remote observability + typed delegate reasons:** The operation layer can reach Level 2 within the existing architecture. Level 3 already exists in `OfflineMapCreateSizeEstimator` and the database migration recovery paths — use those as templates.

---

## The Practice Context

This report is the **Diagnose** step. The next steps:

1. **Specify:** Product defines target maturity per flow, once, platform-agnostic. For example: "Download failure → Level 2: user sees a specific message with actionable context." "Background update failure → Level 1: user is informed the map may be stale."

2. **Verify:** After implementation, confirm each flow meets spec. The typed delegate approach makes this testable.
