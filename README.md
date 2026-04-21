# Error Handling Maturity Model

This repo contains two things:

1. **The model** — a UX-centered framework for specifying, building, and verifying how applications handle failure
2. **The skill** — a Claude Code slash command that automates the diagnosis step against any codebase

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
| File upload | 3 | 1 | 0 |
| User login | 2 | 2 | 1 |
| Background sync | 0 | 0 | 0 |

Uniform low scores = investment needed. Variance across platforms = specification gap (one team figured it out; that knowledge was never shared or required).

### The practice

**Diagnose → Specify → Verify**

- **Diagnose:** Trigger failures across platforms. Fill in the matrix.
- **Specify:** Product defines target maturity per flow, once, platform-agnostic. Error handling becomes a product requirement.
- **Verify:** QA confirms each platform meets the spec. "This flow is Level 1 on Android but the spec says Level 2" is an unambiguous, actionable finding.

The full framework is in [`error-handling-maturity-model.md`](error-handling-maturity-model.md).

---

## The Skill

The skill is a Claude Code slash command that automates the Diagnose step. Point it at a local directory or GitHub repo and it scans the codebase, classifies every error handling site against the model, and produces a full maturity report.

### Installation

Copy `error-maturity/` into `~/.claude/skills/`:

```bash
cp -r error-maturity ~/.claude/skills/
```

The slash command is available immediately in any Claude Code session.

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
