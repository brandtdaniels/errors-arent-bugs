# Error Handling Pattern Reference

## Maturity Levels (0–3)

### Level 0 — Silent
The error is swallowed. The user gets no feedback. Nobody knows something went wrong.

### Level 1 — Acknowledged
The user knows something went wrong but can't do anything about it. This includes both generic messages ("Something went wrong") AND leaked technical details (stack traces, raw exception messages). Both are dead ends from the user's perspective.

### Level 2 — Identifiable
The error has a code, category, or reference the user can use with support, docs, or a community forum. The error has a name that bridges user and organization.

### Level 3 — Guided
The app offers a next step: retry, fallback, offline mode, alternative workflow. The user stays in motion toward their goal.

---

## Swift Patterns

### Level 0 — Silent
```swift
// Empty catch
do { try something() } catch { }

// Catch with only return
do { try something() } catch { return }

// try? discards the error entirely
let result = try? something()

// Catch that logs locally only (print/debugPrint)
do { try something() } catch { print(error) }
do { try something() } catch { debugPrint(error) }

// Catch that sets a boolean/nil without user feedback
do { try something() } catch { self.data = nil }

// Guard with else return (no feedback)
guard let x = try? something() else { return }
```

### Level 1 — Acknowledged
```swift
// Generic alert/toast/message with no actionable info
catch { showAlert("Something went wrong") }
catch { showError("An error occurred") }
catch { self.errorMessage = "Please try again" }
catch { showToast("Error") }

// Displaying raw error.localizedDescription without mapping
catch { showAlert(error.localizedDescription) }
catch { self.errorMessage = "\(error)" }
catch { showAlert("Error: \(error)") }

// Logging remotely but still showing generic message
catch {
    logger.error("Failed: \(error)")  // remote log = good
    showAlert("Something went wrong") // but user gets nothing useful
}
```

### Level 2 — Identifiable
```swift
// Mapped error codes
catch let error as AppError {
    showAlert("Error \(error.code): \(error.userMessage)")
}

// Typed error enums with user-facing codes
enum UploadError: Error {
    case fileTooLarge(code: String)
    case unsupportedFormat(code: String)
}

// Error codes in user-facing messages
catch { showAlert("Upload failed (UP-1042). Contact support if this continues.") }

// Switch on error type to provide specific messages
catch {
    switch error {
    case let e as URLError: showAlert("Network error (NET-\(e.errorCode))")
    case let e as AppError: showAlert("\(e.userMessage) (\(e.code))")
    default: showAlert("Unexpected error (ERR-0000)")
    }
}
```

### Level 3 — Guided
```swift
// Offering retry
catch { showRetryDialog("Upload failed. Try again?", retryAction: { ... }) }

// Fallback to cached data
catch {
    if let cached = cache.load(key) {
        showCachedData(cached, message: "Showing cached data. We'll refresh when connected.")
    }
}

// Automatic retry with backoff
catch {
    retryQueue.enqueue(operation, maxRetries: 3, backoff: .exponential)
    showStatus("Syncing... will retry automatically.")
}

// Alternative workflow
catch {
    showAlert("Couldn't send right now. Save as a draft and try again later?",
              actions: [.saveDraft, .cancel])
}

// State preservation on failure
catch {
    saveDraft()
    showAlert("Your changes are saved. Sign back in to continue.")
}
```

---

## Kotlin Patterns

### Level 0 — Silent
```kotlin
// Empty catch
try { something() } catch (e: Exception) { }

// Catch with only return
try { something() } catch (e: Exception) { return }

// Catch with local-only logging
try { something() } catch (e: Exception) { e.printStackTrace() }
try { something() } catch (e: Exception) { Log.d(TAG, "Error", e) }
try { something() } catch (e: Exception) { println(e) }

// runCatching with getOrNull (discards error)
val result = runCatching { something() }.getOrNull()

// Catch that sets null without user feedback
try { data = something() } catch (e: Exception) { data = null }

// Result.getOrDefault silently substituting
val value = runCatching { fetch() }.getOrDefault(emptyList())
```

### Level 1 — Acknowledged
```kotlin
// Generic toast/snackbar
catch (e: Exception) { Toast.makeText(ctx, "Something went wrong", ...).show() }
catch (e: Exception) { showSnackbar("An error occurred") }
catch (e: Exception) { _errorState.value = "Please try again" }

// Raw exception message shown to user
catch (e: Exception) { showError(e.message ?: "Unknown error") }
catch (e: Exception) { _errorState.value = e.localizedMessage }

// Remote logging but generic UX
catch (e: Exception) {
    Timber.e(e, "Failed to load")
    _uiState.value = UiState.Error("Something went wrong")
}
```

### Level 2 — Identifiable
```kotlin
// Sealed class error hierarchy with codes
sealed class AppError(val code: String, val userMessage: String) {
    class NetworkError(code: String) : AppError(code, "Network issue")
    class AuthError(code: String) : AppError(code, "Authentication failed")
}

// Mapping exceptions to error codes
catch (e: Exception) {
    val appError = errorMapper.map(e)
    _uiState.value = UiState.Error(appError.code, appError.userMessage)
}

// When expression mapping to specific messages
catch (e: Exception) {
    val message = when (e) {
        is HttpException -> "Server error (HTTP-${e.code()})"
        is IOException -> "Connection failed (NET-001)"
        else -> "Unexpected error (ERR-000)"
    }
}
```

### Level 3 — Guided
```kotlin
// Retry action offered
catch (e: Exception) {
    _uiState.value = UiState.Error(
        message = "Upload failed",
        action = RetryAction { uploadFile(file) }
    )
}

// Fallback to cached data
catch (e: Exception) {
    val cached = cacheRepository.get(key)
    if (cached != null) {
        _uiState.value = UiState.Stale(cached, "Showing cached data")
    }
}

// Automatic retry with WorkManager
catch (e: Exception) {
    val work = OneTimeWorkRequestBuilder<SyncWorker>()
        .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
        .build()
    workManager.enqueue(work)
    showStatus("Will retry when connected")
}

// Navigation to recovery flow
catch (e: AuthException) {
    savePendingState()
    navigateTo(LoginScreen(returnTo = currentRoute))
}
```

---

## TypeScript / JavaScript Patterns

### Level 0 — Silent
```typescript
// Empty catch
try { something() } catch (e) { }
try { something() } catch { }

// Promise with empty catch
fetch(url).catch(() => {})
promise.catch(e => {})

// Catch with only console.log/console.debug (local only)
try { something() } catch (e) { console.log(e) }
try { something() } catch (e) { console.debug(e) }
promise.catch(e => console.log(e))

// Catch with return/null and no user feedback
try { return something() } catch { return null }
try { return something() } catch { return undefined }
try { return something() } catch { return [] }
try { return something() } catch { return {} }

// Optional chaining that masks failures (context-dependent)
const data = response?.data?.items ?? []

// async function with no try/catch and no .catch()
async function load() { await fetch(url) } // unhandled rejection
```

### Level 1 — Acknowledged
```typescript
// Generic error message
catch (e) { alert("Something went wrong") }
catch (e) { setError("An error occurred") }
catch (e) { toast.error("Please try again") }
catch (e) { showNotification({ type: 'error', message: 'Error' }) }

// Raw error shown to user
catch (e) { setError(e.message) }
catch (e) { alert(e.toString()) }
catch (e) { setError(`Error: ${e}`) }

// React error boundary with generic fallback
class ErrorBoundary extends React.Component {
    render() { return <div>Something went wrong</div> }
}

// Remote logging but generic UX
catch (e) {
    Sentry.captureException(e)  // observability = good
    setError("Something went wrong")  // but user gets dead end
}
```

### Level 2 — Identifiable
```typescript
// Custom error classes with codes
class AppError extends Error {
    constructor(public code: string, public userMessage: string) { super(userMessage) }
}
class NetworkError extends AppError {
    constructor() { super('NET-001', 'Network connection failed') }
}

// Mapping to error codes
catch (e) {
    const appError = mapError(e)
    setError({ code: appError.code, message: appError.userMessage })
}

// API response with error codes
// { error: { code: "AUTH-2001", message: "Session expired" } }

// Error code displayed to user
catch (e) {
    setError(`Upload failed (UP-1042). Contact support if this persists.`)
}

// Switch/if mapping to specific messages
catch (e) {
    if (e instanceof AuthError) setError(`Session error (${e.code})`)
    else if (e instanceof NetworkError) setError(`Connection issue (${e.code})`)
    else setError(`Unexpected error (ERR-0000)`)
}
```

### Level 3 — Guided
```typescript
// Retry action offered
catch (e) {
    setError({
        message: "Upload failed",
        action: { label: "Try again", handler: () => retry(upload) }
    })
}

// Fallback to cached data
catch (e) {
    const cached = await localCache.get(key)
    if (cached) {
        setData(cached)
        setBanner("Showing cached data. We'll refresh when you're back online.")
    }
}

// Automatic retry with exponential backoff
catch (e) {
    if (retryCount < MAX_RETRIES) {
        setTimeout(() => fetchWithRetry(retryCount + 1), backoff(retryCount))
        setStatus("Retrying...")
    }
}

// Service worker / background sync
catch (e) {
    await navigator.serviceWorker.ready
    await registration.sync.register('retry-upload')
    setStatus("Queued for upload when online")
}

// State preservation + redirect
catch (e) {
    saveDraftToLocalStorage(formData)
    setError("Session expired. Your work is saved. Sign in to continue.")
    redirect('/login?return=' + currentPath)
}

// React error boundary with recovery
class ErrorBoundary extends React.Component {
    render() {
        return (
            <div>
                <p>This section couldn't load.</p>
                <button onClick={this.reset}>Try again</button>
            </div>
        )
    }
}
```

---

## Cross-Language Signals

### Indicators of remote logging (good for engineering, but doesn't affect UX level)
- Sentry, Crashlytics, Datadog, Timber (with remote transport), os.Logger with remote config
- `logger.error()`, `Sentry.captureException()`, `crashlytics.recordException()`
- These are noted in the report but don't elevate the UX maturity level on their own

### Indicators of error wrapping (layer-by-layer architecture)
- Custom error types that wrap underlying errors
- Error mapping functions/classes
- Middleware that transforms errors
- `Result` types that carry typed errors

### Indicators of recovery patterns
- Retry logic, exponential backoff
- Circuit breakers
- Cache fallbacks
- Offline mode switches
- Queue-and-retry (WorkManager, service workers, background tasks)
- State preservation (draft saving, form state persistence)
