# The Claude Agent SDK Bridge: Why I Needed 5 Layers to Call an API

Sometimes the simplest-sounding problem -- "call an API and stream the response" -- turns into a multi-week odyssey through runtime incompatibilities, environment variable landmines, and framework assumptions that only surface at the worst possible moment. This is the story of how my iOS app's chat feature went from a single HTTP call to a five-layer bridge architecture, and why every layer exists because something simpler tried to kill it.

## The Goal

ILS is a native iOS/macOS client for Claude Code. The core feature is straightforward: the user types a message, the app sends it to Claude, and the response streams back in real time. Server-sent events. Partial messages. The kind of thing every chat app does.

I estimated this would take an afternoon.

It took 22 debugging sessions across 10 days.

## Attempt 1: Just Call the API Directly

The obvious first move. Anthropic publishes `@anthropic-ai/sdk` for JavaScript. Import it, pass the API key, stream the response. Three lines of meaningful code.

```javascript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
const stream = await client.messages.stream({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 4096,
  messages: [{ role: 'user', content: prompt }]
});
```

Dead on arrival. Claude Code does not use API keys. It authenticates through OAuth -- a browser-based flow that produces session tokens managed by the CLI itself. There is no `ANTHROPIC_API_KEY` environment variable to read. I could have asked users to create one through the Anthropic console, but that defeats the entire point of ILS: it is a *client for Claude Code*, not a separate Anthropic API consumer. The authentication must flow through Claude Code's own credential chain.

**Failure mode**: No API key available. OAuth tokens are managed internally by the Claude CLI and not exposed to third-party consumers.

**Time lost**: Half a day. The least painful failure on this list.

## Attempt 2: The ClaudeCodeSDK in Swift

Anthropic ships a Swift SDK -- `ClaudeCodeSDK` -- designed for exactly this scenario. It wraps the Claude CLI process, handles authentication, and provides a publisher-based streaming interface using Combine's `PassthroughSubject`.

Perfect. My backend is Vapor (Swift). Same language, same ecosystem. I integrated the SDK, wired up the publisher, and hit run.

Nothing happened.

No errors. No crashes. No output. The publisher simply never emitted a single value.

After hours of debugging, I found the root cause buried in the SDK's implementation. `ClaudeCodeSDK` uses `FileHandle.readabilityHandler` to read stdout from the subprocess. That handler dispatches events through `RunLoop`. The events then flow through a Combine `PassthroughSubject` which also depends on `RunLoop` scheduling.

Vapor runs on SwiftNIO. SwiftNIO uses its own event loop implementation -- `EventLoop`, not `RunLoop`. The NIO event loops never pump `RunLoop`. So the `readabilityHandler` fires, the `PassthroughSubject` tries to emit, and the emission disappears into a scheduling void. The subscriber callback never executes.

This is the kind of bug that produces zero diagnostic output. Everything initializes correctly. The process spawns. Claude receives the prompt and generates a response. The bytes arrive on stdout. They are read by the `FileHandle` handler. They are pushed into the `PassthroughSubject`. And then they vanish, because no `RunLoop` iteration ever delivers them to the subscriber.

```swift
// What the SDK does internally (simplified)
let pipe = Pipe()
pipe.fileHandleForReading.readabilityHandler = { handle in
    let data = handle.availableData
    // This fires correctly...
    subject.send(data)
    // ...but the subscriber never receives it
    // because Vapor's NIO event loops don't pump RunLoop
}
```

I tried workarounds: manually pumping `RunLoop` on a background thread, wrapping the subscription in a `DispatchQueue.main.async`, even creating a dedicated `Thread` with its own `RunLoop`. Each workaround introduced new problems -- deadlocks, event ordering issues, or simply moving the silent failure to a different layer.

**Failure mode**: RunLoop/NIO impedance mismatch. Combine publishers require RunLoop scheduling. Vapor's SwiftNIO event loops do not pump RunLoop. Publisher emissions are silently dropped.

**Time lost**: Two full days. Most of that time was spent verifying that the SDK *was* receiving data -- adding logging to every layer, confirming the `PassthroughSubject.send()` was being called, and slowly narrowing the gap to the subscriber side. The silence was the hardest part. If the SDK had thrown an error or logged a warning, this would have been a 30-minute fix.

## Attempt 3: Bypass the SDK, Use Process Directly

If the SDK's abstraction is the problem, remove the abstraction. I wrote a direct `Process` invocation with `DispatchQueue`-based stdout reading -- no RunLoop dependency, no Combine, just GCD dispatch handlers on a pipe.

```swift
let process = Process()
process.executableURL = URL(fileURLWithPath: "/usr/local/bin/claude")
process.arguments = ["-p", prompt, "--output-format", "stream-json"]

let pipe = Pipe()
process.standardOutput = pipe

pipe.fileHandleForReading.readabilityHandler = { handle in
    let data = handle.availableData
    guard !data.isEmpty else { return }
    DispatchQueue.global().async {
        self.parseAndForward(data)
    }
}

try process.run()
```

This worked -- on my development machine, running the backend standalone. The moment I ran it inside an active Claude Code session, it failed silently again.

Claude CLI has a nesting detection mechanism. When you run `claude` from within a Claude Code session, the CLI checks for the environment variable `CLAUDECODE=1` and several `CLAUDE_CODE_*` prefixed variables. If it detects these, it refuses to spawn a nested instance. The behavior is intentional -- recursive Claude Code sessions could create runaway process trees and circular reasoning loops. But my backend was running *inside* Claude Code (I was developing it in a Claude Code session), and the spawned subprocess inherited the parent environment.

This was the most insidious failure because it was environment-dependent. The code worked perfectly when I ran the backend from a terminal. It failed silently when the backend was spawned from within Claude Code. Same binary, same arguments, same database -- different environment variables. I spent hours comparing the two environments before I noticed the `CLAUDECODE=1` variable in the inherited environment.

The fix was surgical:

```swift
// ClaudeExecutorService.swift -- strip nesting detection env vars
var env = ProcessInfo.processInfo.environment
env.removeValue(forKey: "CLAUDECODE")
env = env.filter { !$0.key.hasPrefix("CLAUDE_CODE_") }
process.environment = env
```

Three lines. Ten hours to find them.

**Failure mode**: Claude CLI nesting detection. The subprocess inherits `CLAUDECODE=1` from the parent environment, causing the CLI to silently refuse execution.

## The NSTask Crash Detour

While debugging Attempt 3, I hit another sharp edge that deserves its own callout because it will bite anyone working with `Process` (née `NSTask`) in Swift.

After reading all output from the process's stdout pipe, I accessed `process.terminationStatus` to check the exit code.

Crash. `NSInvalidArgumentException`.

The issue: reading EOF from stdout does not mean the process has exited. There is a race condition between the pipe closing (all output consumed) and the process actually terminating. Accessing `terminationStatus` on a still-running `Process` throws an Objective-C exception that Swift does not catch gracefully.

The fix is a single line, but you have to know it exists:

```swift
process.waitUntilExit()  // MUST call before accessing terminationStatus
let status = process.terminationStatus
```

This is documented in Apple's `NSTask` reference if you know where to look, but the crash message -- `NSInvalidArgumentException` -- is misleading enough to send you down the wrong diagnostic path. You start looking for argument validation issues when the real problem is a temporal one.

## Attempt 4: The Python Bridge

With the environment variable fix, the direct Process approach worked for invoking the CLI. But the raw Claude CLI output needed careful parsing, error handling, and streaming support with partial message delivery. I was writing an increasingly complex Swift parser for Claude's streaming JSON format when I realized Anthropic had already solved this problem -- just not in Swift.

The `claude-agent-sdk` Python package wraps the CLI interaction cleanly with an async iterator interface and `include_partial_messages=True` for real-time token-by-token streaming.

```python
# scripts/sdk-wrapper.py
import json, sys, asyncio
from claude_agent_sdk import query

async def main():
    prompt = sys.argv[1]
    async for event in query(prompt=prompt, include_partial_messages=True):
        print(json.dumps(event), flush=True)

asyncio.run(main())
```

Eight lines. The wrapper reads a prompt from argv, calls `query()` with partial message streaming enabled, and writes each event as a JSON line (NDJSON) to stdout. The `flush=True` is critical -- without it, Python's output buffering delays events by unpredictable amounts and breaks the real-time streaming experience. Users would see nothing for seconds, then a burst of text, then nothing again.

On the Swift side, `ClaudeExecutorService` spawns `python3 sdk-wrapper.py` with the sanitized environment, reads NDJSON lines from stdout, parses each line into a structured event, and converts them to SSE events that the Vapor route handler streams to the iOS client.

This worked. Every time. In every environment.

## The Final Architecture

Here is what "call an API" became:

```
iOS App (SwiftUI)
    |  HTTP POST /api/v1/chat/stream
    v
Vapor Backend (Swift/NIO)
    |  Process() spawn, sanitized env
    v
Python sdk-wrapper.py
    |  claude_agent_sdk.query() async iterator
    v
Claude CLI
    |  OAuth-authenticated HTTP
    v
Anthropic API
```

Five layers. Each one exists because the layer above it could not talk to the layer below it without an intermediary.

**Layer 1 -- iOS to Vapor**: Standard HTTP with SSE streaming. The only layer that worked on the first try.

**Layer 2 -- Vapor to Python**: Required because the Swift SDK (`ClaudeCodeSDK`) has a RunLoop dependency incompatible with Vapor's NIO runtime. Process spawning with GCD-based stdout reading avoids the impedance mismatch. Environment variable stripping prevents nesting detection failures.

**Layer 3 -- Python wrapper**: Required because the raw Claude CLI output needs structured parsing with partial message support. The Python `claude-agent-sdk` provides this with a clean async iterator. The wrapper also serves as an isolation boundary -- if the SDK changes its interface, only the 8-line Python script needs updating.

**Layer 4 -- Claude CLI**: Required because Claude Code uses OAuth authentication, not API keys. The CLI is the only consumer-accessible interface that handles the OAuth token chain correctly.

**Layer 5 -- Anthropic API**: The actual destination. The only layer that was never a problem.

The response path flows in reverse, with each layer translating the format:

```
Anthropic API  -->  Claude CLI  -->  stdout  -->
Python (json.dumps)  -->  stdout NDJSON  -->
Swift (line parse)  -->  SSE events  -->  iOS (EventSource)
```

## Performance Profile

The five-layer architecture has measurable overhead, and I want to be honest about the cost:

| Path | Latency | Notes |
|------|---------|-------|
| Direct API (hypothetical) | ~1-3s | If API key auth were available |
| Python bridge, cold start | ~12s | Process spawn + interpreter + SDK init |
| Python bridge, warm | ~2-3s | Subsequent calls in same backend session |
| Cost per query | ~$0.04 | Claude Sonnet, typical chat message |

The cold start penalty is real. Twelve seconds for the first message in a session is noticeable. Most of that time is Python interpreter startup and the SDK establishing its connection through the CLI. Subsequent queries drop to 2-3 seconds, which is acceptable for a chat interface where the user is reading the previous response anyway.

I considered keeping a warm Python process alive between requests with a stdin/stdout protocol, but the complexity of managing process lifecycle, health checks, stdin framing, and crash recovery outweighed the cold start savings for now. The current architecture treats each request as stateless -- spawn, execute, collect, terminate. Stateless is debuggable. And when something goes wrong, the blast radius is a single request, not a corrupted long-lived process.

## What I Learned

**Environment variables are ambient authority.** The `CLAUDECODE=1` nesting detection is a sensible safety mechanism, but it demonstrates how environment variables create invisible coupling between processes. The subprocess did not ask to be inside a Claude Code session. It inherited that context silently and failed silently. If you are spawning subprocesses from a server context, audit the inherited environment. Always.

**RunLoop is not a universal scheduler.** Apple's documentation treats `RunLoop` as the default scheduling mechanism for Foundation callbacks. But modern Swift server frameworks -- Vapor, Hummingbird, anything built on SwiftNIO -- use cooperative task executors and NIO event loops. Any library that assumes `RunLoop` availability is quietly incompatible with server-side Swift. This is not documented as a limitation anywhere I could find. It manifests as "the callback never fires" with zero diagnostic output.

**Silent failures are worse than crashes.** The RunLoop/NIO mismatch produced zero errors. The nesting detection produced zero errors. The `terminationStatus` race condition produced a crash, and paradoxically, that was the easiest bug to fix. When something explodes, you know where to look. When something silently does nothing, you question whether you understand how computers work.

**Process lifecycle has sharp edges.** `waitUntilExit()` before `terminationStatus`. `flush=True` on Python stdout. Sanitizing inherited environment variables. Each of these is a one-line fix that takes hours to discover. The gap between "the process ran" and "the process ran correctly in all environments" is surprisingly wide.

**Indirection is not always bad.** Five layers sounds excessive when you draw it on a whiteboard. But each layer is independently testable, the failure mode at each boundary is well-understood, and replacing any single layer requires changes to exactly one integration point. The architecture is a stack of simple translations, not a monolith of coupled concerns. When the Python SDK updates its interface, I change 8 lines. When Anthropic eventually exposes OAuth tokens for third-party use, I can collapse layers 3 and 4 into a direct API call without touching the iOS client or the Vapor route handler.

**The simplest solution is the one that works.** I spent days trying to eliminate layers -- using the Swift SDK directly, calling the API without the CLI, avoiding the Python intermediary. Each simplification introduced a new incompatibility. The five-layer bridge is the simplest architecture that actually functions given the real constraints of OAuth authentication, NIO event loops, and CLI nesting detection. Simplicity is not about minimizing the number of components. It is about minimizing the number of things that can fail in ways you cannot diagnose.

---

*ILS is a native iOS/macOS client for Claude Code. The chat streaming bridge described here shipped in the v2 release and has been running in production for three weeks. The project memory file for this feature, including all 22 debugging session notes, is preserved in the repository's `.omc/evidence/` directory.*
