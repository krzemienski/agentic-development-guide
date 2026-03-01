# I Banned Unit Tests from My AI Workflow — And Found More Bugs Than Ever

*How a "no mocks" mandate across five projects replaced test theater with real evidence.*

---

## The Controversial Decision

Let me start with the rule that makes most engineers uncomfortable:

> **NEVER write mocks, stubs, test doubles, unit tests, or test files. No test frameworks. No mock fallbacks. ALWAYS build and run the real system. Validate through actual user interfaces. Capture and verify evidence before claiming completion.**

That mandate lives in every project instruction file across my AI-assisted development workflow. It is not a guideline. It is not a suggestion. It is an absolute prohibition enforced at the tooling level, baked into agent prompts, and verified at every completion gate.

Five projects. Zero mock files. Zero test doubles. Over 470 validation screenshots captured in a single marathon session. Thirty-seven numbered gates in one project alone. Three browser automation tools deployed in production validation pipelines.

The results have been unambiguous: functional validation catches categories of bugs that unit tests structurally cannot reach.

This post explains why, how it works in practice, and what the real trade-offs are.

---

## Why Mocks Fail in AI-Assisted Development

The conventional testing wisdom is well-established: write unit tests, mock your dependencies, achieve high coverage, ship with confidence. That wisdom was developed for human developers writing code they deeply understand, modifying it incrementally over months and years.

AI-assisted development breaks every assumption that wisdom rests on.

### The Confidence Illusion

When an AI agent writes code and then writes tests for that code, you get a peculiar failure mode: the tests validate the agent's *understanding* of the requirement, not the *actual behavior* of the system. If the agent misunderstood what "export chat as PDF" means, both the implementation and the test will encode the same misunderstanding. The test passes. The feature is wrong.

I watched this happen repeatedly in early projects. An agent would implement a feature, write comprehensive unit tests with 90%+ coverage, report success, and the feature would be subtly broken in ways that only manifest when a real user touches the real UI.

### Integration Boundaries Are Where Bugs Live

In the projects I work on — iOS apps backed by Swift Vapor servers, web applications with complex browser interactions, multi-service architectures — the most consequential bugs live at integration boundaries. The SSE streaming connection between an iOS client and a Vapor backend. The deep link handler that receives a URL from the operating system. The WebSocket reconnection logic when a tunnel drops.

Mocking these boundaries does not test them. It tests your *model* of them. And in AI-assisted development, where the agent may have written both sides of the boundary in the same session, that model is often circular.

### The "Verification Wars"

The breaking point came during the awesome-site project. An AI agent had completed a complex feature set and reported everything working, backed by a full test suite. When I asked for proof that the actual website rendered correctly in a browser, the response was confident but evasive. I pushed back:

> "I don't see any real verification. Show me the browser."

What followed was a multi-hour session where we discovered that the "fully tested" code had rendering bugs visible in the first three seconds of loading the page. The test suite was green. The application was broken. Every mock had faithfully reproduced the wrong behavior.

That was the last day I allowed mocks in any project.

---

## The Functional Validation Alternative

Functional validation replaces the test-then-ship pipeline with a build-run-screenshot-verify pipeline. The difference is not just philosophical — it changes the entire structure of how work is verified.

### The Core Loop

The workflow is straightforward:

1. **Build the real system.** Compile the actual application, start the actual server, load the actual database.
2. **Run it.** Launch in a simulator, open in a browser, hit real endpoints.
3. **Interact through the UI.** Navigate to the feature. Tap the buttons. Fill the forms. Trigger the edge cases.
4. **Capture evidence.** Screenshots, network logs, console output. Timestamped, numbered, stored.
5. **Verify against expectations.** Does the screenshot show what the spec says? Does the response contain the right data? Does the error state render correctly?
6. **Gate progression.** No evidence, no completion claim. Failed evidence, back to implementation.

This loop runs after every meaningful change. Not at the end of a sprint. Not before a release. After every change.

### What "Evidence" Means

Evidence is not a log line that says "test passed." Evidence is a screenshot of the actual screen showing the actual UI state after the actual interaction. It is a `curl` response from the actual server returning actual data. It is an accessibility tree dump showing that the button exists at specific coordinates and responds to taps.

In the ILS iOS project, a single validation marathon produced 470 numbered screenshots. Each one corresponds to a specific claim: "the sidebar renders with the correct highlight state," "the system monitor shows live CPU metrics," "the theme editor presents all 17 color tokens."

You cannot fake a screenshot. You cannot mock a screenshot. The screenshot either shows the feature working or it does not.

---

## Browser Automation: Three Tools for Real Verification

Web projects require browser-level validation, and I deploy three tools depending on the context and complexity.

### agent-browser CLI

The lightest option. A command-line tool that launches a browser, navigates to a URL, and captures screenshots or extracts content. Best for simple "does this page render" checks and quick visual verification.

Use it when: you need a fast screenshot of a rendered page, you want to verify a deployment is live, or you need to check basic layout.

### Puppeteer MCP (107 Actions)

The workhorse. Integrated as an MCP (Model Context Protocol) server, Puppeteer provides 107 distinct browser actions that AI agents can invoke directly. Click elements, fill forms, wait for network idle, extract text content, capture screenshots at specific viewport sizes.

During the awesome-site project, Puppeteer MCP was the tool that finally proved the rendering bugs existed. The agent could navigate to each page, wait for hydration, capture a full-page screenshot, and compare it against the design specification — all without human intervention.

Use it when: you need multi-step browser interactions, form submissions, navigation flows, or detailed DOM inspection.

### Playwright MCP

The heavy artillery. Playwright provides cross-browser testing (Chromium, Firefox, WebKit), mobile viewport emulation, network interception, and sophisticated waiting strategies. When a feature needs to work across browsers or when timing-sensitive interactions need precise control, Playwright is the tool.

Use it when: cross-browser verification matters, you need network-level observation, or the interaction requires precise timing control.

### The Decision Framework

The choice between tools follows complexity:

- **Single page, single screenshot**: agent-browser
- **Multi-step flow, single browser**: Puppeteer MCP
- **Cross-browser, complex timing**: Playwright MCP

All three produce the same thing: real evidence from real browsers rendering real pages.

---

## iOS Automation: From Accessibility Trees to Tap Coordinates

iOS validation cannot use browser tools. It requires a different pipeline, built on Apple's accessibility infrastructure and the `idb` (iOS Development Bridge) toolchain.

### The idb Pipeline

The core insight is that every iOS UI element has an accessibility representation with exact coordinates. The pipeline exploits this:

```bash
# Step 1: Get the full accessibility tree
idb_describe operation:all
# Returns structured data:
# Button "Settings" centerX:390 centerY:45
# Label "System Monitor" centerX:200 centerY:233
# ScrollView centerX:220 centerY:500

# Step 2: Tap using exact coordinates from the tree
idb_tap 390 45

# Step 3: Capture evidence
idb screenshot /tmp/evidence/settings-screen.png

# Step 4: Verify the screenshot shows expected state
# (Agent analyzes the image against the specification)
```

This pipeline is deterministic. The coordinates come from the system's own accessibility representation, not from guessing pixel positions on a screenshot. Early attempts at guessing coordinates were unreliable — elements shift between device sizes, dynamic type changes layout, and system UI elements consume unpredictable space.

### Lessons Learned the Hard Way

Several non-obvious constraints shape iOS functional validation:

**SwiftUI toolbar buttons are invisible to `idb_tap`.** The standard toolbar modifier renders buttons outside the accessibility tree's tappable region. The workaround: use swipe gestures for sidebar access (`idb ui swipe 5 500 300 500 --duration 0.3`) and deep links (`ils://settings`) for direct navigation.

**The dedicated simulator is sacred.** In a multi-agent development environment, each AI session gets its own simulator UDID. The ILS project's dedicated simulator is `50523130-57AA-48B0-ABD0-4D59CE455F14` (iPhone 16 Pro Max, iOS 18.6). Using another simulator risks corrupting another session's state. This constraint is enforced in project instructions with the severity of a production access control.

**Fresh installs clear UserDefaults.** A `simctl uninstall + install` cycle resets cached configuration. This is actually useful — it validates the first-launch experience — but it caught us off guard when a stale Cloudflare tunnel URL persisted across what we assumed was a clean state.

---

## Gate Validation: 37 Numbered Checkpoints

The most structured application of functional validation is the gate system used in the Code Story project. Thirty-seven numbered gates, each with explicit pass/fail criteria, each requiring evidence before progression.

### How Gates Work

A gate is a checkpoint with three components:

1. **The claim.** A specific, verifiable statement: "The navigation sidebar renders with all 6 menu items visible."
2. **The evidence requirement.** What proves the claim: "Screenshot showing sidebar with Home, Sessions, Projects, Skills, MCP, Settings items."
3. **The pass/fail decision.** Binary. The evidence either supports the claim or it does not.

Gates are numbered sequentially. Gate 14 cannot pass until gates 1 through 13 have passed. A failure at any gate sends work back to implementation. There is no "skip" mechanism and no "known failure" exception.

### The Loop

When a gate fails, the cycle is:

1. Analyze the evidence to identify the defect.
2. Fix the implementation.
3. Rebuild the system.
4. Re-run the gate from the beginning (not just the failed check).
5. Capture new evidence.
6. Re-evaluate.

This loop runs until the gate passes. In practice, most gates pass on the first or second attempt. The gates that fail repeatedly are precisely the ones where the most significant bugs hide — integration boundaries, state management edge cases, timing-dependent behavior.

### Why 37 Gates?

The number is not arbitrary. Each gate corresponds to a user-visible behavior or system property that matters. The granularity ensures that no feature ships with a "well, it mostly works" status. Either the sidebar renders correctly or it does not. Either the WebSocket reconnects within the timeout or it does not. Either the theme editor saves changes that persist across app restarts or it does not.

Thirty-seven gates is not excessive. It is precise.

---

## Results: What Functional Validation Actually Catches

Across five projects with the no-mocks mandate in place, the categories of bugs caught by functional validation cluster into patterns that unit tests structurally miss:

### Visual Rendering Defects

A SwiftUI view can compile, pass type checks, and even render in a preview without matching the design specification. Font sizes, spacing, colors, alignment — these are properties that only manifest in a running UI. The ILS project's HIG font compliance audit found 39 instances of sub-11pt fonts across 14 files. Every one compiled cleanly. Every one violated Apple's Human Interface Guidelines. Screenshots caught them all.

### Integration Boundary Failures

The SSE streaming fix in ILS required a two-tier timeout (30 seconds initial, 5 minutes total) that only revealed its necessity under real network conditions. The Claude CLI subprocess hangs silently when spawned inside an active Claude Code session due to environment variable nesting detection. No mock would have reproduced this — it requires the actual process hierarchy.

### State Management Across Navigation

The `ChatView.task(id:)` fix — where messages failed to reload on session switch — is a classic example. The view appeared to work when navigated to directly. It broke when navigating *between* sessions because the SwiftUI lifecycle did not trigger a re-fetch. Only real navigation through the real sidebar with real session data exposed the bug.

### Platform-Specific Behavior

The macOS build and the iOS build share code but diverge in behavior. `NavigationSplitView` on macOS handles sidebar state differently than the sheet-based sidebar on iOS. `AppDelegate` lifecycle on macOS follows different patterns than `SceneDelegate` on iOS. These divergences only surface when you run both platforms. Mocks for one platform tell you nothing about the other.

---

## The Trade-Offs

Functional validation is not free. The costs are real and should be acknowledged honestly.

### Speed

A unit test suite runs in seconds. A functional validation pass through 37 gates takes hours. The 470-screenshot marathon was exactly that — a marathon. Build times, simulator boot times, network latency for real API calls — these add up.

The mitigation is parallelism. Multiple agents validating different gates simultaneously. Background builds while evidence is being analyzed. But the wall-clock time is still longer than `pytest -x`.

### Determinism

Screenshots are subject to rendering variations. A system alert, a low-battery warning, a time-zone difference in a timestamp — these can cause false negatives. The accessibility tree approach reduces this (coordinates are deterministic), but the visual verification step introduces human judgment or AI image analysis, both of which have error rates.

### Coverage Granularity

Unit tests can exercise individual functions with crafted inputs. Functional validation exercises user-visible paths. If a bug hides in a code path that no UI interaction reaches, functional validation will not find it. This is a real gap, partially mitigated by the fact that unreachable code paths are, by definition, not affecting users.

### Initial Setup Cost

Configuring `idb`, establishing simulator conventions, building browser automation pipelines — the upfront investment is significant. The ILS project's simulator automation rules alone span dozens of documented lessons learned, from swipe gesture coordinates to DerivedData path conventions.

---

## When This Approach Makes Sense

Functional validation is not universally superior. It is specifically superior for AI-assisted development workflows where:

- The agent writes both implementation and tests (circular validation risk).
- Integration boundaries dominate the bug surface area.
- Visual correctness matters (UI applications).
- Multiple platforms share code but diverge in behavior.
- Evidence-based completion claims are required (not just "tests pass").

For pure library code with well-defined mathematical properties, unit tests remain appropriate. For algorithm implementations where edge cases are enumerable, property-based testing is strong. The no-mocks mandate applies to *application development* in *AI-assisted workflows*, not to all software development universally.

---

## Conclusion

The decision to ban mocks was not ideological. It was empirical. Across five projects, traditional test suites consistently failed to catch the bugs that mattered most — the ones users would hit in the first thirty seconds of using the application.

Functional validation is slower, harder to set up, and less granular. It is also *honest*. A screenshot does not lie about whether the feature works. A green test suite, backed by mocks that encode the same misunderstandings as the implementation, absolutely can.

Zero mock files. 470 screenshots. 37 gates. Three browser tools. The evidence speaks for itself.

---

*This post is part of a series on AI-assisted development practices. The tools and workflows described are used in production across iOS, macOS, and web projects.*
