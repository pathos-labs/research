# Agent OS — Architecture Decision Record

_Date: 2026-03-02 | Shelf-life: Yellow — Monitor (6-12 months; architectural patterns stabilizing but tooling evolves weekly)_

_Track 8 of 8 in the Agent OS research series. Synthesizes Tracks 1-7 into implementable decisions._

---

## Executive Summary

The Agent OS is a long-running control daemon, written in TypeScript, that manages Claude Code CLI agent processes on a single Hetzner VPS via the Claude Agent SDK. It exposes a bidirectional JSON-RPC protocol over Unix domain socket and WebSocket, with Discord as the first client and a CLI as the second. Agents run inside Docker containers with per-agent filesystem isolation, cost circuit breakers at four levels, and scheduled session rotation every 4-8 hours to prevent context degradation. MCP tool access flows through a single gateway that handles namespace prefixing, lazy loading, and read/write pool separation. The system is designed for a solo operator managing 5-15 agents across research, coaching, content, and consulting — lean enough to maintain alone, structured enough to run for days without intervention.

---

## Decision 1: Language & Runtime

### Decision

**TypeScript on Node.js (Deno optional), using the Claude Agent SDK's `ProcessTransport`.**

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **TypeScript** | Sam already maintains TS projects (whatsapp-mcp-ts). Claude Agent SDK is TypeScript-native. Largest agent ecosystem (OpenClaw, NanoClaw, zebbern/claude-code-discord all TS). discord.js is mature. Async I/O maps well to concurrent agent management. | Node.js memory leaks over multi-day runs (OpenClaw's documented 1GB+ growth). Not the fastest runtime. |
| **Rust** | Memory safety, no GC pauses. ZeroClaw proves the architecture works. Single binary deployment. Best security properties. | Sam has no Rust experience. Ecosystem for agent harnesses is smaller. Development velocity 2-3x slower for unfamiliar devs. Claude Agent SDK has no Rust binding. |
| **Go** | Single binary. PicoClaw proves viability. Good concurrency primitives. Lower memory than Node. | No Claude Agent SDK binding. Agent ecosystem gravitates toward TS/Python. Smaller async ecosystem than Node. |
| **Zig** | NullClaw's 678KB binary is remarkable. Zero dependencies. | Tiny ecosystem. Sam has no Zig experience. No SDK support. Maintenance becomes a solo burden with no community to draw from. |

### Evidence

- Track 7: The three most feature-complete harnesses (OpenClaw, NanoClaw, zebbern/claude-code-discord) are all TypeScript. The Claude Agent SDK is TypeScript-first.
- Track 6: NanoClaw's entire codebase fits in ~35,000 tokens — Claude can understand and help maintain it. This "AI-maintainable codebase" property is strongest in TypeScript.
- Track 1: Node.js memory leaks are real but manageable with scheduled session rotation (4-8 hours) and systemd `MemoryMax` hard caps.
- Track 7: ZeroClaw and NullClaw prove that Rust/Zig produce superior binaries, but both require deep language expertise Sam doesn't have.

### Confidence: ★★★★☆ Strong

The choice is dictated less by technical merit (Rust would be technically superior) and more by practical constraints: Sam needs to maintain this alone, the SDK is TypeScript, and the ecosystem is TypeScript-heavy. TypeScript is the 80/20 choice.

### Risks

- **Memory leaks over multi-day runs.** Mitigation: scheduled session rotation, systemd MemoryMax, monitoring RSS growth.
- **Single-threaded event loop.** Mitigation: each agent runs as a separate process (via ProcessTransport), so the daemon itself handles only coordination, not agent computation.
- **If Sam later wants to contribute upstream to Rust/Go projects, TypeScript skills don't transfer.** Mitigation: the protocol-first design means the daemon could be rewritten without changing any client.

---

## Decision 2: Process Model

### Decision

**Container-per-agent via Docker, supervised by a TypeScript control daemon, supervised by systemd. Three-level supervision tree.**

```
systemd (root supervisor)
  └── agent-os daemon (application supervisor)
        ├── Agent 1 (Docker container → Claude Code SDK process)
        ├── Agent 2 (Docker container → Claude Code SDK process)
        └── Agent N (Docker container → Claude Code SDK process)
```

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Container-per-agent (Docker)** | Strongest isolation short of MicroVMs. Per-agent resource limits (memory, CPU). Clean restart semantics. Blast radius containment. NanoClaw proves months of uptime. | ~50-100MB overhead per container. ~2s cold start. Docker daemon as dependency. |
| **Process-per-agent (no containers)** | Lower overhead (~20-50MB). Instant start. Simpler. Atalay's VPS fleet runs this way. | Shared kernel, shared filesystem (unless manually scoped). One agent's misbehavior can affect others. Weaker security boundary. |
| **MicroVM-per-agent (Firecracker/Kata)** | Strongest possible isolation. Dedicated kernel per agent. | Requires KVM support on VPS (check Hetzner). Higher complexity. ~125ms cold start is fine but adds infrastructure. Overkill for trusted code running your own prompts. |
| **Single process (OpenClaw)** | Simplest. One process, one event loop. | Single point of failure. One agent's OOM or infinite loop kills everything. OpenClaw's documented instability at scale. |

### Evidence

- Track 1: NanoClaw's container-per-agent reports months of uptime. OpenClaw's single-process model has documented memory leak issues over days.
- Track 4: "AI agents are not chatbots — they execute code, modify files, and make network calls. The threat model is closer to running untrusted code." Container isolation is the minimum viable security boundary.
- Track 4: MicroVMs (Firecracker) boot in ~125ms with <5MB overhead, but require KVM and add infrastructure complexity. Docker containers are the pragmatic middle ground for a solo operator.
- Track 1: A 32GB Hetzner VPS running 4-8 agents with Docker overhead of ~50-100MB per container uses under 1GB total for container overhead — acceptable.

### Confidence: ★★★★☆ Strong

Container-per-agent is the consensus approach across NanoClaw, Cursor cloud agents, and Docker's own "Compose for the AI Agent Era" guidance. The only question is whether to upgrade to MicroVMs later if the threat model demands it.

### Risks

- **Docker daemon failure takes down all agents.** Mitigation: systemd restarts Docker automatically. Agents restart with their containers.
- **Container overhead limits max agents.** At ~100MB per container on a 32GB VPS, this caps at roughly 100+ agents — far beyond the 5-15 target.
- **Hetzner VPS may not support nested virtualization (needed for Firecracker/Kata).** Mitigation: Docker containers work without KVM. Upgrade to dedicated server if MicroVMs become necessary.

---

## Decision 3: Control Daemon Architecture

### Decision

**A single TypeScript daemon process that exposes a bidirectional JSON-RPC 2.0 protocol over Unix domain socket (local) and WebSocket (remote). The daemon manages agent lifecycle, enforces budgets, routes approvals, streams events, and persists state to SQLite. It does NOT contain agent logic — agents are Claude Code processes in containers.**

### What the Daemon Does

1. **Agent lifecycle management**: create, configure, start, monitor, rotate, stop, destroy agents.
2. **Approval routing**: receives approval requests from agents, forwards to connected clients, returns decisions with configurable timeout (default-deny).
3. **Event streaming**: all agent events (tool calls, completions, errors, cost updates) streamed to all connected clients in real-time.
4. **Budget enforcement**: four-level cost hierarchy (per-call, per-agent-hourly, per-task, global-daily). Hard circuit breakers.
5. **State persistence**: SQLite WAL for agent state, conversation checkpoints, cost tracking. WAL checkpoint every 10 minutes.
6. **Session rotation**: scheduled rotation every 4-8 hours (configurable per agent). Generates handoff document, stops old session, starts new session with handoff context.
7. **Health monitoring**: heartbeat collection, metric aggregation, alert dispatch.
8. **MCP gateway coordination**: manages the MCP gateway process, routes tool access per agent scoping rules.

### What the Daemon Does NOT Do

- Run agent logic (that's the Claude Code SDK process inside the container).
- Execute user commands directly (that's the agent's job).
- Parse Discord messages (that's the Discord client's job).
- Manage Docker infrastructure (that's Docker and systemd's job).

### Protocol

JSON-RPC 2.0, bidirectional. The daemon both receives requests and sends requests (approval prompts to clients). Transport: Unix domain socket for local CLI, WebSocket for remote clients (Discord bot, future web dashboard).

Study the Codex App Server pattern (Items/Turns/Threads primitives) and ACP spec for design vocabulary, but don't implement full ACP compliance — it's not yet stable enough to bet on.

### Evidence

- Track 3: "Design the Agent OS control plane as a JSON-RPC server first. Every frontend becomes a thin translation layer. This is the load-bearing decision."
- Track 3: Codex App Server's bidirectional JSON-RPC (Items/Turns/Threads) is the most detailed public protocol design for agent control planes.
- Track 3: "The cost of the abstraction layer is modest. The cost of NOT having it when you want to add a CLI or web dashboard is a complete rewrite. The asymmetry strongly favors protocol-first design."
- Track 1: SQLite WAL for state persistence requires zero external infrastructure and handles concurrent reads during writes.

### Confidence: ★★★★★ Established

The protocol-first pattern is universal across Codex, Devin, Warp Oz, and ACP. Every successful multi-surface agent system follows this architecture.

### Risks

- **Over-engineering for a system that may only ever use Discord.** Mitigation: the protocol layer is thin — it's a JSON-RPC wrapper around SDK calls. The additional cost is days, not weeks.
- **WebSocket complexity for remote access.** Mitigation: start with Unix socket only. Add WebSocket when the Discord client needs it.
- **SQLite as single point of state.** Mitigation: WAL mode handles concurrent access. Backup via periodic `VACUUM INTO` or filesystem snapshots.

---

## Decision 4: Agent Lifecycle

### Decision

**Convention-over-configuration: a directory with a CLAUDE.md creates an agent. Optional `agent.yaml` for explicit settings. The daemon watches agent directories and auto-discovers changes.**

### Lifecycle Stages

```
discover → create → configure → run → monitor → rotate → idle → teardown
```

1. **Discover**: Daemon watches `agents/` directory via inotify. New subdirectory with CLAUDE.md triggers agent creation.
2. **Create**: Daemon reads CLAUDE.md (identity, instructions, constraints) and optional `agent.yaml` (model, budget, schedule, tools, autonomy level).
3. **Configure**: Docker container provisioned with agent's workspace mounted. MCP gateway scoped to agent's allowed tools.
4. **Run**: Claude Code SDK `ProcessTransport` spawns inside the container. Agent begins executing per its CLAUDE.md.
5. **Monitor**: Heartbeat every 60 seconds. Metrics: RSS memory, token rate, error rate, context depth, cost accumulation.
6. **Rotate**: At 4-8 hour intervals (or at 75% context depth), generate handoff document, gracefully stop current session, start new session with handoff context.
7. **Idle**: Agent with no active task. Container kept warm for fast reactivation. Idle timeout configurable (default: 1 hour before container is stopped).
8. **Teardown**: Graceful shutdown. Persist final state. Remove container. Keep workspace directory for auditability.

### Agent Configuration (Convention Defaults)

```yaml
# agents/research/agent.yaml (all fields optional)
model: claude-sonnet-4    # default: sonnet-class
budget_hourly: 3.00       # default: $2.00
budget_task: 10.00        # default: $5.00
autonomy: consultant      # default: collaborator (levels: operator, collaborator, consultant, approver, observer)
rotation_hours: 6         # default: 4
schedule: null             # cron expression for recurring tasks
tools:                     # MCP tool scoping
  - exa.*
  - brave-search.*
  - semantic-scholar.*
  - firecrawl.firecrawl_scrape
  - firecrawl.firecrawl_search
```

If no `agent.yaml` exists, all defaults apply. The CLAUDE.md alone is sufficient for a working agent.

### Evidence

- Track 6: "Convention-over-configuration wins for small operators. A directory with a CLAUDE.md is the simplest possible 'add an agent' pattern." Time to add: ~2 minutes.
- Track 6: NanoClaw's CLAUDE.md-as-configuration pattern: "the CLAUDE.md file IS the agent's configuration. Identity, instructions, permissions, behavioral constraints — all in one Markdown file."
- Track 6: Hot-reload via inotify is the recommended pattern for configuration updates without restart.
- Track 1: Scheduled session rotation every 4-8 hours, with structured handoff documents, is the evidence-backed approach for multi-day coherence.

### Confidence: ★★★★☆ Strong

Directory-convention is proven at NanoClaw's scale. The scaling threshold (~50 agents) where explicit config becomes necessary is far above Sam's target of 5-15.

### Risks

- **Convention ambiguity at scale.** Mitigation: `agent.yaml` provides explicit override when conventions aren't enough. This is the Rails pattern — sensible defaults with escape hatches.
- **Inotify watches on large directory trees can miss events.** Mitigation: supplement with a periodic scan (every 60 seconds) as a fallback.
- **Handoff document quality determines rotation success.** Mitigation: invest in prompt engineering for handoff generation. Track 1's template is the starting point. Iterate based on observed quality.

---

## Decision 5: Multi-Agent Coordination

### Decision

**Orchestrator-Worker as the default pattern, with a filesystem-backed blackboard available for collaborative tasks. Filesystem as primary IPC. Lightweight JSON messages for coordination signals.**

### How It Works

1. **Orchestrator-Worker (default)**: A lead agent (Opus-class model) decomposes the task, spawns workers (Sonnet-class) with isolated contexts, and synthesizes results. Workers write artifacts to `/workspace/{task-id}/shared/`. Workers do not communicate with each other directly.

2. **Blackboard (opt-in)**: For tasks requiring incremental collaboration (research synthesis, complex debugging), agents read/write to a shared SQLite-backed blackboard. The blackboard stores structured entries (findings, decisions, hypotheses) with timestamps and agent attribution.

3. **Task DAG**: The orchestrator's decomposition produces a directed acyclic graph of subtasks with declared dependencies. The daemon executes independent tasks in parallel, resolves dependencies, and handles failures (retry, skip, compensate).

### Communication Channels

| Channel | Use Case | Mechanism |
|---------|----------|-----------|
| Filesystem artifacts | Primary output exchange | Agents write to `/workspace/{task-id}/`, daemon watches via inotify |
| Coordination messages | Ready/blocked/failed/done signals | Lightweight JSON via daemon message router |
| Blackboard | Shared state for collaborative tasks | SQLite with agent-attributed entries |
| Direct messaging | Peer-to-peer (rare, for debate/challenge) | Via daemon message router, opt-in |

### Token Economics

Model tiering by role is mandatory:
- **Orchestrator/Planner**: Opus-class (best reasoning for decomposition quality)
- **Specialized workers**: Sonnet-class (good enough for scoped tasks)
- **Validators/Checkers**: Haiku-class (simple verification)

Budget: multi-agent tasks cost ~15x single-agent baseline. A complex research task may cost $5-8 total. The per-task budget in `agent.yaml` governs this.

### Evidence

- Track 2: "The orchestrator-worker pattern is the default. Established (5/5) — Universal across all surveyed production systems."
- Track 2: "Context isolation improves performance. Subagents pattern processes 67% fewer tokens than Skills pattern." (LangChain data)
- Track 2: Blackboard architecture "achieved best average performance across commonsense, reasoning, and math benchmarks while spending fewer tokens." (Han & Zhang 2025)
- Track 2: "Default to isolation. Share only what's necessary, via the narrowest channel possible."
- Track 2: AgentPrune shows 28-73% of inter-agent communication tokens are redundant — design for compression from the start.

### Confidence: ★★★★☆ Strong

Orchestrator-worker is universally proven. Blackboard is academically supported but has limited production deployment for LLM agents. Starting with orchestrator-worker and adding blackboard incrementally is low-risk.

### Risks

- **Orchestrator as single point of failure.** Bad decomposition ruins everything downstream. Mitigation: use the best available model for orchestration. Implement plan review (the lead shows its plan before executing).
- **Multi-agent cost explosion.** Mitigation: per-task budgets, difficulty-aware scaling (simple tasks get 1 agent, not 5), model tiering.
- **Debugging multi-agent failures is hard.** Track 2 cites only 14.2% accuracy in pinpointing failure steps. Mitigation: correlation IDs across all agents in a task, full trace logging to Langfuse.

---

## Decision 6: Memory Architecture

### Decision

**CLAUDE.md as the primary agent identity/instruction store. SQLite for operational state (checkpoints, cost tracking, task state). Structured handoff documents for session rotation. No external memory database for v1.**

### Three Memory Layers

| Layer | Contents | Storage | Lifetime |
|-------|----------|---------|----------|
| **Identity** | Who the agent is, what it does, constraints, voice | CLAUDE.md in agent directory | Permanent (edited by operator) |
| **Working** | Current conversation, task progress, decisions | Claude Code context window | One session (4-8 hours) |
| **Operational** | Checkpoints, cost data, task DAG state, metrics | SQLite WAL | Persistent across sessions |

### Context Degradation Strategy

1. **Proactive rotation** (primary): Every 4-8 hours, or when context depth hits 75%, the daemon triggers session rotation. A structured handoff document captures: current task state, key decisions, pending work, files modified, learned constraints.

2. **Tool result clearing** (within session): Agent is instructed to externalize large outputs to files and keep only references in context.

3. **Compaction** (last resort): If rotation cannot be triggered (e.g., mid-critical-operation), rely on Claude Code's built-in auto-compaction. But never depend on it for multi-day coherence.

### Why Not an External Memory Database (Mem0, Letta, Zep)

Track 1 and prior research on memory daemons show that external memory systems add infrastructure complexity without solving the core problem for this use case. The handoff document + CLAUDE.md + SQLite checkpoint covers 90% of memory needs. Durable semantic memory (knowledge that persists across tasks and projects) can be added in v2 via a lightweight MCP server that reads/writes to the operator's Obsidian vault or a simple Markdown knowledge base.

### Evidence

- Track 1: "Proactive session rotation beats reactive compaction. 1M context window degrades around 256K tokens — well before the limit."
- Track 1: "Scheduled session rotation every 4-8 hours with structured handoff documents is more reliable than relying on compaction alone."
- Track 6: "CLAUDE.md as single source of truth — the NanoClaw pattern. Identity, instructions, permissions, behavioral constraints — all in one Markdown file."
- Track 1: "SQLite WAL as the primary state store requires zero external infrastructure."
- Prior research (memory-daemon-llm-agents.md): External memory systems add 200-500ms latency per retrieval and require their own operational overhead.

### Confidence: ★★★★☆ Strong

The handoff + checkpoint approach is proven at NanoClaw and in Anthropic's internal harness (feature list JSON + progress.txt + git commits). The main uncertainty is handoff document fidelity — no one has rigorously measured how much context survives rotation.

### Risks

- **Handoff document loses critical context.** Mitigation: iterate on the handoff template. Log cases where the new session "forgets" something. Gradually improve the handoff prompt.
- **SQLite WAL growth on long-running daemon.** Mitigation: `PRAGMA wal_checkpoint(TRUNCATE)` every 10 minutes via timer.
- **No semantic memory means agents can't learn across projects.** Mitigation: acceptable for v1. The operator's CLAUDE.md files and skill files serve as manually curated long-term memory. Automated cross-project learning is v2.

---

## Decision 7: Interface Layer

### Decision

**Protocol-first: JSON-RPC daemon as the core. Discord bot as first client. CLI as second client. Web dashboard deferred to v2.**

### Architecture

```
    Discord Bot (TS, discord.js)              CLI (TS or Bash)
         |                                        |
         v                                        v
    WebSocket ←→ Control Daemon (JSON-RPC) ←→ Unix Socket
                        |
              +---------+---------+
              |         |         |
           Agent 1   Agent 2   Agent N
```

### Discord Client Design

- **Channel mapping**: One Discord thread per agent task. Agent output streams into the thread. The thread title includes agent name and status.
- **Approval workflow**: Interactive buttons (Allow/Deny) for permission requests. Configurable timeout with default-deny. Batch approval for similar actions.
- **Notifications**: Priority routing — `info` to channel, `needs-input` to DM, `error` to DM + @mention, `cost-alert` to DM + @mention.
- **Rate limit awareness**: Queue messages when approaching Discord's 5/5s limit. Chunk messages at 2000 chars with code-block awareness.
- **RBAC**: Discord role IDs mapped to permission levels in daemon config.
- **The bot contains zero agent management logic.** It translates Discord interactions to JSON-RPC calls and renders JSON-RPC events as Discord messages.

### CLI Design

```bash
agentctl spawn --agent research --task "Find papers on X"
agentctl list                          # all agents with status
agentctl attach <session-id>           # stream live output
agentctl approve <request-id>          # approve pending action
agentctl kill <session-id>             # terminate agent
agentctl health                        # morning health check summary
agentctl cost                          # cost report (today, this week)
agentctl schedule --agent deploy-check --cron "0 */4 * * *"
```

NDJSON output mode for piping to other tools.

### Autonomy Levels

Configurable per agent, based on Feng et al. (2024):
1. **Operator**: Human types every command
2. **Collaborator** (default): Agent proposes, human approves each step
3. **Consultant**: Agent runs routine actions autonomously, consults on novel situations
4. **Approver**: Agent runs fully, pauses only for explicit approval-required actions
5. **Observer**: Fully autonomous, human monitors after the fact

Start new agents at Collaborator. Graduate to Consultant/Approver as trust builds.

### Evidence

- Track 3: "Protocol-first design is the load-bearing decision. If you skip the protocol layer and build directly on Discord and later need CLI or web access: you rewrite from scratch."
- Track 3: zebbern/claude-code-discord demonstrates comprehensive Discord-to-Claude bridging (45+ slash commands, RBAC, mid-session controls).
- Track 3: Discord imposes hard constraints at scale (5/5s rate limit, 2000 char messages, no structured data). The protocol layer insulates the daemon from these limitations.
- Track 3: Feng et al. (2024) five-level autonomy framework maps directly to agent permission configuration.

### Confidence: ★★★★★ Established

Protocol-first is the unanimous recommendation from Track 3 and is validated by Codex, Devin, Warp Oz, and ACP.

### Risks

- **Discord as primary interface trains the operator to think in chat, which is the wrong abstraction for process management.** Mitigation: the CLI provides the "htop for agents" view. Discord handles notifications and quick commands.
- **Discord platform risk.** Mitigation: the bot is a replaceable client. Switching to Slack, Telegram, or a web UI requires writing a new thin client, not rewriting the core.
- **Prompt injection via Discord messages.** Mitigation: all Discord input is sanitized before passing to agents. The daemon enforces permission boundaries, not the Discord bot.

---

## Decision 8: MCP Integration

### Decision

**MCP gateway as a single point of tool access. Read/write server pool separation. Lazy loading via Tool Search. Per-agent tool scoping via gateway configuration.**

### Architecture

```
Agents → MCP Gateway (single /mcp endpoint)
              |
    +---------+---------+---------+
    |         |         |         |
  Read Pool  Write Pool  Code Pool
  (shared)   (per-agent)  (sandboxed)
```

### Server Pools

| Pool | Servers | Sharing | Rationale |
|------|---------|---------|-----------|
| **Search & Discovery** | Exa, Brave, Tavily, Firecrawl Search | Shared (read-only) | No side effects, safe for concurrent access |
| **Academic** | Paper Search, Semantic Scholar, OpenAlex, Crossref, Unpaywall | Shared (read-only) | No side effects |
| **Communication** | WhatsApp MCP, Fathom | Per-agent | Side effects (message sending) require isolation |
| **Code & Filesystem** | Filesystem, Bash | Per-agent, sandboxed | Must be sandboxed per container |
| **Future Write Tools** | GitHub, Jira, Slack, Notion | Per-agent | Side effects |

### Lazy Loading

When total MCP tool definitions exceed 10K tokens, the gateway presents a single `search_tools` meta-tool instead of dumping all definitions. Agents search for tools by keyword and receive 3-5 relevant tool definitions per query. This achieves 47-95% context reduction (Claude Code's measured data).

For advanced use: auto-generate typed filesystem wrappers for tools (the "code execution with MCP" pattern). The agent explores a `tools/` directory to discover capabilities rather than receiving schema in context. This achieves 98.7% reduction (Anthropic's data).

### Per-Agent Tool Scoping

The `agent.yaml` `tools` field specifies which MCP tools the agent can access, using glob patterns:

```yaml
tools:
  - exa.*                    # all Exa tools
  - brave-search.*           # all Brave tools
  - firecrawl.firecrawl_scrape  # specific Firecrawl tool
```

The gateway filters the tool catalog per-agent based on this configuration. Agents never see tools they aren't authorized to use.

### Gateway Implementation

Start with a custom lightweight gateway (TypeScript, using the MCP SDK as a proxy). If infrastructure needs grow, evaluate AgentGateway (open-source, Kubernetes-native). The gateway handles:
- Tool catalog aggregation with namespace prefixing (`server_name.tool_name`)
- Health checks with automatic failover for unhealthy servers
- Lazy loading / Tool Search endpoint
- Per-agent rate limiting
- Request logging for cost tracking

### Evidence

- Track 5: "The gateway pattern is the critical architectural decision. ★★★★☆"
- Track 5: "Tool schema bloat is the biggest operational problem at scale. Loading all tool definitions upfront consumes 54-85% of available context. ★★★★★"
- Track 5: "MCP is the wrong layer for agent-to-agent coordination. Use MCP for capability, a separate mechanism for coordination. ★★★★☆"
- Track 5: Agent-Zero GitHub #912 documents deadlock from multi-session MCP concurrent access — read/write pool separation prevents this.

### Confidence: ★★★★☆ Strong

The gateway pattern is converging across AgentGateway, Kong, Traefik Hub, and AWS Bedrock AgentCore. Tool bloat is quantified by 5+ independent sources. The main uncertainty is whether a custom gateway or AgentGateway is the better starting point.

### Risks

- **Gateway as single point of failure.** Mitigation: direct-connection fallback for critical tools if the gateway is down. The gateway should be supervised by systemd with auto-restart.
- **Custom gateway maintenance burden.** Mitigation: keep it minimal — tool catalog aggregation, namespace prefixing, health checks, lazy loading. No advanced features until needed.
- **MCP spec evolution may break things.** Mitigation: pin to a specific MCP SDK version. Upgrade intentionally after testing.

---

## Decision 9: Security Model

### Decision

**Defense-in-depth across five layers: container isolation, command allowlists, cost circuit breakers, prompt injection defense, and immutable audit logging. Default-deny on everything.**

### Layer 1: Container Isolation (Docker)

Each agent runs in a Docker container with:
- `--memory 2g` hard cap
- `--cpus 1.0` limit
- Read-only root filesystem except for the agent's workspace directory
- No `--privileged`, no `--cap-add`
- Network: outbound-only to allowed domains (Anthropic API, MCP server hosts). No inbound ports.
- No Docker socket mounted (prevents container escape via Docker API)

### Layer 2: Command Allowlists (PreToolUse Hooks)

Claude Code's PreToolUse hooks enforce an allowlist. The hook script receives tool input JSON and exits with code 2 to block.

Default allowlist (per agent, configurable):
- File reads: allowed within workspace
- File writes: allowed within workspace
- Bash commands: allowed from explicit list (git, npm, python, node, etc.)
- Network: allowed to whitelisted domains only
- Destructive commands (`rm -rf /`, `git reset --hard`, `dd`, `mkfs`): always denied

### Layer 3: Cost Circuit Breakers

Four-level hierarchy, enforced by the daemon:

| Level | Default | Action |
|-------|---------|--------|
| Per-call token ceiling | 100K input, 16K output | Reject the call |
| Per-agent hourly budget | $2.00 | Pause agent, alert operator |
| Per-task budget | $5.00 | Fail task, alert operator |
| Global daily budget | $20.00 | Pause all agents, require manual reset |

Alert thresholds: 50% of hourly → log warning. 70% of daily → Discord notification. Any single call > $1.00 → flag for review. Error rate > 20% in 5 minutes → pause agent.

### Layer 4: Prompt Injection Defense

- All user input from Discord is sanitized before passing to agents (strip markdown injection patterns, limit length).
- Agents run with system prompts that establish identity before processing user input.
- MCP tool responses are treated as untrusted data (tool outputs can contain injection payloads).
- Inter-agent messages pass through the daemon, not directly between agents (prevents agent-in-the-middle attacks per He et al. 2025).

Prompt injection is architecturally unsolvable at the model level (Track 4, OpenAI's own admission). These are mitigations, not solutions. The structural controls (containers, allowlists, budgets) are the real security boundary.

### Layer 5: Immutable Audit Logging

Every tool call, API call, file modification, approval decision, and cost event is logged to an append-only SQLite database. The agent cannot modify its own logs (logs live outside the agent's container mount). Logs include:
- Timestamp, agent ID, session ID, correlation ID (for multi-agent tracing)
- Tool name, input, output (truncated at 10K chars)
- Token usage, cost
- Approval requests and decisions

### Secrets Management

API keys and tokens stored in environment variables injected into containers at runtime. Never mounted as files in the agent's workspace. Never included in CLAUDE.md or agent.yaml. For v1, a `.env` file managed by the operator. For v2, consider HashiCorp Vault for dynamic, short-lived, scoped tokens.

### Evidence

- Track 4: "Treat every AI agent as if it were running arbitrary code from an untrusted source. ★★★★★"
- Track 4: "Allowlists are safer than denylists. ★★★★★"
- Track 4: "A single Claude Code recursion loop consumed 1.67 billion tokens ($16,000-$50,000) in five hours. ★★★★★"
- Track 4: "Prompt injection is architecturally unsolvable at the model level. ★★★★★"
- Track 4: OpenClaw's failures (0.0.0.0 binding, no gateway auth, trusting community skills) provide the precise anti-pattern playbook.

### Confidence: ★★★★★ Established

Every principle here is well-established in security engineering. The only novelty is applying them to AI agents, which Track 4 documents thoroughly.

### Risks

- **Security friction slows agent productivity.** Mitigation: start with a sensible allowlist per agent role. Expand as trust builds. The autonomy levels (Decision 7) provide a graduated trust model.
- **False positives in command blocking.** Mitigation: log all blocked commands. Review and adjust allowlists weekly based on false positive data.
- **Immutable logs grow large.** Mitigation: rotate logs daily. Archive to compressed files. Retain 30 days online, older to cold storage.

---

## Decision 10: Deployment & Operations

### Decision

**Single Hetzner VPS, Docker Compose, systemd supervision, Langfuse for observability, Discord webhooks for alerting.**

### Deployment Stack

```yaml
# docker-compose.yml (simplified)
services:
  agent-os:
    build: .
    restart: unless-stopped
    volumes:
      - ./agents:/app/agents          # agent directories (CLAUDE.md + agent.yaml)
      - ./workspace:/app/workspace    # task workspaces
      - ./data:/app/data              # SQLite databases
      - /var/run/docker.sock:/var/run/docker.sock  # for spawning agent containers
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - DISCORD_TOKEN=${DISCORD_TOKEN}
    ports:
      - "127.0.0.1:9090:9090"        # WebSocket for remote clients (localhost only)

  langfuse:
    image: langfuse/langfuse:latest
    restart: unless-stopped
    volumes:
      - ./langfuse-data:/data
    ports:
      - "127.0.0.1:3000:3000"        # Langfuse UI (localhost only, tunnel for access)

  mcp-gateway:
    build: ./mcp-gateway
    restart: unless-stopped
    # MCP servers managed by the gateway
```

### systemd Wrapper

```ini
[Service]
Type=simple
ExecStart=/usr/bin/docker compose -f /opt/agent-os/docker-compose.yml up
Restart=on-failure
RestartSec=30
WatchdogSec=300
```

### Monitoring

1. **Langfuse (self-hosted)**: Trace collection, prompt logging, cost tracking. Docker Compose deployment.
2. **Discord #monitoring channel**: Agents post daily health summaries. Critical alerts go to separate #alerts channel with @mentions.
3. **`agentctl health` CLI command**: Morning check takes <5 minutes. Shows: agents running, errors since last check, cost since last check, stuck agents.
4. **Cost dashboard**: `agentctl cost` shows daily/weekly/monthly breakdown by agent and task.

### Daily Operations Ritual (~10 minutes)

1. `agentctl health` — scan for errors, stuck agents, cost anomalies
2. Check Discord #alerts for overnight issues
3. Review any pending approval requests
4. Adjust task priorities or agent assignments if needed
5. Done

### Upgrade Strategy

- **Agent OS daemon**: Docker image rebuild + `docker compose up -d`. Agent containers restart gracefully.
- **Agent instructions (CLAUDE.md)**: Edit file, daemon hot-reloads via inotify. No restart needed.
- **MCP servers**: Hot-swap via gateway. New version deployed alongside old. Gateway switches after health check passes.
- **Claude SDK upgrades**: Pin version in package.json. Test in a single agent before rolling to all.

### Backup

- **SQLite databases**: Daily `VACUUM INTO` to a backup file. Rsync to off-site storage.
- **Agent directories**: Git-managed. Push to private repo.
- **Workspace artifacts**: Task-dependent. Git repos are self-backing. Other outputs backed up per schedule.

### Evidence

- Track 6: "A `docker-compose.yml` that brings up the harness process, a Langfuse instance, and a shared volume for agent workspaces. Total time to first working agent: target 20 minutes."
- Track 6: "Langfuse (self-hosted) for trace collection. Discord webhooks for alerting. A morning health-check CLI command. Terminal-based, no browser needed."
- Track 1: "systemd service with Restart=on-failure, RestartSec=30, WatchdogSec=120, MemoryMax=2G. StartLimitBurst=5. This stack uses zero external infrastructure beyond the VPS itself."

### Confidence: ★★★★☆ Strong

Docker Compose on a single VPS is the dominant deployment pattern across NanoClaw, zebbern/claude-code-discord, and community harnesses. Langfuse is the leading open-source observability tool for LLM applications (30K+ GitHub stars).

### Risks

- **Single VPS = single point of failure.** Mitigation: Hetzner has good uptime SLA. Daily backups allow recovery on a new VPS within hours. The system is designed to be reconstructable from git + backups.
- **Docker-in-Docker complexity (daemon managing agent containers).** Mitigation: mount Docker socket, not Docker-in-Docker. The daemon uses the Docker API to manage sibling containers.
- **Langfuse adds operational overhead.** Mitigation: it runs as a Docker service with minimal configuration. If it becomes burdensome, replace with simple file-based logging.

---

## Decision 11: What NOT to Build

### Explicitly Out of Scope for v1

| Feature | Why Not | When to Reconsider |
|---------|---------|-------------------|
| **Web dashboard** | Discord + CLI covers 90% of needs. Dashboard adds frontend maintenance. | v2, when multi-agent monitoring becomes painful in Discord |
| **Multi-model support (OpenAI, Google, local)** | Claude-only simplifies everything. One SDK, one API, one billing model. | v2, if cost pressure demands model routing |
| **Mobile app** | Discord mobile provides notifications. MobileCLI pattern is interesting but premature. | v2, if mobile-first control becomes a real need |
| **Semantic memory database** | Handoff docs + CLAUDE.md + SQLite checkpoints cover v1. External memory adds infra complexity. | v2, when agents need cross-project learning |
| **A2A protocol support** | Too early. The spec is immature and Sam's agents don't need to coordinate with external agent systems. | v2, when/if A2A reaches critical mass |
| **Custom LLM routing/load balancing** | Single provider (Anthropic). No routing decisions to make. | v2, if multi-model support is added |
| **Plugin/skill marketplace** | OpenClaw's ClawHavoc attack proved this is a security liability. Curate tools manually. | Never (for a solo operator) |
| **Automatic agent spawning without human initiation** | Risk of runaway agent creation. All agent creation should be human-initiated in v1. | v2, with proven cost controls and trust in the orchestrator |
| **Full Temporal/durable execution integration** | SQLite checkpoints + handoff documents cover 90% of the use case without the infrastructure overhead. | v2, if multi-day workflow durability becomes critical |
| **GPU/local model support** | Hetzner VPS is CPU-only. Local models not needed when using Anthropic API. | Only if the economics change dramatically |

### The Principle

Build the minimum viable kernel that runs agents reliably for days. Every feature not listed above is scope creep. The Agent OS should do fewer things well rather than many things poorly.

---

## Build Order

### Phase 1: Skeleton (Week 1-2)

**Goal**: A daemon that can spawn one agent, stream its output, and kill it.

1. TypeScript project scaffold with Claude Agent SDK dependency
2. SQLite database schema (agents, sessions, events, costs)
3. JSON-RPC server over Unix domain socket
4. Agent spawning via `ProcessTransport` inside a Docker container
5. Event streaming from agent to daemon
6. systemd service file
7. Basic `agentctl` CLI: `spawn`, `list`, `attach`, `kill`
8. Directory watching for agent discovery (CLAUDE.md)

**Exit criteria**: `agentctl spawn --agent test` starts a Claude Code agent in a Docker container that can execute tasks. `agentctl attach` streams its output. `agentctl kill` stops it cleanly.

### Phase 2: Safety (Week 3)

**Goal**: The system cannot bankrupt you or destroy itself.

1. Cost tracking (per-call, per-agent, per-task, global)
2. Circuit breakers at all four levels
3. PreToolUse hook for command allowlists
4. Container resource limits (memory, CPU)
5. Network allowlists (outbound-only, whitelisted domains)
6. Immutable audit logging
7. Secrets management (.env injection)

**Exit criteria**: A runaway agent is automatically stopped when it exceeds its hourly budget. Destructive commands are blocked. All tool calls are logged.

### Phase 3: Resilience (Week 4)

**Goal**: The system runs for days without intervention.

1. Heartbeat monitoring
2. Health check metrics (RSS, token rate, error rate, context depth)
3. Scheduled session rotation with handoff documents
4. Crash recovery from SQLite checkpoints
5. systemd watchdog integration
6. Alert dispatch (Discord webhook to #alerts channel)
7. Morning health check in CLI (`agentctl health`, `agentctl cost`)

**Exit criteria**: An agent runs for 48+ hours with two session rotations and no human intervention. A forced crash recovers automatically. The operator receives alerts for errors and cost thresholds.

### Phase 4: Discord (Week 5-6)

**Goal**: Full Discord client for remote agent management.

1. Discord bot connecting to the daemon via WebSocket
2. Slash commands: `/spawn`, `/list`, `/kill`, `/approve`, `/health`, `/cost`
3. Thread-per-task mapping
4. Interactive approval buttons (Allow/Deny)
5. Notification routing (info → channel, urgent → DM)
6. Rate limit awareness and message chunking
7. RBAC via Discord role mapping

**Exit criteria**: Sam can manage all agents from his phone via Discord. Approval requests appear as button prompts. Cost alerts arrive as DMs.

### Phase 5: MCP & Multi-Agent (Week 7-8)

**Goal**: MCP gateway and basic multi-agent coordination.

1. MCP gateway with tool catalog aggregation and namespace prefixing
2. Lazy loading / Tool Search endpoint
3. Read/write pool separation
4. Per-agent tool scoping from agent.yaml
5. Orchestrator-worker pattern implementation
6. Task DAG execution engine
7. Model tiering (Opus orchestrator, Sonnet workers, Haiku validators)
8. Filesystem IPC for agent output exchange

**Exit criteria**: An orchestrator agent can decompose a research task into subtasks, spawn worker agents with scoped tool access, execute them in parallel, and synthesize results. Total cost stays within per-task budget.

### Phase 6: Polish (Week 9-10)

**Goal**: Production-ready for daily use.

1. Langfuse integration for trace collection
2. Hot-reload of CLAUDE.md via inotify
3. Scheduled/recurring tasks (cron integration)
4. Autonomy level configuration and progressive trust
5. Documentation (setup guide, operations runbook)
6. Stress testing: 8 agents running for 72 hours
7. Bug fixes from stress testing

**Exit criteria**: Sam uses the Agent OS daily for research, content, and project work. 72-hour stress test passes. Setup guide enables rebuilding from scratch on a new VPS in under 2 hours.

---

## Risk Register

### Risk 1: Claude Agent SDK Instability
**Likelihood**: Medium | **Impact**: High

The SDK was renamed from `@anthropic-ai/claude-code` to `@anthropic-ai/claude-agent-sdk` in February 2026. Breaking changes are expected as Anthropic iterates.

**Mitigation**: Pin to a specific SDK version. Test upgrades in a single agent before rolling to all. Abstract the SDK interface behind a thin wrapper so swapping versions requires changes in one file, not everywhere.

### Risk 2: Cost Runaway Despite Circuit Breakers
**Likelihood**: Low | **Impact**: Critical

The documented $16,000 incident happened because no cost controls existed. Circuit breakers can fail if the cost tracking has a bug or a race condition.

**Mitigation**: Defense in depth. Set Anthropic API-level spend limits (if available) as the outermost safety net. The daemon's circuit breakers are the second layer. Per-agent container memory limits are the third (OOM kill prevents infinite loops). Monitor daily spending manually for the first month.

### Risk 3: Scope Creep Into "Agent Framework"
**Likelihood**: High | **Impact**: Medium

The temptation to add features (web dashboard, multi-model, marketplace, semantic memory) will be strong. Each addition increases maintenance burden on a solo operator.

**Mitigation**: The "What NOT to Build" section is a contract. Before adding any feature, ask: "Does this help agents run reliably for days?" If not, it's v2.

### Risk 4: Warp Oz / Managed Platforms Make Self-Hosting Obsolete
**Likelihood**: Medium | **Impact**: Medium

If Oz pricing becomes reasonable and features match, self-hosting may become unnecessary overhead.

**Mitigation**: The protocol-first design means the daemon's value is portable. If a managed platform emerges that's good enough, the Discord bot and CLI can connect to it instead of the self-hosted daemon. The investment in protocol design is not wasted.

### Risk 5: Context Degradation Not Solved by Rotation
**Likelihood**: Medium | **Impact**: Medium

Session rotation depends on handoff document quality. If the handoff loses critical context, agents will repeat work or make contradictory decisions after rotation.

**Mitigation**: Log every rotation event with before/after context summaries. Track "post-rotation confusion" incidents (agent asks about something covered in the handoff). Iterate on the handoff template based on data. This is the single most important prompt engineering task in the system.

---

## Open Questions

### 1. Optimal Session Rotation Interval
No rigorous empirical study compares agent performance across different rotation intervals. The 4-8 hour recommendation is practitioner consensus. **Resolution**: instrument rotation events with quality metrics. Run A/B tests (4h vs. 6h vs. 8h rotation) on the same task types and measure output quality.

### 2. Handoff Document Fidelity
How much context survives a structured handoff? At what point does information loss from rotation exceed loss from compaction? **Resolution**: measure empirically by having agents complete multi-session tasks and rating output quality versus single-session baselines.

### 3. Docker Socket Security
Mounting the Docker socket into the daemon container gives the daemon full Docker API access, which is a significant privilege escalation vector. **Resolution**: evaluate Docker socket proxies (Tecnativa/docker-socket-proxy) that restrict which Docker API endpoints the daemon can call. The daemon only needs: create container, start, stop, remove, inspect.

### 4. Hetzner VPS KVM Support
MicroVM isolation (Firecracker/Kata) requires KVM. Hetzner shared VPS plans may not support nested virtualization. **Resolution**: test KVM availability on the target Hetzner plan. If unavailable, Docker containers remain the isolation layer. If available, Kata Containers can be added later for high-security agents.

### 5. Agent SDK Concurrent Session Limits
How many simultaneous Claude Code sessions can run on a single Anthropic account? Are there undocumented rate limits on concurrent `ProcessTransport` instances? **Resolution**: test empirically during Phase 1. Start with 2 agents, scale to 8, measure for throttling or errors.

### 6. Multi-Day SQLite WAL Stability
SQLite WAL mode under continuous writes from 5-15 agents for days at a time may encounter checkpoint starvation or unbounded WAL growth. **Resolution**: implement aggressive checkpointing (`PRAGMA wal_checkpoint(TRUNCATE)` every 10 minutes) and monitor WAL file size as a health metric.

### 7. Claude Code Hooks on Remote/Containerized Sessions
The PreToolUse hook system is documented for local Claude Code CLI. Does it work identically when Claude Code runs inside a Docker container via the Agent SDK? **Resolution**: test in Phase 2. If hooks don't work in containers, implement command filtering in the daemon's event stream instead.

---

## Source References

This ADR synthesizes findings from seven research tracks, each with its own detailed source log:

| Track | Document | Key Contribution to ADR |
|-------|----------|------------------------|
| 1 | agent-os-process-architecture.md | Supervision tree, WAL persistence, session rotation, cost circuit breakers, MAPE-K health monitoring |
| 2 | agent-os-multi-agent-orchestration.md | Orchestrator-worker + blackboard hybrid, filesystem IPC, task DAG, model tiering, token economics |
| 3 | agent-os-interface-control-plane.md | Protocol-first design, JSON-RPC daemon, Discord client pattern, autonomy levels, approval workflows |
| 4 | agent-os-security-hardening.md | Container isolation, command allowlists, defense-in-depth, prompt injection as unsolvable, audit logging |
| 5 | agent-os-mcp-architecture.md | Gateway pattern, lazy loading, read/write pool separation, per-agent tool scoping, tool quality |
| 6 | agent-os-developer-experience.md | Convention-over-configuration, CLAUDE.md as agent identity, Langfuse observability, 10-minute daily ritual |
| 7 | agent-os-ecosystem-landscape.md | *Claw ecosystem context, NanoClaw/ZeroClaw/NullClaw architectures, enterprise platform comparison, security lessons from OpenClaw |

Total sources across all tracks: 200+ (see individual track documents for complete source logs with tier ratings and verification notes).

---

_This document is the blueprint. Every decision is backed by evidence, labeled with confidence, and honest about risks. Build in the order specified. Resist scope creep. The goal is not a perfect system — it is a system that runs reliably for days and can be improved iteratively._
