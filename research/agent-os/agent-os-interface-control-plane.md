# Agent OS Track 3: Interface & Control Plane
_Research conducted: 2026-03-02 | Overall shelf-life: Perishable (field moving monthly)_

## Domain Map

The interface layer for AI agent systems sits at the intersection of three fields: developer tooling (terminals, CLIs, IDE integrations), chat-ops platforms (Discord, Slack, Telegram), and human-automation interaction design (supervisory control theory, levels of autonomy). The key players are Warp (Oz orchestration platform, launched February 2026), OpenAI (Codex App Server, published February 2026), Anthropic (Claude Code SDK/Agent SDK), Cognition (Devin's multi-channel interface), HumanLayer (approval workflow infrastructure), and a fast-growing open-source community building Discord-to-agent bridges. The central tension is between convenience (Discord is where humans already are) and capability (purpose-built interfaces offer richer interaction primitives). The Agent Client Protocol (ACP), introduced August 2025, is attempting to standardize the agent-to-client communication layer the way LSP standardized language servers. The question for Agent OS is not "which single interface?" but "what protocol sits beneath all of them?"

## Executive Summary

The evidence points strongly toward an API-first architecture with Discord as the primary human-facing frontend -- but only if the API layer is designed first and Discord is treated as a client, not the platform. The Codex App Server pattern (bidirectional JSON-RPC over stdio, with Items/Turns/Threads as primitives) is the most mature protocol design available and should be studied carefully. Warp's Oz platform validates the cloud-agent orchestration model but is a proprietary SaaS, not something you can self-host freely. The open-source project zebbern/claude-code-discord (122 stars, built on Claude Agent SDK v0.2.45) is the most feature-complete Discord-to-Claude bridge and demonstrates both what is possible and where Discord's limitations bite. The academic literature on supervisory control (Sheridan & Verplank, 1978; Feng et al., 2024) provides a rigorous framework for thinking about autonomy levels that maps directly onto agent permission systems. The biggest risk is building on Discord's proprietary platform and inheriting its rate limits, 2000-character message cap, and prompt injection surface area. The biggest opportunity is building a thin, protocol-first control plane that speaks JSON-RPC internally and can project onto Discord, CLI, web, and mobile simultaneously.

---

## 1. Foundations -- The 80/20

### Principle 1: Protocol First, Interface Second
**Confidence: Strong (4/5)**

Every successful multi-surface agent system (Codex, Devin, Warp Oz) follows the same pattern: define a bidirectional protocol that encapsulates agent lifecycle (sessions, turns, events, approvals), then build thin clients on top. OpenAI's Codex App Server is the clearest articulation of this -- it started as an internal hack to reuse the Codex harness across CLI and VS Code, then evolved into a stable JSON-RPC protocol that now powers CLI, web, desktop, VS Code, JetBrains, and Xcode clients (OpenAI, "Unlocking the Codex Harness," Feb 4, 2026). The key primitives are Items (atomic I/O units with start/delta/completed lifecycle), Turns (one unit of agent work), and Threads (durable session containers). Transport is JSON-RPC over stdio (JSONL framing), which enables both local subprocess spawning and remote tunneling over WebSocket.

The Agent Client Protocol (ACP) independently arrived at the same conclusion: JSON-RPC 2.0 over stdio, with agents as subprocesses of their clients (agentclientprotocol.com; Goose/Block, Oct 2025; Zed integration, Aug 2025). ACP is to coding agents what LSP is to language servers -- a standardization layer that lets any editor talk to any agent. Multiple CLI agents (Claude Code, Codex, OpenCode, Gemini CLI, Goose) are implementing ACP support.

**So what:** Design the Agent OS control plane as a JSON-RPC server first. Every frontend (Discord bot, CLI, web dashboard, mobile app) becomes a thin translation layer. This is not optional architecture astronautics -- it is the load-bearing decision that determines whether you can add new interfaces without rewriting the core.

### Principle 2: Approval Workflows Are the Core Interaction Pattern
**Confidence: Strong (4/5)**

The dominant human-agent interaction in production is not "give command, receive result." It is: agent runs autonomously, encounters a decision point requiring human judgment, pauses, requests approval, human grants/denies, agent continues. This is the pattern in Codex App Server (item/commandExecution/requestApproval with allow/deny response), in zebbern/claude-code-discord (interactive Allow/Deny buttons via Discord embeds), in Devin (confidence estimates with go/no-go), and in HumanLayer's entire product (API and SDK for routing approval requests to Slack, Discord, or email). Anthropic's own research confirms this: Claude Code sessions are getting longer (median autonomous stretch doubled from ~25 to ~45 minutes in 3 months), but they still stop at permission boundaries (Anthropic, "Measuring AI Agent Autonomy in Practice," Feb 18, 2026).

**So what:** The approval workflow is the atomic unit of the control plane. Design it with: (a) multiple priority levels (info, needs-input, error, cost-alert), (b) batch approval capability (approve N similar actions at once), (c) async approval (agent queues work, human reviews later), and (d) timeout with default-deny.

### Principle 3: Humans Supervise, Not Operate
**Confidence: Established (5/5)**

Sheridan and Verplank's (1978) 10-level taxonomy of automation -- from "human does everything" to "computer does everything, ignores human" -- has been the foundational framework for human-automation interaction for nearly 50 years. Applied to AI agents, the sweet spot is levels 5-7: the agent generates options and executes after human approval (level 5), executes unless human vetoes within a time window (level 6), or executes autonomously and informs after the fact (level 7). Feng, McDonald, and Zhang (2024, University of Washington) updated this for AI agents specifically, proposing five roles: operator, collaborator, consultant, approver, and observer. Most production agent systems today operate in the "approver" mode -- the agent proposes, the human rubber-stamps or rejects.

Shneiderman's Human-Centered AI framework (2020) adds a crucial insight: high automation AND high human control are not in tension. Systems can provide both simultaneously through what he calls "reliable, safe, and trustworthy" design patterns. The interface should make it easy to see what the agent is doing (transparency), easy to intervene (controllability), and easy to audit after the fact (accountability).

**So what:** Design the control plane for supervisory control, not direct operation. The default mode should be "agent runs, human monitors and intervenes when needed." This means: real-time event streaming (not polling), clear interruption mechanisms, and a complete audit trail.

### Principle 4: Discord Is a Good-Enough First Client, Not a Good Architecture
**Confidence: Strong (4/5)**

Discord offers compelling advantages as an initial agent control surface: near-universal adoption among developers, rich interaction primitives (threads, embeds, buttons, select menus, modals), built-in persistence (channel history = audit trail), mobile notifications, and RBAC via roles. The zebbern/claude-code-discord project demonstrates this comprehensively -- 45+ slash commands, interactive permission prompts, branch-to-channel mapping, 7 specialized agents, MCP server management, all built on the Claude Agent SDK v0.2.45.

But Discord imposes hard constraints that will become painful at scale:
- **Message rate limits:** 5 messages per 5 seconds per channel; global limit of 50 messages per 10 seconds across all channels (Discord API docs). For a swarm of 10+ agents all streaming output, this is immediately constraining.
- **Message length:** 2000 characters per message. Agent outputs routinely exceed this, requiring chunking that fragments code blocks and breaks formatting.
- **No structured data:** Discord messages are text blobs. Passing structured data (diffs, approval requests with metadata, agent state) requires encoding into embeds and button payloads, which have their own limits (25 buttons per message, 4000 characters per embed).
- **Proprietary platform risk:** Discord can change its API, raise prices, or ban bots at any time. Building your critical infrastructure on a platform you don't control is a known anti-pattern.
- **Security surface:** Every Discord message is potential prompt injection input. Any user who can type in a channel can attempt to manipulate the agent. (Snyk Labs, "Agent Hijacking," Aug 2024; Radware, "Prompt Injection in 2026")

**So what:** Use Discord as the first frontend client, but architect it as a replaceable projection of the underlying protocol. Never put agent management logic in the Discord bot code itself.

---

## 2. Current Evidence Landscape

### 2.1 Discord-to-Agent Bridges: The State of the Art

**zebbern/claude-code-discord** (122 stars, v2.2.0, Feb 2026) is the most complete implementation. Key architectural decisions:
- **SDK integration:** Built on `@anthropic-ai/claude-agent-sdk` v0.2.45 (migrated from deprecated `@anthropic-ai/claude-code` package). Uses `ProcessTransport` to spawn Claude Code sessions.
- **Session model:** One Claude Code process per Discord channel. Branch-aware organization maps Git branches to Discord channels/categories.
- **Permission model:** Six SDK permission modes (accept, deny, delegate, dontAsk, bypassPermissions, default). Interactive Allow/Deny buttons for real-time approval. RBAC via `ADMIN_ROLE_IDS` and `ADMIN_USER_IDS` env vars restricting destructive commands (shell, git, worktree ops) to specific Discord roles.
- **Mid-session controls:** Interrupt running tasks, change model mid-session, rewind conversation, toggle MCP servers, switch thinking modes (standard/think/think-hard/ultrathink), set effort level and budget cap.
- **Hooks system:** Passive SDK callbacks for tool use, notification, and task completion. The hooks enable observability without blocking the agent loop.
- **7 specialized agents:** Code reviewer, architect, debugger, security analyst, performance engineer, DevOps, general -- each with distinct system prompts and tool configurations.
- **Runtime:** Deno, TypeScript. Docker-ready with docker-compose and GHCR images.

**Confidence: Strong (4/5)** -- Directly verified from GitHub repository, commit history, and README.

**claude-code-comm-bot** (MichaelAyles, 7 stars, v0.1.7, Jun 2025) takes a different approach: VS Code extension that mirrors Claude conversations to Discord for remote monitoring. Architecture: child process spawning for Claude CLI with live terminal output, dual processing (terminal for real-time, background for structured JSON streaming), Discord as a read/notification channel rather than a command channel. Notably less ambitious but demonstrates the "Discord as notification layer" pattern.

**Confidence: Moderate (3/5)** -- Verified from repository; project appears dormant since June 2025.

**Disclaude** claims to manage Claude Code sessions via tmux from Discord. The disclaude.com domain appears to be down as of this research session; the project may have been absorbed into the broader ecosystem or discontinued. Multiple similar projects exist: `claude-discord-bridge` (thcapp, uses tmux with SQLite persistence), `Claude-Code-Remote` (JessyTsui, controls via email/Discord/Telegram).

**Confidence: Emerging (2/5)** -- Could not verify Disclaude directly; claims from Reddit threads only.

### 2.2 Warp + Oz: Cloud Agent Orchestration

Oz launched February 10, 2026 as Warp's cloud-based agent orchestration platform. This is the most polished implementation of the "agent control plane" concept currently available.

**Architecture:**
- **Layered design:** CLI (`oz` command) + REST API + SDK + Web app (oz.warp.dev) + Warp terminal integration. Any entry point works; all share the same backend.
- **Environment model:** Docker containers + git repos + startup commands. Environments are shared across teams. Setup takes ~5 minutes with agent assistance.
- **Agent lifecycle:** `oz agent run --prompt "..." --share` (local) or `oz agent run-cloud --prompt "..." --environment <slug>` (cloud). Every agent gets an auto-tracked session sharing link.
- **Session sharing:** Live links that let anyone on the team watch, steer, or take over an agent session. This is the killer UX feature -- it makes agent work visible without requiring everyone to be in the same terminal.
- **Scheduling:** `oz schedule --skill "flag-cleanup" --cron "0 2 * * *"` -- cron-based recurring agent tasks.
- **Skills as agents:** Any Claude Code skill, Codex skill, or `.agents/` directory skill is launchable as an Oz agent.
- **Integrations:** First-party Slack and Linear integrations for triggering agents where work happens (Ry Walker Research, Feb 2026).
- **Self-hosting:** Enterprise option available to keep code execution within your network.

**Internal usage (Warp's own claim):** Oz writes 60% of their PRs. Use cases include: parallelizing mermaid.js-to-Rust port across 15 agents, fraud detection bot running every 8 hours (found $60K fraudulent usage in one run), issue triage app (PowerFixer) that dispatches agents per GitHub issue.

**What makes Oz different from Discord-based approaches:**
1. **Cloud-native execution:** Agents run in cloud containers, not on your laptop. Eliminates local resource constraints.
2. **Purpose-built observability:** Auto-tracking, artifact collection (PRs, branches, plans), session sharing links.
3. **Programmable:** Full CLI/API/SDK for building apps on top of agents, not just commanding them.
4. **Team-oriented:** Shared environments, shared visibility, team-wide agent management.

**What you cannot borrow:**
- Oz is a SaaS product, not an open-source platform. The CLI is available but the cloud infrastructure is proprietary.
- The Warp terminal itself is closed-source (Rust, GPU-rendered).
- No self-hosted version for individual developers; enterprise self-hosting requires a sales conversation.

**Confidence: Strong (4/5)** -- First-party blog post, product documentation, multiple independent reviews.

### 2.3 The Codex App Server Pattern

OpenAI's Codex App Server (published February 4, 2026) is the most detailed public documentation of how to build a protocol-first agent control plane.

**Key design decisions:**
- **Transport:** JSON-RPC over stdio (JSONL framing). Deliberately chose JSON-RPC over MCP because MCP's semantics didn't map well to rich agent interactions like streaming diffs and approval flows.
- **Bidirectional:** Client sends requests, server sends notifications AND requests (e.g., approval prompts). The server can initiate requests when it needs human input, pausing the turn until the client responds.
- **Primitives:** Item (atomic I/O with start/delta/completed lifecycle), Turn (one unit of agent work), Thread (durable session). These three primitives are sufficient to model the full agent interaction surface.
- **Initialize handshake:** Client sends `initialize` with `clientInfo`; server responds with capabilities. Enables protocol versioning and feature negotiation.
- **Backward compatibility:** Older clients talk to newer servers safely. Partners with slower release cycles (JetBrains, Xcode) can decouple from server updates.
- **Code generation:** `codex app-server generate-ts` generates TypeScript bindings; `codex app-server generate-json-schema` produces a JSON Schema bundle for any language.
- **Web runtime:** Codex Web runs the App Server in a container, communicates over stdio tunneled through HTTP+SSE. Browser tab is stateless; server is source of truth. Tab can disconnect and reconnect without losing state.

**Client bindings exist in:** Go, Python, TypeScript, Swift, Kotlin (per OpenAI blog).

**Confidence: Established (5/5)** -- First-party engineering blog, open-source code in codex-rs repository, detailed protocol documentation.

### 2.4 Agent Client Protocol (ACP)

ACP is an open standard for agent-to-editor communication, analogous to LSP for language servers. Launched August 2025, it has gained traction across multiple editors and agents.

**Specification:**
- JSON-RPC 2.0 over stdio (local) or HTTP (remote)
- Agents run as subprocesses; editors spawn them
- Standardized events for thinking, tool execution, approval requests, streaming output
- Implemented by: Claude Code, Codex CLI, Gemini CLI, OpenCode, Goose
- Supported by: Zed (first adopter), Kiro, and others

**Key difference from Codex App Server:** ACP is cross-provider (any agent, any editor), while the Codex App Server is Codex-specific. ACP converges on the common subset; Codex App Server exposes full Codex capabilities including config management and auth flows.

**Confidence: Strong (4/5)** -- Open specification, multiple implementations, Philipp Schmid overview (Feb 2026), Block/Goose documentation.

### 2.5 CLI Tools for Agent Management

**harnesscli** (Rust, on crates.io) is a unified CLI that spawns Claude Code, Codex, OpenCode, or Cursor as subprocesses, translates their native streaming output into NDJSON event streams, and outputs to stdout. This is precisely the "thin translation layer" pattern: one CLI, multiple backends, standardized output format.

**MobileCLI** (mobilecli.app) is a self-hosted Rust daemon that streams AI coding agent sessions to your phone. Supports Claude Code, Gemini CLI, Codex, OpenCode. Features: approve tool calls, browse files, monitor progress. No cloud dependency, no account required. This demonstrates the "mobile client" surface for the protocol-first pattern.

**Codeman** (Ark0N/Codeman) manages Claude Code and OpenCode in tmux sessions with a web UI. Demonstrates the "web dashboard" surface.

**Confidence: Moderate (3/5)** -- Verified from crates.io and GitHub; projects are young with small user bases.

### 2.6 Enterprise/Startup Interfaces

**Devin** (Cognition AI) is the most mature multi-channel agent interface:
- **Slack:** Tag @Devin in any channel; responds in-thread. First-class integration.
- **Linear:** Assign issues to Devin or mention @devin in comments; provides scoped implementation plans with confidence estimates.
- **CLI:** Command-line access to Devin sessions.
- **Web:** Full dashboard at app.devin.ai.
- **API:** Programmatic session management.

**Augment Code** offers a similar multi-channel approach: Intent agents accessible via Slack, CLI, and IDE. Their comparison with Devin (Feb 2026) highlights the philosophical split: Intent keeps humans in the loop through "living specifications," while Devin operates autonomously from a task queue.

**Cursor** added background agents with cloud handoff (Jan 2026 CLI update). Agent modes (Plan and Ask), cloud handoff for background tasks, one-click MCP auth. The UX pattern: start work in IDE, hand off to cloud, get notified when done.

**Confidence: Strong (4/5)** -- Verified from official documentation and product pages.

### 2.7 Notification & Approval Workflow Design

**HumanLayer** (humanlayer.systems, open-source Agent Control Plane) provides the most sophisticated approval infrastructure:
- Distributed agent scheduler for outer-loop agents
- Durable task execution across long-running function calls
- Tool-based interface for requesting human approval or input
- Routes approvals through Slack, Discord, or email
- Maintains context: denied actions are fed back to the agent's context window for learning
- Full MCP support

The "12 Factor Agents" design principles (referenced by HumanLayer) advocate for: agent as process, explicit approval boundaries, stateless agent loops with external state management, and human-on-the-loop as default.

**Anthropic's data on agent autonomy** (Feb 2026): Among the longest-running Claude Code sessions, autonomous work stretches nearly doubled (25 to 45+ minutes) over three months. This increase was smooth across model releases, suggesting it reflects user trust growth, not just capability improvement. The implication: as users gain confidence, approval workflows should be configurable to progressively widen the agent's autonomous authority.

**Confidence: Strong (4/5)** -- HumanLayer verified from GitHub; Anthropic data from first-party research publication.

---

## 3. Practical Tactics

### Tactic 1: Build a JSON-RPC Control Daemon
**Difficulty: High | Impact: Critical**

Implement a long-running daemon process (Rust or TypeScript) that:
- Manages Claude Code sessions via the Agent SDK (`ProcessTransport`)
- Exposes a bidirectional JSON-RPC interface over Unix domain socket and/or WebSocket
- Implements the Item/Turn/Thread primitives from the Codex App Server pattern
- Handles approval requests with configurable timeout and default-deny
- Streams events to all connected clients in real-time
- Persists session state to disk (SQLite) for crash recovery and reconnection

This daemon is the core of Agent OS. Everything else is a client.

### Tactic 2: Discord Bot as First Client
**Difficulty: Medium | Impact: High**

Build the Discord bot as a pure client of the JSON-RPC daemon:
- Translates Discord messages to JSON-RPC requests
- Renders JSON-RPC notifications as Discord embeds/messages
- Maps Discord threads to agent sessions
- Implements RBAC by mapping Discord roles to permission levels
- Handles Discord's 2000-char limit by chunking with code-block awareness
- Rate-limit aware: queues messages when approaching Discord's 5/5s limit

Borrow heavily from zebbern/claude-code-discord's UX patterns (interactive buttons, slash commands, branch-to-channel mapping) but implement them as stateless translations of the underlying protocol.

### Tactic 3: CLI for Operations and Scripting
**Difficulty: Low | Impact: High**

Build a CLI client that speaks to the same daemon:
- `agentctl spawn --prompt "..." --model opus-4.6` -- start a new agent
- `agentctl list` -- show all running agents with status
- `agentctl attach <session-id>` -- stream live output
- `agentctl approve <request-id>` -- approve a pending action
- `agentctl kill <session-id>` -- terminate an agent
- `agentctl schedule --cron "0 */4 * * *" --skill deploy-check` -- recurring tasks
- NDJSON output mode for piping to other tools

The CLI is the primary interface for scripting, automation, and CI/CD integration.

### Tactic 4: Progressive Autonomy Configuration
**Difficulty: Medium | Impact: High**

Implement Feng et al.'s (2024) five levels of autonomy as configurable profiles:
- **Operator** (Level 1): Human types every command; agent is a tool
- **Collaborator** (Level 2): Agent proposes; human approves each step
- **Consultant** (Level 3): Agent executes routine actions autonomously; consults human for novel situations
- **Approver** (Level 4): Agent runs fully, pauses only for explicit approval-required actions
- **Observer** (Level 5): Agent runs fully autonomously; human monitors after the fact

Each agent session gets an autonomy level. The permission hook system determines what requires approval at each level. Allow users to start at Level 2 and graduate to Level 4 as trust builds.

### Tactic 5: Multi-Channel Notification Router
**Difficulty: Medium | Impact: Medium**

Build a notification subsystem with:
- Priority levels: `info` (agent completed task), `needs-input` (agent is blocked), `error` (agent failed), `cost-alert` (approaching budget limit)
- Channel routing: `info` goes to Discord channel; `needs-input` goes to Discord DM + mobile push; `error` goes to Discord + email; `cost-alert` goes to Discord + SMS
- Batch digest mode: instead of 50 individual notifications, send a summary every 15 minutes
- Quiet hours: suppress non-urgent notifications during family time (5-7pm per sam.md)

---

## 4. Contrarian & Minority Views

### Why Discord Might Be Wrong

The strongest argument against Discord as primary interface: **it trains you to think in chat messages, which is the wrong abstraction for agent management.** Agent management is closer to process management (htop, systemd) than to conversation (Slack, Discord). The natural interface for "monitor 8 parallel agents, intervene when one goes off-rails, review artifacts" is a dashboard, not a chat thread. Discord's linear, time-ordered message stream makes it hard to see agent state at a glance -- you have to scroll through conversation history to understand where an agent is. Warp's Oz session-sharing links (with visual artifact tracking and real-time steering) are a fundamentally better UX for this use case.

Counter-counter-argument: dashboards require you to go look at them. Discord notifications come to you. The right answer is probably both: Discord for notifications and quick commands, web dashboard for detailed monitoring and multi-agent oversight.

### Why API-First Might Be Over-Engineered

The case against protocol-first design: **you're building for a future that may never arrive.** If you only ever use Discord as your interface, the JSON-RPC daemon is unnecessary complexity. You could build the Discord bot directly on the Claude Agent SDK and have something working in a weekend instead of a month. The Codex App Server pattern was built by a team of engineers at OpenAI supporting five client surfaces; a solo developer supporting one surface (Discord) doesn't need that abstraction layer.

Counter: the cost of the abstraction layer is modest (a thin JSON-RPC wrapper around the Agent SDK), and the cost of NOT having it when you want to add a CLI or web dashboard is a complete rewrite. The protocol-first approach is insurance against future needs, and the premium is small.

### Why Warp/Oz Might Make Self-Hosting Obsolete

If Oz matures and its pricing is reasonable, self-hosting agent infrastructure on Hetzner might become like self-hosting email: technically possible, practically pointless. Oz already offers: cloud agent execution, session sharing, team management, scheduling, API/SDK, web and mobile access, and enterprise self-hosting. The "Agent OS" project might be solving a problem that Warp is about to solve commercially.

Counter: Oz is proprietary, closed-source, and priced per credit. For a power user running 50+ agent-hours per day, costs will be substantial. Self-hosting on a Hetzner VPS gives you unlimited compute at fixed cost. More importantly, self-hosting gives you full control over the agent's environment, permissions, and data -- no vendor lock-in, no terms-of-service changes, no privacy concerns about code passing through third-party infrastructure.

### Why the Academic Frameworks Might Not Apply

Sheridan & Verplank's levels of automation were designed for physical systems (teleoperation, nuclear plant control) where the cost of error is catastrophic and irreversible. AI coding agents operating in git repositories have fundamentally different risk profiles: most actions are reversible (git revert), the blast radius is bounded (sandboxed environment), and the human can always review before merge. Applying supervisory control theory to coding agents may introduce unnecessary ceremony.

Counter: the frameworks remain useful as design vocabulary even if the risk calculus is different. The five-level autonomy model (Feng et al., 2024) is specifically designed for AI agents and accounts for the lower-stakes context. And the "most actions are reversible" claim is only true if you design the system to make them reversible -- without proper sandboxing, an agent can `rm -rf /` or exfiltrate secrets, which are very much not reversible.

---

## 5. Decision Translation -- "So What?"

### Recommended Architecture for Agent OS Interface Layer

```
                    +-------------------+
                    |   Discord Bot     |  (TypeScript, discord.js)
                    +--------+----------+
                             |
                    +--------v----------+
                    |   Web Dashboard   |  (React/SvelteKit, WebSocket)
                    +--------+----------+
                             |
+----------------+  +--------v----------+  +----------------+
|   CLI Client   +-->   Control Daemon  <--+  Mobile Client  |
|  (Rust/TS)     |  |   (JSON-RPC over  |  |  (MobileCLI    |
+----------------+  |   Unix socket +   |  |   pattern)     |
                    |   WebSocket)      |  +----------------+
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
    +---------v--+  +--------v---+  +-------v----+
    | Agent 1    |  | Agent 2    |  | Agent N    |
    | (Claude    |  | (Claude    |  | (Claude    |
    |  Code SDK) |  |  Code SDK) |  |  Code SDK) |
    +------------+  +------------+  +------------+
```

**Build order:**
1. Control Daemon with JSON-RPC protocol (week 1-2)
2. CLI client (week 2)
3. Discord bot client (week 3-4)
4. Web dashboard (future)
5. Mobile client (future)

**Key decisions:**
- Protocol: JSON-RPC 2.0 over stdio (for local) and WebSocket (for remote). Study ACP spec for standardized event types.
- Session management: One Claude Code process per agent session. Sessions persist in SQLite. Reconnectable after daemon restart.
- Approval system: Inspired by Codex App Server's `item/commandExecution/requestApproval` pattern. Agent pauses, emits approval request, waits for response with configurable timeout.
- Autonomy levels: Configurable per session. Start conservative (Level 2/Collaborator), allow graduation.
- Notifications: Multi-channel with priority routing. Discord for most things; direct push for urgent items.
- Security: Input sanitization on all client messages before passing to agents. Never trust Discord message content as safe input. Implement the principle from Claude Code's hooks system: permission decisions are made by the daemon, not by the client.

### What Changes If This Research Is Wrong

If Discord proves sufficient and the protocol layer is unnecessary: you've built a well-structured daemon that you could simplify. Cost: ~2 weeks of extra engineering. If you skip the protocol layer and build directly on Discord and later need CLI or web access: you rewrite from scratch. Cost: the entire project.

The asymmetry strongly favors protocol-first design.

---

## Key Unknowns & Open Questions

1. **ACP maturity:** Will ACP become the standard, or will each vendor maintain proprietary protocols? If ACP wins, implementing ACP compliance in the daemon gives you free compatibility with Zed, Kiro, and future editors.

2. **Claude Agent SDK stability:** The SDK was renamed from `@anthropic-ai/claude-code` to `@anthropic-ai/claude-agent-sdk` in February 2026. How stable is the API? What breaking changes are expected?

3. **Warp Oz pricing at scale:** If Oz becomes cheap enough, the self-hosting argument weakens. Monitor their pricing closely.

4. **Discord bot verification requirements:** Discord has been tightening bot policies. At scale (100+ servers), verification and review requirements may introduce friction.

5. **Latency budget:** How much latency does the JSON-RPC translation layer add? For interactive use (human types, agent responds), even 100ms is fine. For streaming output, the daemon needs to forward events with minimal buffering.

---

## Source Log

| # | Source | Tier | Found Via | Contribution |
|---|--------|------|-----------|-------------|
| 1 | OpenAI (2026). "Unlocking the Codex Harness: How We Built the App Server." openai.com/index/unlocking-the-codex-harness/ | A | Exa | Core protocol design pattern: JSON-RPC, Items/Turns/Threads, bidirectional communication, approval flows |
| 2 | zebbern/claude-code-discord. GitHub. github.com/zebbern/claude-code-discord | B | Brave | Most complete Discord-to-Claude bridge: SDK integration, RBAC, permission buttons, mid-session controls, 7 agents |
| 3 | Warp (2026). "Introducing Oz: The Orchestration Platform for Cloud Agents." warp.dev/blog/oz-orchestration-platform-cloud-agents | B | Exa | Cloud agent orchestration: CLI/API/SDK/Web, session sharing, scheduling, environment model, 60% PR claim |
| 4 | Agent Client Protocol. agentclientprotocol.com | B | Brave | ACP specification: JSON-RPC 2.0 over stdio, agent-editor standardization |
| 5 | Schmid, P. (2026). "The Agent Client Protocol Overview." philschmid.de/acp-overview | C | Exa | ACP context: comparison to LSP, multi-agent support, list of implementing agents |
| 6 | Discord API Documentation. docs.discord.com/developers/topics/rate-limits | A | Exa | Rate limit specifics: 5/5s per channel, 50/10s global, per-route bucket system |
| 7 | Anthropic (2026). "Measuring AI Agent Autonomy in Practice." anthropic.com/research/measuring-agent-autonomy | A | Exa | Empirical data: autonomous session length doubling, trust growth independent of capability |
| 8 | Feng, K.J., McDonald, D.W., Zhang, A.X. (2024). "Levels of Autonomy for AI Agents." arXiv:2506.12469 | A | Exa | Five-level autonomy framework: operator/collaborator/consultant/approver/observer |
| 9 | Sheridan, T.B. & Verplank, W.L. (1978). "Human and Computer Control of Undersea Teleoperators." MIT Man-Machine Systems Lab | S | Semantic Scholar | Foundational 10-level automation taxonomy |
| 10 | Sheridan, T.B. (2011). "Adaptive Automation, Level of Automation, Allocation Authority, Supervisory Control." IEEE Trans. SMC-A | A | Semantic Scholar | Updated taxonomy with adaptive automation distinctions, 157 citations |
| 11 | Shneiderman, B. (2020). "Human-Centered Artificial Intelligence: Reliable, Safe & Trustworthy." arXiv:2002.04087 | A | Exa | HCAI framework: high automation + high human control simultaneously |
| 12 | HumanLayer/agentcontrolplane. GitHub. github.com/humanlayer/agentcontrolplane | B | Brave, Tavily | Distributed agent scheduler, approval routing via Slack/Discord/email, durable task execution |
| 13 | MichaelAyles/claude-code-comm-bot. GitHub. github.com/MichaelAyles/claude-code-comm-bot | C | Brave | VS Code extension with Discord mirroring, alternative "notification layer" pattern |
| 14 | harnesscli. crates.io/crates/harnesscli | C | Brave | Unified CLI: spawns multiple agent backends, NDJSON event stream output |
| 15 | MobileCLI. mobilecli.app | C | Brave | Self-hosted Rust daemon streaming agent sessions to phone, approval from mobile |
| 16 | Codeman (Ark0N). GitHub. github.com/Ark0N/Codeman | C | Brave | Web UI for managing Claude Code + OpenCode in tmux sessions |
| 17 | Devin/Cognition AI. docs.devin.ai/integrations/slack; linear.app/integrations/devin | B | Exa | Multi-channel agent interface: Slack, Linear, CLI, Web, API patterns |
| 18 | Cursor CLI update (Jan 2026). forum.cursor.com | C | Brave | Background agent + cloud handoff UX pattern |
| 19 | Snyk Labs (2024). "Agent Hijacking: The True Impact of Prompt Injection Attacks." labs.snyk.io | A | Exa | Prompt injection risks in agent systems with tool access |
| 20 | Lasso Security (2026). "Prompt Injection Examples That Expose Real AI Security Risks." lasso.security/blog | B | Exa | Multi-agent infection, indirect prompt injection, authority flow in agentic architectures |
| 21 | Ry Walker Research (2026). "Warp Oz." rywalker.com/research/warp-oz | C | Exa | Independent analysis of Oz: Slack/Linear integrations, multi-model support, enterprise self-hosting |
| 22 | Claude Code Hooks Reference. code.claude.com/docs/en/hooks | A | Brave | Permission hook system: decision block/deny via JSON response, hook lifecycle |
| 23 | Tsamados, A., Floridi, L., Taddeo, M. (2024). "Human Control of AI Systems: From Supervision to Teaming." AI Ethics, 5(2), 1535-1548 | A | Exa | Academic framework: supervision-to-teaming continuum for AI oversight |
| 24 | Browseract (2026). "Clawdbot Guide: Build a Secure Self-Hosted AI Control Plane." browseract.com | C | Tavily | Clawdbot as local control plane pattern: agentic persistence, cron tasks, memory management |
| 25 | Tessl (2026). "As Coding Agents Become Collaborative Co-Workers, Orchestration Takes Center Stage." tessl.io/blog | C | Exa | Industry analysis: Codex, Augment, Warp orchestration approaches compared |
| 26 | JessyTsui/Claude-Code-Remote. GitHub. | C | Brave | Remote control via email/Discord/Telegram: notification + reply-to-command pattern |
| 27 | OneUpTime (2026). "How to Build Agent Supervision." oneuptime.com/blog | C | Exa | Practical supervision patterns: resource limits, escalation paths, loop detection |
| 28 | Permit.io (2026). "Human-in-the-Loop for AI Agents." permit.io/blog | C | Brave | HumanLayer SDK integration patterns, approval routing across channels |

---

## Audit Notes

1. **Sheridan & Verplank (1978) original paper:** Confirmed existence via Semantic Scholar and ResearchGate. The 10-level taxonomy is widely cited (Sheridan 2011 paper has 157 citations). I could not access the original 1978 MIT report directly but the taxonomy is reproduced consistently across multiple secondary sources.

2. **Warp Oz "60% of PRs" claim:** This is a first-party claim from Warp's CEO (Zach Lloyd) in the launch blog post. Not independently verified. Treat with appropriate skepticism -- it is a marketing claim from a company launching a product.

3. **Discord rate limit specifics:** Verified against official Discord API documentation. The 5/5s per channel and 50/10s global limits are documented; however, Discord notes these are subject to change and should not be hard-coded. Actual limits may vary.

4. **Anthropic autonomy data:** First-party research from Anthropic. The finding that autonomous session length doubled over 3 months is based on their internal telemetry. Methodology is described as "privacy-preserving" but full details of the measurement approach are not published.

5. **Feng et al. (2024) autonomy levels paper:** Found via Exa search. The arXiv paper ID (2506.12469) suggests this may be a 2025 publication with a 2024 working paper date. The five-level framework has not been widely cited yet -- it is a working paper, not peer-reviewed.

6. **Disclaude:** Could not verify. The disclaude.com domain did not resolve during this research session. Claims about its tmux-based architecture come from Reddit threads and Brave search descriptions only. Labeled accordingly as Emerging (2/5) confidence.

7. **Potential bias:** This research may over-weight the protocol-first approach because the Codex App Server blog post is exceptionally well-written and detailed, while the "just build a Discord bot" approach is represented by smaller, less documented projects. The simpler approach may be underrepresented in the evidence simply because nobody writes blog posts about simple solutions.

8. **Missing perspective:** This research does not cover Telegram, WhatsApp, or SMS as control channels in depth. For a user in Israel, WhatsApp might be a more natural notification channel than Discord. The multi-channel notification router recommendation addresses this implicitly, but specific WhatsApp integration patterns are not researched here.

---

_Shelf-life markers:_
- Protocol patterns (JSON-RPC, ACP): Monitor -- active evolution, may stabilize in 6-12 months
- Discord bot implementations: Perishable -- specific repos and APIs changing weekly
- Oz/Warp: Perishable -- product and pricing evolving monthly
- Supervisory control theory: Durable -- foundational frameworks stable for decades
- Autonomy levels (Feng et al.): Monitor -- promising framework, not yet widely adopted
