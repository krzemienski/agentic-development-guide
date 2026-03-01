# The Prompt Engineering Stack: CLAUDE.md, Rules, Skills, and Hooks

*How a 7-layer defense-in-depth system ensures AI coding agents never cut corners, skip validation, or ship untested code.*

---

## Why It Matters

Every developer who has used an AI coding assistant has experienced the same failure mode: the AI writes beautiful code that doesn't actually work. It generates mock objects instead of calling real APIs. It writes unit tests that test nothing. It skips the build step entirely. And when you ask it to validate, it tells you everything is fine based on vibes rather than evidence.

The problem is not that AI models are incapable. The problem is that they lack persistent, enforceable standards. A human developer internalizes coding standards over years. An AI agent starts every session with a blank slate unless you tell it otherwise.

The ILS project solves this with what we call the **Prompt Engineering Stack** — a hierarchical system of configuration files, rules, hooks, and skills that create 7 layers of defense-in-depth around AI development standards. The system enforces everything from font size minimums to build verification to a complete ban on mock objects and unit tests.

The result: an AI agent that builds real software, validates it through actual user interfaces, and provides screenshot evidence before claiming completion.

---

## The CLAUDE.md Hierarchy

At the foundation of the stack is `CLAUDE.md` — a markdown file that Claude Code reads automatically at session start. But a single file is not enough for a project with multiple scopes. The ILS project uses a three-tier hierarchy:

### Tier 1: Global Instructions (~/.claude/CLAUDE.md)

These apply to every project the developer works on. They define cross-cutting concerns like orchestration patterns, model routing, and agent delegation rules. In the ILS system, the global CLAUDE.md configures oh-my-claudecode (OMC), a multi-agent orchestration layer that routes tasks to specialized agents based on complexity:

- **Haiku**: lightweight scans, quick lookups
- **Sonnet**: standard implementation, debugging, reviews
- **Opus**: architecture, deep analysis, complex refactors

This tier also defines the agent catalog — over 20 specialized agents from `explore` (codebase discovery) to `security-reviewer` (vulnerability analysis) to `deep-executor` (complex autonomous tasks).

### Tier 2: Project Instructions (./CLAUDE.md)

The project-level file contains the most critical mandate in the entire system:

```markdown
## Functional Validation Mandate

**NEVER:** write mocks, stubs, test doubles, unit tests, or test files.
No test frameworks. No mock fallbacks.

**ALWAYS:** build and run the real system. Validate through actual user
interfaces. Capture and verify evidence before claiming completion.
```

This is not a suggestion. It is a hard constraint that propagates through every layer of the stack. The project CLAUDE.md also enforces skill invocation — before any task, the agent must check available skills and invoke any that might apply. Rationalizing away a skill invocation is explicitly forbidden.

### Tier 3: Rules Directory (.claude/rules/*.md)

The rules directory contains 9 specialized files that provide deep context on specific domains. Each file is automatically loaded alongside the CLAUDE.md files, creating a comprehensive instruction set that the AI agent cannot ignore.

This three-tier system means that standards flow downward: global defaults are overridden by project specifics, which are augmented by domain rules. An agent working on the ILS project receives all three tiers simultaneously, creating a rich context of constraints and guidance.

---

## The Nine Rules Files

The `.claude/rules/` directory contains 9 files, each governing a specific aspect of development:

### 1. project.md — Project Quick Reference
The ground truth for the codebase. Tech stack (Swift 5.10+, SwiftUI, Vapor 4), directory structure, build commands, key constants (simulator UDID, backend port, API prefix), and architecture notes. This file prevents the AI from guessing basic facts.

### 2. agents.md — Agent Orchestration
Defines the agent roster and when to use each one. Critically, it mandates parallel execution for independent operations and multi-perspective analysis for complex problems. It also reinforces the validation approach: build real systems, run in simulators, capture evidence.

### 3. development-workflow.md — Feature Implementation
The four-step workflow: Plan First, Build and Validate, Code Review, Commit and Push. Every plan must include a functional validation phase with evidence checkpoints. Plans must never include steps like "write unit tests."

### 4. git-workflow.md — Commit Standards
Conventional commit format (feat, fix, refactor, docs, chore, perf, ci), PR workflow with full commit history analysis, and validation evidence requirements.

### 5. auto-build-hook.md — Continuous Build Verification
Documents the automatic build system that triggers on every Swift file edit. This is not a CI pipeline — it runs locally, immediately, after every single edit.

### 6. feature-gate.md — Premium Subscription System
Guards against bypassing or duplicating feature gates. Defines the `FeatureGate` service, `FeatureGateView` wrapper, and hard rules: never add `if isPremium` checks directly, never hardcode premium status as true.

### 7. fastlane.md — CI/CD Configuration
Build lanes, TestFlight submission, screenshot generation. Separates when to use Fastlane versus direct xcodebuild.

### 8. performance.md — Model Selection and Context Management
Haiku for 90% of Sonnet capability at 3x cost savings. Context window management strategies. Build troubleshooting protocols.

### 9. coding-style.md / security.md / patterns.md
Additional rules covering code style conventions, security practices, and established patterns in the codebase.

---

## Hooks: The Automated Enforcement Layer

Rules tell the AI what to do. Hooks make sure it actually does it. The ILS project uses two hooks configured in `.claude/settings.local.json`:

### The Auto-Build Hook (PostToolUse)

Every time the AI edits a Swift file, the `auto-build-check.sh` script fires automatically:

```bash
case "$FILE_PATH" in
  */ILSApp/*.swift)    xcodebuild -scheme ILSApp -quiet ;;
  */ILSMacApp/*.swift) xcodebuild -scheme ILSMacApp -quiet ;;
  */Sources/*.swift)   swift build ;;
esac
```

The hook detects which target was edited and builds only that target. If the build fails, the hook outputs `BUILD FAILED ($SCHEME): <errors>` and exits with code 1. Claude Code surfaces this as a hook error, and the AI must fix the build before continuing.

This eliminates the most common AI coding failure: making five changes, then discovering three of them broke the build. With the auto-build hook, every edit is immediately verified. The AI cannot proceed with a broken build.

### The Pre-Commit Security Hook (PreToolUse)

Before every `git commit`, a security scan runs on all staged Swift files:

```
Scans for:
- Hardcoded /Users/ paths (warning)
- API key patterns: sk-*, AKIA*, ghp_* (blocked)
- Database files .sqlite/.db (blocked)
```

This hook blocks the commit entirely if it finds API keys or database files in staged changes. Hardcoded user paths generate warnings but do not block. This prevents secrets from ever reaching the repository.

### The Hook Philosophy

The key insight is that hooks run at the tool level, not the conversation level. The AI does not choose whether to run the build — the hook runs automatically whenever the Edit tool touches a Swift file. This removes the possibility of the AI "forgetting" to verify its work.

---

## Skills: Composable Validation Workflows

While CLAUDE.md files provide static instructions and hooks provide automatic enforcement, skills provide dynamic, composable workflows. The ILS project has over 150 skills that can be invoked individually or chained together.

### Skill Composition

Skills compose hierarchically. A high-level skill like `functional-validation` orchestrates lower-level skills:

```
functional-validation
  --> ios-validation-runner (runs the app in simulator)
    --> ios-ui-automation (interacts with UI elements)
  --> ios-validation-gate (verifies evidence meets criteria)
    --> gate-validation-discipline (enforces evidence standards)
```

Each skill in the chain adds specific capabilities. `ios-validation-runner` knows how to boot the dedicated simulator (UDID `50523130`), install the app, and navigate to specific screens. `ios-ui-automation` knows the exact coordinates for sidebar gestures and tab navigation. `ios-validation-gate` knows what constitutes sufficient evidence for a completion claim.

### The No-Mocks Mandate Through Skills

The ban on mocks and unit tests is not just stated in CLAUDE.md — it is enforced through the skill system. When the `functional-validation` skill runs, it:

1. Builds the real application binary
2. Installs it on the dedicated simulator
3. Navigates to the feature under test
4. Captures screenshots as evidence
5. Verifies the screenshots match expected behavior

There is no step where a mock object is created. There is no step where a test framework is invoked. The validation happens through the actual user interface of the actual application running on an actual simulator.

### Model Routing in Skills

Skills leverage model routing to match task complexity to model capability:

- **Haiku** handles exploration and lightweight scanning within skills
- **Sonnet** handles standard implementation and verification
- **Opus** handles architectural decisions and complex analysis

This routing happens automatically based on the skill's requirements, not the developer's manual selection.

---

## Defense-in-Depth: The 7 Layers

The complete system creates 7 layers of defense, each catching failures that slip through the layers above:

### Layer 1: Global CLAUDE.md
Cross-project standards, agent orchestration, model routing. Sets the foundation for how the AI approaches any task.

### Layer 2: Project CLAUDE.md
Project-specific mandates including the no-mocks rule, skill invocation requirements, and planning constraints.

### Layer 3: Rules Files (9 files)
Deep domain context — build commands, architecture patterns, feature gates, security practices. The AI cannot claim ignorance of project conventions.

### Layer 4: Auto-Build Hook
Immediate, automatic build verification after every Swift file edit. Catches compilation errors in real-time, before they compound.

### Layer 5: Pre-Commit Security Hook
Blocks secrets, database files, and hardcoded paths from reaching version control. Last line of defense before code leaves the machine.

### Layer 6: Skills (150+)
Composable validation workflows that enforce real-system testing through actual UI interaction and screenshot evidence.

### Layer 7: Project Memory (MEMORY.md)
Persistent cross-session knowledge that prevents the AI from repeating past mistakes. Records validated patterns, known pitfalls, and lessons learned.

Each layer is independent. If the AI ignores a CLAUDE.md instruction, the hook catches the build failure. If the hook somehow passes, the skill-based validation catches the functional regression. If the skill is skipped, the project memory reminds the AI about mandatory skill invocations.

---

## Setting Up Your Own Stack

You do not need 150 skills to benefit from this approach. Start with the foundation and build up:

### Step 1: Create Your CLAUDE.md
Start with your project's non-negotiable rules. What must the AI always do? What must it never do? Be explicit and absolute — vague guidance produces vague compliance.

### Step 2: Add Rules Files
Create `.claude/rules/` with files for your most common failure modes. If the AI keeps using the wrong build command, create `build.md`. If it keeps ignoring your code style, create `style.md`.

### Step 3: Add the Auto-Build Hook
Configure a PostToolUse hook that runs your build system after every source file edit. This single addition eliminates the most common class of AI coding failures.

### Step 4: Add the Security Hook
Configure a PreToolUse hook on `git commit` that scans for secrets. This takes 10 minutes to set up and prevents credential leaks permanently.

### Step 5: Build Skills Incrementally
As you identify repeated validation patterns, encode them as skills. Start with your most common workflow and expand from there.

### Step 6: Maintain Project Memory
Use MEMORY.md to record lessons learned, validated patterns, and known pitfalls. This prevents the AI from repeating mistakes across sessions.

---

## The Compound Effect

Any single layer of this stack provides marginal improvement. The compound effect of all 7 layers is transformative. The AI agent operating within this stack does not just write code — it builds real software, verifies it works through actual user interfaces, provides screenshot evidence of functionality, and blocks itself from shipping broken builds or leaked secrets.

The prompt engineering stack turns an AI coding assistant from a code generator into a disciplined development partner. It cannot forget the build command. It cannot skip validation. It cannot ship a mock object. It cannot leak an API key. The stack enforces all of this automatically, on every edit, on every commit, on every session.

The investment is front-loaded — building the stack takes real effort. But once built, it pays dividends on every single AI interaction, across every session, for the lifetime of the project. The AI gets better not because the model improves, but because the constraints around it leave no room for the failures that plague unconstrained AI coding.

That is the prompt engineering stack. Not a single file of instructions, but a layered system of enforcement that makes AI development reliable by design.
