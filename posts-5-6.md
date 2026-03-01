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
