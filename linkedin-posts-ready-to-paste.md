# LinkedIn Posts — Ready to Paste

Open LinkedIn in your regular browser, click "Start a post", paste the text below.
GitHub repo: https://github.com/krzemienski/agentic-development-guide

---

## POST 1: Series Launch (Post this first)

8,481 AI coding sessions. 90 days. 10 blog posts. Here's what I learned.

I've been building software almost entirely with AI coding agents for the past three months. Not the "ask ChatGPT to write a function" kind. The "coordinate 30 agents working in parallel across a shared codebase" kind.

The result: 10 deeply technical blog posts covering patterns that don't exist in any documentation yet.

Here are five that surprised me most:

1. A single AI agent reviewed my streaming code and said "looks correct." Three agents running a structured consensus audit caught a P2 bug on the first pass. The fix was changing += to = on one line.

2. I spawned 194 parallel git worktrees, each with its own AI coding agent. Factory-scale software development is real — but the QA pipeline matters more than the agents.

3. I banned unit tests from my AI workflow entirely. Zero mocks, zero stubs. Instead: 470 screenshots, 37 validation gates, and 3 browser automation tools. Every bug my unit tests would have missed, functional validation caught.

4. Connecting an iOS app to Claude's API required 5 layers and 4 failed attempts. The working architecture exists because of an auth constraint, not a technical one.

5. I built an "operating system" for AI development: 25+ specialized agent types, persistence loops, spec pipelines, and project management — all composing into a single workflow.

Every code snippet comes from real sessions. No fabricated examples.

Full series with diagrams, architecture maps, and code: https://github.com/krzemienski/agentic-development-guide

What patterns are you discovering with AI coding tools?

#AgenticDevelopment #AI #SoftwareEngineering

---

## POST 2: Multi-Agent Consensus (Most engaging topic)

A single AI agent reviewed my code and said "looks correct."

Three agents found a P2 bug on line 926.

I was building a streaming chat interface for an iOS app. Messages arrived, tokens flowed, the UI looked responsive. One agent scanned the file, noted some style inconsistencies, and gave the green light.

The bug: every streamed token appeared twice. The word "Four" rendered as "Four.Four." on screen.

Root cause: message.text += textBlock.text when it should have been message.text = textBlock.text. One character. The += appended to already-accumulated content. The = would have treated each event as authoritative.

A second root cause compounded it: the stream-end handler reset the message index to zero, replaying the entire buffer.

Why did one agent miss it? Pattern matching. The code had the shape of correctness — delta events, accumulation buffers, state updates. Each line made sense in isolation. The bug was in how two mechanisms interacted.

The three-agent approach: Lead, Alpha, and Bravo each get independent review prompts. Lead checks architecture. Alpha checks logic and code paths. Bravo checks visual behavior and states. They vote independently. Unanimous pass required.

Alpha caught both root causes in the first pass because it was specifically looking at data flow interactions — not just whether individual lines were correct.

I've now run this pattern across 3 projects with 10 blocking gates each. The cost per gate is roughly $0.15. The P2 bug would have shipped to users.

The detailed technical writeup with system diagrams: https://github.com/krzemienski/agentic-development-guide/tree/main/04-multi-agent-consensus

#CodeQuality #AI #SoftwareEngineering

---

## POST 3: Functional Validation (Contrarian take)

I banned unit tests from my AI development workflow.

Before you close this tab: I'm not anti-testing. I'm anti-false-confidence.

When an AI agent writes both the implementation AND the unit tests, a passing test suite is not independent evidence of correctness. The agent validates its own assumptions. It's a closed loop.

So I replaced it with functional validation: build the real system, run it, screenshot it, verify against the spec. No mocks. No stubs. No test doubles.

The numbers after 90 days:

- 470 evidence screenshots captured
- 37+ validation gates across multiple projects
- 3 browser automation tools evaluated (Puppeteer MCP, Playwright MCP, agent-browser)
- 4 distinct bug categories caught that unit tests systematically miss

The four categories: visual rendering bugs (elements overlap, wrong colors), integration boundary failures (API returns unexpected format), state management bugs (works once, breaks on second interaction), and platform-specific issues (works on iPhone 16 but not iPad).

None of these show up in unit tests. All of them show up when you actually run the app and look at it.

The tradeoff is real: functional validation is slower, less deterministic, and harder to parallelize. But it catches a fundamentally different class of bugs. And when AI agents are writing your code, that class of bugs is the one that ships.

Full methodology with code and diagrams: https://github.com/krzemienski/agentic-development-guide/tree/main/07-functional-validation

#Testing #AI #SoftwareEngineering

---

## POST 4: Native iOS Client for Claude Code (Blog 01)

I built a native iOS app that streams Claude Code responses to my phone.

It required 4 languages, 5 processes, and 6 serialization boundaries.

The stack: SwiftUI frontend -> Vapor backend -> Python SDK wrapper -> Claude CLI -> Anthropic API. Each token traverses this entire chain as an SSE event. The total path from API to pixel: ~50ms per token.

Why so many layers? One auth constraint broke every simpler approach.

Claude Code authenticates via OAuth through the CLI. There is no way to extract the OAuth token for direct API calls. So the Python SDK wrapper inherits the CLI's auth by spawning it as a subprocess. The Vapor backend bridges Python's stdout (NDJSON) into Server-Sent Events. The SwiftUI client consumes SSE natively.

Three things I learned building this:

1. Python stdout buffering will silently break streaming. Without flush=True on every write, users see nothing for seconds, then a burst of text. This single line took 2 days to find.

2. Claude CLI detects nesting. If you spawn claude inside an active Claude Code session, it refuses to run. Fix: strip CLAUDECODE=1 and CLAUDE_CODE_* env vars before subprocess launch.

3. The two-character bug (message.text += vs =) that duplicated every token was invisible to single-agent review. Three independent agents caught it in one pass.

763 Claude Code sessions went into this. The companion repo has the reusable Swift Package: https://github.com/krzemienski/claude-ios-streaming-bridge

Full writeup: https://github.com/krzemienski/agentic-development-guide/tree/main/01-ils-ios-client

#iOSDevelopment #AI #SwiftUI

---

## POST 5: 5 Layers to Call an API (Blog 02)

I needed 5 layers and 4 failed attempts to call a single API from my iOS app.

Attempt 1: Direct Anthropic API from Swift. Failed — no OAuth token available.
Attempt 2: JavaScript SDK via Node subprocess. Failed — NIO event loops don't pump RunLoop.
Attempt 3: Swift ClaudeCodeSDK in Vapor. Failed — FileHandle.readabilityHandler needs RunLoop, which NIO doesn't provide.
Attempt 4: Direct CLI invocation. Failed — nesting detection blocks claude inside claude.

Attempt 5 worked: Python SDK wrapper, spawned as subprocess, NDJSON stdout, environment variables stripped.

The lesson is counterintuitive: the 5-layer architecture is simpler than any of the "simpler" approaches. Each layer does exactly one translation. Each boundary has one failure mode. Replacing any layer requires changes to one file.

When the Python SDK updates its interface? 8 lines change. When Anthropic eventually exposes OAuth tokens for third-party use? Layers 3 and 4 collapse into a direct call without touching the iOS client.

The failed attempts are documented in the companion repo because they are the most valuable part. Every failure has a specific technical reason. Every "simpler" approach has a specific incompatibility.

The detailed failure archaeology: https://github.com/krzemienski/claude-sdk-bridge

Full writeup: https://github.com/krzemienski/agentic-development-guide/tree/main/02-agent-sdk-bridge

#SystemDesign #AI #SoftwareArchitecture

---

## POST 6: 194 Parallel AI Worktrees (Blog 03)

I spawned 194 git worktrees, each running its own AI coding agent, all building the same codebase in parallel.

Here's what factory-scale AI development actually looks like:

Stage 1: Ideation. Claude analyzes the codebase and generates a task manifest with priorities and dependencies.

Stage 2: Specification. Each task gets a detailed spec with acceptance criteria, file scope, and constraints. Bad specs produce bad code — this stage matters more than execution.

Stage 3: Parallel Execution. Git worktrees provide free isolation. Each agent works in its own directory, on its own branch. No merge conflicts during development. No stepping on each other's work.

Stage 4: QA. A SEPARATE agent reviews each completed task. Self-review doesn't work — the same biases that produced the bug cause the agent to overlook it. Independent QA agents produce three verdicts: Approved, Rejected-with-fixes, or Rejected-permanent.

The numbers: 91 specs generated, 71 QA reports produced, 3,066 sessions total. Not every QA report was an approval on first pass. The rejection-and-fix cycle is where the real quality comes from.

The key insight: the QA pipeline matters more than the agents. Fast code with no review ships bugs. Slower code with independent QA ships features.

Companion repo with the full CLI tool: https://github.com/krzemienski/auto-claude-worktrees

Full writeup: https://github.com/krzemienski/agentic-development-guide/tree/main/03-auto-claude-worktrees

#DevOps #AI #Automation

---

## POST 7: The 7-Layer Prompt Engineering Stack (Blog 05)

Your AI coding agent is only as good as the constraints around it.

I built a 7-layer defense-in-depth system that makes it impossible for my agents to cut corners, skip validation, or ship untested code. Here are the layers:

Layer 1: CLAUDE.md — Global rules loaded into every session. "Never write mocks. Never skip validation."

Layer 2: .claude/rules/ — Project-specific instructions. Build commands, architecture patterns, feature gates.

Layer 3: Skills — Reusable validation protocols invoked by name. Functional validation, iOS automation, gate checking.

Layer 4: Hooks — Auto-build after every .swift edit. Pre-commit security scans that block API keys and database files.

Layer 5: Agent definitions — Specialized roles (planner, executor, reviewer) with scoped prompts and tools.

Layer 6: Prompt libraries — YAML-based prompt templates with variable interpolation for repeatable workflows.

Layer 7: Session memory — Persistent notes across sessions so agents never lose context on project conventions.

Any single layer provides marginal improvement. The compound effect of all 7 is transformative.

The agent cannot forget the build command. Cannot skip validation. Cannot ship a mock. Cannot leak an API key. All enforced automatically, on every edit, every commit, every session.

Template repo to build your own stack: https://github.com/krzemienski/claude-prompt-stack

Full writeup: https://github.com/krzemienski/agentic-development-guide/tree/main/05-prompt-engineering-stack

#PromptEngineering #AI #DeveloperProductivity

---

## POST 8: Ralph Orchestrator — Rust Platform for AI Agent Fleets (Blog 06)

When you need 30 AI agents working simultaneously across a shared codebase, you need an orchestration layer.

Ralph Orchestrator is a Rust platform I built for exactly that: coordinating fleets of AI coding agents with a hat-based context system, event-sourced merge queues, and a Telegram control plane.

The "hat" system is the core insight. Each agent wears a hat that defines its context: which files it can see, what tools it has, what role it plays. Swapping hats changes an agent's entire perspective without restarting the session.

Six tenets emerged from 410 orchestration sessions:

1. Isolation via git worktrees — each agent gets its own filesystem
2. Event-sourced merge queues — deterministic conflict resolution
3. Backpressure gates — agents self-throttle when quality drops
4. Telegram control plane — monitor and steer from your phone
5. Persistent loops — agents that survive session boundaries
6. Evidence-based verification — no self-reported success

The hardest problems aren't technical. They're about trust calibration: when should an agent ask a human? When should it proceed autonomously? How do you tune backpressure so agents stay productive without being reckless?

These are orchestration problems, not AI problems. And that's exactly why a dedicated platform exists.

Getting started guide: https://github.com/krzemienski/ralph-orchestrator-guide

Full writeup: https://github.com/krzemienski/agentic-development-guide/tree/main/06-ralph-orchestrator

#Rust #AI #PlatformEngineering

---

## POST 9: GitHub Repos to Audio Stories (Blog 08)

What if you could listen to a GitHub repository?

I built Code Tales: a platform that clones a repo, analyzes its architecture with Claude, generates a narrative script, synthesizes speech with ElevenLabs, and streams the finished audio. Under 2 minutes for most repos.

9 narrative styles: debate, documentary, executive, fiction, interview, podcast, storytelling, technical, tutorial. Each produces a fundamentally different experience from the same codebase.

The build process was the real story: 636 commits, 90 worktree branches, 91 specs, 37 validation gates. The entire application was built using the parallel AI development patterns from earlier in this series.

The audio debugging saga was the most instructive part. Nine commits to fix race conditions in the audio player. Each identified, implemented, and validated by agents working in parallel. A solo developer would have spent days. The agents compressed it into hours.

Hand it a GitHub URL today and you get back a narrated story that is accurate, well-paced, and delivered in a voice matching the style you chose.

Try it: https://github.com/krzemienski/code-tales

Full writeup: https://github.com/krzemienski/agentic-development-guide/tree/main/08-code-tales

#DeveloperTools #AI #AudioContent

---

## POST 10: 21 Screens, Zero Figma Files (Blog 09)

I described an entire application in plain text and AI generated 21 production screens in one session.

No Figma. No design handoff. No pixel-pushing.

The tool: Stitch MCP — a design-to-code pipeline that takes text descriptions, generates design tokens, produces React components with Tailwind CSS, and validates everything with Puppeteer automation.

What one session produced:
- 21 screens (auth flow, dashboard, admin with 20 tabs, settings, profiles)
- 47 design tokens in a brutalist palette (all borderRadius: 0px)
- Button component with CVA: 8 variants, 8 sizes, forwardRef, Radix Slot
- 105 Puppeteer validation checks across 374 actions

The validation layer is what makes this production-grade, not a demo. Every screen is automatically tested for rendering, navigation, interactive states, and responsive behavior.

The remaining challenges aren't technical — they're procedural. Prompt discipline. Branding governance. Validation coverage. Process problems have process solutions.

Design-to-code is not a future workflow. It happened. In one session.

Workflow template: https://github.com/krzemienski/stitch-design-to-code

Full writeup: https://github.com/krzemienski/agentic-development-guide/tree/main/09-stitch-design-to-code

#DesignSystems #AI #FrontendDevelopment

---

## POST 11: The AI Development Operating System (Blog 10)

After 8,481 sessions over 90 days, I accidentally built an operating system for AI development.

Not a tool. Not a framework. A composable meta-system where specialized agents, persistence loops, spec pipelines, and deliberation protocols work together to ship software.

The layers:

OMC (Oh My Claude Code) — 25+ specialized agent types across build, review, domain, and coordination lanes. Each agent has a specific role, model, and prompt.

Ralph Loop — Persistent execution that survives session boundaries. The agent works, verifies, and continues until the task is done or it's blocked.

Specum Pipeline — Specification-driven development: research -> requirements -> design -> tasks -> execution -> verification.

RALPLAN — Adversarial planning where Planner, Architect, and Critic iterate until consensus. No plan ships without challenge.

GSD (Get Stuff Done) — Project lifecycle: roadmap -> phase planning -> execution -> verification.

Team Pipeline — N coordinated agents with staged execution: plan -> PRD -> exec -> verify -> fix loop.

The key insight: AI-assisted development benefits from the same organizational principles human engineering teams discovered decades ago. Specialization. Persistence. Structured processes. Adversarial review. Evidence-based verification.

The models are capable enough. What they need is a system.

Starter kit: https://github.com/krzemienski/ai-dev-operating-system

Full writeup: https://github.com/krzemienski/agentic-development-guide/tree/main/10-ai-dev-operating-system

#AgenticDevelopment #AI #SoftwareEngineering
