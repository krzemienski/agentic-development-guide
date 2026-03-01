=== POST 1: Series Launch ===

8,481 AI coding sessions. 90 days. 5.5 GB of interaction data. 10 blog posts. Here is what I learned building software almost entirely with AI coding agents.

Not the "ask ChatGPT to write a function" kind of AI-assisted development. The "coordinate 30 specialized agents working in parallel across a shared codebase, with hard consensus gates that block shipping until three independent reviewers agree" kind.

I spent the last three months building real products -- a native iOS client for Claude Code, a Rust orchestration platform, an audio story generator, a design-to-code pipeline -- and documenting every pattern, failure, and architecture decision along the way. The result is 10 deeply technical blog posts totaling 22,489 words, backed by 33 Mermaid diagrams, 10 data visualizations, and code from 10 companion repositories that all went through a four-phase audit: structural review, functional validation with real execution, documentation completeness, and SDK compliance. 12 bugs were found and fixed across those repos before publication. Every code snippet comes from production. No fabricated examples.

Here are all 10 topics and why each one matters.


TOPIC 1: BUILDING A NATIVE iOS CLIENT FOR CLAUDE CODE

763 sessions building a SwiftUI app with a 5-layer streaming bridge. The architecture runs SwiftUI frontend to Vapor backend to Python SDK wrapper to Claude CLI to Anthropic API. Each token traverses this entire chain as a Server-Sent Event. The total path from API response to rendered pixel: roughly 50ms per token. The companion repo (claude-ios-streaming-bridge) ships as a reusable Swift Package with an SSEClient that handles UTF-8 buffer parsing, exponential backoff reconnection, and a complete type system -- StreamMessage, ContentBlock, StreamDelta, UsageInfo. The two-character bug that stopped every token from appearing twice lives in this post. More on that in Topic 4.


TOPIC 2: THE 5-LAYER BRIDGE -- 4 FAILED ATTEMPTS, 1 WORKING ARCHITECTURE

A debugging war story. Direct Anthropic API from Swift -- failed, no OAuth token available. JavaScript SDK via Node subprocess -- failed, NIO event loops do not pump RunLoop. Swift ClaudeCodeSDK in Vapor -- failed, FileHandle.readabilityHandler needs RunLoop which NIO does not provide. Direct CLI invocation -- failed, nesting detection blocks Claude inside Claude. The fifth attempt worked: a Python subprocess bridge with NDJSON stdout and environment variable stripping (specifically removing CLAUDECODE=1 and CLAUDE_CODE_* vars before spawning). The counterintuitive lesson is that the 5-layer architecture is simpler than any of the "simpler" approaches because each layer does exactly one translation with exactly one failure mode.


TOPIC 3: SPAWNING 194 PARALLEL GIT WORKTREES

Factory-scale AI development. 194 isolated git worktrees, each running its own AI coding agent, all building the same codebase in parallel. The pipeline has four stages: spec generation, worktree provisioning with parallel execution, independent QA review, and merge queue processing. The companion repo (auto-claude-worktrees) is a pip-installable Click CLI with a priority-weighted merge queue and stale worktree detection. The numbers: 91 specs generated, 71 QA reports produced, 3,066 sessions total. The key insight is that the QA pipeline matters more than the agents themselves -- the rejection-and-fix cycle between independent QA agents and executors is where the real quality comes from.


TOPIC 4: HOW 3 AI AGENTS FOUND A BUG I WOULD HAVE SHIPPED

Multi-agent consensus. A single agent reviewed my streaming code and said "looks correct." Three agents running a structured consensus audit caught a P2 bug on line 926 of ChatViewModel.swift in the first pass. The bug: message.text += textBlock.text when it should have been message.text = textBlock.text. One character. The += appended to already-accumulated content. The = treats each event as authoritative. A second root cause compounded it -- the stream-end handler reset lastProcessedMessageIndex to zero, replaying the entire buffer. The three-agent pattern uses Lead (architecture and consistency), Alpha (code and logic), and Bravo (systems and functional verification). They vote independently. Unanimous pass required. I have run this across 3 projects with 10 blocking gates each. Cost per gate: roughly $0.15. The P2 bug would have shipped to users.


TOPIC 5: THE 7-LAYER PROMPT ENGINEERING STACK

Defense-in-depth for AI coding agents. Layer 1: CLAUDE.md global rules loaded into every session. Layer 2: .claude/rules/ with project-specific instructions -- build commands, architecture patterns, feature gates. Layer 3: 150+ reusable skills invoked by name. Layer 4: hooks that auto-build after every .swift file edit and pre-commit security scans that block API keys (sk-*, AKIA*, ghp_*) and database files from being committed. Layer 5: 25+ specialized agent definitions with scoped prompts and tools. Layer 6: YAML-based prompt libraries with variable interpolation. Layer 7: persistent session memory so agents never lose context on project conventions. The compound effect is that the agent cannot forget the build command, cannot skip validation, cannot ship a mock, cannot leak an API key -- all enforced automatically on every edit, every commit, every session.


TOPIC 6: RALPH ORCHESTRATOR -- A RUST PLATFORM FOR AI AGENT ARMIES

10 Rust crates coordinating 30+ agents simultaneously. The core innovation is the "hat" system: each agent wears a hat that defines its context -- which files it can see, what tools it has, what role it plays. Swapping hats changes an agent's entire perspective without restarting the session. The platform includes event-sourced merge queues for deterministic conflict resolution, backpressure gates where agents self-throttle when quality drops, a Telegram control plane for monitoring from your phone, and persistent loops where agents survive session boundaries. 410 orchestration sessions went into building this. The hardest problems were not technical -- they were trust calibration: when should an agent ask a human, when should it proceed autonomously, how do you tune backpressure so agents stay productive without being reckless.


TOPIC 7: I BANNED UNIT TESTS FROM MY AI WORKFLOW

Zero mocks. Zero stubs. Zero test doubles. When an AI agent writes both the implementation AND the unit tests, a passing test suite is not independent evidence of correctness. The agent validates its own assumptions in a closed loop. The replacement: functional validation. Build the real system, run it, screenshot it, verify against the spec. The numbers after 90 days: 470 evidence screenshots captured, 37+ validation gates across multiple projects, 3 browser automation tools evaluated (Puppeteer MCP, Playwright MCP, agent-browser). Four bug categories caught that unit tests systematically miss: visual rendering bugs, integration boundary failures, state management bugs that only appear on second interaction, and platform-specific issues. The companion repo (functional-validation-framework) ships a Click CLI where "fvf init --type api" generates a real 5-gate YAML configuration with Playwright, httpx, and screenshot capture built in.


TOPIC 8: FROM GITHUB REPOS TO AUDIO STORIES

Code Tales: a platform that clones a repository, analyzes its architecture with Claude, generates a narrative script, synthesizes speech with ElevenLabs, and streams the finished audio. 9 narrative styles: debate, documentary, executive, fiction, interview, podcast, storytelling, technical, tutorial. The build process was the real story -- 636 commits, 90 worktree branches, 91 specs, 37 validation gates. The audio debugging saga required nine commits to fix race conditions in the audio player, each identified, implemented, and validated by agents working in parallel.


TOPIC 9: 21 AI-GENERATED SCREENS IN ONE SESSION

No Figma. No design handoff. Stitch MCP takes text descriptions, generates design tokens, produces React components with Tailwind CSS, and validates everything with Puppeteer automation. One session produced: 21 screens (auth flow, dashboard, admin with 20 tabs, settings, profiles), 47 design tokens in a brutalist palette (all borderRadius: 0px), a Button component with CVA featuring 8 variants, 8 sizes, forwardRef, and Radix Slot integration, plus 105 Puppeteer validation checks across 374 actions. The validation layer is what makes this production-grade, not a demo.


TOPIC 10: THE AI DEVELOPMENT OPERATING SYSTEM

The capstone. After 8,481 sessions, I accidentally built a composable meta-system with 6 subsystems: OMC orchestration with 25+ specialized agent types across build, review, domain, and coordination lanes. Ralph Loop for persistent execution that survives session boundaries. Specum Pipeline for specification-driven development: research to requirements to design to tasks to execution to verification. RALPLAN for adversarial planning where Planner, Architect, and Critic iterate until consensus. GSD for project lifecycle management. Team Pipeline for N coordinated agents with staged execution. The companion repo (ai-dev-operating-system) ships a Click CLI where "ai-dev-os catalog list" displays the full 25-agent catalog across all lanes.


THE AUDIT THAT VALIDATED ALL OF IT

Before publishing, all 10 companion repositories went through a four-phase audit run by 8 parallel audit agents, 3 documentation agents, and a lead coordinator. Phase 1: structural review -- every file mentioned in the README exists, no stubs, all imports resolve. Phase 2: functional validation -- real execution, no mocks. pip install -e . in fresh venvs, CLI entry points responding to --help, real commands producing real output. Phase 3: documentation completeness -- troubleshooting sections added to all 10 READMEs. Phase 4: SDK compliance -- repos using the Claude Agent SDK audited against official patterns with isinstance() dispatch instead of getattr(). 12 issues found and fixed. 10/10 repos passed.

Every architecture diagram, every code quote, every cost number comes from real sessions. The full series with companion repos, Mermaid diagrams, and social cards is at: github.com/krzemienski/agentic-development-guide

What patterns are you discovering as you scale AI coding tools beyond single-session use?

#AgenticDevelopment #AI #SoftwareEngineering #ClaudeCode #DevTools


=== POST 2: Multi-Agent Consensus ===

A single AI agent reviewed my streaming code and said "looks correct."

Three agents found a P2 bug on line 926. Here is the full story of how a one-character fix exposed a structural flaw in how we review AI-generated code.

THE BUG

I was building ILS, a native iOS client for Claude Code. The streaming chat interface was the core feature: messages arrive from Claude's API as Server-Sent Events, tokens flow in real time, the UI renders them progressively. It looked like it worked. Messages appeared. The UI felt responsive. I ran a single-agent code review. The agent scanned ChatViewModel.swift, noted some minor style inconsistencies, and reported: "Streaming implementation looks correct."

The bug had been in the codebase for three days.

Here is what was actually happening. When a user sent a message and Claude's response streamed back, every token appeared twice. The word "Four" rendered as "Four.Four." on screen. The assistant message handler was using += to append text blocks, but those text blocks already contained the full accumulated content from prior textDelta streaming events. Append plus authoritative full text equals duplication.

The code looked like this:

  // ChatViewModel.swift, line 926 -- the bug
  message.text += textBlock.text

The fix:

  // The correct version -- assignment, not append
  message.text = textBlock.text

One character. The += became =. The += operator makes perfect sense if you are building text incrementally from deltas. The textBlock.text containing the full accumulated content makes perfect sense if the assistant event is authoritative. Both are valid patterns. The bug exists only at their intersection -- two update mechanisms, each individually reasonable, interacting badly when combined.

But there was a second root cause that compounded the first. The stream-end handler reset lastProcessedMessageIndex to zero. On the next observation cycle, the entire SSE message buffer replayed from the beginning, feeding already-processed messages back through the rendering pipeline. Double processing on top of double accumulation.

  // Root cause 2 -- the index reset
  // BEFORE (bug):
  self.lastProcessedMessageIndex = 0   // replays all messages next cycle

  // AFTER (fix):
  let finalCount = sseClient.messages.count
  self.lastProcessedMessageIndex = finalCount  // preserves position

The result in practice: every streaming response stuttered visibly. Tokens appeared, doubled, then resolved to the correct text once streaming completed. It would not crash. It would not fail silently. It would just look broken -- the kind of bug that erodes trust in a product the first time a user sees it.


WHY SINGLE-AGENT REVIEW MISSED IT

When you ask one AI agent to review code, it brings a single perspective. It pattern-matches against known error categories. It reads the streaming implementation and sees the shape of correctness -- delta events, accumulation buffers, state updates. Each line makes sense in isolation.

The mistake is that the agent is checking "does this line look correct?" when the bug is about how two correct-looking lines interact. The += operator is a standard accumulation pattern. The textBlock containing full text is a standard authoritative-update pattern. The individual reviewer sees both patterns, recognizes both as valid, and moves on. The bug lives in the space between the patterns.

This is not unique to AI. Human code review has the same failure mode. Individual reviewers develop blind spots. The pattern recognition that makes you productive is exactly what causes you to miss novel bugs. You see what you expect to see.


THE THREE-AGENT ARCHITECTURE

Multi-agent consensus addresses this structurally. Not by making each agent smarter, but by ensuring that genuinely independent perspectives must all agree before work advances.

The pattern: three agents review independently. All three must vote PASS. Any single FAIL keeps the gate closed.

I built this as a Python CLI framework. The three roles are defined as frozen dataclasses in roles.py, each with a specialized system prompt, focus areas, and a list of what it is calibrated to catch:

  LEAD = RoleDefinition(
      role=Role.LEAD,
      title="Lead (Architecture & Consistency Specialist)",
      focus_areas=[
          "Cross-component consistency",
          "API contract compliance",
          "Pattern adherence",
          "Architectural coherence",
          "Regression detection",
      ],
      catches=[
          "Contract mismatches between layers",
          "Pattern violations",
          "Inconsistent naming or data shapes",
          "Fixes that break other components",
      ],
  )

Lead validates the whole. It looks for cross-component consistency, whether all parts agree on contracts and naming, whether fixes introduced new inconsistencies elsewhere. In the ILS case, Lead cross-checked that both the SDK and CLI execution paths used the same corrected handler after the fix was applied.

  ALPHA = RoleDefinition(
      role=Role.ALPHA,
      title="Alpha (Code & Logic Specialist)",
      focus_areas=[
          "Line-by-line code correctness",
          "State management and accumulation patterns",
          "Operator correctness (+= vs = vs ==)",
          "Off-by-one and boundary errors",
          "API contract compliance",
      ],
      catches=[
          "Logic errors invisible in isolation",
          "Incorrect accumulation (the += bug)",
          "State machine index resets",
          "Race conditions in async code",
      ],
  )

Alpha is the detail-oriented auditor. It reads implementation line by line. Its system prompt includes what I call "THE += vs = PRINCIPLE" -- a direct reference to this exact bug class, embedded as institutional knowledge:

  "The most dangerous bugs look correct in isolation. message.text += delta
  makes sense for incremental accumulation. delta containing full accumulated
  text makes sense for authoritative updates. The bug exists at their
  INTERSECTION. Look for these interactions."

Alpha caught both root causes in the ILS streaming bug on the first pass because it was specifically prompted to look at how data flow mechanisms interact -- not just whether individual lines were correct.

  BRAVO = RoleDefinition(
      role=Role.BRAVO,
      title="Bravo (Systems & Functional Specialist)",
      focus_areas=[
          "Functional correctness under real conditions",
          "Edge case behavior",
          "UI/output verification",
          "Performance under load",
          "Regression detection",
      ],
      catches=[
          "Bugs that only appear at runtime",
          "Visual/output duplication or corruption",
          "Edge cases with real data",
          "Regressions in existing flows",
      ],
  )

Bravo exercises the running system. Its prompt explicitly states: "Alpha reads code. You RUN things." Bravo confirmed the fix by verifying that responses rendered as "Four." and "Six." -- not "Four.Four." and "Six.Six."

The three roles are not arbitrary. They are calibrated so that what Alpha misses in the running UI, Bravo catches, and what both miss at the architectural level, Lead finds.


HOW THE GATE ACTUALLY WORKS

The gate mechanism is implemented in gate.py. Each agent is invoked via subprocess -- the Claude CLI with --print, a model flag, and the role-specific system prompt:

  cmd = [
      "claude", "--print",
      "--model", agent_config.model,
      "--system-prompt", system_prompt,
      user_prompt,
  ]

  result = subprocess.run(
      cmd,
      capture_output=True,
      text=True,
      timeout=agent_config.timeout_seconds,
  )

By default, agents run in parallel using ThreadPoolExecutor with max_workers=3 -- true independence, no shared state, no ability to anchor on each other's conclusions:

  with ThreadPoolExecutor(max_workers=3) as executor:
      future_to_role = {
          executor.submit(
              run_agent_validation, role_def, phase_name, target_path, config,
          ): role_def
          for role_def in roles
      }

Each agent returns a structured JSON vote. The framework parses it into a Pydantic model:

  class Vote(BaseModel):
      role: Role
      outcome: VoteOutcome       # PASS or FAIL
      reasoning: str             # 2-3 sentence summary
      findings: list[str]        # specific issues found
      evidence_paths: list[str]  # file paths, line numbers, command output
      duration_seconds: float

Unanimity is computed with a single line:

  unanimous = all(v.is_pass() for v in votes)

If any agent's JSON response fails to parse, the framework defaults to FAIL for safety -- not PASS. Ambiguity blocks the gate:

  except json.JSONDecodeError as e:
      return Vote(
          role=role,
          outcome=VoteOutcome.FAIL,
          reasoning=f"Vote response parsing failed: {e}",
          findings=["Agent response was not valid JSON -- treating as FAIL for safety"],
      )

The GateResult model aggregates everything:

  class GateResult(BaseModel):
      phase_name: str
      gate_number: int
      votes: list[Vote]
      unanimous_pass: bool
      evidence: list[Evidence]
      fix_cycle_count: int

[DIAGRAM: A flow diagram showing the gate check process. Three parallel lanes (Lead, Alpha, Bravo) each receive the same checkpoint prompt independently. Each lane produces a Vote with outcome, reasoning, findings, and evidence. The three votes converge at a "Unanimity Check" diamond. If all three are PASS, the gate opens and the pipeline advances to the next phase. If any vote is FAIL, the gate stays closed, a fix cycle is triggered, and then ALL THREE agents re-validate -- not just the one that failed. The re-validation loop is bounded by max_fix_cycles (default 3) before hard failure.]


THE FIX CYCLE: WHY ALL THREE RE-VALIDATE

When a gate fails, the fix-and-retry loop is where the pattern earns its overhead cost. The critical design decision: after a fix is applied, ALL THREE agents re-validate, not just the one that failed.

  def run_gate_with_fix_cycles(..., max_fix_cycles=3, fix_callback=None):
      for cycle in range(max_fix_cycles + 1):
          result = run_gate_check(...)
          if result.unanimous_pass:
              return result
          # Apply fixes, then re-validate with ALL THREE agents
          findings = result.all_findings()
          fix_callback(findings)

This catches the failure mode where a fix resolves the original issue but introduces a regression. The agent that previously passed might now fail on something the fix broke. You do not get to carry forward stale passing votes.

In the ILS streaming fix, after Alpha flagged the += bug and the index reset, the fix was implemented. Then all three re-ran. Alpha confirmed the code logic was correct (assignment, not append; index preserved). Bravo ran the app and verified clean single-token rendering. Lead cross-checked that both SDK and CLI execution paths used the same corrected handler. All three PASS. Gate opened.


THE COST AND WHEN IT IS WORTH IT

The default configuration in config.py assigns models by role: Lead runs on Opus (deepest reasoning for architectural analysis), while Alpha and Bravo run on Sonnet (best coding model for detail work):

  lead: AgentConfig = field(default_factory=lambda: AgentConfig(model="opus"))
  alpha: AgentConfig = field(default_factory=lambda: AgentConfig(model="sonnet"))
  bravo: AgentConfig = field(default_factory=lambda: AgentConfig(model="sonnet"))

The pipeline defaults to four phases -- explore, audit, fix, verify -- with a maximum of 3 fix cycles per gate before hard failure. Running three agents in parallel (parallel_agents: true by default) means you pay 3x in compute but roughly 1x in wall clock time.

I have run this pattern across 3 projects with 10 blocking gates each. The cost per gate averages roughly $0.15. For the ILS streaming bug specifically, the consensus pass that caught it cost minutes and cents. The alternative -- a P2 visual duplication bug in a live product's core chat interface -- would have required a hotfix release and weeks of trust repair.

Use three-agent consensus when complex state management has multiple interacting update mechanisms, when multiple system layers must agree on data contracts, when a bug would be immediately visible to users, or when you are running comprehensive audits across 50+ files. Stick with single-agent review for isolated, low-risk changes where speed matters more than thoroughness.


THE BROADER PRINCIPLE

What makes this work is not the number three. It is two structural properties: independent verification and hard gates.

Independent verification means each agent starts from scratch with no anchor on another agent's conclusions. This is the AI equivalent of not showing one code reviewer another reviewer's comments before they have formed their own opinion. It eliminates groupthink.

Hard gates mean that "mostly passing" is not passing. Two-out-of-three does not open the gate. The unanimity requirement, implemented as "all(v.is_pass() for v in votes)", eliminates the reviewer who waves something through because someone else already approved it.

The += operator was right there on line 926 for three days. It looked correct because the pattern -- accumulate text in a streaming handler -- should use append. The bug was that this particular text was already accumulated. A single perspective saw what it expected. Three independent perspectives, forced to agree, found what one would have shipped.

Full framework with CLI, Pydantic models, and pipeline orchestrator: github.com/krzemienski/multi-agent-consensus

Technical writeup with system diagrams: github.com/krzemienski/agentic-development-guide/tree/main/04-multi-agent-consensus

#CodeQuality #AI #SoftwareEngineering #CodeReview #AgenticDevelopment


---

================================================================
POST 3: I BANNED UNIT TESTS FROM MY AI WORKFLOW
Agentic Development: 10 Lessons from 8,481 AI Coding Sessions
================================================================

I said it out loud in a team meeting and watched the room go quiet: "I don't write unit tests anymore. I banned them."

Before you close this tab, hear me out. I didn't stop verifying my code. I stopped pretending that AI-generated tests verify anything.

Over 8,481 coding sessions with Claude Code, I discovered a fundamental problem with the test-driven workflow everyone assumes is best practice. When AI writes both the implementation AND the tests, passing tests are not independent evidence of correctness. They are a mirror reflecting itself.

So I built something different. I built a functional validation framework that forces real systems to prove they work -- with screenshots, accessibility trees, HTTP responses, and timestamped evidence you can audit after the fact.

This post walks through the framework, the code, and the four categories of bugs that unit tests systematically miss.


THE MIRROR PROBLEM

Here is the scenario that broke my faith in AI-generated tests.

I asked Claude to build a sidebar navigation component for an iOS app. It wrote the SwiftUI view. Then I asked it to write tests. It produced 14 unit tests. All green. Ship it.

Except the sidebar was invisible. The view existed in the hierarchy, the state management was correct, the navigation routing worked -- but a z-ordering issue meant the sidebar rendered behind the main content. Every unit test passed because they tested the model layer, not what the user sees.

This is not a contrived example. It happened. I have the screenshots to prove it.

The problem is structural. Unit tests verify implementation contracts. When the same intelligence writes both the contract and the verification, you get circular reasoning. The test suite becomes a tautology: "the code does what the code does."

Functional validation asks a different question: "Does the running system behave the way a human expects?"


THE FOUR BUG CATEGORIES UNIT TESTS MISS

After cataloging bugs across hundreds of sessions, I found they cluster into four categories that unit tests are structurally blind to.

1. VISUAL RENDERING BUGS

The sidebar z-ordering bug above. Colors that don't match the spec. Font sizes below the Human Interface Guidelines minimum (I found 39 instances of sub-11pt fonts in one audit pass). Layout that looks correct in a test harness but breaks on a real device frame.

These bugs exist in the gap between "the view model has the right state" and "the user sees the right thing." Unit tests live on one side of that gap. Users live on the other.

2. INTEGRATION BOUNDARY BUGS

A Vapor backend returns camelCase JSON. The iOS client expects snake_case. Every unit test on both sides passes independently. The system is broken.

I hit this exact bug with two backend binaries. The old backend at one path returned raw Claude Code data with bare arrays and snake_case. The current backend at another path returned proper API response wrappers with camelCase. Unit tests for both were green. The app was non-functional until I validated the actual HTTP response against the actual JSON decoder.

3. STATE MANAGEMENT BUGS

SwiftUI re-renders on state changes. But a @State property initialized in the wrong lifecycle phase, or an @EnvironmentObject accessed before injection completes, produces crashes that no unit test catches -- because unit tests don't have a real SwiftUI lifecycle.

I discovered that changing the default activeScreen to .settings in a NavigationStack crashed the app because @EnvironmentObject wasn't ready during @State init. No test would have found this. The simulator found it in 2 seconds.

4. PLATFORM-SPECIFIC BUGS

Claude CLI includes nesting detection. If CLAUDECODE=1 is in the environment, the CLI silently refuses to execute. When your Vapor backend runs inside a Claude Code session, spawned subprocesses inherit these variables. No error. No stderr. Just a zero-byte response.

The fix was three lines of code. Finding those three lines cost a full debugging session. No unit test would have surfaced this because the test environment wouldn't have the offending env vars.


THE FRAMEWORK: FUNCTIONAL VALIDATION FROM SCRATCH

I formalized all of this into a Python framework. The core idea: define validation gates, run them against live systems, collect timestamped evidence, and generate auditable reports.

The data model starts with Pydantic. Here are the core types:

  # src/fvf/models.py

  class EvidenceType(str, Enum):
      SCREENSHOT = "screenshot"
      CURL_OUTPUT = "curl_output"
      ACCESSIBILITY_TREE = "accessibility_tree"
      LOG = "log"
      VIDEO = "video"
      NETWORK_HAR = "network_har"

  class ValidationResult(BaseModel):
      status: ValidationStatus
      message: str
      evidence: list[EvidenceItem] = Field(default_factory=list)
      duration_ms: float = 0.0
      validator_name: str = ""

      @property
      def passed(self) -> bool:
          return self.status == ValidationStatus.PASSED

      @property
      def failed(self) -> bool:
          return self.status in (ValidationStatus.FAILED, ValidationStatus.ERROR)

Every validation produces a ValidationResult with an explicit status, a human-readable message, and a list of evidence items. Evidence is not optional. If you claim something passed, you need the receipts.

Gates are numbered, ordered, and can declare dependencies:

  # src/fvf/models.py

  class GateDefinition(BaseModel):
      number: int = Field(ge=1, description="Gate number (1-based, determines execution order)")
      name: str
      description: str = ""
      criteria: list[GateCriteria] = Field(default_factory=list)
      depends_on: list[int] = Field(
          default_factory=list,
          description="Gate numbers that must pass before this gate runs",
      )

      @field_validator("depends_on")
      @classmethod
      def no_self_dependency(cls, v: list[int], info: Any) -> list[int]:
          number = info.data.get("number")
          if number is not None and number in v:
              raise ValueError(f"Gate {number} cannot depend on itself")
          return v

The dependency system matters. If Gate 1 ("App Launches") fails, there is no point running Gate 5 ("Chat Messages Render Correctly"). The GateRunner handles this automatically:

  # src/fvf/gates/gate.py

  def run_all(self) -> list[GateResult]:
      completed: list[GateResult] = []
      failed_gate_numbers: set[int] = set()

      for gate in self._gates:
          if not self._check_dependencies(gate, completed, failed_gate_numbers):
              skipped = GateResult(
                  gate=gate,
                  status=ValidationStatus.SKIPPED,
                  results=[
                      ValidationResult(
                          status=ValidationStatus.SKIPPED,
                          message=(
                              f"Skipped -- dependency gate(s) "
                              f"{gate.depends_on} did not pass"
                          ),
                          validator_name="GateRunner",
                      )
                  ],
              )
              completed.append(skipped)
              failed_gate_numbers.add(gate.number)
              continue

          gate_result = self.run_gate(gate)
          completed.append(gate_result)
          if not gate_result.passed:
              failed_gate_numbers.add(gate.number)

      return completed


FOUR VALIDATORS FOR FOUR SURFACES

The framework ships with four validators. Each targets a different validation surface.

THE BROWSER VALIDATOR uses Playwright to drive a real Chromium browser. No JSDOM. No enzyme. A real browser rendering real pixels:

  # src/fvf/validators/browser.py

  class BrowserValidator(Validator):
      def validate(self, criteria: GateCriteria) -> ValidationResult:
          # ...
          with sync_playwright() as pw:
              browser = pw.chromium.launch(headless=True)
              context = browser.new_context()
              page = context.new_page()
              page.set_default_timeout(self._config.browser_timeout)

              response = page.goto(url, wait_until="networkidle")

              for action in vc.get("actions", []):
                  self._execute_action(page, action)

              for assertion in vc.get("assertions", []):
                  atype = assertion.get("type", "")
                  if atype == "status_code":
                      expected_code = int(assertion.get("expected", 200))
                      actual_code = response.status if response else -1
                      ok = self._check_status_code(actual_code, expected_code)
                      # ...
                  elif atype == "element_visible":
                      selector = assertion.get("selector", "")
                      ok = self._check_element_visible(page, selector)
                      # ...

              # Always capture a screenshot as evidence
              page.screenshot(path=str(screenshot_path), full_page=True)

That last line is the key. Every single validation run captures a screenshot. Even if all assertions pass. The screenshot is evidence you can review tomorrow when someone asks "are you sure this worked?"

THE iOS VALIDATOR drives the actual iOS Simulator via idb and simctl. Deep links, tap gestures, swipe gestures, accessibility tree inspection:

  # src/fvf/validators/ios.py

  class IOSValidator(Validator):
      def validate(self, criteria: GateCriteria) -> ValidationResult:
          # ...
          if dl := vc.get("deep_link"):
              self._deep_link(dl)
              time.sleep(1.5)  # Allow app to settle

          for action in vc.get("actions", []):
              self._execute_action(action)

          tree = self._get_accessibility_tree()

          for assertion in vc.get("assertions", []):
              atype = assertion.get("type", "")
              if atype == "element_present":
                  label = assertion.get("label", "")
                  element = self._find_element(tree, label)
                  ok = element is not None

The accessibility tree search is recursive. It walks the entire UI hierarchy looking for elements by label, checking both the label and value attributes:

  # src/fvf/validators/ios.py

  def _find_element(self, tree: dict[str, Any], label: str) -> dict[str, Any] | None:
      if not tree:
          return None

      node_label = str(tree.get("label", "") or tree.get("AXLabel", ""))
      node_value = str(tree.get("value", "") or tree.get("AXValue", ""))
      if label.lower() in node_label.lower() or label.lower() in node_value.lower():
          return tree

      for child in tree.get("children", []):
          found = self._find_element(child, label)
          if found is not None:
              return found

      return None

This is how you verify an iOS app works: you open it, you navigate to a screen, you dump the accessibility tree, and you check that the elements you expect are actually present. Not "the view model contains the right data." The elements. On screen. In the tree.

THE API VALIDATOR makes real HTTP requests with httpx. Status codes, response times, JSON path assertions, schema validation:

  # src/fvf/validators/api.py

  class APIValidator(Validator):
      def validate(self, criteria: GateCriteria) -> ValidationResult:
          # ...
          with httpx.Client(timeout=timeout_s) as client:
              response = client.request(
                  method, url,
                  headers=headers,
                  json=body if isinstance(body, (dict, list)) else None,
              )
          duration_ms_actual = (time.monotonic() - request_start) * 1000

          ok = self._check_status(response.status_code, expected_status)
          ok = self._check_response_time(duration_ms_actual, max_response_time_ms)

Every API validation saves a curl-equivalent command as evidence. You can replay it. You can share it. You can paste it into a terminal six months later and see if the endpoint still works.

THE SCREENSHOT VALIDATOR captures and optionally compares screenshots against reference images using pixel-level similarity scoring:

  # src/fvf/validators/screenshot.py

  def _compare_screenshots(self, actual: Path, reference: Path, threshold: float) -> tuple[bool, float]:
      img_actual = Image.open(actual).convert("RGB")
      img_reference = Image.open(reference).convert("RGB")

      if img_actual.size != img_reference.size:
          img_reference = img_reference.resize(img_actual.size, Image.LANCZOS)

      diff = ImageChops.difference(img_actual, img_reference)
      pixels = list(diff.getdata())
      total_diff = sum(max(r, g, b) for r, g, b in pixels)
      max_possible = len(pixels) * 255
      similarity = 1.0 - (total_diff / max_possible)
      return similarity >= threshold, similarity

This catches the class of visual regression that no amount of unit testing will find. A CSS change that shifts a button 3 pixels. A theme change that makes text unreadable against its background. A z-index change that hides a critical element.


THE EVIDENCE SYSTEM

Evidence is not an afterthought. It is the core of the framework. The EvidenceCollector organizes artifacts into a timestamped directory structure:

  # src/fvf/gates/evidence.py

  class EvidenceCollector:
      """
      Directory structure:

          evidence/
            gate-1/
              20240101-120000/
                manifest.json
                screenshot-browser-1234.png
                api-curl-1234.txt
            gate-2/
              20240101-120010/
                manifest.json
      """

      def collect(self, gate_number: int, items: list[EvidenceItem]) -> Path:
          timestamp = datetime.utcnow().strftime(self._TIMESTAMP_FORMAT)
          attempt_dir = self._gate_dir(gate_number) / timestamp
          attempt_dir.mkdir(parents=True, exist_ok=True)

          saved: list[EvidenceItem] = []
          for item in items:
              saved_item = self._save_item(item, attempt_dir)
              saved.append(saved_item)

          self._generate_manifest(saved, attempt_dir)
          return attempt_dir

Multiple attempts are preserved independently. Each attempt gets its own timestamped directory and a manifest.json describing every artifact. You can diff evidence across attempts. You can see exactly when a regression was introduced.

The report generator produces Markdown, JSON, or self-contained HTML with embedded base64 screenshots:

  # src/fvf/gates/report.py

  class ReportGenerator:
      def to_html(self, report: GateReport) -> str:
          # Screenshots embedded as base64 data URIs
          for item in gr.total_evidence:
              if item.type.value == "screenshot" and item.path.exists():
                  b64 = base64.b64encode(item.path.read_bytes()).decode()
                  # Embed directly in <img src="data:image/png;base64,...">

The HTML report is fully portable. No external dependencies. Drop it in a PR comment, email it, put it in a wiki. The evidence travels with the claim.

[DIAGRAM: Evidence flow -- Gate YAML definition feeds into GateRunner, which dispatches to Browser/iOS/API/Screenshot validators, each producing EvidenceItems that flow into the EvidenceCollector, which writes timestamped directories, which feed into the ReportGenerator for Markdown/JSON/HTML output]


THE CLI: THREE COMMANDS

The whole framework runs from the command line:

  # Scaffold a gate config
  fvf init --type browser

  # Run all gates
  fvf validate --gate gates.yaml

  # Generate a report
  fvf report --evidence-dir ./evidence/ --format html

  # Run a single gate
  fvf gate run 3 --gate-file gates.yaml

  # List gates with evidence status
  fvf gate list gates.yaml --evidence-dir ./evidence/

The validate command is the workhorse. It loads gate definitions from YAML, instantiates the appropriate validators, runs them in dependency order, collects evidence, prints a rich progress bar, and exits with code 0 if all gates pass, 1 if any fail. CI-friendly out of the box.


THE NUMBERS

Across the ILS project (the native iOS client for Claude Code that this series covers), functional validation produced:

- 470+ screenshots as validation evidence
- 37+ validation gates across 10 development phases
- 3 browser automation tools integrated (Playwright, idb, simctl)
- 4 bug categories systematically caught that unit tests miss
- 0 unit tests written. Zero. Not one.

The 470 screenshots are not decorative. Each one is timestamped evidence that a specific screen, in a specific state, on a specific device, rendered correctly at a specific point in time. When a regression appears three weeks later, I can binary-search through evidence directories to find exactly when it broke.


THE AGGREGATED REPORT

At the end of a validation run, the GateReport model computes derived metrics:

  # src/fvf/models.py

  class GateReport(BaseModel):
      project_name: str
      gates: list[GateResult] = Field(default_factory=list)
      total_gates: int = 0
      passed: int = 0
      failed: int = 0
      evidence_count: int = 0

      def model_post_init(self, __context: Any) -> None:
          if self.gates and self.total_gates == 0:
              self.total_gates = len(self.gates)
              self.passed = sum(1 for g in self.gates if g.passed)
              self.failed = self.total_gates - self.passed
              self.evidence_count = sum(len(g.total_evidence) for g in self.gates)

      @property
      def pass_rate(self) -> float:
          if self.total_gates == 0:
              return 0.0
          return self.passed / self.total_gates

This is the final artifact. Not "47 tests pass." Instead: "13/13 gates passed, 42 evidence items collected, 100% pass rate." And behind that number is a directory tree of screenshots, accessibility dumps, and curl outputs that anyone can audit.


WHAT THIS ACTUALLY LOOKS LIKE IN PRACTICE

A typical gate YAML for the iOS app looks like this:

  project: ils-ios
  gates:
    - number: 1
      name: App Launches
      description: Verify the app launches and shows the home screen
      criteria:
        - description: Home screen visible
          evidence_required: [screenshot, accessibility_tree]
          validator_type: ios
          validator_config:
            deep_link: ils://home
            assertions:
              - type: element_present
                label: Home

    - number: 2
      name: Sessions Load
      depends_on: [1]
      criteria:
        - description: Sessions screen shows data
          evidence_required: [screenshot, accessibility_tree]
          validator_type: ios
          validator_config:
            deep_link: ils://sessions
            assertions:
              - type: element_present
                label: Sessions

Gate 2 depends on Gate 1. If the app doesn't launch, we don't bother checking if sessions load. This is obvious logic, but it's logic that unit test frameworks don't give you out of the box.


THE LESSON

I am not saying unit tests are useless in general. I am saying that when AI writes both the code and the tests, the tests lose their epistemic value. They become a ritual, not a verification.

Functional validation restores the independence. The validator doesn't know how the code works. It knows what the user should see. It opens the app, navigates to a screen, and checks if the right elements are there. If they are, it takes a screenshot as proof. If they aren't, it tells you exactly what's missing.

After 8,481 sessions, this is the most counterintuitive lesson I've learned: the way to ship faster with AI is to stop writing tests and start collecting evidence.

The companion repo has the full framework. Clone it, run fvf init, and try it on your own project. I think you'll be surprised how many bugs your test suite has been hiding.

GitHub: https://github.com/nickbaumann98/functional-validation-framework

#AgenticDevelopment #FunctionalValidation #ClaudeCode #AIEngineering #SoftwareQuality



================================================================
POST 4: THE 5-LAYER SSE BRIDGE -- BUILDING A NATIVE iOS CLIENT FOR CLAUDE CODE
Agentic Development: 10 Lessons from 8,481 AI Coding Sessions
================================================================

Every token Claude generates on your behalf traverses five layers before it appears on your iPhone screen. SwiftUI view. Vapor backend. Python SDK wrapper. Claude CLI. Anthropic API. Then the whole chain reverses: API response, CLI stdout, Python NDJSON, Vapor SSE, URLSession async bytes, SwiftUI @Observable.

Ten hops per token. Each one a place where the stream can break, stall, duplicate, or silently disappear.

This is the story of building ILS, a native iOS and macOS client for Claude Code. Not a web wrapper. Not a thin API client. A full SwiftUI application that streams Claude's responses token-by-token over Server-Sent Events, with reconnection logic, heartbeat monitoring, environment variable stripping, and a two-character bug that took an entire session to find.

The companion repo extracts the streaming bridge as a standalone Swift package you can drop into any project.


THE ARCHITECTURE

[DIAGRAM: 5-layer architecture stack -- SwiftUI ChatView at top, arrow down to SSEClient (URLSession async bytes), arrow down to Vapor Backend (SSE endpoint), arrow down to Python SDK Wrapper (sdk-wrapper.py with flush=True), arrow down to Claude CLI (--output-format stream-json), arrow down to Anthropic API. Reverse arrows show the response path. Labels on each arrow: "StreamMessage JSON", "SSE event: data:", "NDJSON lines", "stream-json stdout", "API response chunks"]

Layer 1: SwiftUI ChatView observes an @Observable SSEClient.
Layer 2: SSEClient opens a POST to the Vapor backend's /api/v1/chat/stream endpoint.
Layer 3: Vapor spawns python3 sdk-wrapper.py, which calls the claude-agent-sdk.
Layer 4: The SDK wraps Claude CLI with --output-format stream-json.
Layer 5: Claude CLI calls the Anthropic API.

Responses flow back up the same chain. Each layer adds its own failure modes.


LAYER 1: THE BRIDGE CONFIGURATION

Everything starts with a configuration struct that controls timeouts, reconnection, and network behavior:

  // Sources/StreamingBridge/Configuration.swift

  public struct BridgeConfiguration: Sendable {
      public let backendURL: String
      public let initialTimeout: TimeInterval
      public let totalTimeout: TimeInterval
      public let heartbeatTimeout: TimeInterval
      public let maxReconnectAttempts: Int
      public let reconnectBaseDelay: TimeInterval
      public let allowsExpensiveNetworkAccess: Bool
      public let allowsConstrainedNetworkAccess: Bool
      public let streamEndpoint: String
      public let sdkWrapperPath: String

      public static let `default` = BridgeConfiguration()

      public init(
          backendURL: String = "http://localhost:9999",
          initialTimeout: TimeInterval = 30,
          totalTimeout: TimeInterval = 300,
          heartbeatTimeout: TimeInterval = 45,
          maxReconnectAttempts: Int = 10,
          reconnectBaseDelay: TimeInterval = 2,
          allowsExpensiveNetworkAccess: Bool = true,
          allowsConstrainedNetworkAccess: Bool = false,
          streamEndpoint: String = "/api/v1/chat/stream",
          sdkWrapperPath: String = "scripts/sdk-wrapper.py"
      ) { ... }
  }

Three timeout values, not one. This is not over-engineering. This is the result of watching streams fail in three different ways.

The initialTimeout (30 seconds) catches stuck connections where the backend accepts the HTTP connection but never sends a byte. This happens when Claude is overwhelmed or when the Python subprocess fails to launch.

The totalTimeout (300 seconds) caps the entire streaming session. Some prompts trigger extended thinking that can run for minutes. Five minutes is generous but bounded. Without this, a hung Claude process silently consumes battery forever.

The heartbeatTimeout (45 seconds) is the watchdog. Even after a healthy connection is established, the stream can go stale -- network interruption, backend crash, Claude timeout on the API side. If no data arrives for 45 seconds, the connection is declared dead.

Notice allowsConstrainedNetworkAccess defaults to false. SSE streaming is continuous data. If the user has Low Data Mode enabled, we respect that and don't stream Claude responses over a metered connection.


LAYER 2: THE SSE CLIENT

The SSEClient is an @Observable @MainActor class that manages the HTTP connection lifecycle:

  // Sources/StreamingBridge/SSEClient.swift

  @MainActor
  @Observable
  public class SSEClient {
      public var messages: [StreamMessage] = []
      public var isStreaming: Bool = false
      public var error: Error?
      public var connectionState: ConnectionState = .disconnected

      public enum ConnectionState: Equatable, Sendable {
          case disconnected
          case connecting
          case connected
          case reconnecting(attempt: Int)
      }

The connection state is an explicit enum, not a boolean. This matters for the UI. "Connecting..." is different from "Reconnecting (attempt 3)..." which is different from "Connected." The user needs to understand what is happening.

The stream implementation races the connection against a 60-second timeout using a TaskGroup:

  // Sources/StreamingBridge/SSEClient.swift

  let (asyncBytes, response) = try await withThrowingTaskGroup(
      of: (URLSession.AsyncBytes, URLResponse).self
  ) { group in
      group.addTask {
          try await urlSession.bytes(for: urlRequest)
      }
      group.addTask {
          try await Task.sleep(nanoseconds: 60_000_000_000)
          throw URLError(.timedOut)
      }
      let result = try await group.next()!
      group.cancelAll()
      return result
  }

Whichever finishes first wins. If the connection succeeds, the timeout task is cancelled. If 60 seconds elapse, the connection is killed. This is the initial connection timeout -- separate from the heartbeat watchdog that monitors the established stream.

Once connected, the heartbeat watchdog starts:

  // Sources/StreamingBridge/SSEClient.swift

  let lastActivity = LastActivityTracker()
  let watchdogTimeout = configuration.heartbeatTimeout

  let heartbeatWatchdog = Task.detached { [watchdogTimeout] in
      while !Task.isCancelled {
          try await Task.sleep(nanoseconds: 15_000_000_000)
          if lastActivity.secondsSinceLastActivity() > watchdogTimeout {
              throw URLError(.timedOut)
          }
      }
  }
  defer { heartbeatWatchdog.cancel() }

The LastActivityTracker is a dedicated class using OSAllocatedUnfairLock instead of an actor:

  // Sources/StreamingBridge/SSEClient.swift

  private final class LastActivityTracker: Sendable {
      private let storage = OSAllocatedUnfairLock(initialState: Date())

      func touch() {
          storage.withLock { $0 = Date() }
      }

      func secondsSinceLastActivity() -> TimeInterval {
          let last = storage.withLock { $0 }
          return Date().timeIntervalSince(last)
      }
  }

Why not an actor? Because touch() is called on every single SSE line received. That is the hot path during streaming. Actor-hop overhead on every received line adds up. OSAllocatedUnfairLock gives us thread safety without the context switch.

Every line received calls lastActivity.touch(). Every 15 seconds, the watchdog checks if we've gone silent for longer than the heartbeat threshold. If so, the stream is declared stale and the reconnection logic kicks in.


RECONNECTION WITH EXPONENTIAL BACKOFF

When the stream drops, the client doesn't just retry immediately. It uses exponential backoff capped at 30 seconds:

  // Sources/StreamingBridge/SSEClient.swift

  private func shouldReconnect(error: Error) async -> Bool {
      guard let request = currentRequest,
            reconnectAttempts < configuration.maxReconnectAttempts,
            isNetworkError(error) else {
          return false
      }

      reconnectAttempts += 1
      connectionState = .reconnecting(attempt: reconnectAttempts)

      let baseNanos = UInt64(configuration.reconnectBaseDelay * 1_000_000_000)
      let delay = min(baseNanos * UInt64(1 << (reconnectAttempts - 1)), 30_000_000_000)

      let sleepTask = Task<Void, Never> {
          try? await Task.sleep(nanoseconds: delay)
      }
      backoffSleepTask = sleepTask
      await sleepTask.value

      if Task.isCancelled { return false }

      await performStream(request: request)
      return true
  }

The backoff sleep is stored in a separate task property so that resetAndReconnect() can cancel the sleep and force an immediate retry. This gives the user a "Retry Now" button that actually works, rather than making them wait through the exponential delay.

Only network errors trigger reconnection. Application-level errors (bad JSON, authentication failures, rate limits) fail immediately because retrying won't help.

On iOS, the client also listens for background notifications:

  // Sources/StreamingBridge/SSEClient.swift

  #if os(iOS)
  backgroundObserver = NotificationCenter.default.addObserver(
      forName: UIApplication.didEnterBackgroundNotification,
      object: nil,
      queue: .main
  ) { [weak self] _ in
      Task { @MainActor [weak self] in
          guard let self, self.isStreaming else { return }
          self.cancel()
      }
  }
  #endif

When the app goes to background, the SSE stream is cancelled. This saves battery radio. SSE keeps the radio active continuously -- on cellular, that is significant power drain. The user can reconnect when they return to the app.


THE MESSAGE TYPE SYSTEM

Every message on the SSE stream is decoded into a discriminated union:

  // Sources/StreamingBridge/StreamingTypes.swift

  public enum StreamMessage: Codable, Sendable {
      case system(SystemMessage)
      case assistant(AssistantMessage)
      case user(UserMessage)
      case result(ResultMessage)
      case streamEvent(StreamEventMessage)
      case error(StreamError)

      public init(from decoder: Decoder) throws {
          let container = try decoder.container(keyedBy: CodingKeys.self)
          let type = try container.decode(String.self, forKey: .type)

          switch type {
          case "system":    self = .system(try SystemMessage(from: decoder))
          case "assistant": self = .assistant(try AssistantMessage(from: decoder))
          case "user":      self = .user(try UserMessage(from: decoder))
          case "result":    self = .result(try ResultMessage(from: decoder))
          case "streamEvent": self = .streamEvent(try StreamEventMessage(from: decoder))
          case "error":     self = .error(try StreamError(from: decoder))
          default:
              throw DecodingError.dataCorruptedError(
                  forKey: .type, in: container,
                  debugDescription: "Unknown stream message type: \(type)"
              )
          }
      }
  }

The type field in the JSON drives the decode. This is Claude Code's native streaming format, not something I invented. The bridge has to handle every message type the CLI can produce.

Content blocks are similarly discriminated:

  // Sources/StreamingBridge/StreamingTypes.swift

  public enum ContentBlock: Codable, Sendable {
      case text(TextBlock)
      case toolUse(ToolUseBlock)
      case toolResult(ToolResultBlock)
      case thinking(ThinkingBlock)

      public init(from decoder: Decoder) throws {
          let container = try decoder.container(keyedBy: CodingKeys.self)
          let type = try container.decode(String.self, forKey: .type)

          switch type {
          case "text":
              self = .text(try TextBlock(from: decoder))
          case "tool_use", "toolUse":
              self = .toolUse(try ToolUseBlock(from: decoder))
          case "tool_result", "toolResult":
              self = .toolResult(try ToolResultBlock(from: decoder))
          case "thinking":
              self = .thinking(try ThinkingBlock(from: decoder))
          default:
              self = .text(TextBlock(text: "[Unknown block type: \(type)]"))
          }
      }
  }

Notice both "tool_use" and "toolUse" are accepted. Claude Code uses snake_case internally. The Vapor backend may normalize to camelCase. The bridge handles both, because in production you never control the entire pipeline perfectly.

Stream deltas for character-by-character delivery:

  // Sources/StreamingBridge/StreamingTypes.swift

  public enum StreamDelta: Codable, Sendable {
      case textDelta(String)
      case inputJsonDelta(String)
      case thinkingDelta(String)

      public init(from decoder: Decoder) throws {
          let container = try decoder.container(keyedBy: CodingKeys.self)
          let type = try container.decode(String.self, forKey: .type)
          switch type {
          case "text_delta", "textDelta":
              let text = try container.decode(String.self, forKey: .text)
              self = .textDelta(text)
          case "thinking_delta", "thinkingDelta":
              let thinking = try container.decode(String.self, forKey: .thinking)
              self = .thinkingDelta(thinking)
          // ...
          }
      }
  }

The result message carries token usage and cost:

  // Sources/StreamingBridge/StreamingTypes.swift

  public struct ResultMessage: Codable, Sendable {
      public let type: String
      public let subtype: String
      public let sessionId: String
      public let durationMs: Int?
      public let isError: Bool
      public let numTurns: Int?
      public let totalCostUSD: Double?
      public let usage: UsageInfo?
      public let result: String?
  }

This is how you show "$0.04" next to each response in the UI. The cost data comes from Claude Code itself, through five layers, into a Codable struct on the client.


LAYER 3-4: THE EXECUTOR SERVICE AND THE NESTING DETECTION BUG

The ClaudeExecutorService spawns the Python SDK wrapper as a subprocess. This is where the most painful bug in the entire project lives.

Claude CLI has nesting detection. If the environment variable CLAUDECODE=1 is set, or any CLAUDE_CODE_* variables exist, the CLI assumes it's being called from inside an active Claude Code session and refuses to execute. No error message. No stderr output. Just silence.

When you're developing the backend inside Claude Code (which I was, for 8,481 sessions), the backend process inherits these environment variables. Every subprocess it spawns inherits them too. So Claude CLI, called by the Python SDK, called by the Vapor backend, running inside a Claude Code session... silently does nothing.

The fix is documented in the source as a warning to future developers:

  // Sources/StreamingBridge/ClaudeExecutorService.swift

  // CRITICAL: Strip CLAUDE* env vars to prevent nesting detection.
  // Without this, Claude CLI silently refuses to execute inside
  // an active Claude Code session -- no error, no stderr, just
  // a zero-byte response.
  let cleanCmd = """
      for v in $(env | grep ^CLAUDE | cut -d= -f1); do unset $v; done; \(command)
      """
  process.arguments = ["-l", "-c", cleanCmd]

  // Belt-and-suspenders: also strip from Process.environment
  var env = ProcessInfo.processInfo.environment
  for key in env.keys where key.hasPrefix("CLAUDE") {
      env.removeValue(forKey: key)
  }
  process.environment = env

Belt and suspenders. The shell command unsets them. The Process.environment also strips them. Because I never want to debug this again.

The two-tier timeout is implemented with GCD DispatchWorkItems:

  // Sources/StreamingBridge/ClaudeExecutorService.swift

  let didTimeout = AtomicBool(false)

  let timeoutWork = DispatchWorkItem {
      didTimeout.value = true
      process.terminate()
      outputPipe.fileHandleForReading.closeFile()
  }
  DispatchQueue.global().asyncAfter(
      deadline: .now() + config.initialTimeout,
      execute: timeoutWork
  )

  let totalTimeoutWork = DispatchWorkItem {
      if process.isRunning {
          didTimeout.value = true
          process.terminate()
          outputPipe.fileHandleForReading.closeFile()
      }
  }
  DispatchQueue.global().asyncAfter(
      deadline: .now() + config.totalTimeout,
      execute: totalTimeoutWork
  )

The initial timeout is cancelled as soon as the first byte of stdout data arrives. The total timeout runs for the entire session. AtomicBool shares the timeout state safely across GCD queues.

[DIAGRAM: Two-tier timeout timeline -- horizontal axis is time, initial timeout at 30s mark gets cancelled when first data arrives, total timeout at 300s mark is the absolute cap. Data flow starts after first byte, with the heartbeat watchdog checking every 15s during the data flow period.]

The stdout reader runs on a dedicated GCD queue. Not the main queue. Not an actor. A raw DispatchQueue. This avoids the RunLoop dependency that killed the original ClaudeCodeSDK integration:

  // Sources/StreamingBridge/ClaudeExecutorService.swift

  self.readQueue.async {
      Self.readStdout(
          pipe: outputPipe,
          errorPipe: errorPipe,
          process: process,
          // ...
      )
  }

The SDK uses FileHandle.readabilityHandler and Combine PassthroughSubject, which require RunLoop. Vapor's NIO event loops don't pump RunLoop. The publisher never emits. Months of debugging led to: just use Process with GCD. Direct. No SDK.


THE NSTASK TERMINATION STATUS CRASH

Here is a bug that only manifests under race conditions:

  // Sources/StreamingBridge/ClaudeExecutorService.swift

  // CRITICAL: Always call waitUntilExit() before terminationStatus.
  // Reading EOF from stdout does NOT mean the process has exited.
  // Accessing terminationStatus on a running Process throws
  // NSInvalidArgumentException.
  process.waitUntilExit()
  timeoutWork.cancel()
  totalTimeoutWork.cancel()

  let exitCode = process.terminationStatus

When you read EOF from a pipe, the intuition is "the process is done." Wrong. There is a race between the pipe closing and the process exiting. If you read terminationStatus before the process has actually terminated, Foundation throws NSInvalidArgumentException. Not a Swift error you can catch. An Objective-C exception that crashes your app.

The fix is one line: process.waitUntilExit(). But finding that one line required a crash report, a stack trace, and the realization that pipe EOF and process exit are two different events.


THE TWO-CHARACTER BUG: += VS =

This bug deserves its own section because it is the purest example of how streaming introduces failure modes that don't exist in request-response architectures.

The SSEClient receives two kinds of text events. An assistant event contains the accumulated text so far. A streamEvent with a textDelta contains only the new characters since the last delta.

  // Example/Sources/ExampleApp/ChatViewModel.swift

  case .assistant(let assistantMsg):
      for block in assistantMsg.content {
          switch block {
          case .text(let textBlock):
              // CRITICAL: Use assignment, not append.
              // The assistant event contains accumulated text.
              // Using += would duplicate: "Hello" -> "HelloHello"
              currentMessage.text = textBlock.text
          // ...

  case .streamEvent(let event):
      switch delta {
      case .textDelta(let text):
          // For deltas, += is correct -- each delta is incremental
          currentMessage.text += text

If you use += for assistant events, "Hello" becomes "HelloHello". The first assistant event says "Hello". The second says "Hello world". With +=, you get "HelloHello world". With =, you get "Hello world".

Two characters. The difference between = and +=. One produces a working chat interface. The other produces gibberish that doubles in size with every event.

The comment in the source code is emphatic for a reason. This bug was identified as a P2 severity issue in production. The SSEClient header documents it:

  // Sources/StreamingBridge/SSEClient.swift (class documentation)

  /// ## The Text Duplication Bug (P2)
  ///
  /// A critical lesson from production: the `+=` vs `=` distinction matters
  /// for assistant messages. Each `assistant` event contains the **accumulated**
  /// text, not just the new token. Using `+=` on an already-accumulated string
  /// produces exact duplication. The fix: use `=` (assignment) for assistant
  /// events, `+=` only for `textDelta` stream events.


THE PYTHON STDOUT BUFFERING DISCOVERY

Between the Vapor backend and Claude CLI sits a Python script. Python buffers stdout by default. When you print() in Python, the output doesn't reach the parent process immediately -- it accumulates in a buffer and flushes when the buffer is full or the process exits.

For a streaming application, this is fatal. The user sends "Hi" to Claude. Claude responds immediately. The Python wrapper buffers the response. The Vapor backend sees nothing. The iOS client shows "Connecting..." for 30 seconds. Then everything arrives in one burst.

The fix: flush=True on every print statement, or PYTHONUNBUFFERED=1 in the environment, or python3 -u. Three different solutions to the same problem. We use the explicit flush because it's self-documenting in the code.

This is a bug that doesn't exist if you test the Python script in isolation (because the terminal is line-buffered, not block-buffered). It only appears when Python's stdout is a pipe to another process. Another example of an integration boundary bug that no unit test catches.


PUTTING IT ALL TOGETHER

The ChatViewModel ties the SSE client to the SwiftUI view layer:

  // Example/Sources/ExampleApp/ChatViewModel.swift

  func sendMessage(_ text: String) {
      guard let sseClient else { return }
      messages.append(ChatMessage(isUser: true, text: text))
      let request = ChatStreamRequest(prompt: text)
      sseClient.startStream(request: request)
  }

One line to add the user message. One line to create the request. One line to start the stream. The complexity is in the SSEClient, the executor, the timeout management, the reconnection logic, the message parsing, and the evidence that it all works.

The observation binding uses Swift's withObservationTracking to react to SSEClient state changes:

  // Example/Sources/ExampleApp/ChatViewModel.swift

  observationTask = Task { @MainActor [weak self] in
      while let self, !Task.isCancelled {
          await withCheckedContinuation { continuation in
              withObservationTracking {
                  _ = client.isStreaming
                  _ = client.error
                  _ = client.connectionState
                  _ = client.messages
              } onChange: {
                  continuation.resume()
              }
          }
          // Process state changes...
      }
  }

No Combine. No NotificationCenter for state sync. Pure @Observable with explicit observation tracking. This is the modern SwiftUI pattern for bridging between a service object and a view model.


WHAT I LEARNED

Building a native client for an AI service is fundamentally different from building a REST API client. The streaming nature changes everything. Error handling is not "check the status code." It's "what happens when byte 47,000 of a 50,000-byte response drops silently?" Timeouts are not one number. They are three numbers for three different failure modes.

The five-layer architecture is not what I would have designed on a whiteboard. It's what emerged from constraints: Claude Code doesn't have a public streaming API (the CLI is the interface), the CLI has nesting detection (so you need env var stripping), Python buffers stdout (so you need explicit flushing), SSE connections go stale (so you need heartbeat monitoring), and iOS goes to background (so you need to cancel the radio).

Every layer exists because a real bug in a real system demanded it.

The companion repo is a standalone Swift package. Add it to your project, configure the BridgeConfiguration, instantiate an SSEClient, and call startStream(). The five layers of complexity are encapsulated. Your SwiftUI view just observes messages.

GitHub: https://github.com/nickbaumann98/claude-ios-streaming-bridge

#AgenticDevelopment #SwiftUI #ClaudeCode #SSEStreaming #iOSDevelopment


---

================================================================================
POST 5: 5 LAYERS TO CALL AN API
Agentic Development: 10 Lessons from 8,481 AI Coding Sessions (Part 5)
================================================================================

I needed to call one API. It took five layers, four failed attempts, and thirty hours of debugging to get there.

This is the story of connecting an iOS app to Claude Code -- a problem that sounds like it should take an afternoon and instead became a two-week debugging odyssey. The working solution has five layers of indirection between the user tapping "Send" and Claude receiving the prompt. Every layer exists because I tried to remove it and failed.

The companion repo has all four failed attempts with real code and the working bridge: github.com/krzemienski/claude-sdk-bridge


THE OBVIOUS APPROACH (AND WHY IT DIES IMMEDIATELY)

When you want to call an API from an app, the obvious architecture is: app calls API. Three lines of meaningful code. Here is what that looks like:

  # From: failed-attempts/01-direct-api/attempt.py

  client = anthropic.Anthropic()

  with client.messages.stream(
      model="claude-sonnet-4-20250514",
      max_tokens=4096,
      messages=[{"role": "user", "content": prompt}],
  ) as stream:
      for text in stream.text_stream:
          collected_text += text
          print(text, end="", flush=True)

Clean. Elegant. Dead on arrival.

The error is immediate and clear:

  anthropic.AuthenticationError: No API key provided.
  Set ANTHROPIC_API_KEY environment variable or pass api_key parameter.

Claude Code does not use API keys. It uses browser-based OAuth authentication. The session tokens are managed internally by the CLI and stored in ~/.claude/. They are not exposed through environment variables or any public API.

You could ask users to create a separate API key through the Anthropic Console. But that defeats the purpose of building a client for Claude Code -- users would need separate billing, lose access to Claude Code features like tools and MCP servers, and pay twice. Half a day lost. The least painful failure on the list, because at least the error message was clear.


THE SILENT FAILURE (TWO DAYS, ZERO ERROR MESSAGES)

Attempt 2 was the one that nearly broke me. Anthropic ships a Swift SDK -- ClaudeCodeSDK -- designed for exactly this scenario. Same language as our backend. Same ecosystem. It wraps the Claude CLI process and provides a publisher-based streaming interface. It should have worked.

  // From: failed-attempts/02-claude-code-sdk/attempt.swift

  func attemptClaudeCodeSDK(req: Request) async throws -> Response {
      let claude = ClaudeCodeProcess()
      claude.arguments = ["-p", "--output-format", "stream-json"]

      let stream = claude.stream(prompt: "Say hello")

      var response = ""
      for try await chunk in stream {
          response += chunk  // Never reached
      }

      return Response(status: .ok, body: .init(string: response))
  }

Nothing happened. No errors. No crashes. No output. The AsyncStream returned by the SDK simply never yielded a single value. The loop runs forever, waiting for data that will never arrive.

Here is what is happening under the hood. The SDK internally uses FileHandle.readabilityHandler to read stdout from the Claude CLI subprocess. That handler dispatches through RunLoop. The events flow through a Combine PassthroughSubject, which also depends on RunLoop scheduling. But our backend is Vapor, which runs on SwiftNIO. SwiftNIO uses its own EventLoop implementation -- not RunLoop. The NIO event loops never pump RunLoop. So:

1. The Claude CLI process spawns correctly
2. Claude receives the prompt and generates a response
3. Bytes arrive on stdout
4. readabilityHandler fires and reads the data
5. PassthroughSubject.send() is called with the data
6. The subscriber callback never executes -- no RunLoop iteration delivers the event

The data exists. It was read. It was sent into the publisher. And then it vanishes into a scheduling void.

[DIAGRAM: Two parallel event loops (RunLoop and NIO EventLoop) on the same thread, showing data entering RunLoop's pipeline and never crossing to NIO's subscriber. Labeled "The Impedance Mismatch"]

I tried three workarounds. Manually pumping RunLoop on a background thread -- deadlock. DispatchQueue.main.async wrapper -- moved the silent failure to a different layer. Dedicated Thread with its own RunLoop -- event ordering issues and more deadlocks. None of them worked because this is an architectural incompatibility, not a bug.

Two full days. Most of that time was spent verifying the SDK was receiving data -- adding logging at every layer, confirming PassthroughSubject.send() was being called, slowly narrowing the gap to the subscriber side. The silence was the hardest part. If the SDK had thrown an error or logged a warning, this would have been a thirty-minute fix.


THE WRONG LANGUAGE

Attempt 3 was the JavaScript SDK. Same authentication wall as Attempt 1 (OAuth, not API keys), plus the unnecessary overhead of adding a Node.js runtime to a Swift backend. Even the Node.js Agent SDK has environment variable inheritance issues when spawned from within a Claude Code session.

  // From: failed-attempts/03-js-sdk/attempt.js

  const client = new Anthropic();

  const stream = await client.messages.stream({
      model: "claude-sonnet-4-20250514",
      max_tokens: 4096,
      messages: [{ role: "user", content: prompt }],
  });

Same error. Different language. A few hours wasted before recognizing the pattern and moving on.


THREE LINES, TEN HOURS

Attempt 4 got tantalizingly close. Bypass all SDKs. Spawn the Claude CLI directly as a Process (formerly NSTask) with GCD-based stdout reading. No RunLoop dependency, no Combine, just DispatchQueue handlers on a Pipe.

It worked perfectly from a standalone terminal. It failed silently when running inside an active Claude Code session.

  // From: failed-attempts/04-cli-subprocess/attempt.swift

  func executeClaudeDirectly(prompt: String) async throws -> String {
      let process = Process()
      process.executableURL = URL(fileURLWithPath: "/usr/local/bin/claude")
      process.arguments = ["-p", prompt, "--output-format", "stream-json"]

      let stdoutPipe = Pipe()
      process.standardOutput = stdoutPipe

      // BUG: This inherits ALL environment variables from the parent process,
      // including CLAUDECODE=1 and CLAUDE_CODE_* if running inside Claude Code.
      // The child Claude CLI detects these and silently refuses to execute.

      try process.run()
      process.waitUntilExit()
      // ...
  }

The symptom: Claude CLI exits immediately with no output. No error, no stderr, just a zero-byte response. The cause: environment variable inheritance. Claude Code sets CLAUDECODE=1 and CLAUDE_CODE_* variables in its process environment. Child processes inherit these. The CLI's nesting detection mechanism silently refuses to execute when it detects a parent Claude Code session.

The fix was three lines:

  // From: failed-attempts/04-cli-subprocess/attempt.swift

  var env = ProcessInfo.processInfo.environment
  env.removeValue(forKey: "CLAUDECODE")
  env = env.filter { !$0.key.hasPrefix("CLAUDE_CODE_") }
  process.environment = env

Three lines. Ten hours to discover them. The environment-dependent failure -- works in terminal, fails in Claude Code -- made it extremely hard to diagnose. Environment variables are ambient authority. The subprocess did not ask to be inside a Claude Code session. It inherited that context silently and failed silently.

And there was a bonus bug: accessing process.terminationStatus after reading EOF from stdout caused an NSInvalidArgumentException crash. Reading EOF from a pipe does NOT mean the process has exited. There is a race condition between pipe closure and process termination.


THE WORKING BRIDGE: WHY FIVE LAYERS IS SIMPLER

The working solution has five layers. Here is the full path a user's message travels:

  Layer 1: iOS App (SwiftUI)
      |  HTTP POST /api/v1/chat/stream
      v
  Layer 2: Vapor Backend (Swift/NIO)
      |  Process() spawn, sanitized env
      v
  Layer 3: Python sdk-wrapper (bridge.py)
      |  claude_agent_sdk.query() async iterator
      v
  Layer 4: Claude CLI
      |  OAuth-authenticated HTTP
      v
  Layer 5: Anthropic API

[DIAGRAM: Five vertical layers connected by arrows, with serialization format labels on each arrow: "HTTP/SSE", "Process + GCD (NDJSON on stdout)", "SDK async iter", "OAuth HTTP". Six serialization boundaries highlighted.]

Each layer exists because the layer above cannot talk to the layer below without an intermediary. Layer 1 to 2 is standard HTTP. Layer 2 to 3 exists because Swift cannot use the Swift SDK (RunLoop/NIO mismatch) and cannot call the API directly (OAuth, not API keys). Layer 3 is Python because the claude-agent-sdk package wraps the CLI natively and inherits OAuth authentication. Layer 4 is the CLI because it is the only consumer-accessible interface that handles the OAuth token chain. Layer 5 was never a problem.

The Python bridge itself is around 50 lines of meaningful code. The core is the isinstance() dispatch that converts SDK message types to NDJSON:

  # From: working-bridge/claude_bridge.py

  async for message in query(prompt=prompt, options=options):
      if isinstance(message, SystemMessage):
          pass

      elif isinstance(message, AssistantMessage):
          blocks = [convert_block(b) for b in message.content]
          if blocks:
              got_content = True
          emit({"type": "assistant", "message": {"role": "assistant",
               "content": blocks, "model": getattr(message, "model", None)}})

      elif isinstance(message, ResultMessage):
          got_result = True
          result = {
              "type": "result",
              "subtype": "error" if message.is_error else "success",
              "is_error": message.is_error,
              "session_id": message.session_id or session_id,
              "total_cost_usd": message.total_cost_usd or 0.0,
          }
          emit(result)

The block converter uses isinstance() for each content type -- TextBlock, ToolUseBlock, ToolResultBlock, ThinkingBlock:

  # From: working-bridge/claude_bridge.py

  def convert_block(block) -> dict:
      if isinstance(block, TextBlock):
          return {"type": "text", "text": block.text}
      elif isinstance(block, ToolUseBlock):
          return {
              "type": "tool_use",
              "id": block.id,
              "name": block.name,
              "input": block.input,
          }
      elif isinstance(block, ToolResultBlock):
          return {
              "type": "tool_result",
              "tool_use_id": block.tool_use_id,
              "content": block.content,
              "is_error": block.is_error,
          }
      elif isinstance(block, ThinkingBlock):
          return {"type": "thinking", "thinking": block.thinking}
      else:
          text = getattr(block, "text", None) or str(block)
          return {"type": "text", "text": text}

Every write flushes immediately. Without explicit flushing, Python's stdout buffering delays output by unpredictable amounts when writing to a pipe:

  # From: working-bridge/claude_bridge.py

  def emit(obj: dict) -> None:
      line = json.dumps(obj, separators=(",", ":"))
      sys.stdout.write(line + "\n")
      sys.stdout.flush()  # Critical: force immediate delivery


THE SWIFT SIDE: ENVIRONMENT SANITIZATION AND NDJSON PARSING

The Swift executor spawns the Python bridge, strips the dangerous environment variables, and reads NDJSON from stdout on a dedicated GCD queue:

  // From: working-bridge/executor.swift

  // CRITICAL: Strip Claude Code nesting detection env vars.
  let escaped = configJson.replacingOccurrences(of: "'", with: "'\\''")
  let command = "python3 '\(bridgePath)' '\(escaped)'"
  let cleanCmd = "for v in $(env | grep ^CLAUDE | cut -d= -f1); do unset $v; done; \(command)"
  process.arguments = ["-l", "-c", cleanCmd]

  // Belt-and-suspenders: also strip from Process.environment
  var env = ProcessInfo.processInfo.environment
  for key in env.keys where key.hasPrefix("CLAUDE") {
      env.removeValue(forKey: key)
  }
  process.environment = env

Belt-and-suspenders: the environment is stripped both in the shell command AND in the Process.environment dictionary. Two redundant protections for a bug that took ten hours to find the first time.

The NDJSON reader runs on a dedicated GCD queue with no RunLoop dependency. It handles partial lines, buffer accumulation, and the critical waitUntilExit() pattern:

  // From: working-bridge/executor.swift

  DispatchQueue(label: "bridge-reader", qos: .userInitiated).async {
      let handle = stdoutPipe.fileHandleForReading
      var buffer = Data()

      while true {
          let chunk = handle.availableData
          if chunk.isEmpty { break }

          initialTimeoutWork.cancel()  // Got data, cancel initial timeout
          buffer.append(chunk)

          guard let str = String(data: buffer, encoding: .utf8) else { continue }
          let lines = str.components(separatedBy: "\n")

          if lines.count > 1 {
              for i in 0..<(lines.count - 1) {
                  let line = lines[i].trimmingCharacters(in: .whitespacesAndNewlines)
                  if !line.isEmpty,
                     let data = line.data(using: .utf8),
                     let json = try? JSONSerialization.jsonObject(with: data)
                         as? [String: Any] {
                      continuation.yield(json)
                  }
              }
              buffer = lines.last?.data(using: .utf8) ?? Data()
          }
      }

      // CRITICAL: Always waitUntilExit() before terminationStatus.
      process.waitUntilExit()
      continuation.finish()
  }


THE COUNTERINTUITIVE LESSON

Five layers sounds overengineered. It sounds like the kind of architecture an astronaut builds when a direct call would suffice. But here is the thing: every "simpler" approach failed. The direct API call (1 layer) hits an authentication wall. The Swift SDK (2 layers) hits a RunLoop/NIO mismatch. The CLI subprocess (2 layers) hits nesting detection. The JS SDK (2 layers) hits authentication plus adds an unnecessary runtime.

Five layers works because each layer does exactly one translation:

1. SwiftUI gesture to HTTP request
2. HTTP request to subprocess spawn (with env sanitization)
3. Python SDK call to NDJSON on stdout
4. OAuth-authenticated CLI call
5. HTTP to Anthropic's servers

No layer tries to be clever. No layer combines responsibilities. The bridge.py file is ~50 lines. The executor.swift is ~210 lines. The total bridge code is smaller than any of the failed attempts because each layer has a single, clear job.

Six serialization boundaries exist in the full round trip. Each one is a potential source of bugs. The text duplication P2 bug -- where every response appeared twice -- was caused by using += on accumulated text instead of = at boundary 5. Two characters. Three hours of debugging. But the serialization boundary made it easy to isolate once I knew where to look.

Total debugging time across all failure modes: approximately thirty hours. Cold start latency: ~12 seconds. Warm latency: ~2-3 seconds. Cost per query: ~$0.04. The bridge has been running in production for months without a single failure mode it was not designed to handle.

Sometimes the "wrong" architecture is the only one that works.

Companion repo: github.com/krzemienski/claude-sdk-bridge

#AgenticDevelopment #ClaudeCode #iOSDevelopment #SoftwareArchitecture #AIEngineering


================================================================================
POST 6: 194 PARALLEL AI WORKTREES
Agentic Development: 10 Lessons from 8,481 AI Coding Sessions (Part 6)
================================================================================

I gave an AI 194 tasks, 194 isolated copies of a codebase, and told it to build. The execution agents were not the hard part. The QA pipeline was.

This is the story of auto-claude-worktrees -- a system that uses git worktrees to spin up dozens of parallel AI agents, each working in complete filesystem isolation, then runs independent QA agents to decide what ships and what gets sent back for fixes. The numbers from one real project: 194 tasks ideated, 91 specs generated, 71 QA reports produced, 3,066 sessions consumed, 470 MB of conversation data.

The companion repo has the full pipeline: github.com/krzemienski/auto-claude-worktrees


THE PROBLEM WITH PARALLEL AI DEVELOPMENT

When you ask one AI agent to build a large feature, it works sequentially through a long chain of changes. If it makes a mistake in step 3, steps 4 through 20 compound the error. If you ask it to fix step 3, it often breaks step 7 in the process.

The obvious answer is parallelism. Break the work into independent tasks, run them simultaneously. But naive parallelism hits an immediate wall: merge conflicts. If agent A modifies utils.py at line 40 and agent B modifies utils.py at line 50, you have a conflict that neither agent anticipated.

Git worktrees solve this cleanly. Each worktree is a full copy of the repository on a separate branch. No shared filesystem state. No cross-contamination. The conflicts only surface at merge time, when you can handle them systematically instead of reactively.

  auto-claude full --repo ./my-project --workers 8

That single command runs five pipeline stages that took me months to get right.

[DIAGRAM: Five-stage pipeline as horizontal boxes connected by arrows: "Ideate (194 tasks)" -> "Spec Gen (91 specs)" -> "Worktree Factory (71 worktrees)" -> "QA Review (71 reports)" -> "Merge Queue (~60 merged)". Below each box, the key operation: "Claude analyzes codebase", "Task -> acceptance criteria", "git worktree add + agent spawn", "Independent reviewer vs spec", "Priority-weighted toposort".]


STAGE 1: IDEATION -- OVER-GENERATE DELIBERATELY

The ideation phase uses Claude to analyze the entire codebase and generate a task manifest. The data model for each task:

  # From: src/auto_claude/models.py

  class Task(BaseModel):
      id: str = Field(description="Unique task identifier, e.g. 'modularization', 'reduce-any-types'")
      title: str = Field(description="Human-readable task title")
      description: str = Field(description="Detailed description of what needs to be done")
      scope: list[str] = Field(
          default_factory=list,
          description="Files and modules affected by this task",
      )
      dependencies: list[str] = Field(
          default_factory=list,
          description="Task IDs that must complete before this task",
      )
      priority: TaskPriority = Field(default=TaskPriority.MEDIUM)
      status: TaskStatus = Field(default=TaskStatus.IDEATED)
      tags: list[str] = Field(default_factory=list, description="Categorization tags")

Two methods on the Task model enable the pipeline's dependency and conflict logic:

  # From: src/auto_claude/models.py

  def is_blocked(self, completed_task_ids: set[str]) -> bool:
      """Check if this task is blocked by incomplete dependencies."""
      return bool(set(self.dependencies) - completed_task_ids)

  def has_scope_overlap(self, other: Task) -> bool:
      """Check if two tasks modify overlapping files."""
      return bool(set(self.scope) & set(other.scope))

The priority enum determines merge order later:

  # From: src/auto_claude/models.py

  class TaskPriority(str, enum.Enum):
      CRITICAL = "critical"
      HIGH = "high"
      MEDIUM = "medium"
      LOW = "low"

A critical design decision: the ideation phase over-generates deliberately. Generating 194 task descriptions costs a fraction of executing one. The QA pipeline downstream filters what should not ship. This is the opposite of how most people think about AI task planning -- they try to generate exactly the right set of tasks upfront. That precision is expensive and fragile. Casting a wide net and filtering is cheaper and more robust.


STAGE 2: SPECS -- THE REAL BOTTLENECK

Raw task descriptions are not enough for autonomous agents. "Improve error handling" is too vague. The spec generator produces detailed implementation blueprints:

  # From: src/auto_claude/models.py

  class Spec(BaseModel):
      task_id: str = Field(description="ID of the source task")
      objective: str = Field(description="Single-sentence end-state description")
      files_in_scope: list[str] = Field(
          description="Explicit list of files to create, modify, or delete"
      )
      implementation_steps: list[str] = Field(
          description="Ordered sequence of changes to make"
      )
      acceptance_criteria: list[str] = Field(
          description="Concrete, verifiable conditions for completion"
      )
      risk_notes: list[str] = Field(
          default_factory=list,
          description="Known pitfalls, edge cases, or compatibility concerns",
      )

The acceptance_criteria field is the most important. It is the contract between the execution agent and the QA agent. Every criterion must be concrete and verifiable -- not "code is clean" but "all public functions have docstrings" or "error paths return structured error responses."

Each spec gets formatted into a prompt context that the execution agent can reference:

  # From: src/auto_claude/models.py

  def to_prompt_context(self) -> str:
      lines = [
          f"# Task: {self.task_id}",
          f"\n## Objective\n{self.objective}",
          "\n## Files in Scope",
      ]
      for f in self.files_in_scope:
          lines.append(f"- {f}")
      lines.append("\n## Implementation Steps")
      for i, step in enumerate(self.implementation_steps, 1):
          lines.append(f"{i}. {step}")
      lines.append("\n## Acceptance Criteria")
      for criterion in self.acceptance_criteria:
          lines.append(f"- [ ] {criterion}")
      return "\n".join(lines)

Specs are the bottleneck, not execution. A precise spec passes QA on the first attempt. A vague spec produces work that gets rejected, fixed, re-reviewed, and sometimes rejected again. The ~22% first-pass rejection rate in our data correlates directly with spec precision -- tasks with five or more concrete acceptance criteria had a significantly lower rejection rate than tasks with two or three vague ones.


STAGE 3: THE WORKTREE FACTORY

This is the core of the system. For each spec, the factory creates a git worktree, injects the spec, and spawns a Claude agent:

  # From: src/auto_claude/factory.py

  def create_worktree(
      repo_path: Path,
      branch_name: str,
      base_dir: Path,
  ) -> Path:
      base_dir.mkdir(parents=True, exist_ok=True)
      worktree_path = base_dir / branch_name.replace("/", "-")

      # Remove existing worktree if it exists (from a previous run)
      if worktree_path.exists():
          subprocess.run(
              ["git", "worktree", "remove", "--force", str(worktree_path)],
              cwd=repo_path, capture_output=True,
          )

      # Delete branch if it exists (from a previous run)
      subprocess.run(
          ["git", "branch", "-D", branch_name],
          cwd=repo_path, capture_output=True,
      )

      # Create fresh worktree with new branch
      result = subprocess.run(
          ["git", "worktree", "add", "-b", branch_name, str(worktree_path)],
          cwd=repo_path, capture_output=True, text=True,
      )

      if result.returncode != 0:
          raise RuntimeError(
              f"Failed to create worktree '{branch_name}': {result.stderr.strip()}"
          )

      return worktree_path

After creating the worktree, the spec gets injected as a context file -- both human-readable markdown and machine-readable JSON:

  # From: src/auto_claude/factory.py

  def inject_spec(worktree_path: Path, spec: Spec) -> Path:
      spec_dir = worktree_path / ".auto-claude"
      spec_dir.mkdir(exist_ok=True)

      spec_md = spec_dir / "spec.md"
      spec_md.write_text(spec.to_prompt_context())

      spec_json = spec_dir / "spec.json"
      with open(spec_json, "w") as f:
          json.dump(spec.model_dump(mode="json"), f, indent=2, default=str)

      return spec_md

Then a Claude agent is spawned, scoped entirely to that worktree's filesystem:

  # From: src/auto_claude/factory.py

  def spawn_agent(
      worktree_path: Path,
      spec: Spec,
      config: PipelineConfig,
  ) -> subprocess.Popen[str]:
      prompt = f"""\
  Execute this implementation spec in your current working directory.

  {spec.to_prompt_context()}

  After completing all steps:
  1. Verify each acceptance criterion is met
  2. Commit all changes with descriptive messages
  3. Create a COMPLETION.md file summarizing what was done
  """

      cmd = [
          "claude", "--print",
          "--model", config.model,
          "--system-prompt", EXECUTION_SYSTEM_PROMPT,
          prompt,
      ]

      process = subprocess.Popen(
          cmd,
          cwd=worktree_path,
          stdout=subprocess.PIPE,
          stderr=subprocess.PIPE,
          text=True,
      )

      return process

The parallel execution uses ThreadPoolExecutor. The config controls concurrency:

  # From: src/auto_claude/factory.py

  def run_factory(specs, repo_path, config):
      # ...
      with ThreadPoolExecutor(max_workers=config.max_parallel_workers) as executor:
          future_to_spec = {
              executor.submit(
                  execute_in_worktree, spec, repo_path, config, base_dir,
              ): spec
              for spec in specs
          }

          for future in as_completed(future_to_spec):
              spec = future_to_spec[future]
              try:
                  state = future.result()
                  states.append(state)
              except Exception as e:
                  logger.error("Unexpected error executing '%s': %s", spec.task_id, e)

Each worktree execution is tracked through a lifecycle model:

  # From: src/auto_claude/models.py

  class TaskStatus(str, enum.Enum):
      IDEATED = "ideated"
      SPECIFIED = "specified"
      IN_PROGRESS = "in_progress"
      COMPLETED = "completed"
      QA_PENDING = "qa_pending"
      QA_APPROVED = "qa_approved"
      QA_REJECTED = "qa_rejected"
      QA_REJECTED_PERMANENT = "qa_rejected_permanent"
      MERGE_READY = "merge_ready"
      MERGE_COMPLETE = "merge_complete"
      MERGE_CONFLICT = "merge_conflict"
      FAILED = "failed"

Thirteen states. That might seem excessive until you realize each state represents a decision point where the pipeline needs to know what to do next.


STAGE 4: QA -- WHERE QUALITY ACTUALLY COMES FROM

This is the insight that took months to internalize: the QA pipeline matters more than the execution agents.

The QA system has three possible verdicts:

  # From: src/auto_claude/models.py

  class QAVerdict(str, enum.Enum):
      APPROVED = "approved"
      REJECTED_WITH_FIXES = "rejected_with_fixes"
      REJECTED_PERMANENT = "rejected_permanent"

The first principle, stated explicitly in the QA agent's system prompt:

  # From: src/auto_claude/qa.py

  QA_SYSTEM_PROMPT = """\
  You are an independent QA reviewer for an autonomous development pipeline.

  IMPORTANT: You are NOT the agent that wrote this code. You are a separate reviewer
  with fresh eyes. Do not assume anything works -- verify everything against the spec.

  Your job is to review the code changes in this worktree against the original
  implementation specification and produce a verdict:

  1. "approved" -- All acceptance criteria met, code quality acceptable, no regressions
  2. "rejected_with_fixes" -- Specific issues found, but the approach is sound. Provide
     detailed remediation instructions for the execution agent.
  3. "rejected_permanent" -- Fundamental approach is flawed. Task needs re-specification.
  """

QA agents must be separate from execution agents. Self-review does not work. The same biases that led an agent to write buggy code lead it to overlook those bugs in review. This is not a theoretical concern -- it is a pattern I observed repeatedly. An execution agent that misunderstands a requirement will confidently report that requirement as met. A separate reviewer catches the gap.

The QA context is built from the original spec, the git diff, and any completion notes:

  # From: src/auto_claude/qa.py

  def build_qa_context(state: WorktreeState) -> str:
      parts: list[str] = []

      parts.append("# Original Specification\n")
      parts.append(state.spec.to_prompt_context())

      parts.append("\n\n# Code Changes (git diff)\n")
      diff_result = subprocess.run(
          ["git", "diff", "HEAD~..HEAD", "--stat"],
          cwd=state.worktree_path, capture_output=True, text=True,
      )
      if diff_result.returncode == 0 and diff_result.stdout.strip():
          parts.append(f"```\n{diff_result.stdout.strip()}\n```\n")

      # Full diff (capped at 10000 chars to avoid token explosion)
      full_diff = subprocess.run(
          ["git", "diff", "HEAD~..HEAD"],
          cwd=state.worktree_path, capture_output=True, text=True,
      )
      if full_diff.returncode == 0 and full_diff.stdout.strip():
          diff_text = full_diff.stdout.strip()[:10000]
          parts.append(f"```diff\n{diff_text}\n```")

      return "\n".join(parts)

[DIAGRAM: Circular flow showing: "Execution Agent" -> "Completed Work" -> "QA Agent (separate)" -> decision diamond "Verdict?" with three arrows: "Approved" -> "Merge Queue", "Rejected with Fixes" -> back to "Execution Agent", "Rejected Permanent" -> "Discard". The loop from QA back to Execution is labeled "max_retries: 2".]


THE REJECTION-AND-FIX CYCLE

When QA rejects a task, the system sends it back to the execution agent with specific remediation instructions:

  # From: src/auto_claude/qa.py

  def send_back_for_fixes(state, qa_result, config):
      fix_prompt = f"""\
  Your previous implementation was reviewed and REJECTED. Fix the following issues:

  ## QA Summary
  {qa_result.summary}

  ## Failed Criteria
  {chr(10).join(f'- {c}' for c in qa_result.failed_criteria)}

  ## Specific Issues
  {chr(10).join(f'- {i}' for i in qa_result.issues)}

  ## Remediation Instructions
  {chr(10).join(f'{n+1}. {inst}' for n, inst in enumerate(qa_result.remediation_instructions))}

  Fix ALL issues listed above. Then commit your changes and update COMPLETION.md.
  """

      cmd = [
          "claude", "--print",
          "--model", config.model,
          "--system-prompt", factory.EXECUTION_SYSTEM_PROMPT,
          fix_prompt,
      ]

      result = subprocess.run(
          cmd, capture_output=True, text=True,
          cwd=state.worktree_path, timeout=config.timeout_seconds,
      )
      state.retry_count += 1
      state.session_count += 1

The full QA pipeline runs review, rejection, fix, and re-review cycles:

  # From: src/auto_claude/qa.py

  def run_qa_pipeline(states, config):
      completed = [s for s in states if s.status == TaskStatus.COMPLETED]
      all_results: list[QAResult] = []

      with ThreadPoolExecutor(max_workers=config.max_parallel_workers) as executor:
          # First pass review
          future_to_state = {
              executor.submit(review_single_task, state, config, 1): state
              for state in completed
          }

          for future in as_completed(future_to_state):
              state = future_to_state[future]
              qa_result = future.result()
              all_results.append(qa_result)

              if qa_result.is_approved():
                  state.status = TaskStatus.QA_APPROVED
              elif qa_result.is_permanently_rejected():
                  state.status = TaskStatus.QA_REJECTED_PERMANENT
              else:
                  state.status = TaskStatus.QA_REJECTED

      # Fix-and-retry cycles for rejected tasks
      for retry in range(config.max_retries):
          rejected = [s for s in completed if s.status == TaskStatus.QA_REJECTED]
          if not rejected:
              break

          for state in rejected:
              task_results = [r for r in all_results if r.task_id == state.task_id]
              last_rejection = task_results[-1]

              state = send_back_for_fixes(state, last_rejection, config)

              if state.status == TaskStatus.QA_PENDING:
                  qa_result = review_single_task(state, config, retry + 2)
                  all_results.append(qa_result)

The numbers tell the story. First-pass rejection rate: ~22%. Second-pass approval rate: ~95%. The rejection-and-fix cycle is where quality comes from. Without QA, the execution agents produce code that looks correct but has subtle gaps -- missed edge cases, incomplete error handling, hardcoded values that should be configurable. The QA agent catches these because it reviews against the spec's acceptance criteria with fresh eyes.


STAGE 5: MERGE QUEUE -- PRIORITY-WEIGHTED TOPOLOGICAL SORT

The merge queue is the final piece. Not all approved branches can merge cleanly -- parallel development means potential conflicts. The merge order matters:

  # From: src/auto_claude/merge.py

  def compute_merge_order(tasks, approved_ids):
      approved_tasks = [t for t in tasks if t.id in approved_ids]

      sorted_ids: list[str] = []
      visited: set[str] = set()
      in_progress: set[str] = set()

      task_map = {t.id: t for t in approved_tasks}

      def visit(task_id: str) -> None:
          if task_id in visited:
              return
          if task_id in in_progress:
              logger.warning("Circular dependency detected involving '%s'", task_id)
              return

          in_progress.add(task_id)
          task = task_map[task_id]

          for dep in task.dependencies:
              if dep in approved_ids:
                  visit(dep)

          in_progress.discard(task_id)
          visited.add(task_id)
          sorted_ids.append(task_id)

      for task in sorted(
          approved_tasks,
          key=lambda t: (priority_order[t.priority], len(t.scope), t.id),
      ):
          visit(task.id)

      return sorted_ids

Foundation tasks merge first. Small, focused tasks before broad refactors. Dependencies respected. Within the same priority level, tasks with narrower scope merge before wider ones -- the intuition being that narrow changes are less likely to conflict with later merges.

Before each merge, a dry-run conflict check prevents partial merges:

  # From: src/auto_claude/merge.py

  def check_conflicts(repo_path, branch_name):
      result = subprocess.run(
          ["git", "merge", "--no-commit", "--no-ff", branch_name],
          cwd=repo_path, capture_output=True, text=True,
      )

      conflicts: list[str] = []

      if result.returncode != 0:
          status = subprocess.run(
              ["git", "diff", "--name-only", "--diff-filter=U"],
              cwd=repo_path, capture_output=True, text=True,
          )
          if status.stdout.strip():
              conflicts = status.stdout.strip().split("\n")

      subprocess.run(
          ["git", "merge", "--abort"],
          cwd=repo_path, capture_output=True,
      )

      return conflicts

Tasks with conflicts get flagged for re-execution against the updated main. They do not just fail -- they re-enter the pipeline with fresh context.


THE CONFIGURATION

The whole pipeline is configurable through a TOML file:

  # From: config/default.toml

  [pipeline]
  max_parallel_workers = 4
  model = "sonnet"
  qa_model = "opus"
  timeout_seconds = 600
  max_retries = 2

  [qa]
  criteria = [
      "All acceptance criteria from the spec are met",
      "No regressions introduced in existing functionality",
      "Code quality meets project standards (formatting, naming, structure)",
      "No hardcoded values that should be configurable",
      "Error handling is present for failure paths",
  ]

  [ideation]
  model = "opus"
  max_tasks = 200

  [merge]
  strategy = "priority-weighted"
  auto_resolve_conflicts = false

Notice the model split: opus for ideation and QA (where reasoning quality matters most), sonnet for execution (where speed and volume matter more). This is not arbitrary -- ideation and QA are judgment tasks. Execution is mostly mechanical once the spec is precise.


THE NUMBERS AND WHAT THEY MEAN

From the Awesome List project:

  Tasks ideated:                 194
  Specs generated:                91
  QA reports produced:            71
  Git branches created:           90
  Total sessions:              3,066
  Conversation data:           470 MB
  QA first-pass rejection rate:  ~22%
  QA second-pass approval rate:  ~95%

194 tasks ideated, 91 specs generated. The gap is intentional -- some tasks are blocked by dependencies, some are deferred because spec generation fails, some overlap and get merged. Over-generation at the ideation phase feeds the funnel without waste because ideation is cheap.

The 22% first-pass rejection rate is the most important number. It means more than one in five execution attempts produce work that a separate reviewer deems insufficient. Without the QA pipeline, that 22% would have shipped as-is. Some of those rejections were missing edge cases. Some were incomplete implementations. Some were hardcoded values. All of them were things the execution agent thought were correct.

The 95% second-pass approval rate validates the fix cycle. When an agent gets specific, actionable feedback about what failed and why, it almost always fixes the issues. The rejection is not "try again" -- it is structured remediation with failed criteria, specific issues, and step-by-step fix instructions.

3,066 sessions across 71 tasks means an average of ~43 sessions per task. Session count correlates with task breadth, not difficulty. Wide tasks that touch many files consume more sessions than complex but narrow ones. This is useful for capacity planning -- if you know a task touches 15 files, budget more sessions than a task that deeply refactors 3 files.


FOUR LESSONS FROM THE FACTORY FLOOR

First: isolation is non-negotiable. Git worktrees provide free, total filesystem separation. No shared state. No "oops, agent A overwrote agent B's changes." The isolation is not a nice-to-have -- it is the foundation that makes everything else possible. Without it, parallel AI development devolves into merge conflict whack-a-mole.

Second: specs are the bottleneck, not execution. A precise spec with five concrete acceptance criteria passes QA on the first attempt. A vague spec with two criteria gets rejected, fixed, re-reviewed, and sometimes re-rejected. The spec generation phase determines the quality of everything downstream. Investing in better specs has a higher return than investing in smarter execution agents.

Third: QA agents must be separate from execution agents. Self-review does not work at scale. The same reasoning patterns that produce buggy code produce confident-but-incorrect self-assessments. The independent reviewer catches what the author cannot see.

Fourth: the rejection-and-fix cycle is where quality comes from. The goal is not first-pass perfection. The goal is a tight feedback loop: execute, review, reject with specifics, fix, re-review. The 22% first-pass rejection rate is not a failure -- it is the system working as designed. The 95% second-pass approval rate proves the cycle works.

Companion repo: github.com/krzemienski/auto-claude-worktrees

#AgenticDevelopment #GitWorktrees #AIEngineering #ParallelDevelopment #QualityAssurance


---

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


---

================================================================================
POST 9 OF 10: FROM GITHUB REPOS TO AUDIO STORIES
Agentic Development: 10 Lessons from 8,481 AI Coding Sessions
================================================================================

What if you could listen to a codebase?

Not read documentation. Not scan source files. Listen -- like a podcast episode about FastAPI's architecture, or a nature documentary narrating how Django evolved in the wild.

That is exactly what code-tales does. Give it a GitHub URL, pick one of 9 narrative styles, and in under two minutes you get a fully synthesized audio story about the repository. Clone, analyze, narrate with Claude, synthesize with ElevenLabs. Four pipeline stages. One command.

  code-tales generate --repo https://github.com/tiangolo/fastapi --style documentary

This is post 9 of 10 in the Agentic Development series. The companion repo is at github.com/krzemienski/code-tales. Everything I quote here is real code you can read, run, and extend.


THE PIPELINE ARCHITECTURE

The core abstraction is a four-stage pipeline orchestrated by a single class. Here is the actual orchestrator:

  # From: src/code_tales/pipeline/orchestrate.py

  class CodeTalesPipeline:
      """Orchestrates the full code-tales pipeline.

      Stages:
        1. Resolve repository (clone URL or validate local path)
        2. Analyze repository structure and metadata
        3. Generate narration script via Claude
        4. Synthesize audio via ElevenLabs (or save text only)
      """

      def generate(
          self,
          repo_url_or_path: str,
          style_name: str,
          output_dir: Optional[Path] = None,
      ) -> AudioOutput:

The generate method runs all four stages sequentially with Rich progress bars showing real-time status. The key design decision: every stage is independently importable. You can call analyze_repository without cloning, or generate_script without synthesizing audio. The pipeline composes; it does not mandate.

[DIAGRAM: Four-stage pipeline flow -- Input (GitHub URL or local path) -> Clone (gitpython shallow depth=1) -> Analyze (languages, dependencies, frameworks, patterns) -> Narrate (Claude streaming API) -> Synthesize (ElevenLabs TTS) -> Output (script.md + story.mp3)]

The pipeline resolution logic handles both remote URLs and local paths:

  # From: src/code_tales/pipeline/orchestrate.py

  def _resolve_repo(self, repo_url_or_path: str) -> tuple[Path, Optional[Path]]:
      if repo_url_or_path.startswith(("http://", "https://", "git@")):
          temp_dir = Path(tempfile.mkdtemp(prefix="code-tales-"))
          repo_path = clone_repository(
              url=repo_url_or_path,
              target_dir=temp_dir,
              depth=self.config.clone_depth,
          )
          return repo_path, temp_dir

      # Local path
      local = Path(repo_url_or_path).expanduser().resolve()
      if not (local / ".git").exists():
          raise ValueError(
              f"Local path is not a git repository (no .git dir): {local}"
          )
      return local, None

Shallow clones (depth=1) are the default. We need the file tree, not the full commit history. This keeps clone times under a few seconds for most repositories.


THE ANALYSIS ENGINE

Before Claude ever sees a repository, the analysis stage extracts structured metadata. This is where code-tales distinguishes itself from "just paste the README into Claude." The analyzer detects languages by file extension counts, extracts dependencies from 8 different package manager formats, identifies frameworks, and recognizes architectural patterns.

The dependency extraction alone covers package.json, requirements.txt, pyproject.toml, Cargo.toml, go.mod, Gemfile, Package.swift, and pom.xml:

  # From: src/code_tales/pipeline/analyze.py

  def _extract_dependencies(repo_path: Path) -> list[Dependency]:
      deps: list[Dependency] = []

      # package.json (npm/yarn)
      pkg_json = repo_path / "package.json"
      if pkg_json.exists():
          data = json.loads(pkg_json.read_text(encoding="utf-8"))
          for section in ("dependencies", "devDependencies", "peerDependencies"):
              for name, version in (data.get(section) or {}).items():
                  deps.append(
                      Dependency(name=name, version=str(version), source="package.json")
                  )

      # ... repeated for requirements.txt, pyproject.toml, Cargo.toml,
      #     go.mod, Gemfile, Package.swift, pom.xml

The pattern detection identifies monorepos, microservices, REST APIs, GraphQL schemas, CLI tools, web applications, mobile apps, MVC architectures, test coverage, CI/CD pipelines, and containerization. All heuristic-based, all derived from directory structure and file presence -- no LLM calls required.

  # From: src/code_tales/pipeline/analyze.py

  def _detect_patterns(repo_path: Path) -> list[str]:
      patterns: list[str] = []

      # Monorepo detection
      workspace_files = [
          repo_path / "pnpm-workspace.yaml",
          repo_path / "lerna.json",
          repo_path / "nx.json",
          repo_path / "rush.json",
      ]
      if any(f.exists() for f in workspace_files):
          patterns.append("Monorepo")

This matters because the structured analysis gives Claude specific, quantified data to work with -- not just "here's a bunch of source code, figure it out." The prompt includes file counts, language percentages, dependency lists, detected frameworks, and architectural patterns. Claude narrates from data, not from guesswork.

All of this feeds into a Pydantic model that constrains the data shape throughout:

  # From: src/code_tales/models.py

  class RepoAnalysis(BaseModel):
      name: str
      description: str = ""
      languages: list[Language] = Field(default_factory=list)
      dependencies: list[Dependency] = Field(default_factory=list)
      file_tree: str = ""
      key_files: list[FileInfo] = Field(default_factory=list)
      total_files: int = 0
      total_lines: int = 0
      frameworks: list[str] = Field(default_factory=list)
      patterns: list[str] = Field(default_factory=list)
      readme_content: Optional[str] = None


NINE STYLES, NINE DIFFERENT EXPERIENCES

This is the feature that makes code-tales more than a utility. Each of the 9 narrative styles is defined as a YAML configuration with its own tone directive, section structure template, ElevenLabs voice ID, and voice synthesis parameters. The same repository analyzed in "documentary" style and "fiction" style produces fundamentally different audio experiences.

Here is the documentary style -- think David Attenborough observing code as a living ecosystem:

  # From: src/code_tales/styles/documentary.yaml

  name: documentary
  description: A nature documentary-style narration that observes the codebase
    as a living ecosystem, in the tradition of David Attenborough.
  tone: Measured, observational, and quietly awestruck. Speak as if witnessing
    remarkable natural phenomena. Use present tense for immediacy.
  voice_id: "onwK4e9ZLuTAKqWW03F9"
  voice_params:
    stability: 0.7
    similarity_boost: 0.8
    style: 0.3
  example_opener: |
    Here we observe a remarkable specimen of software engineering,
    quietly going about its work in the digital wilderness...

Now compare that to the fiction style -- literary drama with characters and conflict:

  # From: src/code_tales/styles/fiction.yaml

  name: fiction
  description: A literary fiction narrative that transforms code into a
    dramatic story with characters, conflict, and emotional arc.
  tone: Lyrical, evocative, and dramatic. Write as if describing a novel's
    setting and characters. Use vivid metaphors and literary devices.
  voice_id: "EXAVITQu4vr4xnSDxMaL"
  voice_params:
    stability: 0.3
    similarity_boost: 0.7
    style: 0.8
  example_opener: |
    In the vast digital landscape, there existed a repository that dared
    to dream in a language most could not speak...

Notice the voice parameter differences. Documentary uses high stability (0.7) and low style (0.3) -- controlled, authoritative delivery. Fiction drops stability to 0.3 and cranks style to 0.8 -- more expressive, more emotional range. These are not arbitrary numbers. They map directly to ElevenLabs' voice synthesis parameters that control how much freedom the TTS model has in its delivery.

The full roster: debate (structured argument for and against design decisions), documentary, executive (crisp leadership briefing), fiction, interview (Q&A with an expert), podcast (casual tech talk with hot takes), storytelling (hero's journey), technical (dense engineering review), and tutorial (educational walkthrough).

The podcast style captures a completely different register:

  # From: src/code_tales/styles/podcast.yaml

  name: podcast
  tone: Casual, enthusiastic, and opinionated. Sound like you've actually
    used this project and have genuine thoughts. Use filler phrases
    naturally. Share hot takes.
  voice_params:
    stability: 0.5
    similarity_boost: 0.7
    style: 0.6
  example_opener: |
    Hey everyone, welcome back! So I've been digging into this really
    interesting repo this week and honestly, I had to record an
    episode about it...

Each style also defines a structure_template that guides Claude's section organization. The executive style forces "Executive Summary, Key Metrics, Architecture Overview, Risk Assessment, Recommendations." The storytelling style uses "Once Upon a Time, The Call to Adventure, The Journey, The Breakthrough, The Legacy." Same data, completely different narrative architecture.


THE NARRATION ENGINE

The narration stage builds a detailed prompt from the analysis data and sends it to Claude's streaming API. The system message sets the narrative voice, and the user message provides the full repository analysis:

  # From: src/code_tales/pipeline/narrate.py

  with client.messages.stream(
      model=config.claude_model,
      max_tokens=config.max_script_tokens,
      system=system_message,
      messages=[{"role": "user", "content": prompt}],
  ) as stream:
      raw_text = stream.get_final_message().content[0].text

Streaming prevents HTTP timeouts on long scripts. The system message includes explicit formatting rules for audio output -- no bullet points, no tables, natural speech patterns, contractions, flowing sentences. Critical for TTS quality: the text Claude generates will be spoken aloud, not read on a screen.

The parser then splits Claude's response on markdown headings, extracts voice directions from bracketed text (like "[speak slowly]" or "[with enthusiasm]"), and strips formatting artifacts:

  # From: src/code_tales/pipeline/narrate.py

  def _clean_content(content: str) -> str:
      # Remove bold/italic markers
      content = re.sub(r"\*\*([^*]+)\*\*", r"\1", content)
      content = re.sub(r"\*([^*]+)\*", r"\1", content)
      # Remove inline code backticks
      content = re.sub(r"`([^`]+)`", r"\1", content)
      # Remove code blocks
      content = re.sub(r"```[\s\S]*?```", "", content)
      return content.strip()

This is one of those details that seems trivial but matters enormously. If you leave markdown formatting in the text that goes to TTS, the synthesized audio sounds wrong. Asterisks create unnatural pauses. Backticks confuse the speech model. The cleaning step bridges two fundamentally different output modalities.


THE AUDIO SYNTHESIS LAYER

ElevenLabs integration includes retry logic with exponential backoff, per-style voice selection, and graceful degradation to text-only output:

  # From: src/code_tales/pipeline/synthesize.py

  payload = {
      "text": text,
      "model_id": "eleven_multilingual_v2",
      "voice_settings": {
          "stability": voice_params.get("stability", 0.5),
          "similarity_boost": voice_params.get("similarity_boost", 0.75),
          "style": voice_params.get("style", 0.0),
          "use_speaker_boost": True,
      },
      "output_format": "mp3_44100_128",
  }

  for attempt in range(_MAX_RETRIES):
      with httpx.Client(timeout=120.0) as client:
          response = client.post(url, json=payload, headers=headers)
      if response.status_code == 200:
          return response.content
      if response.status_code == 429:
          delay = _RETRY_BASE_DELAY * (2 ** attempt)
          time.sleep(delay)
          continue

The text output is always generated regardless of whether audio synthesis succeeds. If you do not have an ElevenLabs API key, you still get a complete narration script as markdown.


THE BUILD STORY

Code-tales was built using the parallel AI development patterns described earlier in this series. 636 commits across 90 worktree branches. 91 specification documents. 37 validation gates.

The audio debugging saga deserves special mention: 9 commits to fix race conditions in the synthesis pipeline. The problem was subtle -- when sections were concatenated for TTS, the natural pause markers between sections were being dropped during the markdown cleaning step. This produced audio where one section ran directly into the next without any breathing room. The fix was to build the TTS text separately from the display text, preserving paragraph breaks as natural pauses:

  # From: src/code_tales/pipeline/synthesize.py

  _PAUSE_BETWEEN_SECTIONS = "\n\n"  # Natural paragraph break for TTS

  def _build_tts_text(script: NarrationScript) -> str:
      parts: list[str] = []
      for section in script.sections:
          parts.append(section.heading + ".")
          parts.append(section.content)
      return _PAUSE_BETWEEN_SECTIONS.join(parts)

Simple in retrospect. Nine commits to arrive at two newline characters. That is debugging audio pipelines for you.


THE CLI EXPERIENCE

The user-facing interface is a Click CLI with three commands -- generate, preview, and list-styles:

  # From: src/code_tales/cli.py

  @cli.command()
  @click.option("--repo", "repo_url", help="GitHub repository URL.")
  @click.option("--path", "local_path", type=click.Path(exists=True, path_type=Path))
  @click.option("--style", "-s", required=True)
  @click.option("--output", "-o", "output_dir", type=click.Path(path_type=Path))
  @click.option("--no-audio", is_flag=True, default=False)
  def generate(ctx, repo_url, local_path, style, output_dir, no_audio):
      pipeline = CodeTalesPipeline(config=config)
      result = pipeline.generate(
          repo_url_or_path=target,
          style_name=style,
          output_dir=out_dir,
      )

All configuration flows through environment variables via a Pydantic config model:

  # From: src/code_tales/config.py

  class CodeTalesConfig(BaseModel):
      anthropic_api_key: Optional[str] = None
      elevenlabs_api_key: Optional[str] = None
      output_dir: Path = Field(default_factory=lambda: Path("./output"))
      clone_depth: int = 1
      max_files_to_analyze: int = 100
      max_file_size_bytes: int = 100_000
      claude_model: str = "claude-sonnet-4-6"
      max_script_tokens: int = 4096

The style system is extensible via YAML. Drop a new .yaml file in the styles directory -- or load one from any path at runtime -- and you have a new narrative voice. The README includes a full example of creating a "noir detective" style. The style registry handles loading, validation, and lookup:

  # From: src/code_tales/styles/registry.py

  class StyleRegistry:
      def load_builtin_styles(self) -> dict[str, StyleConfig]:
          yaml_files = sorted(_STYLES_DIR.glob("*.yaml"))
          for yaml_file in yaml_files:
              style = self._load_yaml(yaml_file)
              self._styles[style.name] = style
          return self._styles

      def get_style(self, name: str) -> StyleConfig:
          if name not in self._styles:
              available = ", ".join(sorted(self._styles.keys()))
              raise KeyError(
                  f"Style '{name}' not found. Available styles: {available}"
              )
          return self._styles[name]


WHAT I LEARNED

The most counterintuitive lesson: the analysis stage matters more than the narration stage. Give Claude mediocre data and you get mediocre narration regardless of how good the style prompt is. Give Claude precise, quantified analysis -- "47.3% Python, 12 dependencies including FastAPI and Pydantic, REST API + CLI Tool patterns detected, 23,000 lines across 145 files" -- and the narration practically writes itself.

The second lesson: voice parameters are a design surface. Changing ElevenLabs stability from 0.7 to 0.3 transforms a documentary into a dramatic reading. These three numbers -- stability, similarity_boost, style -- are the audio equivalent of typography choices. They deserve the same attention.

The third lesson: text-to-speech requires a different writing style than text-to-screen. Every markdown artifact that slips through the cleaning stage is audible. Every bullet point that should have been a sentence creates an awkward pause. When you build a pipeline that crosses modalities, the boundary between them is where the bugs live.

Companion repo: github.com/krzemienski/code-tales

#AgenticDevelopment #ClaudeAI #ElevenLabs #TextToSpeech #DeveloperTools


================================================================================
POST 10 OF 10: 21 AI-GENERATED SCREENS, ZERO FIGMA FILES
Agentic Development: 10 Lessons from 8,481 AI Coding Sessions
================================================================================

I described an entire web application in plain English. The AI generated 21 production screens -- with components, design tokens, and validation -- in a single session. No Figma. No hand-written CSS. No designer in the loop.

This is not a prototype. It is a complete brutalist-cyberpunk web application with 47 design tokens, 5 component primitives, 4 example compositions, and a 107-action Puppeteer validation suite that proves every screen renders correctly.

This is post 10 of 10 in the Agentic Development series. The companion repo is at github.com/krzemienski/stitch-design-to-code. Everything I quote here is real code from that repo.


THE WORKFLOW

The cycle looks deceptively simple:

1. Craft a text prompt with the design system embedded
2. Feed it to Stitch MCP (an AI design generation tool)
3. Review the visual output for spec compliance
4. Convert to React components with data-testid attributes
5. Run Puppeteer validation against all 21 screens
6. If anything fails, go back to step 1

[DIAGRAM: Circular workflow -- Craft Prompt (embed design system, describe layout + states) -> Stitch MCP (generate visual design from structured prompt) -> Design Output (review for spec compliance, check brand + colors) -> React Conversion (build components, add data-testid attrs) -> Puppeteer Validation (107 actions / 21 screens) -> All Checks Pass? If yes, ship. If no, loop back to Craft Prompt.]

The critical insight, learned the hard way: the prompt must contain the complete design system every single time. Not "see previous specs." Not "same as before." The full token set, repeated verbatim, in every prompt for every screen. Context window drift is real, and it manifests as subtle visual inconsistencies across screens generated in the same session.


THE DESIGN TOKEN SYSTEM

Every visual decision in the application is derived from 47 tokens defined in a single JSON file. Here is the brutalist-cyberpunk palette:

  // From: design-system/tokens.json

  {
    "colors": {
      "background": "#000000",
      "primary": "#e050b0",
      "secondary": "#4dacde",
      "surface": "#111111",
      "surfaceAlt": "#1a1a1a",
      "surfaceHover": "#222222",
      "text": "#ffffff",
      "textSecondary": "#a0a0a0",
      "textMuted": "#606060",
      "border": "#2a2a2a",
      "borderAccent": "#3a3a3a",
      "success": "#22c55e",
      "warning": "#f59e0b",
      "error": "#ef4444",
      "primaryGlow": "rgba(224, 80, 176, 0.3)",
      "secondaryGlow": "rgba(77, 172, 222, 0.3)"
    }
  }

Pure black background. Hot pink primary accent. Cyan secondary. No gray area -- literally. The surface colors step from #111111 to #1a1a1a to #222222 in tight increments, creating depth through value rather than hue.

But the most distinctive design decision lives in the border radius tokens:

  // From: design-system/tokens.json

  "borderRadius": {
    "none": "0px",
    "sm": "0px",
    "base": "0px",
    "md": "0px",
    "lg": "0px",
    "xl": "0px",
    "2xl": "0px",
    "full": "0px"
  }

Every single border radius value is 0px. Not "mostly square with some rounding." Zero. Everywhere. This is not laziness -- it is a deliberate brutalist design choice. When components from shadcn/ui reference borderRadius.md or borderRadius.lg, they resolve to 0px. The token system enforces the aesthetic at the configuration layer, not the component layer.

Typography is equally opinionated. One font family everywhere:

  // From: design-system/tokens.json

  "typography": {
    "fontFamily": "JetBrains Mono, monospace",
    "fontFamilyFallback": "Courier New, Courier, monospace"
  }

JetBrains Mono for headlines, body copy, buttons, labels, navigation -- everything. Combined with the 0px border radius, this produces an aggressive terminal-aesthetic that looks like nothing else in the React ecosystem.

These tokens flow into Tailwind via a preset that maps every token to a utility class:

  // From: design-system/tailwind-preset.js

  const tokens = require('./tokens.json');

  const preset = {
    theme: {
      extend: {
        colors: {
          background: tokens.colors.background,
          primary: {
            DEFAULT: tokens.colors.primary,
            glow: tokens.colors.primaryGlow,
            subtle: tokens.colors.primarySubtle,
          },
          secondary: {
            DEFAULT: tokens.colors.secondary,
            glow: tokens.colors.secondaryGlow,
            subtle: tokens.colors.secondarySubtle,
          },
          surface: {
            DEFAULT: tokens.colors.surface,
            alt: tokens.colors.surfaceAlt,
            hover: tokens.colors.surfaceHover,
          },
        },
        fontFamily: {
          mono: [tokens.typography.fontFamily, tokens.typography.fontFamilyFallback],
          sans: [tokens.typography.fontFamily, tokens.typography.fontFamilyFallback],
        },
        borderRadius: {
          none: tokens.borderRadius.none,
          sm: tokens.borderRadius.sm,
          DEFAULT: tokens.borderRadius.base,
          md: tokens.borderRadius.md,
          lg: tokens.borderRadius.lg,
          xl: tokens.borderRadius.xl,
          '2xl': tokens.borderRadius['2xl'],
          full: tokens.borderRadius.full,
        },
      },
    },
  };

Notice: fontFamily.sans maps to JetBrains Mono. Even when components use Tailwind's font-sans class, they get the monospaced font. The design system is inescapable.

The preset also defines custom background gradients that use the primary and secondary colors at low opacity:

  // From: design-system/tailwind-preset.js

  backgroundImage: {
    'primary-gradient': `linear-gradient(135deg, ${tokens.colors.primary}20 0%, transparent 60%)`,
    'hero-gradient': `radial-gradient(ellipse at top, ${tokens.colors.primary}15 0%, transparent 60%), radial-gradient(ellipse at bottom right, ${tokens.colors.secondary}10 0%, transparent 50%)`,
    'grid-pattern': `linear-gradient(${tokens.colors.border} 1px, transparent 1px), linear-gradient(90deg, ${tokens.colors.border} 1px, transparent 1px)`,
  },

That grid-pattern utility creates a subtle wire-grid background that reinforces the terminal aesthetic. One line in the config, reusable across every screen.


THE COMPONENT LAYER

The Button component demonstrates how CVA (Class Variance Authority) combines with the token system to produce a flexible, type-safe component API. Eight variants, eight sizes, forwardRef, Radix Slot for composition:

  // From: components/ui/button.tsx

  const buttonVariants = cva(
    // Base styles
    [
      'inline-flex items-center justify-center gap-2',
      'font-mono text-sm font-semibold',
      'border-0 outline-none',
      'transition-all duration-150',
      'cursor-pointer select-none',
      'disabled:pointer-events-none disabled:opacity-50',
      'focus-visible:ring-2 focus-visible:ring-primary focus-visible:ring-offset-2 focus-visible:ring-offset-background',
      // Brutalist — no border radius anywhere
      'rounded-none',
    ].join(' '),
    {
      variants: {
        variant: {
          /** Hot pink fill — primary CTA */
          default: [
            'bg-primary text-black',
            'hover:bg-[#c040a0] hover:shadow-[0_0_20px_rgba(224,80,176,0.4)]',
            'active:scale-[0.98] active:bg-[#b03090]',
          ].join(' '),

          /** Cyan outline */
          'secondary-outline': [
            'border border-secondary text-secondary bg-transparent',
            'hover:bg-secondary hover:text-black hover:shadow-[0_0_16px_rgba(77,172,222,0.3)]',
            'active:scale-[0.98]',
          ].join(' '),

          /** Ghost — subtle surface hover */
          ghost: [
            'bg-transparent text-text-secondary',
            'hover:bg-surface hover:text-text',
            'active:scale-[0.98]',
          ].join(' '),
        },

        size: {
          xs: 'h-7 px-2 text-xs',
          sm: 'h-8 px-3 text-xs',
          default: 'h-10 px-4 text-sm',
          lg: 'h-12 px-6 text-base',
          xl: 'h-14 px-8 text-lg',
          icon: 'h-10 w-10 p-0',
          'icon-sm': 'h-8 w-8 p-0',
          'icon-lg': 'h-12 w-12 p-0',
        },
      },
    }
  );

The hover effects are where the brutalist aesthetic comes alive. The primary button gets a hot pink glow on hover -- shadow-[0_0_20px_rgba(224,80,176,0.4)]. The secondary-outline variant gets a cyan glow. Combined with the sharp 0px corners and JetBrains Mono text, these glow effects create a cyberpunk UI that feels like interacting with a terminal from the future.

The component supports Radix Slot for composition, a loading state with an inline spinner, and proper aria-disabled handling:

  // From: components/ui/button.tsx

  const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
    ({ className, variant, size, asChild = false, loading = false, children, disabled, ...props }, ref) => {
      const Comp = asChild ? Slot : 'button';

      return (
        <Comp
          ref={ref}
          className={cn(buttonVariants({ variant, size }), className)}
          disabled={disabled || loading}
          aria-disabled={disabled || loading}
          {...props}
        >
          {loading ? (
            <>
              <LoadingSpinner />
              <span className="opacity-70">{children}</span>
            </>
          ) : (
            children
          )}
        </Comp>
      );
    }
  );

This is one button component. There are 5 total component primitives (Button, Card, Input, Tabs, Badge) and 4 example compositions (HomeHero, ResourceCard, AuthForm, AdminTabs). From these 9 building blocks, 21 screens are constructed.


THE VALIDATION SUITE

This is where most AI design workflows fall apart. You generate beautiful screens, manually eyeball them, declare victory, and ship something that breaks in production. The stitch-design-to-code approach replaces eyeballing with 107 programmatic Puppeteer actions across all 21 screens.

The checks are defined declaratively:

  // From: validation/puppeteer-checks.js

  {
    name: 'home-design-tokens',
    route: '/',
    actions: [
      { type: 'navigate', expected: 200 },
      {
        type: 'evaluate',
        expected: () => {
          const body = document.body;
          const bg = window.getComputedStyle(body).backgroundColor;
          return bg === 'rgb(0, 0, 0)';
        },
      },
    ],
  },

That check validates that the home page background is literally rgb(0, 0, 0) -- pure black, as specified in the design tokens. Not "dark enough." Not "looks black." Computationally verified pure black.

The suite covers 7 action types: navigate (HTTP status checks), screenshot (visual capture), waitForSelector (DOM presence), assert (visibility), evaluate (arbitrary JS assertions), click (interaction), and fill (form input). Here is a form interaction check:

  // From: validation/puppeteer-checks.js

  {
    name: 'login-form-fill',
    route: '/auth/login',
    actions: [
      { type: 'navigate', expected: 200 },
      { type: 'fill', selector: '[data-testid="email-input"]', value: 'test@example.com' },
      { type: 'fill', selector: '[data-testid="password-input"]', value: 'password123' },
      { type: 'screenshot', screenshot: 'login-filled.png' },
    ],
  },

And here is one that validates the admin dashboard has at least 20 tabs:

  // From: validation/puppeteer-checks.js

  {
    name: 'admin-tab-count',
    route: '/admin',
    actions: [
      { type: 'navigate', expected: 200 },
      { type: 'waitForSelector', selector: '[data-testid="admin-tabs"]' },
      { type: 'evaluate', expected: () => document.querySelectorAll('[data-testid^="tab-"]').length >= 20 },
    ],
  },

The validation runner executes checks sequentially (for deterministic screenshots), reports pass/fail with timing, and writes a JSON report:

  // From: validation/run-validation.js

  for (let i = 0; i < filteredChecks.length; i++) {
    const check = filteredChecks[i];
    const progress = `[${String(i + 1).padStart(2, '0')}/${filteredChecks.length}]`;
    process.stdout.write(`  ${progress} ${check.name.padEnd(40, '.')} `);

    const result = await runCheck(browser, check);
    results.push(result);

    if (result.status === 'pass') {
      process.stdout.write(`\x1b[32mPASS\x1b[0m (${result.duration}ms)\n`);
    } else {
      process.stdout.write(`\x1b[31mFAIL\x1b[0m (${result.duration}ms)\n`);
    }
  }

The distribution across screen groups:

  Public screens (Home, Resources, Search, About, Categories x2, Resource Detail): 36 checks
  Auth screens (Login, Register, Forgot Password): 14 checks
  User screens (Profile, Bookmarks, Favorites, History): 18 checks
  Admin screens (Admin Dashboard with 20 tabs, Suggest Edit): 29 checks
  Legal screens (Privacy Policy, Terms of Service): 8 checks
  Total: 105 checks across 107 actions (some checks validate multiple assertions)

The Admin Dashboard alone has 25 checks -- one per tab plus KPI card validation. When you have a 20-tab admin interface, you cannot visually verify all 20 tabs by hand every time you make a change. The Puppeteer suite clicks each tab, waits for the content to render, and screenshots the result. Every time.


THE BRANDING BUG

Here is the most instructive failure from the entire project. During generation, Stitch MCP started producing screens with the product name "Awesome Video Dashboard" instead of the correct name "Awesome Lists." The substitution happened at screen 1 and propagated to 8 screens before it was caught.

The root cause analysis from the branding checklist:

  // From: docs/branding-checklist.md

  ## Why It Happens in AI Workflows

  ### 1. Context Window Drift
  When generating multiple screens in a long session (21+ screens),
  the AI model's "attention" to early instructions diminishes. The
  product name specified in the first prompt may be remembered less
  faithfully by the 15th screen generation.

  ### 2. Training Data Default Patterns
  AI models have seen millions of example UIs with placeholder names
  like "Awesome App," "My Dashboard," "Video Platform." When the
  specific product name isn't prominent in the prompt, the model
  falls back to these common patterns.

  ### 3. Semantic Similarity Substitution
  "Awesome Lists" and "Awesome Video Dashboard" share the word
  "Awesome." The model sometimes pattern-matches on the adjective
  and completes with a more "common" noun phrase from its training data.

The fix is treating branding as a first-class design token, not as free text scattered across prompts. The branding checklist proposes adding brand tokens to the design system:

  // From: docs/branding-checklist.md

  {
    "brand": {
      "name": "Awesome Lists",
      "tagline": "The Ultimate Curated Resource Directory",
      "domain": "awesomeLists.dev",
      "copyrightHolder": "Nick Krzemienski"
    }
  }

And then injecting them programmatically via a prompt builder, so the correct product name is never dependent on human memory or AI attention span.

The automated prevention is a simple grep:

  grep -rn "Video Dashboard\|Placeholder\|lorem ipsum" src/ components/ app/

Run this after every generation session. Add it as a prebuild script. The branding bug is a procedural problem, not a technical one. The AI is perfectly capable of using the correct product name -- it just needs to be told explicitly, every time, in every prompt.


THE PROMPT ENGINEERING PATTERN

Every screen prompt follows the same structure: product name in the first sentence, full design system spec inline, layout description, and key elements list. This pattern worked for all 21 screens:

  Design a [SCREEN NAME] for "Your Product" -- [one-line description].

  DESIGN SYSTEM:
  - Background: #000000 (pure black)
  - Primary Accent: #e050b0 (hot pink)
  - Secondary Accent: #4dacde (cyan)
  - Surface/Card: #111111 background, #1a1a1a elevated
  - Text: #ffffff primary, #a0a0a0 secondary
  - Font: JetBrains Mono, monospaced, used everywhere
  - Border radius: 0px -- brutalist aesthetic
  - Component library: shadcn/ui
  - Borders: 1px solid #2a2a2a

  LAYOUT:
  [Describe structure here]

  KEY ELEMENTS:
  [List all UI elements with specifics]

The redundancy is intentional. Yes, the design system spec is identical across all 21 prompts. Yes, it increases token usage. But it eliminates context window drift entirely. The 15th screen generation has the same design fidelity as the 1st because it has the same context.


THE NUMBERS

  21 screens generated from text descriptions
  47 design tokens governing every visual decision
  107 Puppeteer validation actions (374 individual assertions within those actions)
  5 shadcn/ui component primitives
  4 example compositions
  8 button variants, 8 button sizes
  0 Figma files opened
  0 lines of hand-written CSS
  ~13,432 lines of session transcript
  1 branding bug caught, documented, and solved with a grep script


WHAT THE REMAINING CHALLENGES LOOK LIKE

The interesting conclusion from this project: the remaining challenges are procedural, not technical.

The AI can generate production-quality React components from text descriptions. It can apply design tokens consistently. It can produce 20-tab admin dashboards with proper data-testid attributes. The technology works.

What does not work yet is the process around the technology. Branding drift requires explicit prevention. Long sessions require redundant context. Validation requires deliberate data-testid architecture planned before generation, not added after. These are workflow problems, not capability problems.

The gap between "AI can generate UI" and "AI reliably generates the right UI" is entirely a prompt engineering and validation infrastructure gap. Stitch MCP handles the generation. Puppeteer handles the validation. The hard part -- the part this repo documents -- is everything in between: the token system that constrains the design space, the prompt structure that ensures consistency, the branding safeguards that prevent drift, and the validation suite that proves correctness.

This is the final post in the series. Ten posts. Ten lessons. 8,481 sessions. The tools are here. The patterns are documented. The companion repos have real code you can run today.

The question is no longer whether AI can build production software. It is whether you have the workflows to make it do so reliably.

Companion repo: github.com/krzemienski/stitch-design-to-code

#AgenticDevelopment #StitchMCP #DesignToCode #ReactComponents #PuppeteerValidation


---

Agentic Development: 10 Lessons from 8,481 AI Coding Sessions

Blog 10: The AI Development Operating System


I did not set out to build an operating system.

Over the past 90 days, across 8,481 Claude Code sessions, I kept hitting the same failures. An agent would lose context mid-task. A plan would survive first contact with the codebase for about six minutes before collapsing. Code review happened after the PR was already merged. Verification was a human squinting at a terminal and saying "looks good."

Each time I hit a failure mode, I built a small system to prevent it from happening again. An agent catalog here. A persistence layer there. An adversarial planning protocol after the third time a confident plan led to a dead end.

Ninety days later, I looked at what I had and realized: this is not a collection of scripts. It is an operating system for AI development. Six composable subsystems, each solving a specific class of failure, each usable independently but dramatically more powerful together.

This is the capstone post in the series. Everything I have learned, distilled into architecture you can run yourself.

The companion repository is at the end of this post. Every code block below is quoted from real, running source files.


THE THESIS

Here it is, plainly stated: the models are capable enough. What they need is a system.

A single Claude session can write excellent code. But software development is not a single session. It is research, planning, implementation, review, verification, and iteration -- spread across hours, days, and weeks. It requires specialization, persistence, adversarial challenge, and evidence-based verification.

These are not AI problems. These are organizational problems. Human engineering teams solved them decades ago with role specialization, code review, QA gates, and project management. The insight is that AI agents benefit from exactly the same organizational principles.

The AI Development Operating System encodes those principles into six composable subsystems.

[DIAGRAM: High-level architecture showing the six subsystems (OMC, Ralph Loop, Specum, RALPLAN, GSD, Team Pipeline) arranged as layers, with arrows showing how they compose. OMC at the base providing agents, Ralph Loop providing persistence, and the others as workflow orchestrators that consume both.]


SUBSYSTEM 1: OMC -- THE AGENT CATALOG

Failure mode it solves: using the wrong model for the job, or using a generalist when you need a specialist.

The first thing I built was a catalog of specialized agents. Not because specialization is theoretically elegant, but because I watched a $15/million-token Opus model spend 45 seconds doing file searches that Haiku could do in 1.5 seconds at one-twentieth the cost.

OMC (Oh My Claude Code) defines 25 agents organized into four lanes. Every agent has a full system prompt, a model tier assignment, and explicit capabilities. Here is the catalog structure from the YAML definition:

  # src/ai_dev_os/omc/catalog.yaml

  version: "1.0.0"
  description: "Complete OMC agent catalog for Claude Code orchestration"

  agents:
    - name: explore
      lane: build
      model_tier: haiku
      description: "Codebase discovery, file/symbol mapping, dependency tracing"
      capabilities:
        - "Glob and Grep across large codebases efficiently"
        - "Map module boundaries and import graphs"
        - "Identify entry points, interfaces, and public APIs"

    - name: architect
      lane: build
      model_tier: opus
      description: "System design, component boundaries, interfaces, long-horizon architectural tradeoffs"
      capabilities:
        - "Design system boundaries and component responsibilities"
        - "Define API contracts and interface specifications"
        - "Evaluate architectural patterns: CQRS, event sourcing, hexagonal, etc."

The four lanes:

BUILD LANE: explore (haiku), analyst (opus), planner (opus), architect (opus), debugger (sonnet), executor (sonnet), deep-executor (opus), verifier (sonnet). This is your development team -- from discovery through implementation through verification.

REVIEW LANE: quality-reviewer (sonnet), security-reviewer (sonnet), code-reviewer (opus). Three independent review perspectives that can run in parallel.

DOMAIN LANE: test-engineer (sonnet), build-fixer (sonnet), designer (sonnet), writer (haiku), qa-tester (sonnet), scientist (sonnet), document-specialist (sonnet). Specialists you call when the task demands domain expertise.

COORDINATION LANE: critic (opus). One agent whose sole purpose is to find flaws in plans. More on this in the RALPLAN section.

Each agent is loaded as a typed model with full metadata:

  # src/ai_dev_os/omc/catalog.py

  class AgentDefinition(BaseModel):
      """A single agent definition from the catalog."""

      name: str = Field(description="Unique agent identifier")
      lane: str = Field(description="Organizational lane: build, review, domain, coordination")
      model_tier: str = Field(description="Preferred model tier: haiku, sonnet, opus")
      description: str = Field(description="Brief one-line description")
      capabilities: list[str] = Field(default_factory=list)
      system_prompt: str = Field(description="Full system prompt for this agent")

The model routing table maps each agent to its canonical model tier, with cost and latency characteristics:

  # src/ai_dev_os/omc/routing.py

  AGENT_MODEL_MAP: dict[str, ModelTier] = {
      # Build lane
      "explore": ModelTier.HAIKU,
      "analyst": ModelTier.OPUS,
      "planner": ModelTier.OPUS,
      "architect": ModelTier.OPUS,
      "debugger": ModelTier.SONNET,
      "executor": ModelTier.SONNET,
      "deep-executor": ModelTier.OPUS,
      "verifier": ModelTier.SONNET,
      # Review lane
      "quality-reviewer": ModelTier.SONNET,
      "security-reviewer": ModelTier.SONNET,
      "code-reviewer": ModelTier.OPUS,
      # Domain lane
      "test-engineer": ModelTier.SONNET,
      "build-fixer": ModelTier.SONNET,
      "designer": ModelTier.SONNET,
      "writer": ModelTier.HAIKU,
      # Coordination lane
      "critic": ModelTier.OPUS,
  }

The routing is not just about cost. It encodes a fundamental principle: match cognitive depth to task complexity. The explore agent runs on Haiku because codebase discovery is breadth-first -- you want speed, not deep reasoning. The architect runs on Opus because system design requires holding many constraints in mind simultaneously. The executor runs on Sonnet because it is the best coding model -- fast enough to iterate, smart enough to get the implementation right.

The router also supports dynamic complexity scoring for tasks that do not map to a known agent:

  # src/ai_dev_os/omc/routing.py

  COMPLEXITY_SIGNALS = {
      "high": [
          "architecture", "design", "system", "distributed", "scalab",
          "security", "critical", "production", "migrate", "refactor entire",
          "adversarial", "deep analysis", "comprehensive", "cross-cutting",
      ],
      "medium": [
          "implement", "debug", "review", "test", "fix", "build",
          "integrate", "api", "database", "async", "concurrent",
      ],
      "low": [
          "search", "find", "list", "summarize", "document", "format",
          "check", "scan", "quick", "simple", "lookup",
      ],
  }

The self-hosting story: OMC started with 8 agents. Over 90 days, the system identified gaps -- situations where no existing agent was the right fit -- and I added specialists. The catalog grew from 8 to 25 agents, each one born from a real failure that the existing catalog could not handle.


SUBSYSTEM 2: RALPH LOOP -- PERSISTENT EXECUTION

Failure mode it solves: work dying when a session ends.

This is the subsystem with the most personality. Its motto: "The boulder never stops."

The problem is simple. Claude Code sessions end. The context window fills up. The laptop lid closes. But the work is not done. Ralph Loop provides persistent iterative execution that survives session boundaries by writing its entire state to disk after every iteration.

  # src/ai_dev_os/ralph_loop/state.py

  class RalphTask(BaseModel):
      """A single task tracked within the Ralph Loop."""

      id: str = Field(description="Unique task identifier")
      title: str = Field(description="Short task title")
      description: str = Field(default="", description="Full task description")
      status: TaskStatus = Field(default=TaskStatus.PENDING)
      phase: Optional[str] = Field(default=None)
      created_at: datetime = Field(default_factory=datetime.utcnow)
      started_at: Optional[datetime] = None
      completed_at: Optional[datetime] = None
      attempts: int = Field(default=0, description="Number of execution attempts")
      error: Optional[str] = None

  class RalphState(BaseModel):
      """Complete state for a Ralph Loop execution session."""

      iteration: int = Field(default=0)
      max_iterations: int = Field(default=100)
      task_list: list[RalphTask] = Field(default_factory=list)
      goal: str = Field(default="")
      started_at: datetime = Field(default_factory=datetime.utcnow)
      status: LoopStatus = Field(default=LoopStatus.RUNNING)
      linked_team: Optional[str] = Field(default=None)
      stop_reason: Optional[str] = None

The iterate() method is the heart of the system. Each call processes one iteration, logs progress, checks completion, and persists state:

  # src/ai_dev_os/ralph_loop/loop.py

  def iterate(
      self,
      task_runner: Optional[Callable[[RalphTask], bool]] = None,
  ) -> bool:
      state = self.state
      state.iteration += 1

      # Check completion
      if self.check_completion():
          return False

      # Process next pending task
      pending = state.pending_tasks()
      if pending and task_runner:
          next_task = pending[0]
          next_task.mark_started()

          success = task_runner(next_task)
          if success:
              next_task.mark_completed()
          else:
              next_task.mark_failed("Task runner returned False")

      # Check max iterations
      if state.iteration >= state.max_iterations:
          state.status = LoopStatus.FAILED
          state.stop_reason = f"Max iterations ({state.max_iterations}) reached"
          self.persist_state()
          return False

      self.persist_state()
      return True

The key design choice: state is a flat JSON file at .omc/state/ralph-state.json. Not a database. Not a message queue. A JSON file. Because the most important property of persistence is that it actually works, and a JSON file on disk is the most reliable thing in computing. A new session reads it, picks up where the last one left off, and keeps going.

The completion check is deliberately strict -- all tasks must reach COMPLETED status:

  # src/ai_dev_os/ralph_loop/state.py

  def is_complete(self) -> bool:
      if not self.task_list:
          return False
      return all(t.status == TaskStatus.COMPLETED for t in self.task_list)


SUBSYSTEM 3: SPECUM PIPELINE -- SPEC BEFORE CODE

Failure mode it solves: jumping straight to implementation without understanding what to build.

I named it Specum because it forces you to specify before you implement. The pipeline enforces a mandatory sequence:

  # src/ai_dev_os/specum/pipeline.py

  STAGE_ORDER = [
      PipelineStage.NEW,
      PipelineStage.REQUIREMENTS,
      PipelineStage.DESIGN,
      PipelineStage.TASKS,
      PipelineStage.IMPLEMENT,
      PipelineStage.VERIFY,
      PipelineStage.COMPLETE,
  ]

  STAGE_DESCRIPTIONS = {
      PipelineStage.NEW: "Pipeline initialized, ready to begin",
      PipelineStage.REQUIREMENTS: "Gathering user stories and acceptance criteria",
      PipelineStage.DESIGN: "Creating schema definitions and API contracts",
      PipelineStage.TASKS: "Breaking design into ordered implementation tasks",
      PipelineStage.IMPLEMENT: "Executing tasks with Ralph Loop persistence",
      PipelineStage.VERIFY: "Validating completion with evidence collection",
      PipelineStage.COMPLETE: "Pipeline complete — all stages verified",
  }

Each stage produces a markdown artifact that the next stage consumes. You cannot skip ahead. The pipeline will not let you implement before you have a design, and it will not let you design before you have requirements.

Notice stage five: "Executing tasks with Ralph Loop persistence." Specum and Ralph compose. The specification pipeline generates the task list; Ralph Loop executes it persistently. Two subsystems, each doing one thing well, combining into something neither could do alone.

Each stage also maps to its expected artifact file:

  # src/ai_dev_os/specum/pipeline.py

  STAGE_ARTIFACTS = {
      PipelineStage.REQUIREMENTS: "requirements.md",
      PipelineStage.DESIGN: "design.md",
      PipelineStage.TASKS: "tasks.md",
      PipelineStage.IMPLEMENT: "implementation-report.md",
      PipelineStage.VERIFY: "verification-report.md",
  }


SUBSYSTEM 4: RALPLAN -- ADVERSARIAL PLANNING

Failure mode it solves: plans that sound confident but collapse on contact with reality.

RALPLAN is adversarial deliberation. A Planner creates a plan. A Critic tears it apart. The Planner revises. The Critic reviews again. This continues until the Critic approves or the maximum iteration count is reached.

The critical design constraint: the Critic can only identify problems. It cannot suggest solutions. This is not an arbitrary restriction -- it is the key to the whole protocol. When the critic suggests solutions, the planner becomes a secretary taking dictation. When the critic can only say "this is wrong," the planner must actually think about how to fix it.

  # src/ai_dev_os/ralplan/critic.py

  class CriticFinding:
      """A specific finding from the critic's review."""

      severity: str  # CRITICAL, MODERATE, MINOR
      category: str  # completeness, feasibility, hand_waving, risk_coverage
      description: str
      plan_reference: Optional[str] = None

  class CriticVerdict:
      """Complete verdict from the CriticAgent."""

      verdict: VerdictType
      critical_findings: list[CriticFinding] = field(default_factory=list)
      moderate_findings: list[CriticFinding] = field(default_factory=list)
      minor_findings: list[CriticFinding] = field(default_factory=list)
      rationale: str = ""

      @property
      def is_approved(self) -> bool:
          return self.verdict == VerdictType.APPROVE

The critic evaluates across four categories:

COMPLETENESS: Does the plan cover all required areas? Are there phases without verification criteria? Are there empty phases with no tasks?

FEASIBILITY: Can this actually be built? Are tasks described with enough detail for an implementer to execute? Are high-complexity tasks flagged with risks?

HAND-WAVING: The critic scans for vague language that hides implementation gaps:

  # src/ai_dev_os/ralplan/critic.py

  hand_wave_phrases = [
      "implement the logic",
      "handle appropriately",
      "etc",
      "and so on",
      "as needed",
      "when necessary",
      "somehow",
      "figure out",
  ]

RISK COVERAGE: Are enough risks identified? Is there a proof-of-concept task for unknown territory?

The verdict is binary. Any critical finding means REJECT:

  # src/ai_dev_os/ralplan/critic.py

  verdict = VerdictType.REJECT if critical else VerdictType.APPROVE

The deliberation loop runs the full protocol:

  # src/ai_dev_os/ralplan/deliberate.py

  for iteration in range(1, MAX_ITERATIONS + 1):
      # Planner creates or revises the plan
      if current_plan is None:
          current_plan = self._planner.create_plan(task)
      else:
          criticism = "\n".join(
              f.description for f in current_verdict.critical_findings
          )
          current_plan = self._planner.revise_plan(current_plan, criticism)

      # Critic reviews the plan
      current_verdict = self._critic.review(current_plan)

      if current_verdict.is_approved:
          break

In extended deliberation mode (--deliberate), RALPLAN adds pre-mortem analysis -- imagining the plan has already failed and working backward to identify the most likely causes -- and expanded test planning across unit, integration, e2e, and observability layers.

[DIAGRAM: Sequence diagram showing the RALPLAN deliberation loop: Planner creates plan -> Critic reviews -> REJECT with findings -> Planner revises incorporating findings -> Critic reviews again -> APPROVE or loop continues until MAX_ITERATIONS]


SUBSYSTEM 5: GSD -- THE 10-PHASE PROJECT LIFECYCLE

Failure mode it solves: projects that start strong and then drift into undefined territory.

GSD (Get Stuff Done) is a 10-phase lifecycle with evidence gates. You cannot advance to the next phase without proving the current phase is complete. Not claiming it is complete. Proving it.

  # src/ai_dev_os/gsd/phases.py

  class ProjectPhase(str, Enum):
      NEW_PROJECT = "new_project"
      RESEARCH = "research"
      ROADMAP = "roadmap"
      PLAN_PHASE = "plan_phase"
      EXECUTE_PHASE = "execute_phase"
      VERIFY_PHASE = "verify_phase"
      ITERATE = "iterate"
      INTEGRATION = "integration"
      PRODUCTION_READINESS = "production_readiness"
      COMPLETE = "complete"

Each phase has required evidence types that must be collected before the gate opens:

  # src/ai_dev_os/gsd/phases.py

  PHASE_REQUIRED_EVIDENCE = {
      ProjectPhase.RESEARCH: ["research_document", "feasibility_assessment"],
      ProjectPhase.ROADMAP: ["roadmap_document", "success_criteria"],
      ProjectPhase.PLAN_PHASE: ["implementation_plan", "task_list"],
      ProjectPhase.EXECUTE_PHASE: ["build_log", "implementation_report"],
      ProjectPhase.VERIFY_PHASE: ["verification_report", "acceptance_criteria_results"],
      ProjectPhase.INTEGRATION: ["integration_test_results", "e2e_scenario_results"],
      ProjectPhase.PRODUCTION_READINESS: ["deployment_guide", "runbook", "monitoring_setup"],
      ProjectPhase.COMPLETE: ["final_report"],
  }

The evidence collection system is its own module. Evidence is typed, timestamped, and traceable:

  # src/ai_dev_os/gsd/evidence.py

  class EvidenceType(str, Enum):
      BUILD_LOG = "build_log"
      SCREENSHOT = "screenshot"
      API_RESPONSE = "api_response"
      TEST_OUTPUT = "test_output"
      MANUAL_VERIFICATION = "manual_verification"
      DOCUMENT = "document"
      METRIC = "metric"
      LOG_OUTPUT = "log_output"
      BENCHMARK_RESULT = "benchmark_result"

GSD also tracks assumptions explicitly through an AssumptionTracker. Assumptions made during planning that are never validated represent hidden risks. The tracker surfaces them at phase gates:

  # src/ai_dev_os/gsd/assumptions.py

  class AssumptionTracker:
      """
      Tracks assumptions made during GSD project phases.

      Assumptions that are never validated represent hidden risks.
      This tracker surfaces unvalidated assumptions at phase gates,
      preventing projects from proceeding with invalid foundations.
      """

      def critical_unvalidated(self) -> list[Assumption]:
          """Return unvalidated assumptions with critical or high impact."""
          return [
              a for a in self.list_unvalidated()
              if a.impact in ("critical", "high")
          ]

This is the mechanism that prevents completion theater -- the common pattern where a team says "we are done" without anyone actually verifying the claim. With GSD, "done" is not a feeling. It is a collection of evidence artifacts that prove the work was completed.


SUBSYSTEM 6: TEAM PIPELINE -- COORDINATED MULTI-AGENT EXECUTION

Failure mode it solves: agents working in isolation without a shared understanding of state.

Team Pipeline is the orchestration layer. It coordinates N agents through a staged pipeline with a state machine that handles both the happy path and the inevitable failures:

  # src/ai_dev_os/team_pipeline/pipeline.py

  PIPELINE_PROGRESSION = {
      PipelineStage.PLAN: PipelineStage.PRD,
      PipelineStage.PRD: PipelineStage.EXEC,
      PipelineStage.EXEC: PipelineStage.VERIFY,
      PipelineStage.VERIFY: PipelineStage.COMPLETE,  # on pass
      PipelineStage.FIX: PipelineStage.VERIFY,  # fix feeds back to verify
  }

The pipeline: PLAN -> PRD -> EXEC -> VERIFY -> FIX (loop, bounded).

Each stage uses specialized agents from the OMC catalog. The PLAN stage uses explore (haiku) for fast codebase discovery and planner (opus) for task sequencing. The EXEC stage dispatches to the appropriate specialist based on the task:

  # src/ai_dev_os/team_pipeline/stages.py

  class ExecStage(BaseStage):
      stage_name = "exec"
      required_agents = ["executor", "designer", "build-fixer", "writer", "test-engineer"]

      def _select_specialist(self, task: str) -> str:
          task_lower = task.lower()
          if any(kw in task_lower for kw in ["ui", "design", "ux", "interface", "layout"]):
              return "designer"
          elif any(kw in task_lower for kw in ["test", "coverage", "spec", "tdd"]):
              return "test-engineer"
          elif any(kw in task_lower for kw in ["doc", "readme", "guide", "changelog"]):
              return "writer"
          elif any(kw in task_lower for kw in ["complex", "architecture", "refactor entire", "migrate"]):
              return "deep-executor"
          else:
              return "executor"

The transition logic handles the critical verify-fix loop. When verification fails, the pipeline enters the FIX stage. Fix feeds back to VERIFY. If verification fails again and the fix loop count exceeds the bound, the pipeline transitions to FAILED:

  # src/ai_dev_os/team_pipeline/pipeline.py

  def _transition(self, current: PipelineStage, result: StageResult) -> None:
      state = self.state

      if current == PipelineStage.VERIFY:
          if result.success:
              state.current_stage = PipelineStage.COMPLETE
              state.status = PipelineStatus.COMPLETE
          else:
              if state.fix_loop_count >= state.max_fix_loops:
                  state.current_stage = PipelineStage.FAILED
                  state.status = PipelineStatus.FAILED
              else:
                  state.fix_loop_count += 1
                  state.current_stage = PipelineStage.FIX

      elif current == PipelineStage.FIX:
          state.current_stage = PipelineStage.VERIFY

The bounded fix loop is important. Without it, a failing pipeline could loop forever. Three fix attempts is the default. If you cannot fix a verification failure in three passes, the problem is likely architectural, not a simple bug.

[DIAGRAM: State machine diagram showing Team Pipeline transitions: PLAN -> PRD -> EXEC -> VERIFY -> COMPLETE (on PASS) or VERIFY -> FIX -> VERIFY (loop, max 3x) -> FAILED (if exceeded)]


THE UNIFIED CLI

All six subsystems are accessible through a single CLI:

  # src/ai_dev_os/cli.py

  @click.group()
  @click.version_option(version="1.0.0", prog_name="ai-dev-os")
  def main() -> None:
      """
      AI Development Operating System CLI.

      Orchestrate Claude Code agents across complex software projects
      using OMC, Ralph Loop, Specum, RALPLAN, GSD, and Team Pipeline.

      Quick Start:
        ai-dev-os catalog list           # See all 25+ agents
        ai-dev-os ralph start --task "Build auth system"
        ai-dev-os spec new --goal "Implement payment processing"
        ai-dev-os gsd new-project --name myapp
      """

This is not six disconnected tools bolted together. The subsystems compose:

- RALPLAN produces plans. Ralph Loop persists their execution.
- Specum generates task lists. Team Pipeline coordinates agents to execute them.
- GSD manages the project lifecycle. Each phase can invoke any subsystem.
- OMC provides the agent catalog that every other subsystem draws from.
- Evidence collection in GSD verifies work done by Team Pipeline.
- The Critic from RALPLAN can review plans generated by any subsystem.


THE COMPOSABILITY INSIGHT

The most important thing I learned is that composability beats integration. Each subsystem is useful on its own. Ralph Loop can persist any iterative workflow, not just those generated by Specum. RALPLAN can critique any plan, not just those from GSD. The OMC catalog serves any orchestrator, not just Team Pipeline.

This is the same lesson the Unix community learned decades ago: small, composable tools that do one thing well are more powerful than monolithic systems that try to do everything.

Here is a concrete example. To build a new feature using the full system:

  ai-dev-os plan --task "Add payment processing" --consensus
  ai-dev-os spec new --goal "Implement Stripe integration"
  ai-dev-os ralph start --task "Execute Specum task list"
  ai-dev-os team start --task "Parallel agent implementation"
  ai-dev-os gsd new-project --name payments --goal "Production payment system"

Each command invokes a different subsystem. Each subsystem reads state from the others. The result is a coordinated development workflow where no single component knows about all the others, but together they cover the full lifecycle.


THE SELF-HOSTING LOOP

There is one more thing worth mentioning. This system built itself.

The first version had 8 agents and no adversarial planning. I used it to build features. The features had bugs because the plans were not challenged. So I built RALPLAN to challenge plans. RALPLAN found gaps in the agent catalog. So I added agents. The new agents needed coordination. So I built Team Pipeline. Team Pipeline needed persistence. So I connected it to Ralph Loop.

Each subsystem was built using the subsystems that already existed. The agent catalog was designed by the architect agent. The RALPLAN protocol was planned by the planner agent and critiqued by the critic agent -- before the critic agent formally existed, which required manually playing the critic role and then encoding the pattern.

Over 90 days, the system grew from 8 agents to 25. From one subsystem to six. Each addition was driven by a real failure, not theoretical elegance. The operating system evolved through the same process it now manages: research, plan, execute, verify, iterate.


THE BOTTOM LINE

After 8,481 sessions, here is what I know: AI development benefits from the same organizational principles that human teams discovered decades ago.

Specialization works. Not every task needs the most powerful model. An explore agent on Haiku finds files faster than an architect on Opus, because it is built for that specific job.

Persistence works. Work that survives session boundaries is qualitatively different from work that dies when the terminal closes.

Adversarial review works. Plans that face a critic before implementation are plans that survive implementation.

Evidence-based verification works. "It looks done" is not the same as "here is the build log, the API response, and the screenshot proving it works."

Coordination works. Multiple specialized agents working through a structured pipeline produce better results than one generalist agent trying to do everything.

None of these are novel ideas. They are the accumulated wisdom of decades of software engineering practice. The novelty is applying them to AI agents -- and discovering that they work just as well for silicon as they do for carbon.

The models are capable enough. What they need is a system.


Companion Repository: github.com/nickbaumann98/ai-dev-operating-system

All code quoted in this post is from real, running source files. The repository includes the full OMC catalog (25 agents with system prompts), Ralph Loop, Specum Pipeline, RALPLAN deliberation, GSD lifecycle, and Team Pipeline. MIT licensed.

#AgenticDevelopment #AIEngineering #ClaudeCode #SoftwareArchitecture #DeveloperTools

