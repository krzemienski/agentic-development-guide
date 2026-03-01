# Spawning 194 Parallel AI Worktrees to Build a Codebase

*How we built an automated system that ideates tasks, generates specs, spins up isolated git worktrees, and runs independent Claude agents to develop and QA an entire application in parallel.*

---

## The Scale Problem

Software projects don't fail because the work is too hard. They fail because the work is too wide. A codebase with hundreds of pending improvements, refactors, and feature additions creates a coordination bottleneck that no single developer -- human or AI -- can push through sequentially. The queue grows faster than it drains.

We hit this wall building an application called Awesome List. The backlog wasn't tens of tasks. It was hundreds. Modularization across dozens of files. Type system cleanups touching every layer. Storage refactors that required splitting entire subsystems. A deployment pipeline that didn't exist yet. Test coverage that was effectively zero.

The traditional approach -- one task at a time, one branch, one review cycle -- would take weeks of sequential AI sessions. Even with a fast model, you're bottlenecked by the serial nature of "start task, finish task, start next task."

So we asked a different question: what if we didn't serialize at all?

What if we spawned a separate Claude agent for every single task, each operating in its own isolated git worktree, and let them all run in parallel?

The answer became Auto-Claude: an industrial-scale automated development system that ideated 194 tasks, generated 91 detailed specifications, produced 71 QA reports, and operated across thousands of sessions spanning nearly half a gigabyte of conversation data.

---

## The Architecture

Auto-Claude is built on a simple but powerful insight: git worktrees provide free isolation. A single repository can have hundreds of active worktrees, each with its own working directory, its own branch, and its own state. No worktree can interfere with another. No merge conflicts during development. No stepping on each other's changes.

The system has four stages, each operating as an independent pipeline phase:

1. **Ideation** -- Analyze the codebase and generate a comprehensive task list
2. **Spec Generation** -- For each task, produce a detailed implementation specification
3. **Worktree Execution** -- Spin up an isolated worktree and Claude agent per task
4. **QA Pipeline** -- Automated quality review with rejection and resubmission cycles

Each stage feeds the next, but within each stage, work fans out to maximum parallelism. The ideation phase produces a manifest of 194 tasks. The spec generator processes them in batches. The worktree factory spins up isolated environments. The QA pipeline reviews completed work and sends rejections back for fixing.

---

## Ideation: From Codebase to Task Manifest

The first phase is the broadest. A dedicated Claude agent receives the entire codebase context -- directory structure, file contents, existing architecture, known pain points -- and generates a comprehensive task decomposition.

This isn't a simple TODO list. Each ideated task includes:

- A descriptive identifier (e.g., `modularization`, `reduce-any-types`, `split-storage`)
- A scope boundary defining which files and modules are affected
- Dependency declarations noting which other tasks must complete first
- A priority ranking based on impact and risk

For the Awesome List project, ideation produced **194 distinct tasks**. These ranged from surgical fixes (updating a single configuration value) to sweeping refactors (modularizing the entire application into a plugin architecture). The distribution wasn't uniform -- some areas of the codebase attracted clusters of related tasks, while others needed only a single pass.

The key insight here is that AI ideation at scale is cheap. Generating 194 task descriptions costs a fraction of what executing even one of them costs. So you over-generate deliberately, knowing that the QA pipeline downstream will catch anything that shouldn't have been attempted.

---

## Spec Generation: From Ideas to Blueprints

Raw task descriptions aren't enough to hand to an autonomous agent. "Modularize the storage layer" is too vague. The agent needs to know which files to touch, what the target architecture looks like, which interfaces to preserve, and what constitutes "done."

The spec generation phase takes each ideated task and produces a detailed implementation specification. For the Awesome List project, this phase generated **91 specs** out of the 194 ideated tasks. The remaining 103 tasks were either deferred (blocked by dependencies), merged (overlapping scope with another task), or deprioritized.

Each spec includes:

- **Objective**: A single sentence describing the end state
- **Files in scope**: Explicit list of files to create, modify, or delete
- **Implementation steps**: Ordered sequence of changes
- **Acceptance criteria**: Concrete, verifiable conditions for completion
- **Risk notes**: Known pitfalls, edge cases, or compatibility concerns

The spec generator runs as a batch process. It can process multiple tasks in parallel since specs for independent tasks don't interact. A task specifying changes to the storage layer doesn't need to wait for the spec describing UI component extraction.

This phase is where Code Story's data becomes visible: **90 branches** were created across the project, each corresponding to a specified task ready for execution.

---

## The Worktree Factory

This is the core of Auto-Claude. For each specified task, the system:

1. **Creates a git worktree** from the current main branch state
2. **Checks out a dedicated branch** named after the task (e.g., `auto/modularization`, `auto/reduce-any-types`)
3. **Injects the task spec** into the worktree as context
4. **Spawns a Claude agent** scoped exclusively to that worktree
5. **Monitors execution** for completion, failure, or timeout

Each agent operates in complete isolation. It can read and write files in its worktree without affecting any other agent's work. It can make commits to its branch without creating conflicts on other branches. It can even break things catastrophically within its worktree, and no other task is affected.

The session data tells the story of how much work each agent performed:

| Task | Sessions | Description |
|------|----------|-------------|
| modularization | 62 | Plugin architecture extraction |
| reduce-any-types | 60 | Type system cleanup across all layers |
| test-suite | 57 | Comprehensive test coverage from zero |
| deployment | 48 | CI/CD pipeline and release automation |
| split-storage | 47 | Storage layer decomposition |
| api-cleanup | 41 | REST endpoint standardization |
| error-handling | 38 | Consistent error propagation |
| accessibility | 35 | VoiceOver and Dynamic Type support |
| logging | 32 | Structured logging infrastructure |
| documentation | 28 | API docs and architecture guides |

The total across all tasks: **3,066 sessions** consuming **470 MB** of conversation data. These ran in parallel across isolated worktrees, compressing what would be months of serial work into a fraction of the wall-clock time.

---

## The QA Pipeline

Autonomous execution without quality control is just automated chaos. The QA pipeline is what transforms raw agent output into mergeable code.

Every completed task enters a review queue. A dedicated QA agent -- separate from the execution agent -- examines the changes against the original spec's acceptance criteria. The review produces one of three outcomes:

- **Approved**: Changes meet all acceptance criteria, code quality is acceptable, no regressions detected
- **Rejected with fixes**: Specific issues identified, sent back to the execution agent for remediation
- **Rejected permanently**: Fundamental approach is flawed, task needs re-specification

The system generated **71 QA reports** across the completed tasks. Not every report was an approval on the first pass. The rejection-and-fix cycle is a critical part of the pipeline.

### Case Study: Public API Access

The Public API Access task illustrates the QA pipeline's value. The execution agent implemented an API key system for external access to the application's data. The QA review identified two critical failures:

1. **Broken bcrypt integration**: The password hashing implementation used an incompatible library version, producing hashes that couldn't be verified on subsequent requests. Authentication would succeed on creation but fail on every subsequent use.

2. **Missing `lastUsedAt` tracking**: The spec required tracking when each API key was last used for security auditing. The implementation created the database column but never wrote to it. The field was always `null`.

The QA agent rejected the task with specific remediation instructions. The execution agent received the rejection, fixed both issues in its worktree, and resubmitted. The second pass was approved.

Without the QA pipeline, both bugs would have been merged into main. The bcrypt issue would have manifested as a production authentication failure -- the kind of bug that's obvious in hindsight but invisible in a code review that doesn't actually run the authentication flow.

---

## Merge Strategy and Conflict Resolution

With dozens of branches completing in parallel, the merge phase requires careful orchestration. Auto-Claude uses a priority-weighted merge queue:

1. **Foundation tasks first**: Changes to shared infrastructure (error handling, logging, type definitions) merge before feature work that depends on them
2. **Conflict detection**: Before merging, the system checks for textual conflicts against the current main branch state
3. **Re-execution on conflict**: If a branch conflicts with already-merged work, the task re-enters the execution phase with the updated main branch as its base
4. **Incremental integration**: Small, focused tasks merge before large refactors to minimize conflict surface area

This strategy means that some tasks execute more than once. A task that was specified against main at commit `abc123` might need to re-execute after main has advanced to `def456` due to other merges. The worktree isolation makes this painless -- spin up a new worktree from the updated main, re-inject the spec, and let the agent work.

---

## Results by the Numbers

The Awesome List project's Auto-Claude run produced these aggregate metrics:

| Metric | Value |
|--------|-------|
| Tasks ideated | 194 |
| Specs generated | 91 |
| QA reports produced | 71 |
| Git branches created | 90 |
| Total sessions | 3,066 |
| Conversation data | 470 MB |
| Top task by sessions | modularization (62) |
| QA rejection rate (pass 1) | ~22% |
| QA approval rate (pass 2) | ~95% |

The rejection rate on first pass -- roughly one in five tasks needing fixes -- validates the QA pipeline's necessity. If one-fifth of autonomous agent output has issues significant enough to reject, shipping without review would introduce dozens of bugs per project.

The near-total approval rate on the second pass validates the fix cycle's effectiveness. Agents given specific, actionable feedback from the QA review almost always produce acceptable code on the retry. The feedback loop is tight enough that agents rarely need a third attempt.

---

## What We Learned

**Isolation is non-negotiable.** Every experiment we ran without strict worktree isolation produced cross-contamination. Agent A's half-finished refactor would break Agent B's feature implementation. Git worktrees solve this completely -- they are lightweight, fast to create, and provide total filesystem separation.

**Specs are the bottleneck, not execution.** Generating a good spec takes more care than executing against one. A vague spec produces vague code that the QA pipeline rejects. A precise spec with clear acceptance criteria produces code that passes review on the first attempt. The investment in spec quality pays dividends in reduced QA cycles.

**QA agents must be separate from execution agents.** Self-review doesn't work. The same biases that led an agent to write buggy code lead it to overlook those bugs in review. A fresh agent with no context beyond the spec and the diff catches issues that the original agent is blind to.

**Session count correlates with task breadth, not difficulty.** The modularization task consumed 62 sessions not because it was hard but because it was wide -- touching dozens of files across multiple modules. Simple but broad tasks consume more sessions than complex but narrow ones.

**The economics are compelling.** 3,066 sessions sounds expensive until you calculate the alternative. A senior developer working sequentially through 91 specified tasks, with code review on each, would spend weeks. The parallel execution model compresses wall-clock time by an order of magnitude while maintaining quality through automated QA.

---

## Beyond Awesome List

Auto-Claude isn't specific to any one project. The pipeline -- ideate, specify, isolate, execute, review -- is a general-purpose pattern for applying AI development at scale. Any codebase with a large backlog of well-defined improvements is a candidate.

The constraint isn't the AI's capability. It's the quality of the task decomposition and the rigor of the QA pipeline. Get those right, and the execution phase is almost mechanical: spin up worktrees, inject specs, let agents work, review results, merge approved changes.

We are moving from an era where AI assists a developer to an era where AI operates as a managed fleet. The developer's role shifts from writing code to defining tasks, reviewing outputs, and maintaining the pipeline that connects ideation to integration.

194 worktrees. 91 specs. 71 QA reports. 3,066 sessions. One codebase, built in parallel.

The worktree factory is running.

---

*Auto-Claude is part of the ILS (Intelligent Local Server) ecosystem, built on Claude Code with orchestration through the oh-my-claudecode framework.*
