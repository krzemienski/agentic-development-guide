# How 3 AI Agents Found a Bug I Would Have Shipped

*Nick Krzemienski -- March 2026*

---

## The Bug That Almost Shipped

Line 926 of `ChatViewModel.swift`. That is where the P2 bug lived.

I had been building the streaming chat interface for ILS -- a native iOS client for Claude Code -- and everything appeared to work. Messages arrived, tokens flowed in real time, the UI felt responsive. I ran a single-agent code review. The agent scanned the file, noted some minor style inconsistencies, and reported: "Streaming implementation looks correct."

The bug was already in the codebase. It had been there for three days.

Here is what was happening. When a user sent a message to Claude and the response streamed back, every token appeared twice. The word "Four" would render as "Four.Four." on screen. The assistant message handler was using `+=` to append text blocks -- but those text blocks already contained the full accumulated content from prior `textDelta` streaming events. Append plus authoritative full text equals duplication.

There was a second, subtler cause. The stream-end handler reset `lastProcessedMessageIndex` to zero. On the next observation cycle, the entire SSE message buffer replayed from the beginning, feeding already-processed messages back through the rendering pipeline. Double processing on top of double accumulation.

```swift
// ChatViewModel.swift -- the two root causes

// Root cause 1: append vs assignment
// BEFORE (bug):
message.text += textBlock.text    // appends to already-accumulated content

// AFTER (fix):
message.text = textBlock.text     // assignment -- assistant event is authoritative

// Root cause 2: index reset
// BEFORE (bug):
self.lastProcessedMessageIndex = 0   // replays all messages next cycle

// AFTER (fix):
let finalCount = sseClient.messages.count
self.lastProcessedMessageIndex = finalCount  // preserves position
```

The result in practice: every streaming response stuttered visibly. Tokens appeared, doubled, then resolved to the correct text once streaming completed. It would not crash. It would not fail silently. It would just look broken -- the kind of bug that erodes trust in a product the first time a user sees it.

A single agent reviewed this code and said it looked fine. Three agents running a structured consensus audit caught both root causes in the first pass.

That gap is what this post is about.

---

## Why Single-Agent Review Falls Short

When you ask one AI agent to review code, it brings a single perspective. It is fast, it is cheap, and for many classes of errors it works well. But it has a structural weakness: it reviews what it is looking for.

An agent prompted to check "does this code look correct?" will pattern-match against common errors. It reads the streaming implementation and sees the shape of correctness -- delta events, accumulation buffers, state updates. The logic *looks* right. Each line makes sense in isolation. The mistake is that two update mechanisms, each individually reasonable, interact badly when combined.

The `+=` operator on `message.text` makes sense if you are building text incrementally from deltas. The `textBlock.text` containing the full accumulated content makes sense if the assistant event is authoritative. Both are valid patterns. The bug exists only at their intersection.

Human code review has the same failure mode. Individual reviewers develop blind spots. The pattern recognition that makes you productive is exactly what causes you to miss novel bugs. You see what you expect to see.

Multi-agent consensus addresses this structurally. Not by making each agent smarter, but by ensuring that genuinely independent perspectives must all agree before work advances.

---

## The Consensus Pattern

The pattern is simple in concept: **three agents review independently, all three must vote PASS, or the gate stays closed.**

Here is how it works in practice. Each agent receives the same checkpoint prompt but starts with zero shared context:

```
CONSENSUS VALIDATION - Phase 1 Complete
You are BACKEND VALIDATOR. Independently verify:
  1. Run lint && typecheck && build
  2. Check that all API endpoints return 200
  3. Report PASS or FAIL with evidence
```

The key word is "independently." Each agent starts fresh -- no ability to anchor on another agent's conclusions. They produce separate evidence artifacts. Only when all three evidence files show PASS does the gate open.

The three roles are specialized by what they catch best:

**Alpha (code and logic specialist)** -- reads implementation line by line. Looks for incorrect accumulation patterns, off-by-one errors in state machines, race conditions, API contract violations. Alpha caught the ChatViewModel `+=` vs `=` bug. It flagged: "Line 926 appends the content delta AND sets the full accumulated text. The assistant event is authoritative; this should be assignment, not append."

**Bravo (visual and functional specialist)** -- exercises the running system. Looks for UI behavior under real conditions, edge cases that only appear with actual data, regressions in previously working flows. Bravo confirmed the fix by running the app and verifying responses rendered as "Four." and "Six." -- not "Four.Four." and "Six.Six."

**Lead (architecture and consistency specialist)** -- validates the whole. Looks for cross-component consistency, pattern compliance, whether a fix introduced new inconsistencies elsewhere. Lead confirmed Alpha's finding, cross-checked the fix across both the SDK and CLI execution paths, and verified `lastProcessedMessageIndex` was correctly preserved.

The three roles are not arbitrary. They are calibrated so that what Alpha misses in the running UI, Bravo catches, and what both miss at the architectural level, Lead finds. The specialization creates genuine independence.

---

## Three Projects, Three Implementations

I have applied this pattern across three different projects. Each implementation taught me something new about where consensus adds value and where it creates overhead.

### ILS iOS: The 10-Phase Sequential Audit

The most structured deployment. ILS iOS underwent a 10-phase audit with gates at each phase boundary. Three agents -- Lead, Alpha, Bravo -- reviewed each phase independently before work advanced.

The phases covered distinct concerns:

- **Phases 1-3**: Architecture, navigation, layout verification
- **Phases 4-6**: Feature implementation -- skills, plugins, system monitor, themes
- **Phases 7-9**: Integration, cross-platform (iOS + macOS), functional bug hunting
- **Phase 10**: Final gate with full requirement traceability (REQ-01 through REQ-15)

Each gate required explicit evidence artifacts -- not just "PASS" text, but build output, screenshots, curl responses, process verification. The streaming duplication bug was caught during the backend/API audit phase, when Alpha performed an independent code review of the SSE streaming pipeline.

What the sequential structure added: **a bug caught at Phase 4 cannot slip into Phase 5.** The gate is a hard stop. There is no "we will fix it later" because later is gated on this phase passing cleanly. When Alpha flagged the duplication, work stopped. The fix was implemented. All three agents re-validated. Only then did the pipeline advance.

The evidence from the fix verification is concrete. The commit message reads: "Verified: 3-agent consensus PASS, Bravo visual confirmation shows 'Four.' and 'Six.' (not 'Four.Four.' / 'Six.Six.')."

### Awesome Site: Cross-Validator Consensus

A web project with a different decomposition: `backend-validator`, `frontend-validator`, and `cross-validator`. The cross-validator is the interesting addition -- it explicitly looks for mismatches *between* layers.

A common failure mode in web development is that the API works, the UI works, but they disagree on a field name or a data shape. Single-agent review catches this poorly because it tends to examine one layer at a time. The cross-validator is specifically tasked with finding seams.

The cross-validator caught two issues that the other validators each independently missed:

1. **camelCase vs snake_case mismatch** in error response fields. The backend serialized `error_code`; the frontend expected `errorCode`. Both validators saw correct serialization and correct deserialization -- in isolation, each was right. The cross-validator caught the contract disagreement.

2. **Pagination parameter naming**. The frontend sent `page` as a query parameter. The backend expected `offset`. Both worked in their own test contexts (defaulting to page 1 or offset 0). The cross-validator found the mismatch by tracing a request from button click through to database query.

### Code Story: 37+ Numbered Validation Gates

The most granular application. Rather than one consensus gate per phase, Code Story used numbered validation gates throughout -- 37 across the full implementation, each a standalone agent checkpoint.

The density was intentional. Code Story had a complex transformation pipeline where each stage's output fed the next stage's input. The cost of a bug propagating through multiple stages was high -- a schema error at stage 5 could corrupt everything downstream through stage 15 before anyone noticed.

A narrowly-scoped gate at each stage caught problems before they compounded. The lesson: gate density should match the cost of late detection. In a linear pipeline where stage N feeds stage N+1, bugs compound exponentially. Catching them early is dramatically cheaper than unwinding.

---

## Gate Structure in Detail

The gate itself is the critical mechanism. Here is how a gate actually executes:

1. **Phase N tasks complete** -- all implementation work for the phase is done
2. **Lead signals gate check** -- the orchestrator sends identical checkpoint prompts to all three agents
3. **Agents run independently** -- no shared state, no visibility into each other's work
4. **Each produces an evidence artifact** -- build logs, screenshots, curl output, code analysis
5. **System checks unanimity** -- all three artifacts must contain explicit "PASS"
6. **Gate opens or stays closed** -- unanimous PASS advances; any FAIL triggers fix cycle

The fix cycle is where the pattern earns its overhead cost:

1. The failing agent's evidence identifies the specific issue
2. A fix is implemented
3. **All three agents re-validate** -- not just the one that failed

That third step is critical. Re-validating all three after a fix catches the failure mode where a fix resolves the original issue but introduces a regression. The agent that previously passed might now fail on something the fix broke. You do not get to carry forward stale passing votes.

In the ILS streaming fix, after Alpha flagged the `+=` bug and the `lastProcessedMessageIndex` reset, the fix was implemented. Then all three re-ran:

- Alpha confirmed the code logic was correct (assignment, not append; index preserved)
- Bravo ran the app and verified clean single-token rendering
- Lead cross-checked that both SDK and CLI execution paths used the same corrected handler

All three PASS. Gate opened. Phase advanced.

---

## The TeamDelete Gotcha

One practical lesson from implementing multi-agent orchestration with Claude Code's native Teams API: **teammates must call the `shutdown_response` tool explicitly to terminate.**

Plain text acknowledgment does not count. If an agent responds "Understood, shutting down" in natural language, it does not actually terminate. The system waits. Your orchestration hangs. I hit this on the first multi-agent run -- two of three agents shut down cleanly, the third had acknowledged the shutdown in prose and stayed alive indefinitely.

The correct pattern:

```python
# Orchestrator sends shutdown request via SendMessage
SendMessage(team_name, "shutdown_request")

# Teammate MUST call the tool -- not just acknowledge in text
shutdown_response()  # explicit tool call required

# Only then can TeamDelete proceed
TeamDelete(team_name)
```

The fix is to make the shutdown tool call a mandatory part of each agent's task completion protocol. Not a polite request -- a required terminal action, enforced in the agent's system prompt.

---

## Results: What the Pattern Actually Caught

Across the three projects, the consensus pattern caught bugs that single-agent review reliably missed:

**ILS iOS (10-phase, 4-gate)**: The P2 streaming text duplication -- two root causes in `ChatViewModel.swift`. The `+=` vs `=` operator choice and the `lastProcessedMessageIndex` reset to zero. Caught at the backend audit gate by Alpha. Would have shipped: the single-agent review three days earlier explicitly approved the streaming implementation.

**Awesome Site (phased consensus)**: Two API contract mismatches across the frontend/backend boundary. The camelCase/snake_case error field and the page/offset pagination parameter. Both caught by the cross-validator role -- a perspective that has no equivalent in single-agent review.

**Code Story (37 gates)**: Multiple schema drift issues where transformation stages produced subtly wrong output that only manifested when consumed by downstream stages. The granular gates made each issue trivially easy to isolate -- the gate number told you exactly which stage introduced the problem.

The pattern does not just catch more bugs. It catches them earlier. A bug found at a phase gate costs one fix cycle. The same bug found in production costs incident response, user-facing degradation, and a hotfix deployment.

---

## When to Use This Pattern (and When Not To)

Three-agent consensus is not free. Running three independent agents takes roughly 3x the compute of a single-agent review. If run concurrently you pay in compute but not wall clock time; if sequential, you pay both. The question is whether the overhead is justified.

For the ILS streaming bug: absolutely yes. A P2 bug in a live product's core chat interface would have required a hotfix release and trust repair that takes weeks. The consensus pass that caught it cost minutes.

**Use three-agent consensus when:**

- Complex state management where multiple update mechanisms interact (streaming, SSE, accumulation buffers)
- Multiple system layers must agree on data contracts (field names, serialization formats, pagination schemes)
- A bug would be immediately visible to users and degrade product trust
- Comprehensive audits across a large surface area (10+ phases, 50+ files)
- Sequential pipelines where bugs compound across stages

**Stick with single-agent review when:**

- The change is isolated to one file with clear, simple logic
- The risk of a missed bug is low -- non-critical path, easily caught in use
- Speed matters more than thoroughness -- prototyping, exploratory work

---

## The Broader Principle

What makes the consensus pattern work is not the number three specifically. It is the combination of two structural properties: **independent verification** and **hard gates**.

Independent verification means each agent starts from scratch with no anchor on another agent's conclusions -- the AI equivalent of not showing one code reviewer another reviewer's comments before they have formed their own opinion. This eliminates groupthink.

Hard gates mean that "mostly passing" is not passing. Two-out-of-three does not open the gate. All three must report PASS with evidence. This eliminates the reviewer who waves something through because the previous reviewer already approved it.

Together, these properties catch what any individual perspective misses. The `+=` operator was right there on line 926. It looked correct because the pattern -- accumulate text in a streaming handler -- *should* use append. The bug was that this particular text was already accumulated. You need a fresh perspective to see what pattern matching missed.

The P2 streaming bug lived in my codebase for three days. One structured consensus pass found both root causes in minutes. That is the value of making agents disagree with each other before letting your code ship.

---

*The ILS iOS source is at [github.com/krzemienski/ils-ios](https://github.com/krzemienski/ils-ios). The 10-phase audit plans are in `.planning/phases/`, and the consensus evidence artifacts are referenced in each phase's verification documents.*

---

## Companion Repository

**[`multi-agent-consensus`](https://github.com/krzemienski/multi-agent-consensus)** — Framework for 3-agent consensus validation with hard gates

---

*Part 4 of 10 in the **Agentic Development** series — [View all posts](https://github.com/krzemienski/agentic-development-guide)*

*Nick Krzemienski — March 2026*
