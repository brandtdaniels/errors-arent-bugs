# The Error Handling Maturity Model

_A UX-Centered Framework for Specifying, Building, and Verifying How Applications Handle Failure_

_Brandt Daniels_

---

## Where This Idea Came From

Throughout my computer science education, maturity models kept showing up. Capability Maturity Model for software processes. Test maturity models. DevOps maturity models. The pattern was always the same: take a discipline, define a progression from ad hoc to optimized, and give organizations a way to measure where they stand and where to go next.

When I entered the industry, I started noticing something. Every codebase I worked in handled errors differently — not just between projects, but within the same project. One screen would gracefully guide users through a failure. The next screen would silently swallow an exception and leave the user staring at a blank page. Some flows showed "Something went wrong" with no further guidance. Others leaked raw stack traces to end users. A few offered retry buttons or fell back to cached data.

And then there was the cross-platform problem. The same product, the same failure, handled completely differently on iOS, Android, and web. A user who got a helpful retry prompt on one platform would get a silent failure on another. The product felt like it was built by three different companies — because, from an error handling perspective, it was.

I kept looking for a maturity model that described this progression — something that named the levels and gave teams a shared vocabulary for talking about how they handle failure. I couldn't find one. There are maturity models for testing, for deployment, for observability, for incident response. But not for the thing that determines what a user actually experiences when something goes wrong in your application.

So I built one.

---

## The Problem

There are two problems with error handling in most software organizations, and they compound each other.

### Problem 1: Errors are a bad user experience

Most error handling falls into one of two categories: invisible or useless. Either the user gets no feedback at all, or they get a generic message that tells them nothing. In both cases, the user is stranded with no path forward. Trust erodes. Support tickets pile up. Users learn to distrust the product.

### Problem 2: Error handling is inconsistent across platforms

The same product, the same user flow, the same failure condition — handled differently on every platform. iOS shows a retry dialog. Android shows a toast. Web shows nothing. This happens because error handling decisions are made by individual engineers during implementation, not specified as product requirements. Each platform team makes their own judgment calls, and nobody is comparing notes.

### Why this keeps happening

These problems are almost never caused by bad engineering. They are caused by **the absence of sad path specifications.**

When a product team defines a feature, they describe the happy path in detail. "The user taps Upload, a progress bar appears, the file is saved to the cloud." What happens when the upload fails? What happens when the device runs out of storage? What happens when the user loses connectivity midway through? These are not edge cases. They are foreseeable, common conditions. But they are rarely specified.

Without a specification, error handling becomes an implementation detail left to individual developers. The result is inconsistency — between flows within a single platform, and between platforms within a single product.

---

## Errors Are Not Defects

This leads to a foundational distinction: **poor error handling is not a defect.**

A defect — the industry often calls them "bugs," which is a terrible name for what is really a defect — is the difference between expected behavior and actual behavior. The specification says one thing; the software does another. That's a defect. It's a failure of implementation.

Errors, on the other hand, are expected. Networks time out. Servers return 500s. Connections drop mid-request. Disks fill up. Sessions expire. These are not surprises. They are known, predictable conditions that every application will encounter.

When an application handles these conditions poorly — or doesn't handle them at all — that is usually not a defect. It's a **lack of sad path specification.** The happy path was designed, specified, and implemented. The sad path was not. The feature is simply unfinished.

This reframing matters because it changes how teams talk about the work. Filing a "bug" for poor error handling implies something was built wrong. In reality, something was never built at all. Error handling work isn't fixing defects — it's completing features.

---

## The Model

This model provides a shared vocabulary for specifying, building, and verifying how an application handles failure. It is framed entirely around user experience — the primary question at each level is: when something goes wrong, what does the user experience?

Level 0 represents the absence of error handling — you haven't started yet. Levels 1 through 3 represent increasing maturity in how the application serves the user through failure.

| Level | User Experience | What the User Thinks |
|-------|-----------------|----------------------|
| **Level 0** — _Silent_ | Nothing happens. The action silently fails. | "Did I tap that? Is this thing frozen? Did it work or not?" |
| **Level 1** — _Acknowledged_ | The user knows something went wrong, but has no way to act on it. | "OK, it broke. Now what? I guess I'll try again and hope for the best." |
| **Level 2** — _Identifiable_ | The user gets a code or reference they can use with support, docs, or a forum. | "I can look this up or tell support exactly what happened." |
| **Level 3** — _Guided_ | The app offers a next step. The user stays in motion toward their goal. | "That didn't work, but I know what to do next." |

---

## Levels in Detail

### Level 0 — Silent

**User Experience**

The user performs an action and nothing visibly happens. There is no feedback that anything went wrong. The user is left guessing whether their action was received, whether the app is still working, or whether they need to try again. This is not a level of error handling — it is the absence of it. The user cannot even begin to help themselves.

**Examples:**

- A save button is tapped but the save fails silently. The user leaves the screen believing their data was saved.
- An image fails to load, leaving a blank area with no indication anything is missing.
- A background sync fails, and the user discovers hours later that their data is out of date.

**Engineering Strategies**

At this level, errors are typically either unhandled or caught with an empty handler. There may be no logging at all, or only local debug logging that is invisible in production.

**The local vs. remote logging distinction matters here.** Local debug logging (print statements, console.log, debugPrint) is only visible to a developer on their own device. It provides zero visibility into what users experience in the field. Remote logging — sending errors to an observability platform like Sentry, Crashlytics, or Datadog — is the engineering foundation that makes all higher levels possible. Without it, you are flying blind.

**Moving to Level 1 requires:**

- Identifying all catch blocks that silently swallow errors.
- Adding remote logging so errors are visible in production.
- Surfacing any form of feedback to the user when an action fails.

---

### Level 1 — Acknowledged

**User Experience**

The user knows something went wrong. They see a message, a toast, a dialog, or an error screen. However, the information provided does not help them understand the cause or take any meaningful action.

**An important design note:** generic error messages and leaked technical details are the same maturity level. "Something went wrong" and "NullPointerException at com.example.service.UploadService.java:142" are both dead ends from the user's perspective. One is masked, one is leaked, but neither helps.

**Examples:**

- "An error occurred. Please try again later."
- "Error: NullPointerException at com.example.service.UploadService.send(UploadService.java:142)"
- A red banner that says "Connection failed" with no further context.

**Engineering Strategies**

Errors are caught and surfaced, but without classification or context enrichment. A global error handler may catch everything and display a single generic message regardless of what went wrong.

**Moving to Level 2 requires:**

- Classifying errors into categories (network, authentication, data, permissions).
- Creating a mapping system between internal errors and user-facing error codes.
- Ensuring support documentation and customer service tooling can reference those codes.

---

### Level 2 — Identifiable

**User Experience**

The user receives an error with a code or reference that they can use to get help. They can search a help center, quote the code to customer support, or look it up in a community forum. The error has a name, and that name bridges the gap between the user and the organization that built the software.

**Examples:**

- "Unable to upload file. (Error UP-1042)" — support can immediately look up this code and guide the user.
- "Your session has expired. (AUTH-2001)" — help docs explain exactly what to do for this code.
- "Sync failed: SYNC-3010. Visit help.example.com/errors for details."

**Engineering Strategies**

This level requires a layer-by-layer approach to error handling rather than a single global handler. Each layer of the application catches errors and wraps them with additional context before passing them upward. The network layer knows the request timed out. The service layer knows it was an upload request. The UI layer knows the user was trying to share a specific file.

**Moving to Level 3 requires:**

- Designing recovery paths for each error category.
- Understanding the user's intent at the point of failure.
- Building UI flows that offer next steps rather than just reporting the problem.

---

### Level 3 — Guided

**User Experience**

The app takes ownership of the failure and helps the user move forward. The error message includes a clear next step, an alternative path, or an automatic recovery. The user is not left stranded. This is the level where error handling stops being damage control and becomes a feature.

**Examples:**

- "This content couldn't be loaded. Want to save it for later and try again?"
- "Your session expired. Tap here to sign back in. Your unsaved changes are preserved."
- A failed upload is automatically queued for retry when connectivity returns, with a status indicator the user can check.
- "You're offline. Showing cached data. We'll refresh automatically when you reconnect."

**Engineering Strategies**

This is the highest level of maturity and requires deep integration between error handling and application logic. The system must understand user intent, maintain state across failures, and offer context-appropriate recovery paths.

**Key engineering capabilities:**

- **Layer-by-layer error reframing:** Each architectural layer catches, wraps, and enriches the error with context about what the user was doing.
- **State preservation:** User work is not lost when an error occurs. Drafts are saved, queues are maintained, and progress is recoverable.
- **Graceful degradation:** The app offers reduced functionality rather than full failure. Cached data and fallback workflows keep the user productive.
- **Automatic recovery:** Retry logic, circuit breakers, and background recovery processes handle transient failures without requiring user action.
- **Remote observability:** All errors, including those automatically recovered from, are logged remotely so engineering has full visibility into failure patterns.

---

## Engineering Architecture: Bubble-Up vs. Layer-by-Layer

A critical architectural decision underpins this model: how errors propagate through the application. This decision determines the ceiling on the maturity level an application can achieve.

### Bubble-Up (Global Handler)

Errors propagate to a top-level handler that decides what to show the user. This is simpler to implement but limits maturity. By the time the error reaches the top, the context of what the user was trying to do is often lost. A network timeout looks the same whether the user was uploading a file, syncing their inbox, or checking their subscription status.

**Maturity ceiling: Level 1.** The global handler can acknowledge the error but cannot offer context-specific guidance or recovery.

### Layer-by-Layer (Contextual Wrapping)

Each layer of the application catches errors, wraps them with context, and passes them upward. The network layer knows the request timed out. The service layer knows it was an upload for a specific file. The UI layer knows the user was trying to share that file with a collaborator.

This approach produces errors rich enough in context to support both error codes (Level 2) and guided recovery (Level 3). Meanwhile, the full error chain remains available for engineering through remote logging.

**Maturity ceiling: Level 3.** The layered context is what makes guided recovery possible.

---

## Putting It Into Practice: Diagnose, Specify, Verify

The model is only useful if it drives action. A one-time audit that produces a report and sits in a wiki is a snapshot, not a solution. To create lasting change, the model needs to be embedded into the product development process through three activities: diagnose, specify, and verify.

### Diagnose: Understand where you are

Run the model against your codebase and your user experience. For each major user flow, trigger failures deliberately and document what the user sees. Do this on every platform your product ships on.

The output is a matrix: flows on one axis, platforms on the other, maturity levels in the cells.

| Flow | iOS | Android | Web |
|------|-----|---------|-----|
| File upload | 3 | 1 | 0 |
| User login | 2 | 2 | 1 |
| Background sync | 0 | 0 | 0 |
| Subscription purchase | 1 | 1 | 2 |

This matrix makes two things immediately visible: the overall maturity of each flow, and the variance between platforms. A flow that is uniformly Level 0 across all platforms needs investment. A flow that is Level 3 on one platform and Level 0 on another reveals a specification gap — one team figured out the right experience, but that knowledge was never shared or required.

### Specify: Define the target

For each flow, product defines the target maturity level as a product requirement — once, platform-agnostic. This is the critical shift. Error handling moves from an implementation detail left to individual engineers to a product specification with the same weight as the happy path.

The specification should include the flow (what the user is trying to accomplish), the failure conditions (what can go wrong), the target level, and the user experience (what the user should see and be able to do when each failure condition occurs).

**Not every flow needs Level 3.** The appropriate level depends on the business impact of the failure. A flow that generates high support ticket volume or causes users to churn deserves more investment than one that fails rarely and has low impact. The diagnosis matrix helps prioritize.

### Verify: Confirm the implementation

After implementation, verify that every platform meets the specification. The model gives QA and product a shared language to do this precisely. "This flow is Level 1 on Android but the spec says Level 2" is an unambiguous, actionable finding.

Verification can take several forms:

- **Manual QA:** Trigger each failure condition on each platform and compare the experience against the specification.
- **Cross-platform review:** Demonstrate the same failure on all platforms side by side. Inconsistencies become immediately obvious.
- **Static analysis:** Scan the codebase for error handling patterns that indicate the maturity level.
- **Observability:** Monitor remote error logs for silent failures (Level 0) that should have been caught.

Verification is not a one-time event. As new features ship and codebases evolve, error handling can regress. The diagnosis should be repeated periodically — or better yet, error handling requirements should be part of every feature's definition of done.

---

## Integrating Into the Product Development Process

For this model to solve the problem permanently — not just diagnose it once — it needs to become part of how features are built.

**During feature specification:** When a product team writes a feature spec, they describe the happy path. The model asks them to also describe the sad paths — and to do so with the same level of detail. For each sad path, the team specifies the target maturity level and the expected user experience.

**During implementation:** Engineers implement error handling to the level specified. The model gives them a clear target rather than leaving the decision to their own judgment. It also gives them a shared vocabulary with product and design: "the spec says Level 3, which means we need a retry mechanism with state preservation, not just a toast."

**During code review:** Reviewers can assess error handling against the specification. "This catch block shows a generic error message, but the spec says Level 2 — we need an error code here." The model makes error handling reviewable rather than subjective.

**During QA:** Testers verify that each failure condition on each platform matches the specification. The diagnosis matrix becomes a test plan. Inconsistencies between platforms are caught before release, not after users report them.

**During retrospective:** When support ticket data or user feedback reveals error handling problems, the team can reference the model to classify the gap and specify the fix. "This flow is Level 0 — users are reporting silent failures. We need to bring it to at least Level 1 and add remote logging."

---

## How to Start

If you are reading this and recognizing your own product, here is how to begin.

**1. Pick three flows.** Don't try to audit the entire product at once. Pick three user flows that are high-impact: ones that fail often, generate support tickets, or represent critical moments in the user journey.

**2. Diagnose them across platforms.** Trigger failures in each flow on every platform. Document what the user sees. Fill in the matrix.

**3. Specify the target.** For each flow, decide what level it should be at. Write the sad path specification: what the user should experience for each failure condition.

**4. Build to the spec.** Implement the error handling on every platform to match the specification. Use the model's vocabulary during implementation and code review.

**5. Verify the result.** Test each failure condition on each platform. Confirm the experience matches the spec and is consistent across platforms.

**6. Expand.** Once the team has the muscle memory of specifying sad paths, expand to more flows. Over time, sad path specification becomes a natural part of feature development rather than a separate initiative.

---

## Quick Reference

### The Four Levels

| Level | Name | User Experience | What the User Thinks |
|-------|------|-----------------|----------------------|
| 0 — Silent | Silent | Nothing happens. The action silently fails. | "Did I tap that? Is this frozen? Did it work?" |
| 1 — Acknowledged | Acknowledged | User knows something broke but can't act on it. | "OK, it broke. Now what?" |
| 2 — Identifiable | Identifiable | User gets a code or reference they can use with support. | "I can look this up or tell support exactly what happened." |
| 3 — Guided | Guided | App offers a next step. User stays in motion. | "That didn't work, but I know what to do next." |

> _Key insight: Poor error handling is not a defect — it's a lack of sad path specification. The happy path was designed and built. The sad path was never specified. This is unfinished work, not broken work. Error handling work isn't fixing defects — it's completing features._

### Architecture Maturity Ceilings

| Architecture | Ceiling | Why |
|--------------|---------|-----|
| Bubble-up (global handler) | Level 1 | Context is lost before errors reach the handler. Generic messages are the best possible outcome. |
| Layer-by-layer (contextual) | Level 3 | Each layer enriches errors with context. Codes (L2) and guided recovery (L3) become achievable. |

### The Practice Framework: Diagnose → Specify → Verify

**Diagnose:** Trigger each failure on each platform. Fill in the matrix (flows × platforms × levels). Inconsistency between platforms signals a specification gap, not an engineering failure.

**Specify:** Product defines the target maturity level per flow, once, platform-agnostic. Error handling becomes a product requirement with the same weight as the happy path.

**Verify:** After implementation, QA confirms each platform meets the spec. "This flow is Level 1 on Android but the spec says Level 2" is an unambiguous, actionable finding.

### Diagnosis Matrix

| Flow | iOS | Android | Web |
|------|-----|---------|-----|
| [Critical flow] | 0–3 | 0–3 | 0–3 |

Fill in each cell with 0–3. Uniform low scores = investment needed. Variance across platforms = specification gap.

### How to Start

1. **Pick three flows.** High-impact: frequent failures, high support volume, or critical moments in the user journey.
2. **Diagnose across platforms.** Trigger failures. Document what the user sees. Fill in the matrix.
3. **Specify the target.** For each flow, write the sad path spec: what the user should experience for each failure condition.
4. **Build to the spec.** Implement on every platform. Use the model's vocabulary during code review.
5. **Verify the result.** Test each failure condition on each platform. Confirm the experience matches the spec.
6. **Expand.** Once the team has muscle memory for sad path specs, expand to more flows.

> _Not every flow needs Level 3. Prioritize by business impact: support ticket volume, churn risk, failure frequency. A low-traffic flow with minimal impact may be fine at Level 1._
