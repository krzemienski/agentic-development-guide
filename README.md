# Agentic Development: 10 Lessons from 8,481 AI Coding Sessions

**A technical blog series on building with AI coding agents at scale.**

Over 90 days, across 10+ projects and 8,481 Claude Code sessions generating 5.5GB of interaction data, I discovered patterns for agentic development that fundamentally changed how I build software. This repository contains 10 deeply technical blog posts — each grounded in real code, real terminal output, and real problem-solving from actual development sessions.

Every diagram is generated as Mermaid or SVG. Every social card is self-contained HTML. Every code snippet comes from production.

---

## The Series

### 1. [Building a Native iOS Client for Claude Code](./01-ils-ios-client/)
763 sessions building a SwiftUI app with a 5-layer streaming bridge, SSE real-time rendering, and the two-character fix that stopped every token from appearing twice.

### 2. [The 5-Layer Bridge: 4 Failed Attempts, 1 Working Architecture](./02-agent-sdk-bridge/)
A debugging war story: Direct SDK, ClaudeCodeSDK, JS SDK, and raw CLI all failed before a Python subprocess bridge connected iOS to Claude's API.

### 3. [Auto-Claude: Spawning 194 Parallel Git Worktrees](./03-auto-claude-worktrees/)
Factory-scale AI development — 194 isolated worktrees, 91 specs, 3,066 sessions, and a QA pipeline that caught what agents missed.

### 4. [How 3 AI Agents Found a Bug I Would Have Shipped](./04-multi-agent-consensus/)
Multi-agent consensus: Lead, Alpha, and Bravo each run independent reviews. A single agent said "looks correct." Three agents caught the P2 streaming duplication bug.

### 5. [The Prompt Engineering Stack: 7 Layers of Defense-in-Depth](./05-prompt-engineering-stack/)
CLAUDE.md hierarchy, 9 rules files, auto-build hooks, 150+ skills, and project memory — a complete system for enforcing development standards with AI agents.

### 6. [Ralph Orchestrator: A Rust Platform for AI Agent Armies](./06-ralph-orchestrator/)
10 Rust crates, a Hat-based context system, event-sourced merge queues, Telegram control plane, and a companion iOS app — coordinating 30+ agents simultaneously.

### 7. [I Banned Unit Tests from My AI Workflow](./07-functional-validation/)
Zero mocks, zero stubs. 470 screenshots, 37+ validation gates, and 3 browser automation tools. Why functional validation catches bugs that unit tests systematically miss.

### 8. [From GitHub Repos to Audio Stories](./08-code-tales/)
A platform that clones repositories, generates narrative scripts with Claude, converts them to audio with ElevenLabs, and delivers 9 storytelling styles — built with 636 commits across 90 worktree branches.

### 9. [21 AI-Generated Screens in One Session](./09-stitch-design-to-code/)
Stitch MCP + Gemini 3 Pro: text prompts to validated React components. A complete web application designed, generated, and validated without Figma.

### 10. [The AI Development Operating System](./10-ai-dev-operating-system/)
The meta-system: OMC orchestration (25+ agent types), Ralph persistence loops, ralph-specum spec pipelines, and GSD project management — composing into a development OS.

---

## What's Inside Each Post

Every topic directory contains:

```
{topic}/
├── README.md              # The full blog post (1,500-2,500 words)
└── visuals/
    ├── hero.html          # 1200x630 hero image (self-contained HTML)
    ├── *.mermaid          # 2-5 architecture/flow diagrams
    ├── *.svg              # Data visualization charts
    ├── social-card-twitter.html   # 1200x628 Twitter/X card
    └── social-card-linkedin.html  # 1200x627 LinkedIn card
```

**Totals:** 22,489 words | 10 hero images | 33 Mermaid diagrams | 10 SVG charts | 20 social cards

## Rendering Visuals

**Mermaid diagrams:** Paste into [mermaid.live](https://mermaid.live) or any Mermaid-compatible renderer (GitHub renders `.mermaid` files natively in markdown).

**HTML cards:** Open in any browser — all CSS is inline, no external dependencies. Screenshot at the specified dimensions for social media.

**SVG charts:** Open in browser or embed directly in web pages.

## About

Written by [Nick Krzemienski](https://github.com/krzemienski) — March 2026.

Built entirely with Claude Code, validated through the same agentic development patterns described in the series.

---

*Every code snippet in this series comes from real production sessions. No fabricated examples, no hypothetical architectures, no made-up metrics.*
