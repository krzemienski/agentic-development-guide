# From GitHub Repos to Audio Stories: Building Code Tales

*How we turned source code repositories into narrated audio experiences with Claude AI, ElevenLabs, and an army of autonomous agents across 636 commits and 90 worktree branches.*

---

## The Vision: Code You Can Listen To

Every GitHub repository tells a story. Behind the commit history, the folder structure, and the dependency graph, there is a narrative about decisions made, problems solved, and systems designed. But that narrative is locked inside files that demand your full visual attention.

Code Tales started with a simple question: what if you could hand a GitHub URL to a system and get back a narrated audio story about what that codebase does, how it works, and why it matters?

Not a text-to-speech dump of a README. Not a robotic recitation of API documentation. An actual story -- with narrative arc, pacing, and a voice that sounds like a human who genuinely understands the code and wants to explain it to you.

The result is a platform that transforms any public GitHub repository into a narrated audio experience. You pick a repo, choose a narrative style, and the system clones the code, analyzes its structure, generates a script with Claude AI, synthesizes speech with ElevenLabs, and streams the finished audio to your device. The entire pipeline runs in under two minutes for most repositories.

This is the story of building that system.

## Architecture: The Generation Pipeline

The core of Code Tales is a linear pipeline that moves a GitHub URL through six discrete stages. Each stage is stateless relative to the others, communicating through well-defined intermediate artifacts.

**Stage 1: URL Intake and Cloning.** The user submits a GitHub repository URL. The backend validates the URL format, checks for accessibility, and performs a shallow clone into a temporary workspace. Shallow cloning keeps the operation fast -- we need the file tree and current content, not the full commit history.

**Stage 2: Repository Analysis.** The cloned repository is scanned to extract structural metadata. This includes the language breakdown, framework detection, directory hierarchy, key configuration files (package.json, Cargo.toml, go.mod, etc.), and a sample of representative source files. The analyzer prioritizes files that reveal architectural intent: entry points, route definitions, model schemas, and configuration.

**Stage 3: Style Selection.** The user chooses one of nine narrative styles. Each style carries a distinct system prompt, tone profile, structural template, and pacing guide. The style selection determines everything about how the final audio will sound -- from vocabulary choices to segment length to the emotional register of the narration.

**Stage 4: Script Generation with Claude.** The repository analysis and the selected style profile are combined into a structured prompt sent to Claude. The model generates a narration script that follows the style's template while accurately representing the codebase. Scripts are typically 1,500 to 4,000 words depending on repository complexity and style requirements.

**Stage 5: Speech Synthesis with ElevenLabs.** The generated script is sent to the ElevenLabs API for text-to-speech synthesis. Each narrative style maps to a specific voice ID and voice settings (stability, similarity boost, style) that match the intended tone. The audio is generated as a streamable format.

**Stage 6: Audio Delivery.** The synthesized audio is streamed to the client. On mobile, a custom audio player handles playback with full transport controls. On web, a standard HTML5 audio element with custom UI handles the stream.

The elegance of this pipeline is its composability. Swapping Claude for a different LLM requires changing one stage. Adding a new voice provider means modifying stage five alone. The intermediate artifacts -- the analysis JSON, the style profile, the script text -- are all inspectable and cacheable.

## Nine Narrative Styles: One Codebase, Nine Stories

The same repository sounds completely different depending on which narrative style you choose. This is not cosmetic variation. Each style represents a fundamentally different approach to explaining code.

**Fiction.** The codebase becomes the setting for a short narrative. Functions become characters. Data flows become journeys. A React application might be narrated as an adventure where the Hero Component ventures through the Router's labyrinth to deliver a Payload to the distant Server Kingdom. It sounds absurd on paper, but the metaphors make architectural patterns surprisingly memorable.

**Documentary.** A measured, authoritative narration in the style of a nature documentary. "Here we observe the load balancer in its natural habitat, distributing incoming requests across a cluster of worker processes..." The tone is observational and precise, treating code as a subject worthy of careful study.

**Tutorial.** A patient, step-by-step walkthrough aimed at someone encountering the codebase for the first time. The narration follows a logical learning path, building understanding incrementally. This style generates the longest scripts because it prioritizes completeness over brevity.

**Podcast.** Casual and conversational, as if two developers were discussing the repo over coffee. The script includes natural digressions, opinions, and the kind of "oh, interesting" moments that make technical podcasts engaging. It is the most accessible style for non-technical listeners.

**Technical.** Precise and information-dense. No metaphors, no narrative framing -- just a clear, systematic description of what the code does, how it is organized, and what patterns it employs. This is the style for developers who want facts delivered efficiently.

**Debate.** Two perspectives argue about the architectural decisions in the codebase. One voice defends the choices made; the other challenges them. "The decision to use a monorepo here introduces coupling that..." / "But the shared type definitions eliminate an entire class of integration bugs..." This style surfaces tradeoffs that a single-perspective narration would gloss over.

**Interview.** Structured as a Q&A session where an interviewer asks questions about the codebase and an expert answers. The format naturally handles the "why" questions that straight narration sometimes skips. "Why did they choose SQLite over Postgres for this use case?"

**Executive.** A high-level summary designed for technical leadership. Focuses on system boundaries, team implications, scaling characteristics, and risk areas. Skips implementation details in favor of strategic observations. Scripts in this style are the shortest -- typically under 1,500 words.

**Storytelling.** A warm, narrative-driven approach that weaves the codebase into a broader context. How does this project fit into the ecosystem? What problem was it born to solve? The storytelling style treats code as a human artifact with history and purpose.

## The Audio Debugging Saga

Building an audio player sounds straightforward until you actually build one. The audio subsystem consumed more debugging effort than any other part of Code Tales -- nine or more commits dedicated solely to fixing race conditions in playback state management.

The root problem was concurrent state mutation. The audio player needed to track multiple overlapping concerns: playback position, buffering status, loading state, error conditions, and user-initiated transport commands (play, pause, seek). These states were initially managed by separate handlers that could fire in any order.

Consider this sequence: the user taps play. The player begins buffering. Before buffering completes, the user taps a different story. The first story's buffer callback fires and attempts to begin playback -- but the player context has already switched to the second story. The result: audio from story A plays while the UI shows story B.

The fix was architectural, not incremental. We replaced the distributed state handlers with a unified `AudioPlayerContext` -- a single source of truth for all playback state, with serialized state transitions that prevented interleaving. Every state change flows through the context, and the context enforces valid transitions. You cannot go from "loading story B" to "playing story A" because the context rejects the transition.

This pattern -- discovering that distributed state management creates race conditions, then consolidating into a single serialized context -- is one of the oldest lessons in concurrent programming. But audio playback has a particular talent for surfacing these bugs because the consequences are immediately perceptible. A visual glitch might go unnoticed. Playing the wrong audio is impossible to miss.

The debugging saga reinforced a design principle we now apply everywhere in Code Tales: state that crosses async boundaries gets a dedicated context object with explicit transition rules. No exceptions.

## Mobile: From SwiftUI to Expo React Native

The mobile client went through a complete platform pivot during development. The initial implementation used SwiftUI -- a natural choice for an iOS-first application with rich audio playback requirements. But SwiftUI was abandoned in favor of Expo React Native.

The decision was pragmatic, not ideological. Code Tales needed to ship on both iOS and Android, and the team's velocity with React Native and Expo was significantly higher than maintaining parallel SwiftUI and Kotlin implementations. The audio playback layer, which had already been debugged extensively, was reimplemented using Expo's AV module with the same unified context pattern.

The final mobile app has nine screens covering the full user journey: repository input, style selection, generation progress, audio playback, library management, and settings. Each screen went through a validation process -- 37 or more validation gates total -- ensuring that every interaction path was tested against real generated audio, not mock data.

The 37 validation gates deserve emphasis. Every screen transition, every error state, every edge case (what happens when generation fails mid-script? what happens when the network drops during audio streaming?) was validated as a discrete checkpoint. This was not unit testing in the traditional sense. It was functional validation: run the real system, trigger the real condition, verify the real behavior.

## Building with AI Agents: 636 Commits Across 90 Worktree Branches

Code Tales was not built by a single developer typing in a single branch. It was built by a coordinated fleet of AI agents operating across 90 worktree branches, generating 636 commits, driven by 91 specification documents.

The orchestration system, which we call Auto-Claude, works as follows. A specification document describes a discrete unit of work: "implement the documentary narrative style with ElevenLabs voice ID X and stability parameter Y" or "add error recovery to the cloning stage when the repository exceeds 500MB." Each spec is precise enough that an AI agent can execute it without ambiguity.

From 91 specs, the system creates worktree branches -- isolated git worktrees that allow parallel development without branch conflicts. Each worktree gets assigned to an agent that reads the spec, implements the changes, runs validation, and marks the task complete. At peak parallelism, dozens of agents operate simultaneously on different worktrees.

The pipeline looks like this: specs are authored (either by humans or by a planning agent), worktrees are created, agents execute in parallel, a QA phase validates each branch independently, and passing branches are merged into the main line. Failed branches get diagnostic reports and re-enter the queue.

This is not theoretical. Code Tales literally could not have been built at this pace without parallel agent execution. 636 commits across 90 branches in a compressed timeline is not achievable by a single developer, regardless of skill. The agents handled the volume; the humans handled the direction.

## RALPLAN: Consensus-Driven Planning

Before agents could execute, someone had to decide what to build. That planning process used RALPLAN -- a structured deliberation protocol where a Planner agent and a Critic agent iterate through three rounds of proposal and challenge.

Round one: the Planner proposes an implementation approach. Round two: the Critic identifies risks, gaps, and unstated assumptions. Round three: the Planner revises the proposal to address the Critic's concerns, and the Critic either approves or escalates remaining issues.

Three iterations is the sweet spot. Fewer rounds leave obvious gaps in the plan. More rounds produce diminishing returns as the Planner and Critic converge. The output of RALPLAN is a plan document that has survived adversarial review -- not a first draft, but a third draft that has already addressed its own weaknesses.

For Code Tales, RALPLAN was used for every major architectural decision: the pipeline stage boundaries, the style system design, the audio context unification, and the mobile platform pivot. Each decision has a paper trail showing what was proposed, what was challenged, and what was ultimately decided.

## AGENTS.md: A Map for AI Navigation

With 91 specs and 90 branches, AI agents needed a way to understand the codebase without reading every file. The solution was `AGENTS.md` -- a hierarchical navigation document placed at the repository root that describes the project structure, key files, architectural patterns, and conventions.

Think of AGENTS.md as a README written specifically for AI agents rather than human developers. It answers the questions an agent asks when it lands in an unfamiliar codebase: Where is the entry point? What is the directory structure? What naming conventions are used? Where do new features go? What are the testing expectations?

The hierarchical structure means that AGENTS.md files can exist at multiple levels of the directory tree. The root AGENTS.md provides the global overview. A subdirectory AGENTS.md provides local context for that module. An agent working on the audio player reads the root AGENTS.md for project context and the `audio/AGENTS.md` for module-specific guidance.

This pattern proved essential for agent productivity. Without AGENTS.md, agents spent significant time exploring the codebase before making changes. With it, they could begin productive work almost immediately.

## Product Strategy as Machine-Readable JSON

An unconventional decision in Code Tales was encoding product strategy as machine-readable JSON rather than prose documents. Feature priorities, target user segments, success metrics, and roadmap items all live in structured JSON files that both humans and agents can parse.

This means that when a planning agent needs to decide whether a feature request aligns with product goals, it can query the strategy JSON directly rather than interpreting a natural-language product document. The result is more consistent prioritization and fewer "that's not what the product doc meant" misunderstandings between human stakeholders and AI agents.

## Results and Reflections

Code Tales ships with nine narrative styles, a sub-two-minute generation pipeline, a battle-tested audio player, and native mobile apps for iOS and Android. The system has processed repositories ranging from tiny CLI utilities to large monorepos with hundreds of thousands of lines of code.

The numbers tell one story: 636 commits, 90 worktree branches, 91 specs, 9 narrative styles, 37 validation gates. But the more interesting story is about the process. Code Tales is an existence proof that AI-agent-driven development at scale is not a future possibility -- it is a present reality. The agents did not replace human judgment. They amplified human intent across a surface area that no individual could cover alone.

The audio debugging saga is perhaps the most instructive episode. Nine commits to fix race conditions in an audio player is a lot. But each of those commits was identified, implemented, and validated by agents working in parallel. A human developer would have spent days on the same problem. The agents compressed it into hours.

If you hand Code Tales a GitHub URL today, you will get back a narrated audio story in under two minutes. That story will be accurate, well-paced, and delivered in a voice that matches the narrative style you chose. Behind that experience is a pipeline built by agents, planned through adversarial deliberation, and validated through 37 gates that accept nothing less than real, functional evidence.

The code tells the story. Code Tales just gives it a voice.

---

## Companion Repository

**[`code-tales`](https://github.com/krzemienski/code-tales)** — Transform GitHub repositories into narrated audio stories with 9 narrative styles

---

*Part 8 of 10 in the **Agentic Development** series — [View all posts](https://github.com/krzemienski/agentic-development-guide)*

*Nick Krzemienski — March 2026*
