---
name: pushengage-debug
description: Use when invoked by a PushEngage platform spoke (pushengage-ios, etc.) to diagnose a broken integration. Runs a static audit, matches against a version-pinned known-issues table, then falls back to a symptom-driven decision tree. Not invoked directly by customers.
license: MIT
compatibility: Works with any agent that supports the Agent Skills (SKILL.md) standard, including Claude Code and Cowork. Needs filesystem read access and a shell to detect the project type; some steps fetch allowlisted documentation URLs.
metadata:
  author: PushEngage
  version: "1.0.0"
---

# PushEngage — Diagnostic Skill

You arrive here because a platform spoke (e.g., `pushengage-ios`) failed verification or the customer reported a symptom. Your job:

1. Run the platform's static audit first (cheap, often resolves the issue).
2. If audit finds nothing, match the customer's reported version against `known-issues.md`.
3. If still unresolved, walk the customer through the symptom-driven decision tree.
4. If still unresolved, package a bug-report bundle for PushEngage support.

## Phase 1 — Static audit

Read the platform spoke's `audit-checks.md`. Run every check (read the listed files, grep for the listed patterns). Output findings grouped by category (Likely cause / Worth fixing / Cosmetic) with rule IDs.

**If the invoking spoke already ran the audit** (the Flutter and React Native spokes pre-run their `audit-checks.md` before escalating), take its findings as this phase's output — don't re-run the audit. Apply the same stop-gate below to those findings.

If audit returns findings categorized as **Likely cause**, propose fixes and stop here — don't go deeper into Phase 2/3 until those are resolved. The customer can opt to investigate further if they think the audit findings aren't the real issue.

## Phase 2 — Known-issues match

Read `known-issues.md`. Detect the customer's SDK version from their Podfile / Package.resolved / package.json / pubspec.lock. Match the version against the affected-range column. For any hit, present:

- Symptom description.
- Versions affected.
- Whether it's fixed (and in which version) → if so, propose upgrade.
- If not yet fixed → propose workaround from the entry.

## Phase 3 — Symptom-driven tree

Read `diagnostic-tree.md`. Ask the customer to clarify the symptom in one sentence. Branch the tree accordingly. The tree terminates at root cause or "escalate to support."

When branching requires SDK debug logs, read `log-readers.md` for what to ask the customer to enable and what log lines to look for.

## Phase 4 — Support bundle

If the symptom tree terminates at "escalate," generate a paste-able support bundle:

```
PushEngage Bug Report
=====================

Platform: <iOS/Android/...>
SDK version: <x.y.z>
App framework: <UIKit/SwiftUI/...>
Integration mode: <swizzling/manual>

Audit findings: <copy from Phase 1>
Known-issue matches: <copy from Phase 2>
Symptom tree path: <copy the questions and customer's answers>
Captured logs (last 50 lines, sensitive identifiers redacted):
<paste here>

What I tried: <bulleted list>
What I expected: ...
What actually happens: ...
```

Show this to the customer. Ask them to send it to PushEngage support (link from the platform spoke's `references.md`).

## Boundaries

- This skill never edits customer code unless the spoke explicitly authorizes it.
- This skill cites rule IDs from the platform spoke's `best-practices.md` — it doesn't reinvent rules.
- This skill does not invoke other platform spokes.
