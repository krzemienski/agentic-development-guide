# Ralph Orchestrator: Building a Rust Platform for AI Agent Armies

When a single AI coding agent can modify a handful of files per session, that feels like magic. When you need thirty agents working simultaneously across a shared codebase -- each planning, coding, and merging in parallel -- the magic breaks down into an engineering problem. Ralph Orchestrator is a Rust-based platform built to solve that problem: coordinating fleets of AI coding agents at scale, with a hat-based context system, event-sourced merge queues, a Telegram control plane, and a companion iOS app validated across 1,705 screenshots.

This is the story of how it was built, what it learned, and why orchestrating agents is fundamentally different from orchestrating containers.

---

## The Scale Problem

Most AI coding tools operate in a single-session model. You open a chat, describe a task, and the agent modifies files in your working directory. This works until your project has fifty open tasks and you want them done before lunch.

The naive approach -- running thirty Claude Code sessions in thirty terminal tabs -- collapses immediately. Agents overwrite each other's files. They lose context mid-task and start contradicting earlier decisions. Merge conflicts pile up faster than any human can resolve them. And you have no visibility into what any of them are actually doing.

Ralph started as a question: what if there were a thin coordination layer that treated AI agents the way Kubernetes treats containers? Not by micromanaging their work, but by giving them isolated environments, clear objectives, and a structured way to hand off results.

The answer became a Rust workspace of ten crates, a Telegram bot, a web dashboard, and a native iOS companion app -- all built around six tenets that emerged from running 410 orchestration sessions totaling over a gigabyte of interaction data.

---

## Architecture: A Thin Coordination Layer

Ralph's architecture follows a deliberate philosophy: the orchestrator is a thin coordination layer, not a platform. Agents are smart; let them do the work.

The Rust workspace is organized into focused crates:

- **ralph-core**: The heart of the system. Event loop orchestration, hat management, memory persistence, task tracking, merge queues, and worktree coordination.
- **ralph-cli**: The command-line entry point. Commands for `run`, `plan`, `task`, `loops`, and `web`.
- **ralph-adapters**: Backend integrations for Claude, Kiro, Gemini, Codex, and other AI providers.
- **ralph-telegram**: The Telegram bot for human-in-the-loop communication.
- **ralph-tui**: A ratatui-based terminal UI for real-time monitoring.
- **ralph-mobile-server**: The API server powering the companion iOS app.
- **ralph-api**: RPC protocol definitions for the web client.
- **ralph-proto**: Shared protocol types -- events, hats, topics.
- **ralph-e2e**: End-to-end test framework with mock and live modes.
- **ralph-bench**: Benchmarking infrastructure.

The event loop in `ralph-core` is the central nervous system. Each iteration coordinates hat selection, prompt construction, agent execution, event parsing, and state transitions. The loop terminates on completion promises, max iterations, cost limits, or explicit signals -- each with distinct exit codes (0 for success, 1 for failure, 2 for limits, 130 for interrupt).

Everything runs on Tokio. The Cargo workspace enforces `unsafe_code = "forbid"` and pedantic Clippy lints across all crates. The edition is Rust 2024.

---

## The Hat System: Context as a First-Class Concept

The most distinctive architectural decision in Ralph is the hat system. Instead of giving an agent a monolithic prompt and hoping for the best, Ralph decomposes work into "hats" -- focused roles that an agent wears during a single iteration.

A `HatlessRalph` coordinator is always present, acting as the universal fallback. It holds the objective, the hat topology, skill indices, and any human guidance from the Telegram bot. When a new iteration begins, the coordinator selects which hat the agent should wear based on the current event bus state.

Each hat carries:
- A **name** and **description** defining its role
- **Subscriptions**: which events trigger this hat
- **Publications**: which events this hat can emit
- **Instructions**: the specialized prompt injected when wearing this hat
- **Event receivers**: a map showing what downstream hats will be activated by each published event

This creates a directed graph of work. A `planner` hat subscribes to `project.start` and publishes `plan.complete`. A `coder` hat subscribes to `plan.complete` and publishes `code.complete`. A `reviewer` hat subscribes to `code.complete`. The event publishing guide in each hat's prompt tells the agent exactly what happens when it publishes -- which hat picks up next, and what that hat does.

The topology is configurable via YAML. You can define custom hat graphs for different project types. A web frontend project might flow through `designer -> coder -> reviewer`. A systems project might use `architect -> coder -> tester -> security-reviewer`. The hat system doesn't prescribe workflow; it provides the rails.

This matters because of Ralph's first tenet: **Fresh Context Is Reliability**. Each iteration clears context. The agent re-reads specs, plans, and code every cycle. The hat system ensures that each fresh-context iteration has a focused, well-scoped objective rather than a vague "continue working on the project."

---

## Merge Queues: Parallel Work Without Chaos

Running thirty agents in parallel means thirty branches of changes that eventually need to become one codebase. Ralph's merge queue is an event-sourced system persisted as an append-only JSONL log at `.ralph/merge-queue.jsonl`.

The lifecycle of a parallel loop:

1. **Isolation**: Each parallel loop gets its own git worktree under `.worktrees/<loop-id>/`. Full filesystem isolation, shared `.git` history. Memories, specs, and tasks are symlinked back to the main repo so agents share project knowledge.

2. **Execution**: The loop runs independently. It has its own event bus, its own hat transitions, its own iteration counter. The primary loop (the one holding `.ralph/loop.lock`) manages the Telegram bot and processes the merge queue.

3. **Completion**: When a worktree loop finishes, it enqueues itself with `MergeEventType::Queued`, recording the prompt that was executed.

4. **Merge**: The primary loop picks up pending entries, marks them `Merging` with the merge-ralph PID, performs the git merge, and records either `Merged` (with the commit SHA) or `Failed` (with the error message).

File locking via `flock()` ensures concurrent access safety. The event-sourced design means the full history of every merge attempt is preserved for debugging. You can replay the merge queue to understand exactly what happened during a complex parallel session.

In practice, this system has managed up to 30 simultaneous worktrees -- thirty independent agents, each modifying the codebase in isolation, with their changes flowing through the queue into the main branch.

---

## Telegram: A Remote Control Plane

When you have a fleet of agents running overnight, you need a way to check in from your phone. Ralph's Telegram integration (`ralph-telegram` crate, built on teloxide) provides exactly that.

The bot supports two interaction patterns:

**Agent-to-Human** (`human.interact`): When an agent hits a decision point it cannot resolve autonomously, it sends the question via Telegram and **blocks** the event loop until a response arrives or the timeout (default 300 seconds) expires. This is the "human-in-the-loop" pattern -- the agent does the work, but defers judgment calls.

**Human-to-Agent** (`human.guidance`): Proactive guidance messages sent via Telegram are injected as `## ROBOT GUIDANCE` sections in the agent's next prompt. These are squashed into a numbered list, so multiple guidance messages don't fragment the context.

Commands like `/build`, `/status`, and `/approve` give you a remote control plane. You can check on a running loop from your phone, send course corrections, approve merge operations, or restart loops -- all without touching a terminal.

The bot runs only on the primary loop (the one holding `.ralph/loop.lock`). Parallel loops route messages via reply-to threading, `@loop-id` prefixes, or default to the primary loop. Send failures retry with exponential backoff (three attempts); if all fail, the interaction is treated as a timeout and the loop continues.

---

## RalphMobile: A Companion iOS App

Monitoring agents from a terminal is fine when you are at your desk. RalphMobile is a native SwiftUI iOS app that connects to `ralph-mobile-server` (a Rust HTTP/SSE server) and provides real-time visibility into running orchestration loops.

The app displays:

- **Session list**: All active and historical Ralph sessions with iteration counts, current hats, and elapsed time.
- **Unified session view**: A ten-section detail view showing events, hat transitions, token metrics, scratchpad, tasks, and streaming output.
- **SSE streaming**: Real-time Server-Sent Events flowing from active loops, with hat transition colors and event type indicators.
- **Configuration management**: Server URL, API key, and preset selection from the device.

The app follows a cyberpunk-inspired dark theme and was validated through 1,705 screenshots across multiple development phases. Five validation gates govern every iOS change: visual inspection, real backend integration, video recording for streaming features, real Ralph loop testing (minimum ten minutes of SSE observation), and build verification.

The API contract is intentionally simple:
- `GET /api/sessions` returns lightweight session metadata.
- `GET /api/sessions/{id}/status` returns full session data including mode and elapsed time.
- `GET /api/sessions/{id}/events` returns an SSE stream (only when a loop is active).

---

## The Self-Improvement Loop

The most philosophically interesting aspect of Ralph is that it improves itself using its own orchestration. When a bug is found in the hat system, Ralph runs a loop to fix it -- wearing hats to plan the fix, implement it, and verify it. The orchestrator orchestrates its own development.

This is enabled by the memory system. Memories are classified into four types -- Patterns (how this codebase does things), Decisions (why something was chosen), Fixes (solutions to recurring problems), and Context (project-specific knowledge) -- and persisted in `.ralph/agent/memories.md`. Each new iteration reads these memories, so hard-won lessons from previous sessions survive context resets.

Over 410 sessions, Ralph accumulated a rich memory store. Combined with 41 context continuations in a single arc (the longest sustained chain of work), these memories represent a form of institutional knowledge that no individual agent session possesses.

The task system complements memories. Tasks in `.ralph/agent/tasks.jsonl` provide runtime work tracking. The loop terminates when no open tasks remain and a consecutive `LOOP_COMPLETE` signal is detected. This replaces the older scratchpad approach with a more structured completion mechanism.

---

## Lessons from 410 Sessions

Running an orchestration platform at this scale reveals patterns that are invisible in single-agent workflows:

**Fresh context is more reliable than long context.** Ralph's first tenet emerged from observing that agents with 150K tokens of accumulated context make worse decisions than agents with 40K tokens of fresh, relevant context. The hat system enforces this by resetting context each iteration and injecting only what the current hat needs.

**Backpressure beats prescription.** Rather than giving agents detailed step-by-step instructions, Ralph creates gates that reject bad work: type checks, builds, lints, tests. For subjective criteria, it uses LLM-as-judge with binary pass/fail. The agent figures out *how*; the gates ensure the result meets the bar.

**The plan is disposable.** Regenerating a plan costs one planning loop. It is cheap. Never fight to save a plan that is not working. This is Ralph's third tenet and it applies equally to the orchestrator's own development.

**Disk is state, git is memory.** Memories and tasks are the handoff mechanism between iterations. No sophisticated coordination protocol is needed -- just files on disk and commits in git. The merge queue is a JSONL file. The loop lock is a PID file. Simplicity scales.

**Steer with signals, not scripts.** When Ralph fails a specific way, the fix is not a more elaborate retry mechanism. It is a sign -- a memory, a lint rule, a gate -- that prevents the same failure next time. The codebase is the instruction manual.

**Let Ralph Ralph.** The final tenet. Sit on the loop, not in it. Tune like a guitar, don't conduct like an orchestra. The orchestrator's job is to set up the environment and get out of the way.

---

## What Comes Next

Ralph is at version 2.6.0, with a web dashboard (React + Vite + TailwindCSS) alongside the terminal UI and iOS app. The RPC v1 control plane is landing, shell completions are shipping, and the adapter layer supports Claude, Kiro, Gemini, Codex, and custom backends via per-hat configuration.

The harder problems ahead are not technical. They are about trust calibration: when should an agent ask a human? When should it proceed autonomously? How do you tune the backpressure gates so agents are productive without being reckless?

These are orchestration problems, not AI problems. And that is exactly why Ralph exists -- to make the orchestration layer boring enough that the interesting work happens inside the hats.

---

*Ralph Orchestrator is open source at [github.com/mikeyobrien/ralph-orchestrator](https://github.com/mikeyobrien/ralph-orchestrator). It is built in Rust, ships via cargo-dist to macOS and Linux, and is available as an npm package at `@ralph-orchestrator/ralph`.*
