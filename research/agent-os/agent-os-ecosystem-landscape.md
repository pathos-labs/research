# Agent OS Ecosystem Landscape Scan
_Research conducted: 2026-03-02 | Overall shelf-life: Red — Perishable (likely outdated within 4-8 weeks)_
_Track 7 of 8: Agent OS Research Project_

## Executive Summary

The agent infrastructure landscape has undergone a tectonic shift in the five weeks since February 26, 2026. Three developments dominate: (1) OpenClaw's security crisis deepened with the ClawHavoc supply chain attack compromising 1,184+ skills on ClawHub, accelerating a Cambrian explosion of security-conscious alternatives; (2) enterprise platforms went to cloud-native execution, with Cursor launching cloud agents on isolated VMs (35% of internal PRs), Warp shipping Oz for cloud-native agent sandboxes, and GitHub making its Copilot coding agent GA with model picker and self-review; (3) Claude Code itself expanded significantly with Remote Control (Feb 24), Agent Teams (TeammateTool), and a maturing SDK ecosystem that makes the "build your own harness" path increasingly viable.

The *Claw ecosystem now contains 30+ named forks and reimplementations across six programming languages. NullClaw (Zig, 678KB binary), ZeroClaw (Rust, ~16MB binary), and PicoClaw (Go, 21K+ stars) have emerged as the most architecturally distinct alternatives. The abstraction layer pattern is gaining traction, with projects like harness-cli and agent-mux enabling cross-engine orchestration. Meanwhile, established agent frameworks (LangGraph, CrewAI, AutoGen) continue evolving but remain complementary to, not competitive with, the terminal-native harness paradigm.

The direction is clear: agent infrastructure is converging on cloud-native execution with local control planes, multi-agent orchestration as a first-class primitive, and security as a non-negotiable baseline rather than an afterthought.

---

## Domain Map

This scan covers the ecosystem of tools, frameworks, and platforms for running AI coding agents in production. The field sits at the intersection of developer tooling, cloud infrastructure, and LLM orchestration. Key players include Anthropic (Claude Code, Agent SDK), OpenAI (Codex CLI), Google (Gemini CLI, Jules), Cursor/Anysphere, Warp, Cognition (Devin), GitHub (Copilot coding agent), and a fast-growing open-source ecosystem centered around the *Claw family of OpenClaw forks. The major tension is between fully managed cloud agents (Devin, Cursor Cloud, Warp Oz) and self-hosted, composable harnesses (Claude Code SDK, OpenClaw derivatives). Security has become the defining fault line after OpenClaw's CVE-2026-25253 and the ClawHavoc supply chain attack.

---

## 1. *Claw Ecosystem Updates (Since Feb 26)

### 1.1 OpenClaw: Security Crisis and Resilience

OpenClaw remains the category-defining project (200K+ GitHub stars, 35K+ forks) but has been battered by security incidents that fundamentally changed the ecosystem's trajectory. **[Confidence: Strong, well-documented across multiple security research firms]**

**CVE-2026-25253** (CVSS 8.8): A critical remote code execution vulnerability disclosed in mid-February. The attack chain exploits OpenClaw's WebSocket gateway, which fails to validate connection origins. A malicious webpage can steal the authentication token during the WebSocket handshake, then replay it to gain full control of the agent — including shell command execution. Because OpenClaw runs with full system access, token theft equals machine compromise. Patches shipped in v2026.2.1 (Feb 19). Multiple security firms (SonicWall, runZero, Immersive Labs) published detailed analyses.

**CVE-2026-26329**: A path traversal vulnerability in OpenClaw's file handling, discovered shortly after.

**CVE-2026-25474**: An additional vulnerability affecting versions prior to 2026.2.1, patched alongside the others.

**ClawHavoc Supply Chain Attack**: The most damaging incident. Threat actors registered as ClawHub developers and uploaded 1,184+ malicious skills to OpenClaw's marketplace. The campaign used ClickFix-style social engineering, tricking users into executing commands that downloaded malware. An independent audit found 41.7% of 2,890+ ClawHub skills contain serious security vulnerabilities. The attack specifically targeted always-on machines (Mac minis running OpenClaw 24/7) — the exact "Agent OS" deployment pattern. **[Confidence: Established -- documented by SC Media, Aryaka, eSecurity Planet, Immersive Labs]**

**Release 2026.2.23** (Feb 23): Security patches plus Claude Opus 4.6 support. Anthropic clarified that Claude accounts can still be used with OpenClaw, NanoClaw, and similar tools — settling community fears after initial TOS confusion.

**Net assessment**: OpenClaw's feature set remains unmatched in breadth (430K+ lines, dozens of channels, companion apps, voice, canvas, sandboxing), but its security posture is a serious liability for production deployment. The ClawHavoc attack demonstrated that the skill marketplace model is fundamentally difficult to secure at scale. For an Agent OS build, OpenClaw's architecture is instructive but its security model is disqualifying without significant hardening.

### 1.2 NanoClaw: Maturing Quietly

NanoClaw, the ~500-line minimalist fork by Gavriel Cohen (Qwibit), has continued steady development since Feb 26. **[Confidence: Strong]**

**Key updates since Feb 26**:
- **Channel expansion**: Discord and Signal support added via the skills system (not merged into core — by design). Slack also supported. The canonical channels remain WhatsApp, Telegram, and headless.
- **Skills-based extension model hardened**: NanoClaw explicitly discourages direct PRs for feature additions. Instead, contributors create Skills (.claude/skills/) that teach a local AI assistant how to extend the codebase. This "AI-native development" approach means the codebase stays small while capabilities grow.
- **Container isolation**: Each agent runs in its own Docker container with filesystem isolation, separate memory, and its own context window. This directly addresses OpenClaw's security failures.
- **Claude Agent SDK integration**: Runs directly on Anthropic's official SDK, keeping it within TOS.

**Limitations**: Single-provider (Anthropic only). No OpenRouter, no local models. The VentureBeat profile (Feb 2026) notes this as a deliberate constraint, not a gap — NanoClaw optimizes for Claude specifically.

**Star count**: Moderate (lower than PicoClaw or ZeroClaw) but punches above its weight in production use. Cohen runs his own marketing agency (Qwibit) on NanoClaw agents.

### 1.3 NullClaw (Zig): Full Architecture Analysis

NullClaw is the most technically ambitious reimplementation, written entirely in Zig and compiling to a single 678KB static binary with zero runtime dependencies. **[Confidence: Strong -- verified across GitHub repo, multiple technical reviews, TikTok developer coverage]**

**Architecture**:
- **Language**: Zig (zero-dependency, no allocator overhead, no garbage collection)
- **Binary size**: 678KB (ReleaseSmall), static — no libc, no runtime
- **Boot time**: Sub-2ms on commodity hardware
- **Memory**: Extremely low footprint, runs on $5 ARM boards
- **Subsystem design**: Vtable interfaces for every subsystem. Providers, channels, tools, memory, sandbox, tunnels — all swappable through compile-time interfaces
- **Feature scope**: 22+ AI providers, 17-18 messaging channels, 18+ tools, hybrid vector+FTS5 memory, multi-layer sandbox, tunnels, hardware peripherals, MCP support, subagents, streaming, voice
- **Observability**: Built-in telemetry, health registry, trace compression, cost auditing — no external dependencies or sidecar processes

**What it can do that others can't**: NullClaw's value proposition is deploying the full assistant infrastructure stack on hardware that would choke on Node.js or Python. A $5 VPS, a Raspberry Pi, an IoT device — NullClaw boots in milliseconds and uses kilobytes of RAM where OpenClaw needs gigabytes. The Zig compiler's cross-compilation support means a single `zig build` produces binaries for ARM, x86, and RISC-V.

**Current state**: Active development, growing community (4K+ stars per earlier reports). GitHub mirror and Codeberg mirror both active. The r/Zig community is engaged.

**Assessment for Agent OS**: NullClaw is the most deployment-friendly option for resource-constrained environments. For a remote server harness, the single static binary with zero dependencies eliminates an entire class of deployment issues. The tradeoff is that Zig's ecosystem is smaller than Rust or Go, making contributions harder to attract.

### 1.4 ZeroClaw (Rust): Security-First Architecture

ZeroClaw positions itself as "claw done right" — a Rust-based agent runtime emphasizing security, portability, and clean architecture. **[Confidence: Strong -- verified across official sites, GitHub (17K stars, 2K forks), DEV Community, Cloudron forum]**

**Architecture**:
- **Language**: Rust (memory safety, no GC)
- **Binary size**: ~16MB single binary (larger than NullClaw but still very lean)
- **Three operational modes**: Agent (CLI), Gateway (HTTP), Daemon (gateway + channels + heartbeat + scheduler)
- **Trait-driven design**: Every subsystem (providers, channels, tools, memory, tunnels) swappable through Rust traits
- **Security model**: Pairing-based gateways, strict sandboxing, explicit allowlists, workspace scoping, encrypted secrets at rest
- **Memory options**: SQLite, markdown, or none — with configurable vector+keyword hybrid search
- **Autonomy levels**: readonly, supervised (default), full — configurable per deployment
- **Config**: TOML-based (~/.zeroclaw/config.toml), simple onboarding wizard

**OpenClaw migration compatibility**: ZeroClaw supports the AI Entity Object Specification (AIEOS) for portable, standardized AI personas. Migration guides exist. The Rust trait architecture means OpenClaw integrations can be ported incrementally.

**Platforms**: ARM, x86, RISC-V (portable Rust architecture). 27+ contributors.

**Assessment for Agent OS**: ZeroClaw is the strongest candidate for a security-conscious production deployment. The trait-driven architecture makes it genuinely extensible without touching core code. The Rust ecosystem ensures memory safety without GC pauses. The three operational modes (agent/gateway/daemon) map directly onto Agent OS deployment patterns.

### 1.5 PicoClaw (Go): The Edge Computing Play

PicoClaw is backed by Sipeed, a RISC-V hardware company, and has exploded to 21,337 GitHub stars — making it the fastest-growing *Claw alternative by raw numbers. **[Confidence: Strong -- verified via GitHub, CNX Software hardware review, Medium technical deep-dives]**

**Architecture**:
- **Language**: Go (93.3% of codebase)
- **Origin**: Refactored from Nanobot (Python) by AI agents in a self-bootstrapping process — 95% of code was agent-generated with human-in-the-loop refinement
- **Binary**: Single static binary, cross-compiles to RISC-V, ARM, x86
- **Memory**: <10MB RAM (99% less than OpenClaw, 90% less than Nanobot)
- **Boot time**: 1 second on 600MHz core (400x faster than OpenClaw on equivalent hardware)
- **Target hardware**: $10 boards (Sipeed LicheeRV Nano), Raspberry Pi, cheap VPS
- **Codebase size**: ~4,000 lines of Go — fully auditable

**Comparison table (from Sipeed)**:

| | OpenClaw | Nanobot | PicoClaw |
|---|---|---|---|
| Language | TypeScript | Python | Go |
| RAM | 1GB | 100MB | <10MB |
| Boot (800MHz) | 500s | ~30s | ~1s |

**What makes it different**: PicoClaw is the first *Claw project with a hardware company behind it. Sipeed sells the boards it runs on. This creates a unique business model: sell cheap RISC-V hardware as a "personal AI appliance" running PicoClaw. The self-bootstrapping origin story (AI ported the codebase from Python to Go) is both a marketing hook and a genuine architectural experiment.

**Caution**: The project explicitly warns it is in early development with unresolved network security issues, and recommends against production deployment before v1.0. Multiple `.ai/.org/.com` domains registered by third parties (not official), and there are cryptocurrency scams trading fake PicoClaw tokens. **[Confidence: Established -- official repo warning]**

### 1.6 The Long Tail: BabyClaw, TurboClaw, ClawSwarm, IronClaw, TinyClaw, and Others

The *Claw ecosystem now contains 30+ named projects. A dedicated comparison site (clawclones.com) tracks them across metrics including boot time, memory usage, security score, and community sentiment. **[Confidence: Moderate -- some projects are very new with low star counts]**

**BabyClaw** (TypeScript, 1 star): "Same lobster spirit, ~5% of the complexity." Built on Vercel AI SDK, SQLite via Drizzle ORM, Telegram via grammY. Scheduler with cron support. ClawHub-compatible skills. Explicitly targets developers who want to understand the full codebase end-to-end. Very early, barely a proof of concept.

**TurboClaw** (TypeScript/Bun, 2 stars): Clean rewrite focused on documentation and reliability. Runs multiple Claude Code agents simultaneously via Telegram. Built on Bun for fast startup. Key differentiator: uses Claude Code directly (not the API), staying within subscription TOS. Simple by design — one config, one daemon, one CLI.

**ClawSwarm** (Rust, 2 stars): Built by The Swarm Corporation on the Swarms framework. Natively multi-agent, compiles to Rust, unified messaging across Telegram/Discord/WhatsApp. Architectural interest as a multi-agent-first design rather than single-agent-with-orchestration.

**IronClaw**: Mentioned in multiple comparison lists alongside NanoClaw and ZeroClaw. Launched Jan 31, 2026 under MIT license, crossed 7K stars in first week. Container-isolated design where each chat group gets its own sandboxed Docker container.

**TinyClaw** (TypeScript, ~2.8K stars): Positioned between OpenClaw and the minimalist alternatives. 85MB memory, 180ms boot time, 72/100 security score per ClawClones comparison. A middle-ground option.

**Other named projects**: MimiClaw (C, 3.6K stars, 8ms boot, 0.5MB memory — the extreme minimalist), RuVector, Moltiszclaw, Picobot, Ourorobos, n8nClaw, SmallClaw, memU Bot, LightClaw, SafeClaw, OpenGork, Carapace, AndyClaw, FreeClaw, TitanClaw, KafClaw, BashoBot, grip-ai.

**Net assessment**: Most of these projects are experiments or weekend hacks (single-digit stars). The ecosystem is healthy in the sense that it demonstrates intense interest in the problem space, but the signal-to-noise ratio is low. Only NanoClaw, ZeroClaw, NullClaw, PicoClaw, and IronClaw show signs of sustained development and community adoption.

---

## 2. Enterprise Agent Platforms

### 2.1 Warp + Oz: Deep Dive

Warp launched Oz on February 10, 2026, marking its pivot from "AI terminal" to "agentic development environment." **[Confidence: Strong -- verified via Fast Company, Stackademic, official Warp site]**

**What Oz is**: A cloud-native agent platform that provides secure, sandboxed environments for AI agents to run asynchronously. Each agent gets its own cloud sandbox with full terminal control, connected to your codebase.

**Architecture**:
- Cloud-based sandboxes (not local execution)
- Agents accessible via Warp app, web interface, or mobile
- All operations logged and accessible
- Designed for team use — multiple developers can observe and interact with agents
- WARP.md files (equivalent of CLAUDE.md/AGENTS.md) control agent behavior
- Model-agnostic — not locked to a single provider

**Key capabilities**: Agents can write code, run tests, process customer feedback, handle bug reports, and perform various tasks autonomously. Scheduled agent runs with prompts and playbooks. Cloud agents positioned as potential n8n/workflow-automation replacement.

**Security model**: Purpose-built to address the security gaps Warp's CEO identifies in existing tools — particularly around agents properly configured and securely handling company code in the face of prompt injection attacks.

**Is it open source?** No. Warp is a commercial product. The agent infrastructure is proprietary cloud.

**Assessment for Agent OS**: Oz represents the "fully managed" end of the spectrum. For teams wanting agent infrastructure without self-hosting, it's a compelling option. For an Agent OS build on your own server, Oz's architecture is instructive (cloud sandboxes, async execution, team observability) but not directly usable.

### 2.2 Devin (Cognition): Maturing Into Enterprise

Devin has evolved significantly, moving from experimental to enterprise-grade. **[Confidence: Strong -- documented by Cognition, Nader Dabit (joined Cognition), Ry Walker Research]**

**Key metrics**:
- 67% PR merge rate (up from 34% year-over-year)
- Merged hundreds of thousands of PRs across customers
- Enterprise customers: Goldman Sachs, Santander, Nubank, Infosys, Cognizant
- Pricing: ~$500/seat/month enterprise, $20/month Core plan (Devin 2.0)
- 83% more productive than Devin 1.x per internal benchmarks

**Latest capabilities**:
- **Playbooks**: Reusable instruction sets (like recipes) that encode how Devin should approach specific task types. Community gallery at app.devin.ai/playbooks.
- **Session Insights**: Analytics on agent performance per session.
- **Scheduled Devins**: Recurring agent runs on cron — moving toward "always-on" agents.
- **MCP Marketplace**: Third-party tool integrations via MCP.
- **DeepWiki**: Knowledge management for agent context.
- **Multi-interface**: Web, Slack, Linear, CLI, API — "wherever the work starts."
- **Data Analyst Agent**: Expanding beyond pure code tasks.

**Internal dogfooding**: Cognition merged 659 Devin PRs into their own codebase in a single week (Feb 2026), up from 154 in their best week in 2025.

**Acquisition of Windsurf/Codeium**: Cognition acquired Windsurf (formerly Codeium) in a three-way split. Windsurf is now under Cognition. This consolidates Devin's cloud-first agent approach with Windsurf's IDE-native agent (Cascade, SWE-1 model family).

### 2.3 Cursor: Cloud Agents on VMs

Cursor's February 2026 update is a paradigm shift — from IDE-embedded agents to fully autonomous cloud agents running on isolated virtual machines. **[Confidence: Established -- CNBC, DevOps.com, NxCode, multiple Medium posts]**

**Launch**: February 24, 2026 (cloud agents); February 12, 2026 (long-running agents for Ultra/Teams/Enterprise).

**Architecture**:
- Each agent gets its own isolated VM with full development environment
- Agents can build software, test it, record video demos of their work, and produce merge-ready PRs
- Triggerable from web, desktop app, mobile, Slack, or GitHub
- Agents run in parallel — multiple VMs, multiple tasks
- Long-running: 25-52+ hours of autonomous operation
- CLI with agent modes (Plan, Ask), cloud handoff for background tasks

**Results**: More than one-third (35%) of PRs at Cursor itself are now created by cloud agents. "The factory model" — fleets of agents as teammates.

**Sandboxing**: Agents run freely inside controlled boundaries, surfacing permission requests only when stepping outside (most often for internet access).

**Assessment for Agent OS**: Cursor Cloud Agents represent the gold standard for what "agents with their own computers" looks like. The VM-per-agent pattern, video demo recording, and multi-trigger-point design are reference architecture for any production agent deployment.

### 2.4 GitHub Copilot: Agent Infrastructure Goes Enterprise

GitHub has been aggressive in February 2026, shipping agent infrastructure features across the entire platform. **[Confidence: Established -- all from GitHub Changelog, official blog]**

**Key updates**:
- **Copilot Coding Agent GA**: Asynchronous, autonomous background agent running in GitHub Actions environments. Available to all paid Copilot subscribers since Sept 2025, now with:
  - Model picker (Claude Opus 4.6, GPT-5.3-Codex available alongside default)
  - Self-review capability
  - Built-in security scanning
  - Custom agents (defined in .github-private/agents/*.md)
  - CLI handoff (delegate from local to cloud)
- **Copilot CLI GA** (Feb 25): Specialized agents — Explore (codebase analysis), Task (builds/tests), Code Review, Plan (implementation planning)
- **Windows project support** (Feb 18)
- **GitHub Agentic Workflows** (Feb 13, technical preview): Markdown files in .github/workflows/ describe automation goals in natural language. The `gh aw` CLI converts them into GitHub Actions workflows
- **Enterprise AI Controls & Agent Control Plane GA** (Feb 26): Centralized policy management for all AI agents across an organization

**Assessment for Agent OS**: GitHub's approach is the most enterprise-ready agent infrastructure. The agent-control-plane pattern (centralized policy, custom agents via markdown, model picker) is exactly what organizations need. The tight coupling to GitHub Actions is both a strength (existing CI/CD) and a limitation (GitHub lock-in).

### 2.5 Google: Jules + Gemini CLI

Google's agent infrastructure is split between Jules (autonomous cloud agent) and Gemini CLI (terminal-native). **[Confidence: Moderate -- Jules still experimental, Gemini CLI more established]**

**Jules**: Clones repos to secure Google Cloud VMs, plans changes with Gemini models, executes edits, runs tests, opens PRs. Available as a standalone tool and as a Gemini CLI extension. Focused on scoped tasks: framework upgrades, mechanical refactors, dependency bumps, documentation fixes.

**Gemini CLI**: Open-source terminal agent, accepted into Google Summer of Code 2026. Supports MCP, Software Agent patterns, A2A (Agent-to-Agent) protocol. The Jules extension enables Gemini CLI to delegate tasks to Jules VMs asynchronously.

**Assessment**: Google's agent infrastructure is less mature than Anthropic's or GitHub's for production use, but the Gemini CLI + Jules combination mirrors the Claude Code + cloud execution pattern emerging elsewhere.

---

## 3. Claude Code Official: Latest State

### 3.1 Agent Teams (TeammateTool)

Agent Teams is an experimental feature (requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) that enables multi-session collaboration. **[Confidence: Strong -- multiple independent guides, official documentation]**

**Architecture**:
- One session acts as **Team Lead** — creates the team, spawns teammates, assigns tasks, synthesizes results
- **Teammates** are separate Claude Code instances, each with their own context window
- 3-5 independent instances collaborating on a shared project
- Shared task system for coordination
- Each teammate costs separately (separate Claude instance)

**TeammateTool capabilities**: Spawn teammates, assign tasks, send messages between agents, manage task lists. The Team Lead orchestrates while teammates work independently in parallel.

**Current state**: Research preview. Functional but requires understanding of Claude Code's context window limitations and cost implications.

### 3.2 Claude Code SDK

The Claude Agent SDK (TypeScript, npm `@anthropic-ai/claude-agent-sdk`) is at v0.2.63 (Feb 28, 2026). 875 stars, 91 forks, 52 releases. **[Confidence: Established]**

Provides a `query()` function that streams events, includes the same built-in tools as Claude Code (file editing, shell execution, search), and enables headless agent construction. Production patterns include:
- Headless mode (`-p` flag): Skip interactive UI, optimal for CI/CD and background tasks. Recent optimizations: skips unnecessary startup API calls, caches MCP auth failures.
- New environment variables: `CLAUDE_CODE_ACCOUNT_UUID`, `CLAUDE_CODE_USER_EMAIL`, `CLAUDE_CODE_ORGANIZATION_UUID` for SDK callers to provide account info synchronously (eliminates race condition in telemetry).

### 3.3 Remote Control

Launched February 24, 2026. Bridges local Claude Code terminal sessions with claude.ai/code, Claude iOS app, and Claude Android app. **[Confidence: Established -- Simon Willison, multiple Medium articles, GitHub issue confirms headless Linux VM via SSH + tmux]**

**Key insight**: This is a synchronization layer, not a cloud migration. Files stay on your machine. The remote interfaces (web, mobile) connect to your running local session. Confirmed working on headless servers via SSH + tmux — directly relevant to Agent OS patterns where you want a phone/web interface to a server-based agent.

### 3.4 Hooks

Claude Code now supports 12 lifecycle events with three handler types. **[Confidence: Strong -- PixelMojo comprehensive guide, ksred practical guide]**

- **Handler types**: Command hooks (shell commands via stdin/JSON), Prompt hooks, Agent hooks
- **Lifecycle events**: PreToolUse, PostToolUse, Stop, SessionStart, and more
- **Production patterns**: Auto-formatting on file write (PostToolUse), blocking dangerous commands (PreToolUse), injecting context at session start (SessionStart), automated quality checks

---

## 4. Open Source Agent Frameworks

### 4.1 LangGraph, AutoGen, CrewAI: The Big Three

These frameworks continue to evolve but serve a fundamentally different purpose than terminal-native agent harnesses. **[Confidence: Strong -- numerous 2026 comparison articles]**

**LangGraph** (LangChain): Graph-based agent workflows. Strongest at complex multi-step pipelines with conditional branching. The most architecturally flexible of the three but also the most complex.

**AutoGen** (Microsoft): Multi-agent conversation framework. Best for scenarios where agents need to discuss and negotiate (code review between agents, planning sessions). AG2 (the community fork) has been gaining traction alongside Microsoft's official version.

**CrewAI**: Role-based agent teams. Most intuitive mental model (assign roles, define tasks, let crews execute). Lower ceiling than LangGraph but faster time-to-value.

**How they relate to Agent OS**: These frameworks are orchestration layers, not harnesses. You'd use LangGraph or CrewAI to coordinate what multiple Claude Code instances do, but you'd still need the agent harness (Claude Code SDK, *Claw runtime) as the execution layer. They're complementary, not competitive.

### 4.2 Pydantic AI: Type-Safe Agents

Pydantic AI has carved a niche as the "FastAPI of agent frameworks" — type-safe, model-agnostic, built for production. **[Confidence: Moderate -- well-documented but smaller community than the Big Three]**

Key differentiators: Declare a Pydantic model as `output_type` and the framework handles structured output extraction. Built-in `TestModel` for deterministic testing without API calls. First-class dependency injection. Supports switching between providers with a single string change.

### 4.3 Emerging: Strands (AWS), Microsoft Agent Framework

AWS's Strands Agents framework and Microsoft's Agent Framework (with Semantic Kernel underpinnings) are entering the space with enterprise backing. Both support structured output via Pydantic models, and Microsoft's framework integrates with Azure OpenAI natively.

---

## 5. Abstraction Layer Projects

### 5.1 harness-cli (Rust)

A Rust CLI (69 downloads on crates.io as of Feb 2026, v0.1.6) that spawns any supported agent as a subprocess and translates its native streaming output into a unified NDJSON event stream. **[Confidence: Moderate -- very early, low adoption, but architecturally interesting]**

- Supports: Claude Code, Codex, OpenCode, Cursor
- Philosophy: "Write one integration, and it works with any supported agent backend"
- Website: harness.lol

**Assessment**: The right idea at the right time, but far too early and too small to bet on. The concept of a unified agent event stream is sound, but the implementation lacks the community needed for sustainability.

### 5.2 agent-mux: Cross-Engine Subagents

agent-mux (by Nick Oak) is the more compelling abstraction project. It solves a specific, well-articulated problem: Claude Code has Task subagents but can only dispatch Claude. Codex has no subagent system at all. agent-mux provides one CLI and one JSON contract that lets any engine dispatch work to any other engine. **[Confidence: Moderate -- single developer, but technically sound and well-explained]**

**Key insight from agent-mux**: "Mode collapse between Claude and OpenAI models is roughly orthogonal. The blind spots don't overlap." Running Opus for orchestration and Codex for surgical code changes isn't redundancy — it's coverage.

### 5.3 Open Harness

Max Gfeller's Open Harness (Feb 24, 2026) is a code-first, composable SDK for building AI agents, inspired by the capabilities of Claude Code and Codex but designed to be lighter weight. 3 stars on GitHub. **[Confidence: Low -- brand new, single developer]**

### 5.4 Harnss (Desktop Client)

OpenSource03/harnss is a desktop client for the Agent Client Protocol, Claude Agent SDK, and Codex App Server. 4 stars but 38 releases in two weeks — active development. Runs multiple AI agents side by side with tool visualization and MCP integrations. **[Confidence: Low -- very early but interesting as a GUI abstraction]**

### 5.5 ergo (Planning CLI)

ergo is a planning abstraction — not a harness but a task management layer that any agent can use. Stores task backlogs as JSONL in the repo, supports sequencing, epics, and concurrent agent access. Agent-agnostic by design. **[Confidence: Low -- early stage]**

---

## 6. Infrastructure Layer: Sandboxes, Compute, and Hosting

### 6.1 E2B: Purpose-Built Agent Sandboxes

E2B provides isolated Linux microVM sandboxes specifically designed for AI agent code execution. Each sandbox is an isolated environment where agents can execute code, install packages, read/write files, and run processes. **[Confidence: Strong -- well-established, multiple comparisons]**

- microVM isolation (Firecracker-based)
- Sub-second cold starts
- SDK for Python and TypeScript
- Integrates directly into agent loops

**Assessment for Agent OS**: E2B is the strongest candidate for the execution sandbox layer if you want to separate the agent orchestration (your harness) from the code execution environment. Rather than giving Claude Code direct shell access on your server, you'd route `Bash` tool calls through E2B sandboxes.

### 6.2 Modal: Serverless GPU + Agent Infrastructure

Modal provides a Python SDK for deploying AI workloads with serverless GPUs. Not agent-specific, but increasingly used as agent infrastructure. $30/month free compute tier. **[Confidence: Moderate -- primarily a compute platform, agent use is secondary]**

- gVisor-based sandbox isolation
- Serverless: scale to zero, auto-scale up
- Supports inference, training, and batch compute
- Good developer experience with Python-native API

### 6.3 The Sandbox Landscape

The broader sandbox market for AI agents in 2026 includes E2B, Modal, Northflank (Kata/Firecracker/gVisor microVMs), Daytona (sub-90ms cold starts, Docker containers), Koyeb (serverless containers), and Runpod (GPU-focused). The pattern is consistent: microVM or container isolation, ephemeral or persistent sessions, direct integration with agent SDKs. **[Confidence: Strong -- well-documented across multiple comparison articles]**

---

## 7. Community Patterns and Ecosystem Signals

### 7.1 What People Are Building on GitHub

The GitHub community around Claude Code has spawned several distinct patterns:

**The Living Architect Model (LAM)**: A framework that transforms Claude Code from a "coding machine" into an "architect" through 3-phase approval gates, decision-making from 3 perspectives, mandatory TDD, and automatic subagent assignment. 8 specialized subagents for different workflow phases.

**Multi-agent coordination tools**: Projects like ergo (task management), agent-mux (cross-engine dispatch), and various dashboard tools for monitoring multiple Claude Code sessions.

**Skill ecosystems**: NanoClaw's skills pattern (extend via .claude/skills/ rather than direct code changes) is being adopted more broadly. Claude Code's official Skills system (SKILL.md with frontmatter, lifecycle events, context forking) provides the foundation.

### 7.2 VS Code as Multi-Agent Hub

Microsoft is positioning VS Code as "the home for multi-agent development." The January 2026 release (1.109) added the ability to run Claude and Codex agents alongside GitHub Copilot, with an Agent Sessions view as a unified management interface. **[Confidence: Established -- official VS Code blog]**

### 7.3 The "Agent Factory" Mental Model

A recurring pattern across Cursor, Devin, and GitHub: teams describe their workflow as a "factory" where fleets of agents work as teammates. Developers provide direction, equip agents with tools, and review output. This is not one human + one agent — it's one human managing many agents in parallel.

---

## 8. Trend Analysis

### What's Accelerating

1. **Cloud-native agent execution**: Every major platform shipped cloud/VM-based agents in Feb 2026. Local-only execution is becoming the fallback, not the default. The pattern: local control plane, cloud execution sandbox. **[Confidence: Established]**

2. **Multi-agent as default**: Agent Teams (Claude Code), cloud agent parallelism (Cursor), Scheduled Devins, Copilot custom agents — single-agent workflows are already legacy. **[Confidence: Strong]**

3. **Security-first design**: Post-OpenClaw-CVE and post-ClawHavoc, every new project leads with its security model. Container isolation, pairing-based auth, workspace scoping, encrypted secrets — table stakes. **[Confidence: Established]**

4. **Agent-agnostic tooling**: harness-cli, agent-mux, ergo, AGENTS.md, MCP — the ecosystem is standardizing interfaces so agents and tools are interchangeable. **[Confidence: Moderate -- the trend is clear but adoption is still early]**

5. **"AI-native" development**: NanoClaw's skills pattern, PicoClaw's self-bootstrapping, the Living Architect Model — a meta-pattern where AI agents build and extend the tools that AI agents use. **[Confidence: Moderate]**

### What's Stalling

1. **Single-language lock-in**: OpenClaw's TypeScript-only codebase (430K+ lines) is showing its limits. The most interesting new projects are in Zig, Rust, and Go — languages optimized for the deployment constraints of always-on agents. **[Confidence: Strong]**

2. **Marketplace trust models**: ClawHavoc destroyed confidence in open skill/extension marketplaces. No one has solved the "how do you trust third-party agent extensions?" problem. GitHub's agent control plane (centralized policy) is the enterprise answer; the open-source ecosystem has no equivalent. **[Confidence: Strong]**

3. **Single-provider agents**: NanoClaw (Claude only), Codex (OpenAI only) — these work for committed users but create fragility. The trend favors multi-provider architectures (OpenRouter, model routers, agent-mux). **[Confidence: Moderate]**

### What's Dying

1. **Manual agent configuration**: One-time, manual setup flows are being replaced by interactive onboarding wizards, AI-native configuration (tell the agent what you want), and infrastructure-as-code patterns. **[Confidence: Moderate]**

2. **IDE-only agent access**: Cursor's cloud agents are triggerable from Slack, mobile, and web. Claude Code has Remote Control. Devin runs via Slack, Linear, CLI, and API. The agent doesn't live in your editor anymore — it lives in the cloud with multiple access points. **[Confidence: Strong]**

---

## 9. Where the Puck Is Going

### 3-6 Month Predictions

1. **Claude Code will ship official cloud execution** (not just Remote Control). The pattern is too clear — Cursor, Devin, GitHub, Warp, Google all have cloud agents. Anthropic will follow. Remote Control is the bridge. **[Confidence: Moderate -- extrapolation from clear trends, but no inside information]**

2. **The *Claw ecosystem will consolidate to 3-5 survivors**. Of the 30+ projects, expect NanoClaw, ZeroClaw, and PicoClaw to gain ground while most others fade. NullClaw's Zig niche may sustain it. OpenClaw will remain relevant by sheer install base but continue losing mindshare on security grounds. **[Confidence: Moderate]**

3. **Agent-to-Agent (A2A) protocols will mature**. Google's A2A protocol, MCP's tool-level interop, and projects like agent-mux point toward agents dispatching work to other agents across organizational boundaries. **[Confidence: Emerging -- technically possible, adoption uncertain]**

4. **Security certification for agent skills/extensions will emerge**. Post-ClawHavoc, some form of signing, review, or certification for third-party agent capabilities is inevitable. Likely led by GitHub (signing for custom agents) or Anthropic (vetted skill marketplace). **[Confidence: Moderate]**

5. **The "one human, many agents" pattern becomes the default workflow for professional developers**. Cursor's 35% internal PRs, Cognition's 659 Devin PRs/week, GitHub's agent factory — the data supports this as the emerging norm, not the exception. **[Confidence: Strong]**

### The End State

The "Agent OS" end state looks like this: a control plane that manages agent lifecycles, routes tasks to the right agent/model combination, provides sandboxed execution environments, maintains persistent memory across sessions, and exposes a unified interface (terminal, web, mobile, API). The execution layer is cloud-native by default with local fallback. Multiple LLM providers serve different purposes (orchestration, code generation, review, planning). Security is enforced at every boundary — network, filesystem, secrets, API access.

This is not one product. It's a stack. The question for any Agent OS build is which layers you own and which you delegate. **[Confidence: Moderate -- architecturally sound, but the specific products that win are unpredictable]**

---

## Key Unknowns & Open Questions

1. **Will Anthropic ship official multi-agent orchestration?** Agent Teams is experimental. If it goes GA with proper tooling, the entire *Claw ecosystem becomes less relevant.

2. **Can open skill marketplaces be secured?** ClawHavoc showed the attack surface. Code signing? Sandboxed skill execution? Reputation systems? No one has a proven answer.

3. **What's the right isolation boundary for agent execution?** VM (Cursor), container (NanoClaw, E2B), process sandbox (Claude Code), or no isolation (OpenClaw default)? The performance/security tradeoff isn't settled.

4. **Will the agent harness commoditize?** If Claude Code SDK, Codex SDK, and Gemini CLI provide everything needed, do custom harnesses still make sense? Or do they become thin wrappers?

5. **What happens when agents manage agents manage agents?** The recursion depth of multi-agent systems is increasing. Governance and observability at depth > 2 are unsolved problems.

---

## Source Log

| # | Source | Tier | Found Via | Key Contribution |
|---|--------|------|-----------|-----------------|
| 1 | SonicWall Labs. "OpenClaw Auth Token Theft: CVE-2026-25253." sonicwall.com | A | Tavily | Detailed CVE-2026-25253 attack chain analysis |
| 2 | runZero. "OpenClaw RCE vulnerability." runzero.com | A | Tavily | Independent CVE assessment |
| 3 | NVD. "CVE-2026-25474." nvd.nist.gov | S | Tavily | Official vulnerability database entry |
| 4 | SC Media. "Massive OpenClaw supply chain attack." scworld.com | B | Tavily | ClawHavoc attack details (1,184 malicious skills) |
| 5 | Aryaka. "Securing OpenClaw Against ClawHavoc." aryaka.com | B | Tavily | ClawHavoc attack chain analysis |
| 6 | eSecurity Planet. "Hundreds of Malicious Skills." esecurityplanet.com | B | Tavily | 41.7% vulnerability rate across 2,890+ skills |
| 7 | Taskade Blog. "15 Best OpenClaw Alternatives." taskade.com | C | Firecrawl | Comprehensive alternatives comparison |
| 8 | GitHub. nullclaw/nullclaw repo. github.com | B | Brave, Exa | NullClaw architecture, 678KB binary, 22+ providers |
| 9 | NullClaw official. nullclaw.co, nullclaw.org, nullclaw.net | C | Brave | Architecture details, vtable interfaces, telemetry |
| 10 | BitDoze. "NullClaw Deploy Guide." bitdoze.com | C | Brave | Deployment patterns, vtable architecture details |
| 11 | GitHub. sipeed/picoclaw repo (21K stars). github.com | B | Exa | PicoClaw architecture, <10MB RAM, Go, self-bootstrapping |
| 12 | CNX Software. "PicoClaw ultra-lightweight." cnx-software.com | B | Exa | Hardware comparison table, RISC-V deployment |
| 13 | l3dlp (Medium). "PicoClaw: Bare Metal AI Agent." medium.com | C | Exa | Go architecture deep dive, 4K lines of code |
| 14 | ZeroClaw official. zeroclaw.net, zeroclaws.io, zeroclaw.space | C | Brave, Firecrawl | Trait-driven architecture, AIEOS, three modes |
| 15 | DEV Community. "ZeroClaw: Lightweight Secure Rust Agent Runtime." dev.to | C | Brave | Plugin-oriented Rust trait architecture |
| 16 | GitHub Gist (yanji84). "ZeroClaw Migration Assessment." gist.github.com | C | Brave | 17K stars, 2K forks, 27+ contributors, 16MB binary |
| 17 | GitHub. qwibitai/nanoclaw repo. github.com | B | Brave | NanoClaw channels (WhatsApp, Telegram, Discord, Slack, Signal) |
| 18 | VentureBeat. "NanoClaw solves OpenClaw's biggest security issue." venturebeat.com | B | Brave | AI-native development pattern, skills-only extension |
| 19 | The New Stack. "NanoClaw minimalist AI agents." thenewstack.io | B | Brave | NanoClaw architecture interview with Cohen |
| 20 | Fast Company. "Warp unveils Oz." fastcompany.com | A | Tavily | Oz architecture, cloud sandboxes, team features |
| 21 | Stackademic. "Oz by Warp." blog.stackademic.com | C | Tavily | Oz launch date (Feb 10, 2026), new paradigm framing |
| 22 | Nader Dabit (Substack). "How Cognition Uses Devin to Build Devin." nader.substack.com | B | Exa | 659 Devin PRs/week internal, multi-interface usage |
| 23 | Ry Walker Research. "Devin (Cognition)." rywalker.com | B | Exa | $500/seat pricing, 67% merge rate, enterprise customers |
| 24 | Cognition Blog. "Devin's 2025 Performance Review." cognition.ai | A | Exa | Official capabilities assessment, customer list |
| 25 | Cognition Docs. "Creating Playbooks," "Using Playbooks." docs.devin.ai | A | Exa | Official playbook documentation |
| 26 | CNBC. "Cursor announces major update." cnbc.com | A | Brave | Cloud agents on VMs, multi-trigger, parallel execution |
| 27 | DevOps.com. "Cursor Cloud Agents." devops.com | B | Brave | 35% of internal PRs by agents |
| 28 | NxCode. "Cursor Cloud Agents: Autonomous Coding on VMs." nxcode.io | C | Brave | Feb 24 launch, video demo recording, VM architecture |
| 29 | Digital Applied. "Cursor Cloud Agents Guide." digitalapplied.com | C | Brave | 30%+ merge rate, isolated VM details |
| 30 | GitHub Blog. Multiple changelogs (Feb 2026). github.blog | A | Brave | Copilot coding agent GA, model picker, CLI, agentic workflows |
| 31 | GitHub Blog. "What's new with Copilot coding agent." github.blog | A | Brave | Self-review, security scanning, custom agents, CLI handoff |
| 32 | VS Code Blog. "Your Home for Multi-Agent Development." code.visualstudio.com | A | Exa | VS Code as multi-agent hub, agent sessions view |
| 33 | ClaudeFa.st. "Agent Teams Complete Guide." claudefa.st | C | Brave | TeammateTool architecture, setup instructions |
| 34 | GitHub (anthropics). claude-agent-sdk-typescript repo. github.com | A | Exa | SDK v0.2.63, 875 stars, 52 releases |
| 35 | Simon Willison. "Claude Code Remote Control." simonwillison.net | B | Brave | Remote Control launch confirmation (Feb 25) |
| 36 | GitHub Issues (anthropics/claude-code). "#29479 Remote Control on headless Linux." github.com | B | Brave | SSH + tmux headless server confirmation |
| 37 | ClaudeFa.st. "Remote Control Setup Guide." claudefa.st | C | Brave | Architecture: sync layer, not cloud migration |
| 38 | PixelMojo. "Claude Code Hooks Guide: All 12 Lifecycle Events." pixelmojo.io | C | Tavily | 12 events, 3 handler types, production patterns |
| 39 | ksred.com. "Claude Code Hooks Complete Guide." ksred.com | C | Tavily | Practical hook patterns (formatting, blocking, context) |
| 40 | Nick Oak. "agent-mux: Cross-Engine Subagents." nickoak.com | C | Exa | Cross-engine dispatch, mode collapse insight |
| 41 | Crates.io. harnesscli v0.1.6. crates.io | C | Exa | 69 downloads, unified NDJSON event stream |
| 42 | Max Gfeller. "Introducing Open Harness." maxgfeller.com | C | Exa | Composable SDK for agent building |
| 43 | Koyeb Blog. "Top Sandbox Platforms for AI Code Execution 2026." koyeb.com | B | Tavily | E2B, Modal, sandbox comparison |
| 44 | Northflank. "E2B vs Modal." northflank.com | B | Tavily | microVM vs gVisor isolation comparison |
| 45 | Google Developers Blog. "Jules Extension for Gemini CLI." developers.googleblog.com | A | Exa | Jules + Gemini CLI integration, async delegation |
| 46 | Point of AI. "Jules by Google." pointofai.com | C | Exa | Jules architecture: cloud VM, plan/approve/execute |
| 47 | Awesome Agents. "Windsurf Review." awesomeagents.ai | C | Brave | Cognition acquisition, SWE-1 model family |
| 48 | Fundesk. "Windsurf vs Cursor 2026." fundesk.io | C | Brave | SWE-1.5 model, autonomous agent flow |
| 49 | GitHub. babyclaw/babyclaw repo. github.com | D | Exa | BabyClaw architecture, Vercel AI SDK, Drizzle |
| 50 | GitHub. ikmolbo/TurboClaw repo. github.com | D | Exa | TurboClaw architecture, Bun-based, Claude Code native |
| 51 | GitHub. The-Swarm-Corporation/ClawSwarm repo. github.com | D | Exa | ClawSwarm multi-agent-first design |
| 52 | ClawClones.com. Various comparison pages. clawclones.com | D | Exa | 30+ project comparison metrics |
| 53 | Zylos Research. "AI Agent CLI Frameworks." zylos.ai | B | Exa | Comprehensive CLI framework landscape survey |
| 54 | Calvin French-Owen. "Coding Agents in Feb 2026." calv.info | B | Exa | Practitioner comparison (Codex, Claude Code) |
| 55 | Releasebot. "Anthropic Release Notes Feb 2026." releasebot.io | C | Brave | SDK env variables, race condition fix |
| 56 | Joget. "AI Agent Adoption 2026." joget.com | B | Brave | Forrester/Gartner: 2026 breakthrough year for multi-agent |

---

## Audit Notes

1. **Star counts verified where possible** but are volatile. PicoClaw's 21K+ stars, ZeroClaw's 17K stars, and NullClaw's 4K+ stars were confirmed via GitHub repo data and cross-referenced with third-party reports. These numbers will change rapidly.

2. **OpenClaw security claims triangulated** across 5+ independent sources (SonicWall, runZero, NVD, SC Media, Immersive Labs). The 41.7% vulnerability rate in ClawHub skills comes from a single audit cited by eSecurity Planet — treat with moderate confidence as independent verification was not found.

3. **Cursor's 35% internal PR statistic** is self-reported (via Cursor's own blog/changelog). No independent verification found, but the claim is specific and falsifiable.

4. **Devin's 67% merge rate and 659 PRs/week** come from Cognition's own blog and an employee (Nader Dabit). These are self-reported metrics. The pricing (~$500/seat) comes from customer intel via Ry Walker Research, not official Cognition pricing pages.

5. **Windsurf/Cognition acquisition**: Multiple sources confirm Cognition acquired Windsurf, but the "three-way acquisition split" detail comes from a single review (Awesome Agents). Treat with moderate confidence.

6. **PicoClaw security warning**: The project's own README warns against production deployment before v1.0. This is often missed in enthusiastic coverage. The crypto scam warnings are also directly from the official repo.

7. **harness-cli and agent-mux are very early-stage projects** with minimal adoption. Included for architectural interest, not as proven solutions. Confidence is deliberately low.

8. **No fabricated citations**: All sources were found via search during this session. No studies or statistics are cited from training data alone.

9. **Potential bias**: The *Claw ecosystem generates enormous content volume (blog posts, comparisons, tutorials) because it's trending. This may create an impression of greater maturity than actually exists. Many of these projects have more blog posts about them than active contributors.

10. **Missing perspectives**: This scan undercovers the Chinese AI agent ecosystem (aside from PicoClaw's Sipeed connection). There are likely significant agent infrastructure projects in the Chinese open-source ecosystem not captured by English-language search tools.
