POST 7: THE 7-LAYER PROMPT ENGINEERING STACK — DEFENSE-IN-DEPTH FOR AI AGENTS
================================================================================

Agentic Development: 10 Lessons from 8,481 AI Coding Sessions (Post 7 of 10)


I used to think prompting an AI coding agent was about writing a good initial message. Describe the task clearly, give it some context, hit enter, hope for the best.

After 8,481 sessions I can tell you: the initial prompt is maybe 10% of what determines whether an agent produces reliable code. The other 90% is the invisible system of rules, hooks, skills, and memory that surrounds every interaction -- a defense-in-depth architecture where each layer catches failures that slip through the layers above.

I call it the prompt engineering stack. It has 7 layers. Any single layer provides marginal improvement. All 7 together are transformative.


THE PROBLEM: AI AGENTS CUT CORNERS

Here is the failure mode that cost me the most time across thousands of sessions: an AI agent makes five file edits in rapid succession without verifying any of them compile. By the time you discover the build is broken, the errors have cascaded across three files and the agent's context is polluted with its own mistakes.

Or this one: the agent helpfully commits code that includes a hardcoded API key from an environment variable you pasted during debugging. Now sk-proj-abc123 is in your git history forever.

Or this: you ask the agent to fix a CSS bug and it decides the real problem is your entire component architecture, refactoring 15 files you did not ask it to touch.

Every one of these failures happened to me. Some of them happened dozens of times before I built the systems to prevent them. The prompt stack is the result of that scar tissue.


[DIAGRAM: 7-layer stack visualization showing Layer 1 (Global CLAUDE.md) at the top through Layer 7 (Session Memory) at the bottom, with arrows showing how each layer catches failures that slip through the layers above. Left side shows "Session Start" loading Layers 1-3, "During Editing" triggering Layer 4, "At Commit" triggering Layer 5, and "Validation" invoking Layers 6-7.]


LAYER 1: GLOBAL CLAUDE.md — THE CONSTITUTION

The global CLAUDE.md lives at ~/.claude/CLAUDE.md and loads for every Claude Code session regardless of project. This is where you encode standards that apply universally -- your agent orchestration strategy, model routing preferences, delegation rules.

Think of it as the constitution. Individual projects can add laws, but they cannot override the constitution.

In my setup, the global file defines agent routing:

  # From ~/.claude/CLAUDE.md (global)
  # Model routing:
  # - Haiku: quick lookups, lightweight scans, narrow checks
  # - Sonnet: standard implementation, debugging, reviews
  # - Opus: architecture, deep analysis, complex refactors

It also sets the non-negotiable behavioral rules that every session inherits. The most important one, learned after watching agents waste entire sessions writing elaborate test harnesses instead of building the actual feature:

  # From CLAUDE.md (project-level template)
  # claude-prompt-stack/CLAUDE.md

  ## Functional Validation Mandate

  NEVER: write mocks, stubs, test doubles, or fake implementations
  when validating features.

  ALWAYS: build and run the real system. Validate through actual
  user interfaces or real API calls. Capture evidence (screenshots,
  logs, curl output) before claiming completion.

That single rule -- six lines of text -- changed the character of every AI session. The agent stops trying to prove correctness through abstraction and starts proving it through demonstration.


LAYER 2: PROJECT CLAUDE.md — THE LOCAL LAW

Each project gets its own CLAUDE.md in the repository root. This file contains project-specific mandates, build commands, common pitfalls, and working style rules.

The working style rules address the behavioral anti-patterns I observed across thousands of sessions:

  # From CLAUDE.md (project template)
  # claude-prompt-stack/CLAUDE.md

  ## Working Style

  ### Implement Immediately
  When asked to fix or implement something, start with implementation
  immediately. Do NOT spend the entire session in planning/discovery
  mode reading files repeatedly.

  ### One Change, One Verify
  Make a change. Verify it works. Then make the next change. Do not
  batch 5 changes and then discover 3 of them broke the build.

  ### Stay Focused -- No Scope Creep
  When fixing a specific bug or build error, stay focused on that issue.
  Do NOT escalate into full workspace reorganization, architecture
  changes, or broad refactoring unless explicitly asked.

These rules emerged from specific, painful sessions. The "Implement Immediately" rule came from watching an agent spend 40 minutes reading every file in a 150-file project before making its first edit. The "No Scope Creep" rule came from asking for a one-line CSS fix and getting back a 15-file component refactor.


LAYER 3: RULES FILES — DEEP DOMAIN CONTEXT

The .claude/rules/ directory contains markdown files that provide deep domain knowledge about specific aspects of the project. Claude Code auto-loads all files in this directory at session start.

The companion repo ships 9 rules files:

  # From .claude/rules/project.md
  # claude-prompt-stack/.claude/rules/project.md

  ## Tech Stack
  - Language: [LANGUAGE_AND_VERSION]
  - Framework: [FRAMEWORK_AND_VERSION]
  - Build System: [BUILD_TOOL]
  - Package Manager: [PACKAGE_MANAGER]

  ## Key Constants
  | Item            | Value           |
  |-----------------|-----------------|
  | Default Port    | [PORT]          |
  | API Prefix      | [API_PREFIX]    |
  | Config Path     | [CONFIG_PATH]   |

When I fill in these templates for a real project, the agent stops guessing. It knows the port is 9999, not 3000. It knows the API prefix is /api/v1 and does not double-prefix routes. It knows the build command and does not try npm run build on a Swift project.

The agents.md rules file defines specialized agent roles and when to invoke them:

  # From .claude/rules/agents.md
  # claude-prompt-stack/.claude/rules/agents.md

  ## Parallel Task Execution

  ALWAYS use parallel agent execution for independent operations:

  # GOOD: Parallel execution
  Launch 3 agents in parallel:
  1. Agent 1: Security analysis of auth module
  2. Agent 2: Performance review of data layer
  3. Agent 3: Build verification across all targets

  # BAD: Sequential when tasks are independent
  First agent 1... wait... then agent 2... wait... then agent 3

The development-workflow.md file encodes the anti-patterns I have seen most frequently:

  # From .claude/rules/development-workflow.md
  # claude-prompt-stack/.claude/rules/development-workflow.md

  | Anti-Pattern                  | Do This Instead                  |
  |-------------------------------|----------------------------------|
  | Planning without executing    | Plan once, then execute          |
  | Skipping validation           | Always verify with real system   |
  | Batching too many changes     | One change, one verify           |
  | Reading files repeatedly      | Read once, track what you know   |
  | Scope creep during bug fixes  | Fix the bug, move on             |


LAYER 4: AUTO-BUILD HOOK — THE GUARDRAIL THAT CHANGED EVERYTHING

This is the layer that had the single biggest impact on code quality. The auto-build hook fires automatically after every source file edit and runs the appropriate build command based on file type.

Here is the hook configuration that wires it up:

  // From .claude/settings.local.json
  // claude-prompt-stack/.claude/settings.local.json

  {
    "hooks": {
      "PostToolUse": [
        {
          "matcher": "Edit|Write|MultiEdit",
          "command": "bash .claude/hooks/auto-build-check.sh \"$CLAUDE_FILE_PATH\"",
          "timeout": 120000
        }
      ]
    }
  }

And here is the routing logic that detects file type and runs the right build:

  # From .claude/hooks/auto-build-check.sh
  # claude-prompt-stack/.claude/hooks/auto-build-check.sh

  case "$FILE_PATH" in
      # TypeScript / JavaScript
      *.ts|*.tsx|*.js|*.jsx)
          build_typescript || build_result=$?
          ;;
      # Python
      *.py)
          build_python || build_result=$?
          ;;
      # Rust
      *.rs)
          build_rust || build_result=$?
          ;;
      # Go
      *.go)
          build_go || build_result=$?
          ;;
      # Swift
      *.swift)
          build_swift || build_result=$?
          ;;
      # C / C++
      *.c|*.cpp|*.cc|*.h|*.hpp)
          if [ -f "Makefile" ]; then
              make 2>&1 | tail -20 || build_result=$?
          elif [ -f "CMakeLists.txt" ]; then
              cmake --build build 2>&1 | tail -20 || build_result=$?
          fi
          ;;
      # Non-source files: no build needed
      *)
          exit 0
          ;;
  esac

  if [ $build_result -ne 0 ]; then
      echo ""
      echo "BUILD FAILED: Fix the errors above before continuing."
      exit 1
  fi

The key insight: when the hook exits with code 1, Claude Code surfaces it as an error that the agent must address before continuing. The agent cannot ignore a failed build. It cannot say "I will fix that later." It must fix it now, on this edit, before making the next one.

This single mechanism eliminated the cascading-build-failure problem that plagued my first thousand sessions.


LAYER 5: PRE-COMMIT SECURITY HOOK — THE LAST LINE OF DEFENSE

The pre-commit hook scans every staged file for secrets, credentials, and sensitive data before any commit reaches the repository. It runs automatically when the agent executes git commit.

The blocking patterns -- these reject the commit entirely:

  # From .claude/hooks/pre-commit-check.sh
  # claude-prompt-stack/.claude/hooks/pre-commit-check.sh

  # OpenAI / Anthropic API keys
  if grep -nE 'sk-[a-zA-Z0-9]{20,}' "$file" 2>/dev/null; then
      echo "BLOCKED: Possible API key found in $file"
      BLOCKED=1
  fi

  # AWS Access Keys
  if grep -nE 'AKIA[A-Z0-9]{16}' "$file" 2>/dev/null; then
      echo "BLOCKED: AWS access key found in $file"
      BLOCKED=1
  fi

  # GitHub tokens
  if grep -nE '(ghp_|ghu_|ghs_|gho_|ghr_)[a-zA-Z0-9]{36,}' "$file" 2>/dev/null; then
      echo "BLOCKED: GitHub token found in $file"
      BLOCKED=1
  fi

  # GitLab tokens
  if grep -nE 'glpat-[a-zA-Z0-9\-]{20,}' "$file" 2>/dev/null; then
      echo "BLOCKED: GitLab token found in $file"
      BLOCKED=1
  fi

  # Private keys
  if grep -nE '-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----' "$file" 2>/dev/null; then
      echo "BLOCKED: Private key found in $file"
      BLOCKED=1
  fi

It also catches sensitive file types that should never be committed:

  # From .claude/hooks/pre-commit-check.sh
  # claude-prompt-stack/.claude/hooks/pre-commit-check.sh

  case "$file" in
      *.sqlite|*.sqlite3|*.db)
          echo "BLOCKED: Database file staged for commit: $file"
          BLOCKED=1
          ;;
      .env|.env.*|*.env)
          echo "BLOCKED: Environment file staged for commit: $file"
          BLOCKED=1
          ;;
      *.pem|*.key|*.p12|*.pfx)
          echo "BLOCKED: Certificate/key file staged for commit: $file"
          BLOCKED=1
          ;;
  esac

And warning patterns that flag suspicious content but allow the commit to proceed:

  # Hardcoded absolute paths
  if grep -nE '/Users/[a-zA-Z]|/home/[a-zA-Z]' "$file" 2>/dev/null; then
      echo "WARNING: Hardcoded absolute path in $file"
      WARNED=1
  fi

  # Hardcoded passwords
  if grep -niE '(password|passwd|pwd)\s*=\s*"[^"]+"|password\s*:\s*"[^"]+"' "$file"; then
      echo "WARNING: Possible hardcoded password in $file"
      WARNED=1
  fi

I have caught real API keys with this hook. During debugging sessions, it is natural to paste credentials into code temporarily. The hook ensures they never reach the repository, even when the agent (or you) forget to clean up.


LAYER 6: SKILLS — COMPOSABLE VALIDATION WORKFLOWS

Skills are markdown files that define reusable validation protocols. They are more than documentation -- they are executable workflows that the agent follows step by step.

The functional-validation skill enforces evidence-based completion claims:

  # From skills/functional-validation.md
  # claude-prompt-stack/skills/functional-validation.md

  ## Workflow

  ### Step 1: Build the Real Application
  Verify the build succeeds with zero errors. If it fails, fix the build first.

  ### Step 2: Start the Application
  Wait for the application to be fully ready (health check, UI loaded).

  ### Step 3: Exercise the Feature
  Interact with the feature through its actual interface:
  - For web apps: Navigate to the page, click buttons, fill forms
  - For APIs: Send real HTTP requests with curl
  - For mobile apps: Navigate the real UI in the simulator

  ### Step 4: Capture Evidence
  Collect at least one form of evidence:
  - Screenshots, API responses, log output, terminal output

  ### Step 5: Verify Against Requirements
  Compare captured evidence against the original requirements.

  ## Evidence Standards
  | Claim                  | Minimum Evidence                           |
  |------------------------|--------------------------------------------|
  | "Feature works"        | Screenshot or API response showing it works |
  | "Bug is fixed"         | Before/after evidence showing the fix       |
  | "No regressions"       | Key existing features still function        |
  | "Performance improved" | Measurable metrics (timing, memory, etc.)   |

  ## Anti-Patterns
  - NEVER claim "it should work" without evidence
  - NEVER create mock objects to simulate behavior
  - NEVER write unit tests as a substitute for functional validation
  - NEVER assume the feature works based on reading the code alone

The iOS validation runner skill adds platform-specific steps for simulator-based verification:

  # From skills/ios-validation-runner.md
  # claude-prompt-stack/skills/ios-validation-runner.md

  ### Step 4: Navigate to Feature

  Use one of these approaches (in order of reliability):

  1. Deep links (if supported):
     xcrun simctl openurl [SIMULATOR_UDID] "[URL_SCHEME]://[ROUTE]"

  2. Accessibility tree (most reliable for taps):
     # Get accessibility tree with exact coordinates
     idb_describe operation:all
     # Then tap using coordinates from the tree
     idb_tap [x] [y]

  3. Direct state modification (for screenshots only):
     Temporarily modify @State defaults to auto-present the target view.

Skills compose hierarchically. A high-level skill orchestrates lower-level skills, each adding specific capabilities. The functional-validation skill provides the general framework; the iOS validation runner adds platform-specific mechanics.


LAYER 7: PROJECT MEMORY — CROSS-SESSION KNOWLEDGE

MEMORY.md persists knowledge across AI sessions. Every session starts by reading it. Every session can append to it. Over time, it becomes the institutional memory of the project.

The template provides structure:

  # From MEMORY.md
  # claude-prompt-stack/MEMORY.md

  ## Key Facts
  - Project: [PROJECT_NAME]
  - Build command: [BUILD_COMMAND]
  - Port: [PORT_NUMBER]

  ## Architecture Decisions
  <!-- Record important choices so future sessions don't re-debate them -->

  ## Known Pitfalls
  <!-- Things that have gone wrong before -->

  ## Lessons Learned
  <!-- Insights from debugging sessions, failed approaches -->

In my production project, MEMORY.md grew to over 200 lines across hundreds of sessions. It records things like: "The old backend binary at /Users/nick/ils/ILSBackend/ returns raw data -- ALWAYS use the new binary at /Users/nick/Desktop/ils-ios/." Or: "Deep link UUIDs must be LOWERCASE -- uppercase causes failures." Or: "Always call process.waitUntilExit() before accessing process.terminationStatus."

Each of those entries represents a debugging session that cost 30-60 minutes. With memory, the agent learns from the first occurrence and never repeats the mistake.


THE COMPOUND EFFECT

Here is why the stack works as a system and not just a collection of tips:

The AI cannot forget the build command (Layer 3 documents it, Layer 4 runs it automatically).

The AI cannot skip validation (Layer 2 mandates it, Layer 6 provides the workflow).

The AI cannot ship broken code (Layer 4 catches it on every single edit).

The AI cannot leak secrets (Layer 5 blocks them at commit time with pattern matching for sk-*, AKIA*, ghp_*, glpat-*, and Bearer tokens).

The AI cannot repeat past mistakes (Layer 7 remembers them across sessions).

The setup script makes the investment front-loaded:

  # From setup.sh
  # claude-prompt-stack/setup.sh

  bash setup.sh --target /path/to/your/project

  # Creates:
  # Layer 2: CLAUDE.md
  # Layer 3: .claude/rules/ (9 files)
  # Layer 4: .claude/hooks/auto-build-check.sh
  # Layer 5: .claude/hooks/pre-commit-check.sh
  # Layer 6: skills/ (2 files)
  # Layer 7: MEMORY.md

One command. Then customize the templates for your project. The defense-in-depth system is operational from the first session.


WHAT I WOULD TELL MYSELF 8,000 SESSIONS AGO

Start with Layer 4 (auto-build hook). It has the highest impact-to-effort ratio. A 149-line bash script that runs after every edit eliminated more bugs than any other single intervention.

Then add Layer 5 (pre-commit security). Another 149-line script that runs before every commit. It costs nothing and prevents catastrophic mistakes.

Then build out Layers 1-3 (CLAUDE.md and rules). This is where you encode your project's institutional knowledge so the agent stops guessing.

Layers 6 and 7 (skills and memory) grow organically as you work. Do not try to write them all upfront. Let them emerge from real sessions and real failures.

The prompt engineering stack is not about writing better prompts. It is about building a system that makes it structurally impossible for an AI agent to cut corners, skip verification, leak secrets, or repeat past mistakes. Defense in depth. Every layer matters.


Companion repo with all templates, hooks, and skills:
https://github.com/krzemienski/claude-prompt-stack

#AgenticDevelopment #PromptEngineering #ClaudeCode #AIAgents #DefenseInDepth


================================================================================


POST 8: RALPH ORCHESTRATOR — A RUST PLATFORM FOR AI AGENT FLEETS
================================================================================

Agentic Development: 10 Lessons from 8,481 AI Coding Sessions (Post 8 of 10)


When you are running one AI coding agent, you manage it by watching the terminal. When you are running five agents in parallel across three projects, you need an orchestration layer. When you are running 30 agents overnight with backpressure gates, event-sourced merge queues, and a Telegram control plane so you can steer them from your phone at 2 AM -- you need Ralph.

Ralph Orchestrator is a Rust platform that treats AI agents the way Kubernetes treats containers: isolated environments, clear objectives, structured handoffs, and a persistent control loop that survives session boundaries. I have run 410 orchestration sessions through it, coordinating fleets of AI agents across iOS, backend, and web projects. The hardest problems were never technical. They were trust calibration -- learning when to let the agent run and when to intervene.

This post covers the architecture, the six tenets that emerged from those 410 sessions, and real TOML configurations and Python scripts you can use today.


[DIAGRAM: Ralph architecture overview showing the HatlessRalph Coordinator at center, connected to a Hat Graph (Planner -> Coder -> Reviewer -> Deployer with review.fail looping back to Coder). Below the coordinator: Event Bus, Merge Queue, and State Persistence. To the right: Telegram Control Plane connected to a phone icon. Below: multiple Worktree boxes representing parallel agents.]


THE CORE INSIGHT: HATS

Ralph's most distinctive feature is the hat system. Each agent wears a "hat" -- a focused role that defines its capabilities, context, event subscriptions, and model selection for each iteration.

A monolithic prompt that tries to cover planning, coding, reviewing, and deploying produces mediocre results in all four areas. A planner hat with Opus-level reasoning produces specialist-quality architecture. A coder hat with Sonnet produces fast, focused implementation. A reviewer hat with Opus catches subtle bugs that Sonnet misses.

Here is what a hat configuration looks like in TOML:

  # From docs/hat-system.md
  # ralph-orchestrator-guide/docs/hat-system.md

  [hats.planner]
  description = "Plans the implementation approach"
  model = "opus"
  subscribes = ["project.start", "review.fail"]
  publishes = ["plan.complete", "human.interact"]
  instructions = """
  You are a technical planner. Analyze the objective and create
  a step-by-step implementation plan. Consider edge cases,
  dependencies, and potential blockers.
  """

  [hats.coder]
  description = "Implements the plan"
  model = "sonnet"
  subscribes = ["plan.complete"]
  publishes = ["code.complete", "human.interact"]
  instructions = """
  You are an implementation engineer. Follow the plan exactly.
  Build one component at a time. Verify the build after each change.
  """

  [hats.reviewer]
  description = "Reviews code quality and correctness"
  model = "opus"
  subscribes = ["code.complete"]
  publishes = ["review.pass", "review.fail"]
  instructions = """
  You are a code reviewer. Check for bugs, security issues,
  performance problems, and adherence to the plan. Be specific
  in your feedback.
  """

Hats form a directed graph connected by events. When the planner publishes plan.complete, the coordinator transitions to the coder hat. When the reviewer publishes review.fail, the coordinator routes back to coder. The hat graph is the execution topology of your agent fleet:

  planner                    coder                     reviewer
  Subscribes:                Subscribes:               Subscribes:
    project.start  -------->   plan.complete -------->   code.complete
  Publishes:                 Publishes:                Publishes:
    plan.complete              code.complete             review.pass
    human.interact             human.interact            review.fail
                                     ^                         |
                                     +--- review.fail ---------+

Different project types get different hat graphs. The companion repo ships four pre-built configurations:

  # From configs/ios-mobile.toml
  # ralph-orchestrator-guide/configs/ios-mobile.toml

  [agent]
  name = "ios-mobile"
  description = "iOS/macOS specialist for Swift and SwiftUI development"
  model = "sonnet"
  max_tokens = 8192
  temperature = 0.1

  [agent.identity]
  role = "Senior iOS Engineer"
  expertise = [
      "Swift", "SwiftUI", "UIKit",
      "Combine", "async/await", "Actors",
      "Core Data", "SwiftData",
      "Xcode", "XcodeGen", "SPM",
      "StoreKit", "CloudKit", "WidgetKit",
      "Accessibility", "Localization",
      "App Store Review Guidelines"
  ]

  [loop.backpressure]
  build_ios = "xcodebuild -scheme $SCHEME -destination 'id=$SIMULATOR_UDID' -quiet"
  build_macos = "xcodebuild -scheme $MACOS_SCHEME -destination 'platform=macOS' -quiet"
  lint = "swiftlint lint --quiet"

The systems configuration shows the same pattern for backend work:

  # From configs/systems.toml
  # ralph-orchestrator-guide/configs/systems.toml

  [agent.identity]
  role = "Senior Backend Engineer"
  expertise = [
      "Rust", "Go", "Python", "Node.js",
      "PostgreSQL", "Redis", "SQLite",
      "REST APIs", "gRPC", "GraphQL",
      "Docker", "Kubernetes", "Terraform",
      "Security", "Performance", "Observability"
  ]

  [loop.backpressure]
  build = "cargo build"
  test = "cargo test"
  lint = "cargo clippy -- -D warnings"
  security = "cargo audit"


THE SIX TENETS

These tenets emerged from 410 orchestration sessions. They are not theoretical principles. They are scar tissue.


TENET 1: THE BOULDER NEVER STOPS

A Ralph loop continues working until all tasks are complete or an explicit kill signal is received. Session boundaries are checkpoints, not endpoints.

This is implemented through the stop hook -- a bash script that runs whenever a stop signal fires. If tasks remain, the hook injects a continuation message and the loop keeps going:

  # From examples/persistence-loop/stop-hook.sh
  # ralph-orchestrator-guide/examples/persistence-loop/stop-hook.sh

  #!/usr/bin/env bash
  # Ralph Stop Hook -- "The Boulder Never Stops"

  STATE_FILE=".ralph/loop-state.json"
  TASKS_FILE=".ralph/agent/tasks.jsonl"
  GUIDANCE_FILE=".ralph/guidance.jsonl"

  # Count remaining tasks
  if [ -f "$TASKS_FILE" ]; then
      PENDING=$(grep -c '"status":\s*"pending"' "$TASKS_FILE" 2>/dev/null || echo "0")
      IN_PROGRESS=$(grep -c '"status":\s*"in_progress"' "$TASKS_FILE" 2>/dev/null || echo "0")
      REMAINING=$((PENDING + IN_PROGRESS))
  else
      REMAINING=0
  fi

  # Check for explicit kill signal
  if [ -f ".ralph/kill.signal" ]; then
      echo "Kill signal detected. Stopping." >&2
      rm -f ".ralph/kill.signal"
      echo "stop"
      exit 0
  fi

  # The boulder never stops -- if tasks remain, keep going
  if [ "$REMAINING" -gt 0 ]; then
      echo "Tasks remaining: $REMAINING. The boulder never stops." >&2

      TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
      mkdir -p "$(dirname "$GUIDANCE_FILE")"
      echo "{\"type\":\"continuation\",\"timestamp\":\"$TIMESTAMP\",\"message\":\"The boulder never stops. $REMAINING tasks remain. Continue working.\"}" >> "$GUIDANCE_FILE"

      echo "continue"
      exit 0
  fi

  # All tasks complete -- stop gracefully
  echo "All tasks complete. Loop can stop." >&2
  echo "stop"
  exit 0

The persistence configuration makes this work across session restarts:

  # From examples/persistence-loop/persistence.toml
  # ralph-orchestrator-guide/examples/persistence-loop/persistence.toml

  [loop]
  max_iterations = 200
  checkpoint_interval = 3
  timeout_minutes = 480              # 8 hours

  [loop.persistence]
  enabled = true
  state_file = ".ralph/loop-state.json"
  resume_on_start = true
  checkpoint_includes = [
      "iteration",
      "current_hat",
      "task_progress",
      "memory_snapshot",
      "cost_accumulator",
  ]

  [loop.stop_hook]
  script = "bash stop-hook.sh"
  behavior = "check_and_decide"

The state manager provides visibility into the persistence system:

  # From examples/persistence-loop/state-manager.py
  # ralph-orchestrator-guide/examples/persistence-loop/state-manager.py

  def write_state(state: dict[str, Any]) -> None:
      """Write loop state to disk atomically."""
      STATE_FILE.parent.mkdir(parents=True, exist_ok=True)

      # Write to temp file first, then rename (atomic on POSIX)
      tmp = STATE_FILE.with_suffix(".tmp")
      tmp.write_text(json.dumps(state, indent=2))
      tmp.rename(STATE_FILE)

  def cmd_status() -> None:
      """Show the current loop state."""
      state = read_state()

      print("=== Ralph Loop State ===")
      print(f"Status:     {state.get('status', 'unknown')}")
      print(f"Iteration:  {state.get('iteration', 0)}/{state.get('max_iterations', '?')}")
      print(f"Hat:        {state.get('current_hat', 'none')}")

      tasks = get_task_summary()
      if tasks["total"] > 0:
          print(f"\nTasks:      {tasks['completed']}/{tasks['total']} complete")

      cost = state.get("total_cost", 0)
      if cost:
          print(f"Cost:       ${cost:.4f}")

Atomic writes via tmp-file-then-rename prevent state corruption from mid-write crashes. The state manager supports status, checkpoint, resume, reset, and history commands. When a loop runs overnight and you want to check progress in the morning, python state-manager.py status gives you the full picture.


TENET 2: HATS DEFINE CAPABILITY

Covered in detail above. The key insight: focused roles produce specialist-quality output. A hat that plans AND codes AND reviews is just a monolithic prompt with extra steps.


TENET 3: THE PLAN IS DISPOSABLE

Regenerating a plan costs one planning iteration -- about $0.05 and 30 seconds. Clinging to a failing plan costs hours of wasted iterations.

This sounds obvious written down. In practice, I watched agents fight failing plans for 15-20 iterations before I learned to encode this as a tenet. When a plan hits a dead end -- build fails repeatedly, a core assumption turns out to be wrong, the approach leads to unnecessary complexity -- the right move is to discard the plan and regenerate from current state.


TENET 4: TELEGRAM AS CONTROL PLANE

When agents run overnight or in parallel, you need a way to steer them from your phone. The Telegram bot is not a notification system -- it is a remote control plane for human-in-the-loop orchestration.

The bot configuration defines what notifications fire and when:

  # From examples/telegram-bot/bot-config.toml
  # ralph-orchestrator-guide/examples/telegram-bot/bot-config.toml

  [telegram]
  bot_token_env = "RALPH_TELEGRAM_BOT_TOKEN"
  chat_id_env = "RALPH_TELEGRAM_CHAT_ID"
  enabled = true

  [telegram.notifications]
  on_iteration_start = false         # Too noisy
  on_iteration_complete = false      # Also noisy
  on_checkpoint = true               # Every N iterations
  on_human_interact = true           # Agent needs input -- ALWAYS notify
  on_error = true
  on_loop_complete = true
  on_merge_complete = true
  on_merge_failed = true

  [telegram.interaction]
  timeout_seconds = 300              # Wait 5 min for human response
  timeout_action = "continue"        # On timeout: continue with default

The command handlers implement the control plane:

  # From examples/telegram-bot/commands.py
  # ralph-orchestrator-guide/examples/telegram-bot/commands.py

  def cmd_status() -> str:
      """/status -- Show current loop state."""
      state = read_state()

      tasks = read_tasks()
      total = len(tasks)
      completed = sum(1 for t in tasks if t.get("status") == "completed")
      in_progress = sum(1 for t in tasks if t.get("status") == "in_progress")
      pending = sum(1 for t in tasks if t.get("status") == "pending")

      paused = " [PAUSED]" if PAUSE_FLAG.exists() else ""

      return (
          f"Status: {state.get('status', 'unknown')}{paused}\n"
          f"Iteration: {state.get('iteration', 0)}/{state.get('max_iterations', '?')}\n"
          f"Hat: {state.get('current_hat', 'none')}\n"
          f"Tasks: {completed}/{total} done, {in_progress} active, {pending} pending"
      )

  def cmd_guidance(text: str) -> str:
      """/guidance [text] -- Send guidance to the agent."""
      response = {
          "type": "guidance",
          "timestamp": datetime.now(timezone.utc).isoformat(),
          "message": text.strip(),
      }
      _write_guidance(response)
      return f"Guidance sent. Agent will receive it on next iteration."

  def cmd_kill() -> str:
      """/kill -- Force-stop the loop immediately."""
      kill_file = RALPH_DIR / "kill.signal"
      kill_file.parent.mkdir(parents=True, exist_ok=True)
      kill_file.write_text(datetime.now(timezone.utc).isoformat())
      return "Kill signal sent. Loop will terminate."

  # Command dispatch table
  COMMANDS = {
      "status": cmd_status,
      "pause": cmd_pause,
      "resume": cmd_resume,
      "approve": cmd_approve,
      "reject": cmd_reject,
      "metrics": cmd_metrics,
      "kill": cmd_kill,
      "guidance": cmd_guidance,
      "logs": cmd_logs,
  }

The interaction flow: agent hits a decision point, sends a Telegram message, blocks the event loop. You review on your phone, send a reply or /approve, the agent continues. Timeout after 300 seconds means the agent continues with a default action -- it does not block forever if you are asleep.

The /guidance command is the most powerful. It injects arbitrary text into the agent's next prompt. "Use SQLite instead of PostgreSQL for simplicity." "Skip the caching layer, ship without it." "The auth endpoint is at /api/v2/auth, not /api/v1/auth." Real-time course correction from your phone.


TENET 5: WORKTREES AS ISOLATION

Each parallel agent gets its own git worktree -- full filesystem isolation with shared git history. The parallel configuration:

  # From examples/parallel-agents/parallel.toml
  # ralph-orchestrator-guide/examples/parallel-agents/parallel.toml

  [parallel]
  max_workers = 10
  worktree_base = ".worktrees"
  isolation = "worktree"
  merge_strategy = "queue"

  [parallel.shared]
  symlinks = [
      ".ralph/agent/memories.md",    # Shared memories across agents
      ".ralph/agent/tasks.jsonl",    # Shared task tracking
      "specs/",                       # Shared specifications
  ]

  [merge_queue]
  log_path = ".ralph/merge-queue.jsonl"
  lock_path = ".ralph/loop.lock"
  auto_merge = true
  retry_on_conflict = true
  max_retry = 3

The task splitter decomposes high-level objectives into parallelizable work units:

  # From examples/parallel-agents/task-splitter.py
  # ralph-orchestrator-guide/examples/parallel-agents/task-splitter.py

  def create_task(task_id: str, title: str, description: str,
                  dependencies: list[str] | None = None,
                  priority: int = 0) -> dict:
      """Create a single task entry for the JSONL task file."""
      return {
          "id": task_id,
          "title": title,
          "description": description,
          "status": "pending",
          "priority": priority,
          "dependencies": dependencies or [],
          "assigned_to": None,
          "created_at": datetime.now(timezone.utc).isoformat(),
          "completed_at": None,
      }

  def split_from_objective(objective: str) -> list[dict]:
      tasks = [
          create_task("task-001", "Set up project structure", ...),
          create_task("task-002", "Implement core data models",
                      dependencies=["task-001"], ...),
          create_task("task-003", "Build API endpoints",
                      dependencies=["task-002"], ...),
          create_task("task-004", "Build UI components",
                      dependencies=["task-002"], ...),
          create_task("task-005", "Integration and validation",
                      dependencies=["task-003", "task-004"], ...),
      ]
      return tasks

Note the dependency graph: tasks 003 and 004 both depend on 002 but not on each other -- they can run in parallel. Task 005 depends on both 003 and 004 -- it waits until both complete. The merge queue coordinates the handoff:

  1. Worktree completes -> Queued
  2. Primary loop picks up -> Merging
  3. Git merge succeeds -> Merged (commit SHA)
  4. Or fails -> Failed (error message, retry up to 3 times)

Running 30 agents in the same directory causes immediate chaos. Worktrees provide true filesystem isolation while preserving shared git history. Shared memories, task files, and specs are symlinked so agents coordinate without stepping on each other's files.


TENET 6: QA IS NON-NEGOTIABLE

Backpressure gates run after every iteration. They provide binary pass/fail on objective quality criteria. The agent has freedom in implementation but zero tolerance for quality failures:

  # From docs/tenets.md
  # ralph-orchestrator-guide/docs/tenets.md

  [loop.backpressure]
  build = "cargo build"
  lint = "cargo clippy -- -D warnings"
  test = "cargo test"
  security = "cargo audit"
  format = "cargo fmt --check"

This is backpressure, not prescription. You do not tell the agent how to write code. You tell it what quality bar the code must meet. If it meets the bar, proceed. If it does not, iterate until it does. The agent figures out how; the gates ensure the result.

The philosophy from the tenets document captures it: "Steer with signals, not scripts. When Ralph fails a specific way, the fix is not a more elaborate retry mechanism -- it is a sign (a memory, a lint rule, a gate) that prevents the same failure next time."


THE BASIC LOOP: FROM ZERO TO RUNNING

The simplest Ralph configuration is surprisingly minimal:

  # From examples/basic-loop/loop.toml
  # ralph-orchestrator-guide/examples/basic-loop/loop.toml

  [agent]
  name = "basic-worker"
  description = "General-purpose development agent"
  model = "sonnet"
  max_tokens = 4096

  [loop]
  max_iterations = 20
  checkpoint_interval = 5
  timeout_minutes = 30
  completion_signal = "LOOP_COMPLETE"

  [loop.backpressure]
  build = "npm run build"

  [context]
  always_include = ["README.md", "package.json"]
  index_dirs = ["src/"]

The agent instructions teach it the iterative workflow:

  # From examples/basic-loop/instructions.md
  # ralph-orchestrator-guide/examples/basic-loop/instructions.md

  ## Rules

  - Build after every change. Run the build command and verify it passes.
  - One change at a time. Make a single logical change, verify it, proceed.
  - Use memories. Read .ralph/agent/memories.md at the start of each iteration.
  - Write memories. Before ending each iteration, record what you accomplished.
  - Be honest about completion. Only emit LOOP_COMPLETE when truly done.

  ## Memory Format

  ## Iteration [N] - [Date]
  - Did: [What was accomplished]
  - Verified: [How it was verified]
  - Next: [What remains, or "DONE"]

Running it:

  ralph run --config examples/basic-loop/loop.toml "Fix the login page CSS"

That is it. Ralph iterates until the task is complete, the iteration limit is reached, or the timeout fires. The backpressure gate ensures every iteration produces buildable code. The memory system ensures the agent remembers what it did across iterations.


WHAT 410 SESSIONS TAUGHT ME ABOUT TRUST

The hardest problems in agent orchestration are not technical. They are trust calibration.

Trust too little: you micromanage every iteration, sending /guidance messages that contradict the agent's plan, breaking its flow, wasting tokens on context switching. The agent becomes a typing assistant, not an autonomous agent.

Trust too much: you let the agent run 50 iterations overnight and wake up to a codebase that builds but has silently rewritten your authentication system in a way that looks correct but has a subtle timing vulnerability.

The right calibration: start with tight backpressure gates (build, lint, test, security audit) and generous human interaction triggers. As you build confidence in a specific agent configuration for a specific project type, loosen the interaction triggers while keeping the quality gates tight.

The Telegram control plane is what makes this calibration possible. You do not have to choose between "sit at the terminal watching every line" and "let it run unsupervised." You can check /status from your phone, send /guidance when the agent is heading in the wrong direction, and /approve when it asks for confirmation. The agent does the work. You steer.

After 410 sessions, my default configuration uses checkpoint_interval = 5 (Telegram notification every 5 iterations), on_human_interact = true (always notify when the agent needs a decision), and timeout_seconds = 300 (continue with default after 5 minutes). This gives me enough visibility to catch problems early without drowning in notifications.

The six tenets are not about making AI agents autonomous. They are about making AI agents manageable at scale. The boulder never stops, but you always hold the reins.


Companion repo with TOML configs, Python scripts, and documentation:
https://github.com/krzemienski/ralph-orchestrator-guide

#AgenticDevelopment #RalphOrchestrator #AIAgents #Rust #MultiAgent
