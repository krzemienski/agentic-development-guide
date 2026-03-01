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
