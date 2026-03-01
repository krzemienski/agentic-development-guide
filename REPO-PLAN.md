# GitHub Companion Repos — Production Applications

Each repo is a standalone, usable tool/framework that demonstrates the blog post's core concept.

## 1. `claude-ios-streaming-bridge`
**Post**: Building a Native iOS Client for Claude Code
**What it is**: A reusable Swift package + Python bridge for connecting iOS/macOS apps to Claude Code via SSE streaming
**Key files**:
- `Sources/StreamingBridge/SSEClient.swift` — Two-tier timeout SSE client
- `Sources/StreamingBridge/ClaudeExecutorService.swift` — Process spawning with env sanitization
- `scripts/sdk-wrapper.py` — Python Agent SDK wrapper (8 lines)
- `Example/` — Minimal SwiftUI chat app demonstrating the bridge
- `README.md` — Architecture diagram, setup guide, usage

## 2. `claude-sdk-bridge`
**Post**: The 5-Layer Bridge: 4 Failed Attempts, 1 Working Architecture
**What it is**: A documented reference implementation showing the 5-layer bridge pattern with failure case examples
**Key files**:
- `failed-attempts/` — Code from each failed approach with explanations
- `working-bridge/` — The production Python subprocess bridge
- `docs/failure-modes.md` — Catalog of RunLoop/NIO, nesting detection, NSTask pitfalls
- `README.md` — Failure timeline diagram, architecture guide

## 3. `auto-claude-worktrees`
**Post**: Auto-Claude: Spawning 194 Parallel Git Worktrees
**What it is**: A CLI tool for automated parallel AI development using git worktrees
**Key files**:
- `src/ideate.py` — Task ideation from codebase analysis
- `src/specgen.py` — Spec generation from task descriptions
- `src/factory.py` — Worktree creation and agent spawning
- `src/qa.py` — QA pipeline with approve/reject/fix cycles
- `src/merge.py` — Priority-weighted merge queue
- `README.md` — Pipeline diagram, usage guide, configuration

## 4. `multi-agent-consensus`
**Post**: How 3 AI Agents Found a Bug I Would Have Shipped
**What it is**: A Claude Code framework for 3-agent consensus validation with hard gates
**Key files**:
- `consensus/gate.py` — Gate implementation (independent validation + unanimity check)
- `consensus/roles.py` — Alpha, Bravo, Lead role definitions with specialized prompts
- `consensus/evidence.py` — Evidence artifact collection and verification
- `consensus/orchestrator.py` — Phase pipeline with gate checkpoints
- `examples/streaming-audit/` — The actual ILS P2 bug detection as a worked example
- `README.md` — Consensus flow diagram, when-to-use guide

## 5. `claude-prompt-stack`
**Post**: The Prompt Engineering Stack: 7 Layers of Defense-in-Depth
**What it is**: A template repository for setting up the full 7-layer defense-in-depth system
**Key files**:
- `CLAUDE.md` — Example project-level CLAUDE.md with no-mocks mandate
- `.claude/rules/` — 9 template rules files (project, agents, workflow, git, build-hook, feature-gate, fastlane, performance, security)
- `.claude/hooks/` — Auto-build hook + pre-commit security hook scripts
- `skills/` — Example skill definitions for functional validation
- `MEMORY.md` — Example project memory template
- `setup.sh` — One-command setup for the full stack
- `README.md` — 7-layer diagram, setup walkthrough

## 6. `ralph-orchestrator-guide`
**Post**: Ralph Orchestrator: A Rust Platform for AI Agent Armies
**What it is**: Getting started guide and example configurations for Ralph Orchestrator
**Key files**:
- `configs/` — Example hat topologies (web-frontend, systems, data-pipeline)
- `examples/basic-loop/` — Minimal Ralph loop configuration
- `examples/parallel-agents/` — Multi-worktree parallel execution
- `examples/telegram-bot/` — Telegram control plane setup
- `docs/tenets.md` — The 6 tenets with practical examples
- `README.md` — Architecture overview, quickstart, hat system guide

## 7. `functional-validation-framework`
**Post**: I Banned Unit Tests from My AI Workflow
**What it is**: A validation framework for AI-assisted development — browser + iOS automation + gate system
**Key files**:
- `validators/browser.py` — agent-browser, Puppeteer MCP, Playwright MCP integration
- `validators/ios.py` — idb-based iOS accessibility tree validation
- `gates/gate.py` — Numbered gate system with evidence requirements
- `gates/evidence.py` — Screenshot capture, curl verification, accessibility tree dumps
- `templates/` — Gate definition templates for common patterns
- `CLAUDE.md` — The no-mocks mandate as a reusable instruction
- `README.md` — Philosophy, tool comparison, gate workflow

## 8. `code-tales`
**Post**: From GitHub Repos to Audio Stories
**What it is**: The Code Tales platform — GitHub repos to narrated audio stories
**Key files**:
- `pipeline/clone.py` — Repository cloning and analysis
- `pipeline/analyze.py` — Structure extraction and metadata generation
- `pipeline/narrate.py` — Claude script generation with 9 narrative styles
- `pipeline/synthesize.py` — ElevenLabs TTS integration
- `styles/` — 9 style definitions (fiction, documentary, tutorial, podcast, technical, debate, interview, executive, storytelling)
- `app/` — React Native/Expo mobile app shell
- `README.md` — Pipeline diagram, style showcase, API guide

## 9. `stitch-design-to-code`
**Post**: 21 AI-Generated Screens in One Session
**What it is**: A workflow template for Stitch MCP + AI design-to-code with Puppeteer validation
**Key files**:
- `design-system/tokens.json` — Design system as machine-readable tokens
- `prompts/` — Prompt templates for each screen type (public, auth, user, admin, legal)
- `validation/puppeteer-checks.js` — 107-action validation suite
- `components/` — Example shadcn/ui components generated from the workflow
- `docs/branding-checklist.md` — Lessons from the branding propagation bug
- `README.md` — Workflow diagram, design system guide, session economics

## 10. `ai-dev-operating-system`
**Post**: The AI Development Operating System
**What it is**: A complete starter kit for building your own AI development OS
**Key files**:
- `omc/` — Agent catalog definitions and model routing configuration
- `ralph-loop/` — Persistence loop implementation with stop hooks
- `specum/` — Specification pipeline (requirements → design → tasks → implement → verify)
- `ralplan/` — Adversarial planner-critic deliberation protocol
- `gsd/` — 10-phase project management with assumption tracking
- `team-pipeline/` — Staged multi-agent execution (plan → prd → exec → verify → fix)
- `README.md` — Meta-system architecture, incremental adoption guide

---

## GitHub Organization

All repos under: `github.com/krzemienski/`

Naming convention: lowercase-kebab-case, descriptive, no abbreviations

Each repo gets:
- MIT license
- Proper .gitignore
- README with architecture diagrams (Mermaid)
- "Part of the Agentic Development series" badge linking to main repo
