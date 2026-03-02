# Agent OS Track 1: Process Architecture & Self-Healing
_Research conducted: 2026-03-02 | Overall shelf-life: 🟡 Monitor (12-18 months — patterns are stabilizing but tooling evolves weekly)_

## Executive Summary

Production-grade AI agent harnesses that run for days without human intervention require three interlocking systems: process supervision that keeps the agent alive, health monitoring that detects degradation before failure, and state persistence that enables seamless recovery when failure occurs anyway. The field has converged on a hybrid architecture — OS-level process supervision (systemd, containers) handles catastrophic crashes, while application-level self-healing (checkpointing, circuit breakers, context rotation) handles logical failures and degradation. The Erlang/OTP "let it crash" philosophy, combined with IBM's MAPE-K autonomic computing loop, provides the most robust theoretical foundation for designing these systems. Durable execution frameworks (Temporal, LangGraph, DBOS) have matured to production-grade status in 2025-2026, with Temporal's $5B valuation and 9.1 trillion lifetime action executions validating the approach at scale. For a single-VPS Agent OS, the optimal architecture is a layered supervision tree: systemd at the base, a lightweight coordinator process in the middle, and per-agent context management at the top — with WAL-based state persistence, proactive context rotation, and multi-level cost circuit breakers preventing runaway spend.

---

## Domain Map

Process architecture for long-running AI agents sits at the intersection of three established fields: **distributed systems** (supervision trees, fault tolerance, checkpoint/restore), **autonomic computing** (self-healing, self-optimization, the MAPE-K loop from IBM's 2003-2006 research), and the emerging discipline of **AI agent infrastructure** (context engineering, durable execution, token economics).

Key players and their contributions:

- **Temporal** ($5B valuation, Feb 2026): The enterprise standard for durable execution. Journal-based replay ensures agents resume exactly where they left off after crashes. Used by OpenAI, Netflix, JPMorgan Chase.
- **LangGraph** (LangChain): Database checkpointing at every super-step. The dominant framework for graph-shaped agent workflows. Production users include Uber, LinkedIn, Klarna.
- **OpenClaw** (Peter Steinberger): The most popular self-hosted agent harness (219K+ GitHub stars). Single Node.js process with heartbeat scheduling and session management. Known memory leak issues over multi-day runs.
- **NanoClaw** (QwibitAI): Container-per-agent isolation with ~700 lines of host orchestrator code. Reports of months of uptime. Radically minimal approach.
- **Anthropic** (context engineering team): Published the definitive guide on context compaction and sub-agent architectures. Claude Agent SDK includes file checkpointing.
- **OpenAI** (Codex team): Server-side auto-compaction, skills system, and shell primitives for long-running agents.
- **VIGIL** (Christopher Cruz, arXiv:2512.07094): Academic paper proposing a "reflective runtime" — a supervisor agent that monitors a sibling agent's behavioral logs and performs autonomous maintenance.
- **IBM Research** (Kephart, Chess, White et al.): The MAPE-K loop (Monitor-Analyze-Plan-Execute over shared Knowledge) remains the canonical framework for self-managing systems, 20+ years after publication.
- **Erlang/OTP** (Joe Armstrong et al.): Supervision trees and the "let it crash" philosophy provide the most battle-tested patterns for building fault-tolerant process hierarchies.

Major debates center on: (1) how much supervision intelligence should live in the supervisor vs. the agent itself, (2) journal-based replay vs. database checkpointing for crash recovery, (3) whether context compaction is sufficient or whether proactive session rotation is necessary, and (4) the right granularity of cost controls (per-request vs. per-workflow vs. per-agent vs. global).

---

## 1. Foundations — The 80/20

### Principle 1: The Supervision Tree Is the Right Abstraction
**★★★★★ Established**

Erlang/OTP's supervision tree pattern — where supervisor processes monitor worker processes and restart them according to configurable strategies — has proven itself across 40+ years of telecom-grade systems. The pattern maps directly to agent harness requirements: a root supervisor manages agent-group supervisors, which manage individual agent processes. Each level has independent restart strategies (one-for-one, one-for-all, rest-for-one) and configurable thresholds for maximum restart intensity.

The core insight, articulated in Joe Armstrong's thesis and codified in OTP behaviors: **separating error-handling code from normal-case code produces more reliable systems than trying to handle every error at the point it occurs.** Rather than writing defensive code at every call site, you build a hierarchy of supervisors whose sole job is to decide what to do when things fail. This is the "let it crash" philosophy — not recklessness, but a principled separation of concerns.

For an Agent OS on a single VPS, the tree maps to:
- **Root supervisor** (systemd): Manages the harness process. Restarts on crash with configurable delay and max-restart limits.
- **Harness supervisor** (application-level): Manages individual agent sessions. Decides whether to restart, rotate context, or escalate to human.
- **Agent worker** (the Claude Code CLI process): Does the actual work. Crashes are expected and handled by the layer above.

[Sources: Erlang OTP design principles documentation; Armstrong, "Making Reliable Distributed Systems in the Presence of Software Errors" (2003); Stack Overflow supervision tree explanations; adoptingerlang.org/docs/development/supervision_trees/]

### Principle 2: Hybrid Supervision — OS-Level + Application-Level — Is Non-Negotiable
**★★★★★ Established**

The industry has converged on a hybrid approach where process-level supervision (systemd, PM2, Kubernetes) keeps the process alive, while application-level supervision (checkpointing, circuit breakers, context management) handles logical failures. Neither alone is sufficient.

**★★★★★** Process supervisors know nothing about agent state. They can restart a crashed process but cannot resume a multi-turn conversation from where it left off. Application-level self-healing understands business logic but provides no protection if the process is killed by the OS.

The practical architecture, confirmed across multiple independent sources (Zylos Research, Oguzhan Atalay's VPS fleet writeup, ClawTank deployment guide, Bix-Tech Docker/K8s guide):

```
Process Supervisor (systemd)          <- Keeps process alive
  +-- Application Self-Healing        <- Checkpoints, circuit breakers, retries
      +-- Health Monitoring            <- Heartbeat, observability, alerting
```

Process-level handles: OOM kills, segfaults, kernel panics, unresponsive event loops.
Application-level handles: API timeouts, context exhaustion, token budget overruns, model degradation, tool failures.

[Sources: Zylos "AI Agent Self-Healing and Auto-Recovery Patterns" (2026-02-17); Atalay "Architecting a Multi-Agent AI Fleet on a Single VPS" (2026-02-25); ClawTank "Running OpenClaw as a systemd Service" (2026-02-18)]

### Principle 3: The MAPE-K Loop Provides the Right Mental Model for Agent Self-Healing
**★★★★☆ Strong**

IBM's autonomic computing research (White, Hanson, Whalley, Chess, Kephart — 2003-2006) introduced the MAPE-K control loop: **Monitor** system state, **Analyze** for anomalies, **Plan** corrective action, **Execute** the plan, all operating over shared **Knowledge**. This framework, originally designed for managing computing infrastructure, maps remarkably well to AI agent harness design.

Applied to an Agent OS:
- **Monitor**: Collect metrics (memory RSS, token consumption rate, response latency, error rate, context depth percentage).
- **Analyze**: Compare against baselines and thresholds. Detect degradation patterns (rising latency, increasing error rate, memory growth trajectory).
- **Plan**: Decide corrective action (rotate context, restart agent, fall back to cheaper model, alert human).
- **Execute**: Apply the correction with appropriate safety checks.
- **Knowledge**: Historical baselines, learned failure patterns, cost budgets, agent configuration.

The MAPE-K model has been adopted in 2025 by frameworks including Esposito et al.'s "Autonomic Microservice Management via Agentic AI and MAPE-K Integration" (ECSA 2025, 3 citations) and multiple IoT/edge computing systems. Its value for Agent OS design is that it provides a complete decision framework, not just a restart loop.

[Sources: White et al. "An architectural approach to autonomic computing" ICAC 2004 (DOI: 10.1109/icac.2004.1301340, verified via Crossref); White et al. "Autonomic computing: Architectural approach and prototype" Integrated Computer-Aided Engineering, 2006 (DOI: 10.3233/ica-2006-13206); Esposito et al. "Autonomic Microservice Management via Agentic AI and MAPE-K Integration" ECSA 2025; Balditsyn et al. "Uncertainty-Driven Monitoring for ML-Based Autonomic Systems" ACSOS 2025]

### Principle 4: State Persistence Must Be Write-Ahead, Not Post-Hoc
**★★★★☆ Strong**

The Write-Ahead Log (WAL) pattern — writing changes to a durable log before applying them to the primary data store — is the foundational technique for crash recovery in databases (PostgreSQL, SQLite, MySQL). For agent harnesses, this means persisting agent actions and state transitions before executing them, not after.

Applied to Agent OS, this means:
- **Before** an agent executes a tool call, log the intended action and current state to a WAL.
- **Before** committing an agent's output, log the completion state.
- On crash recovery, replay the WAL to reconstruct the last consistent state.

OpenClaw's `bulletproof-memory` skill implements a version of this: a Write-Ahead Log protocol that writes user-provided details before the agent replies, using a SESSION-STATE.md file that survives compaction, restarts, and context resets. The `agentstate` project on GitHub (Apache-2.0, Python/Rust) implements WAL+snapshots with watch streams and idempotency keys specifically for AI agent state management.

SQLite's WAL mode is the go-to for embedded agent databases, but introduces its own failure class: checkpoint starvation and unbounded WAL growth when long-running readers prevent checkpointing. For an always-on agent, this requires explicit `PRAGMA wal_checkpoint(TRUNCATE)` on a timer.

[Sources: OpenClaw/skills bulletproof-memory SKILL.md (playbooks.com); ayushmi/agentstate GitHub repo; Zylos "SQLite WAL Mode: Patterns and Pitfalls for AI Agent Systems" (2026-02-20); Gajula "Persistence & Crash Recovery Using WAL" (Medium, 2026-01)]

### Principle 5: Proactive Context Rotation Beats Reactive Compaction
**★★★★☆ Strong**

Context degradation — the gradual loss of coherence as a context window fills with accumulated conversation history — is the primary failure mode for long-running agents. It manifests not as a crash but as a silent quality decline: the agent drifts, repeats itself, loses architectural decisions made hours earlier, and proposes changes that contradict its own earlier analysis.

The field has established a clear hierarchy of mitigation strategies:

1. **Tool result clearing** (lightest touch): Remove verbose tool outputs, keeping only summaries. Anthropic's context engineering guide identifies this as the safest first step.
2. **Compaction** (medium): Summarize older conversation turns while preserving recent ones in full detail. Manus keeps the most recent tool calls in raw format to preserve "rhythm." Trigger at ~50% of available window, not at the limit.
3. **Proactive session rotation** (strongest): Before degradation begins, start a new agent session with a carefully constructed handoff context — a summary of state, pending tasks, and key decisions. This is a "warm handoff" rather than a "cold restart."

The critical threshold: Philipp Schmid's analysis of Manus reports that a 1M context window degrades around 256K tokens — **well before the limit**. OpenAI's Codex team has implemented "robust auto compaction" that triggers proactively, and their engineers note the goal is to "eventually remove" manual compaction entirely, suggesting confidence in automated approaches.

For multi-day agent runs, the evidence points toward scheduled rotation — creating a new session every N hours or N tokens with explicit state handoff — rather than relying solely on compaction within a single session.

[Sources: Anthropic "Effective context engineering for AI agents" (2025-09); Schmid "Context Engineering for AI Agents: Part 2" (2025-12); OpenAI Codex team (GitHub issue #11325 comment by etraut-openai); Vincent van Deth "Context Rot Is Silently Killing Your Claude Code Sessions"; Google ADK "Context compression" documentation; Jason Liu "Two Experiments We Need to Run on AI Agent Compaction" (2025-08)]

---

## 2. Current Evidence Landscape

### 2.1 Process Supervision Patterns for Agent Harnesses

#### systemd: The De Facto Standard for Single-VPS Deployment
**★★★★★ Established**

systemd provides OS-level process supervision with zero additional infrastructure. Every major agent framework (OpenClaw, NanoClaw, custom harnesses) uses systemd units in production on Linux VPS deployments. The key configuration:

```ini
[Unit]
Description=Agent OS Harness
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/agent-harness --config /etc/agent-os/config.yaml
Restart=on-failure
RestartSec=30
WatchdogSec=120
StartLimitIntervalSec=300
StartLimitBurst=5
MemoryMax=2G
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

Key directives and their rationale:
- `Restart=on-failure` with `RestartSec=30`: Restarts on non-zero exit with 30s backoff. Avoids tight crash loops.
- `StartLimitIntervalSec=300` + `StartLimitBurst=5`: Maximum 5 restarts in 5 minutes before systemd stops trying — prevents infinite restart storms.
- `WatchdogSec=120`: The application must send `sd_notify("WATCHDOG=1")` every 120 seconds or systemd kills and restarts it. This catches event loop hangs that don't produce a crash.
- `MemoryMax=2G`: Hard memory ceiling. When an agent leaks memory (OpenClaw's known issue: 1GB+ growth over days), systemd kills it before the VPS runs out of RAM.

**OpenClaw's systemd watchdog gap** (now closed): Issue #1731 on OpenClaw's GitHub requested native `sd_notify` watchdog support because external HTTP health checks only catch unresponsiveness at the HTTP layer, not internal event loop hangs. The issue was closed as completed in January 2026.

**Multi-agent fleet pattern** (from Atalay's VPS writeup): Each agent gets its own systemd user service on a dedicated port, spaced 20 apart. This provides process isolation without containers — one agent crashing doesn't take down the fleet. A cron-based watchdog script runs every 15 minutes checking liveness, feeding failures to a fast LLM for diagnostics, and falling back to config restoration if the LLM fix fails.

[Sources: ClawTank systemd guide; Atalay dev.to multi-agent VPS article; OpenClaw GitHub issue #1731; caishengold dev.to systemd + watchdog patterns article]

#### Container Isolation: NanoClaw's Approach
**★★★★☆ Strong**

NanoClaw demonstrates that container-per-agent isolation provides stronger guarantees than process-per-agent for multi-agent systems. The architecture: a ~700-line host orchestrator manages WhatsApp connectivity, SQLite persistence, and container lifecycle. When a message arrives, the Container Runner spawns an isolated Linux container (Apple Container on macOS, Docker on Linux) with only the relevant group's directory mounted. Inside, the Claude Agent SDK runs with filesystem-based IPC back to the host.

Benefits confirmed by community reports:
- **Months of uptime** without intervention (multiple Reddit reports from NanoClaw users).
- **Blast radius containment**: A misbehaving agent can only damage its own mounted directory.
- **Resource limits per agent**: Docker's `--memory`, `--cpus` flags provide per-agent resource ceilings.
- **Clean restart semantics**: Destroying and recreating a container is a reliable full-state reset.

The trade-off: container overhead (~50-100MB RAM per container, ~2s cold start) vs. process-only overhead (~20-50MB, instant start). For a Hetzner VPS with 32GB RAM running 4-8 agents, the container overhead is acceptable.

[Sources: NanoClaw GitHub repo (qwibitai/nanoclaw); VentureBeat "NanoClaw solves one of OpenClaw's biggest security issues"; The New Stack "NanoClaw's answer to OpenClaw is minimal code, maximum isolation"; nanoclaw.net documentation]

### 2.2 Health Monitoring for Agent Harnesses

#### Metrics That Matter
**★★★★☆ Strong**

Agent harness monitoring requires metrics beyond traditional application monitoring. The evidence converges on these categories:

| Metric | Why It Matters | Threshold Pattern |
|--------|---------------|-------------------|
| **Memory RSS** | Node.js/Python agents leak memory over days. OpenClaw grows 1GB+ over a week. | Alert at 70% of MemoryMax; kill+restart at 90%. |
| **Token consumption rate** | Detects runaway loops and context bloat. | Per-hour rolling average; alert at 2x baseline. |
| **Response latency (P95)** | Rising latency signals degradation before failure. | Alert at 2x normal P95; circuit-break at 5x. |
| **Error rate (5xx / tool failures)** | Distinguishes transient from persistent failures. | Circuit-break at 5 failures in 60 seconds. |
| **Context depth %** | How full is the context window. | Trigger compaction at 50%; rotate session at 75%. |
| **Cost accumulation** | Running $/hour vs. budget. | Alert at 70%; pause at 100%. |
| **Heartbeat age** | Time since last successful agent response. | Alert at 2x expected interval; restart at 5x. |

**Grafana AI Observability** is the most complete monitoring solution for AI agent stacks as of early 2026. It provides performance tracking (LLM response times, throughput, availability across providers), safety monitoring (toxicity detection, bias assessment), and tool analytics (MCP usage patterns). Combined with Prometheus for metric collection and OpenTelemetry for trace instrumentation, this stack provides full observability without vendor lock-in.

Agent-specific metrics beyond standard MTBF/MTTR: time to detection (how quickly failures are identified), graceful degradation rate (partial failures handled without full restart), recovery success rate (percentage of failures recovered without human intervention), and checkpoint frequency vs. size trade-off.

[Sources: Zylos "AI Agent Self-Healing" (2026-02); Grafana Cloud AI Observability docs; webrtc.ventures Prometheus + Grafana for AI agents; kagent.dev observability agent]

#### Heartbeat vs. Watchdog: Use Both
**★★★★☆ Strong**

Two complementary liveness detection patterns:

**Heartbeat** (bidirectional): The agent sends periodic signals indicating it's alive and healthy, often including metadata (current task, context depth, memory usage). Enables nuanced, preventive action — you know not just whether the system is running, but how well it's running.

**Watchdog** (unidirectional deadline): A timer that must be reset by the application before it expires. If the agent hangs, the watchdog fires and triggers a restart. Simpler, providing autonomous recovery without external input.

Production systems use both: heartbeat for health visibility and preventive action, watchdog as the last-resort safety net. systemd's `WatchdogSec` provides the watchdog. Application-level heartbeats (e.g., OpenClaw's heartbeat scheduler sending periodic pings) provide the health visibility.

The critical gap: **detecting degradation before failure.** A watchdog catches hangs. A heartbeat catches crashes. Neither catches an agent that is technically responsive but producing poor-quality output due to context rot. This is where the VIGIL pattern (Section 2.4) and proactive context rotation (Section 2.5) become essential.

[Sources: Zylos "AI Agent Self-Healing" (2026-02); Saulius.io "OpenClaw Agentic Framework: How Autonomous AI Agents Execute Long-Running Tasks with Heartbeat Monitoring" (2026-02-06)]

### 2.3 Crash Recovery & State Persistence

#### Durable Execution: The Three Competing Paradigms
**★★★★★ Established**

The durable execution landscape has consolidated around three approaches, each with distinct trade-offs:

**1. Journal-Based Replay (Temporal, Restate)**
The runtime maintains a journal of every completed step. On crash recovery, the workflow function re-executes from the beginning, but each previously-completed step returns its cached result instead of re-executing. Temporal's Event History records every decision and activity result.

- **Pro**: Strongest correctness guarantees. Exact state reconstruction.
- **Con**: Requires deterministic workflow code. LLM calls (inherently non-deterministic) must be wrapped as "activities" whose results are journaled.
- **Infrastructure**: Requires a Temporal cluster or Restate server.
- **Scale validation**: 9.1 trillion lifetime executions on Temporal Cloud; 1.86 trillion from AI-native companies.

**2. Database Checkpointing (LangGraph, DBOS)**
State is persisted to a database after each workflow node completes. On crash, the last checkpoint is loaded and execution continues from there.

- **Pro**: Simple. Works with existing Postgres/SQLite. DBOS adds durability with zero new infrastructure.
- **Con**: Checkpoint granularity is per-node, not per-instruction. Some work between checkpoints may be lost.
- **Infrastructure**: Just a database.
- **Scale validation**: LangGraph production users include Uber, LinkedIn, Klarna.

**3. Event Sourcing (Kafka, Akka)**
An immutable, append-only log of events. Current state is derived by replaying all events. Agent actions map naturally to events: `tool_called`, `llm_responded`, `document_retrieved`.

- **Pro**: Complete audit trail. Time travel to any prior state. Natural fit for multi-agent coordination.
- **Con**: More complex to implement. Replay can be slow for long histories.
- **Infrastructure**: Event store or Kafka.

For a single-VPS Agent OS, **database checkpointing (SQLite)** is the pragmatic choice. It requires zero additional infrastructure, and SQLite's WAL mode provides concurrent reads during writes with sub-millisecond latency. Temporal is the right choice if the agent needs cross-service orchestration or week-long workflow durability, but it adds significant infrastructure complexity.

[Sources: Zylos "Durable Execution Patterns for AI Agents" (2026-02-17); Temporal blog "Of course you can build dynamic AI agents with Temporal"; Temporal blog "Managing very long-running Workflows"; LangGraph docs; Pydantic AI Temporal integration docs]

#### Git as Crash Recovery Mechanism
**★★★★☆ Strong**

Anthropic's internal harness pattern (initializer agent + coding agent, feature list JSON, progress.txt, git commits for state) uses git commits as checkpoints. The Claude Agent SDK includes file checkpointing that snapshots file state at execution checkpoints, enabling rewind to any previous state.

The pattern is straightforward: instruct the agent to commit frequently with descriptive messages. Each commit becomes a rollback point. If the agent makes a mistake at commit N, you can `git reset --hard HEAD~1` to the last known good state.

This is complementary to, not a replacement for, conversation-level state persistence. Git captures file-system state; it does not capture conversation history, pending decisions, or intermediate reasoning. A full crash recovery system needs both.

[Sources: Eleanor Berger "Use Git for Automated Checkpointing" (2025-10); Anthropic Claude Agent SDK docs "Rewind file changes with checkpointing"; Zylos "Git Worktree Isolation Patterns for Parallel AI Agent Development" (2026-02-22)]

### 2.4 The VIGIL Pattern: Supervisor-as-Agent
**★★★☆☆ Moderate**

VIGIL (Verifiable Inspection and Guarded Iterative Learning, arXiv:2512.07094, December 2025) proposes a novel architecture: a **reflective runtime that supervises a sibling agent**, performing autonomous maintenance rather than task execution. This is the first published system that treats the supervisor itself as an LLM agent rather than a deterministic process.

VIGIL's architecture:
- Ingests behavioral logs from the supervised agent.
- Appraises each event into a structured emotional representation (EmoBank).
- Maintains per-agent memory traces with temporal decay.
- Performs stage-gated diagnosis using a "Roses-Buds-Thorns" framework.
- Suggests prompt and code fixes so the supervised agent can recover.

The confidence is moderate because: (1) VIGIL is a single academic paper with no reported production deployments, (2) using an LLM to supervise another LLM introduces a second failure point, (3) the paper was published on arXiv without peer review confirmation via Crossref (no DOI found).

However, the concept is directionally correct: for an Agent OS running for days, a lightweight "meta-agent" that periodically reviews the primary agent's behavioral logs and flags degradation patterns would catch failure modes that heartbeats and watchdogs miss. The key is keeping the supervisor deterministic where possible and using LLM judgment only for ambiguous cases.

Atalay's production VPS fleet implements a simpler version: a bash watchdog script running via cron every 15 minutes that feeds failure logs to a fast LLM (Groq) for diagnostics. A separate "oversight agent" running on a cheap model checks every 5 minutes whether agents are following their operational checklists.

[Sources: Cruz "VIGIL: A Reflective Runtime for Self-Healing LLM Agents" arXiv:2512.07094 (2025-12); Atalay multi-agent VPS fleet article (2026-02); Ghosh "Self-Correcting Multi-Agent AI Systems" Medium (2026-02)]

### 2.5 Context Window Management for Multi-Day Runs

#### The Compaction Hierarchy
**★★★★★ Established**

The evidence converges on a clear ordering of context management strategies, from lightest-touch to most aggressive:

| Strategy | Information Loss | When to Use | Source |
|----------|-----------------|-------------|--------|
| **Artifact externalization** | None | Always. Large data uses "handle pattern" — lightweight references in context, full data on disk. | Google ADK docs |
| **Tool result clearing** | Minimal | When tool outputs are verbose but results are summarized in agent reasoning. | Anthropic context engineering guide |
| **Reversible compaction** | None (reversible) | When agent has written files to disk. Remove file contents from context; agent can re-read later. | Schmid/Manus analysis |
| **Lossy summarization** | Moderate | When reversible compaction can't free enough space. Keep recent turns in full; summarize older ones. | Manus, Claude Code auto-compaction |
| **Session rotation** | High (mitigated by handoff document) | When context depth exceeds 75%, or on a fixed schedule for multi-day runs. | Multiple practitioner reports |

**Pre-rot threshold**: Schmid's analysis states that a 1M context window degrades around 256K tokens. **★★★★☆ Strong** — this is based on practitioner observation from the Manus team, not controlled experimentation, but is consistent with Anthropic's guidance that compaction should trigger well before the limit.

**The 50% rule**: Trigger compaction when context reaches 50% of available window. **★★★☆☆ Moderate** — cited by multiple practitioners but no rigorous empirical validation. It provides adequate safety margin.

**Codex auto-compaction**: OpenAI's Codex team has implemented server-side auto-compaction and considers it robust enough to plan removing manual `/compact` commands. Their approach uses the `/responses/compact` endpoint to summarize conversation history server-side. **★★★★☆ Strong** — first-party implementation by a major provider.

**Known failure mode**: Auto-compaction causes agents to forget previously answered questions, repeat work, and lose architectural decisions. Multiple GitHub issues document this (e.g., openai/codex#11174). The van Deth "Context Rot" analysis provides the clearest diagnosis: compaction is lossy compression on working memory, and the losses accumulate over hours.

For multi-day Agent OS runs, the recommendation is: **scheduled session rotation every 4-8 hours** (configurable based on task intensity), with a structured handoff document that captures: (1) current task state, (2) key decisions made, (3) pending work items, (4) files modified, (5) any learned constraints. This is more reliable than relying on compaction alone.

[Sources: Anthropic context engineering guide; Schmid context engineering part 2; Google ADK context compression docs; OpenAI Codex auto-compaction (GitHub issue #11325); van Deth "Context Rot" blog; lethain.com "Building an internal agent: Context window compaction"]

### 2.6 Resource Budgeting and Cost Circuit Breakers

#### Multi-Level Cost Controls
**★★★★☆ Strong**

The evidence strongly supports implementing cost controls at multiple independent levels. A single layer is insufficient — different failure modes manifest at different granularities.

Mehmet Erturk's production architecture implements three independent mechanisms:

**1. Circuit Breakers** (for provider failures):
- Three states: Closed (normal) -> Open (fail fast after 5 failures in 60s) -> Half-Open (test one request after 30s timeout).
- Critical insight: **fast failure is better than slow failure.** A user seeing "service temporarily unavailable" in 10ms is better than waiting 30 seconds for a timeout.
- Pairs with provider fallback: when primary provider's circuit opens, requests route to secondary. Rule: never re-execute tools during fallback — only the model call falls back.

**2. Rate Limiting** (for volume control):
- Gateway layer: 100 requests/minute per user, token bucket algorithm.
- Node layer: Per-LLM-call limits (max 10K input tokens, max 4K output, max $0.50 per call). Per-execution limits (max 50K total tokens, max $2.00, max 25 iterations).
- Both layers needed: gateway catches abuse, node catches expensive individual operations.

**3. Cost Ceilings** (for cumulative spend):
- Per-workflow budgets based on observed P95 costs plus margin. Quick lookup: $0.10. Customer support: $0.50. Complex analysis: $2.00. Full research: $5.00.
- Alert at 70%, dashboard alert at 90%, pause at 100%.
- Anomaly detection: cost > 2x average for workflow type triggers investigation.
- **"Be strict by default"**: Start with $1 (not $100), 10 iterations (not 1000), 5 minutes (not unlimited). Easier to loosen than to explain a $10,000 bill.

The a16z 2025 AI Infrastructure report found that **34% of organizations deploying AI agents experienced at least one budget overrun in the first year**. ★★★☆☆ Moderate — cited by AgentC2 blog but original a16z report not independently verified in this session.

For Agent OS specifically, the budget hierarchy should be:
1. **Global daily cap**: Hard ceiling for total API spend across all agents. Prevents catastrophic bills.
2. **Per-agent hourly budget**: Detects individual runaway agents.
3. **Per-task budget**: Prevents any single task from consuming disproportionate resources.
4. **Per-call token limit**: Catches context window bloat at the individual API call level.

[Sources: Erturk "Circuit Breakers, Rate Limits, and Cost Ceilings" (2026-02-08); Zylos "AI Agent Cost Optimization: Token Economics and FinOps" (2026-02-19); Athenic "AI Agent Rate Limiting: Implementing Token Budgets" (2025-12); agentgateway.dev spending control docs]

---

## 3. Practical Tactics

### Tactic 1: The systemd + WAL + Rotation Stack
**Difficulty: Medium | Impact: High**

The minimum viable self-healing agent harness on a single VPS:

1. **systemd service** with `Restart=on-failure`, `RestartSec=30`, `WatchdogSec=120`, `MemoryMax=2G`, `StartLimitBurst=5`.
2. **SQLite WAL** for state persistence. Each agent action logged before execution. `PRAGMA wal_checkpoint(TRUNCATE)` every 10 minutes via timer.
3. **Scheduled session rotation** every 4-8 hours. A cron job or internal timer triggers: (a) generate handoff document from current session state, (b) gracefully terminate current agent session, (c) start new session with handoff document as initial context.
4. **Heartbeat endpoint** that the agent touches every 60 seconds, reporting context depth, memory RSS, and current task. A lightweight monitoring script checks the heartbeat and alerts (or restarts) on stale heartbeats.

This stack uses zero external infrastructure beyond the VPS itself.

### Tactic 2: The Watchdog Cascade
**Difficulty: Low | Impact: High**

Three layers of watchdog, each catching what the layer below misses:

1. **systemd WatchdogSec** (120s): Catches event loop hangs. Requires `sd_notify("WATCHDOG=1")` integration.
2. **Cron script** (every 15 min): Checks process liveness AND output quality (is the agent producing meaningful responses?). Feeds failure logs to a fast/cheap LLM for diagnostics. Falls back to config restoration if LLM fix fails.
3. **Daily health audit** (once per 24h): Reviews agent logs for degradation patterns (rising error rates, increasing latency trends, growing memory usage). Triggers proactive maintenance (session rotation, memory cleanup) before failures occur.

### Tactic 3: Checkpoint-Commit-Resume Pattern
**Difficulty: Medium | Impact: High**

Combine git commits with conversation checkpointing:

1. Agent commits after every meaningful unit of work (file creation, test passing, config change).
2. Alongside the git commit, write a `CHECKPOINT.md` file capturing: current task, progress percentage, key decisions, next steps, known issues.
3. On crash recovery: (a) read `CHECKPOINT.md`, (b) read recent git log for context, (c) initialize new agent session with checkpoint context, (d) resume from last known state.

This is the pattern Anthropic uses internally: feature list JSON + progress.txt + git commits. The key insight is that git provides file-state persistence for free, and the checkpoint file bridges the gap to conversation-state persistence.

### Tactic 4: Cost Tripwires
**Difficulty: Low | Impact: Critical**

Implement before anything else:

```
Per-agent hourly budget: $2.00 (default, configurable per agent)
Global daily budget: $20.00 (hard cap, requires manual reset)
Per-task iteration limit: 25 (prevents infinite loops)
Per-call token ceiling: 100K input, 16K output

Alert thresholds:
  - 50% of hourly budget consumed -> log warning
  - 70% of daily budget consumed -> send notification
  - Any single call > $1.00 -> flag for review
  - Error rate > 20% in 5 minutes -> pause agent, alert human
```

The specific dollar amounts are calibrated for Claude Sonnet-class models at early-2026 pricing. Adjust based on your model tier and task complexity.

### Tactic 5: The Handoff Document Protocol
**Difficulty: Medium | Impact: High**

For session rotation to work, the handoff between sessions must be structured. Template:

```markdown
# Session Handoff — [Agent Name] — [Timestamp]

## Current State
- Task: [What the agent is working on]
- Progress: [X of Y steps complete]
- Branch: [Git branch name]
- Last commit: [Hash + message]

## Key Decisions Made
- [Decision 1 and rationale]
- [Decision 2 and rationale]

## Pending Work
- [ ] [Next task]
- [ ] [Subsequent task]

## Known Constraints
- [Constraint learned during this session]
- [Edge case discovered]

## Files Modified This Session
- [path/to/file1] — [what changed]
- [path/to/file2] — [what changed]

## Active Context
- [Any important context that would be lost in compaction]
```

This document serves as both the handoff context for the next session and the crash recovery checkpoint if the agent fails before the next scheduled rotation.

---

## 4. Contrarian & Minority Views

### "Let It Crash" Doesn't Work for AI Agents the Way It Works for Erlang
**★★★☆☆ Moderate**

The Erlang "let it crash" philosophy assumes that restarting a process returns it to a known-good initial state. For AI agents, this assumption is fragile. Restarting a Claude Code session doesn't restore the conversation context, the agent's understanding of the codebase, or the intermediate reasoning that led to the current approach. A naive restart is closer to amnesia than recovery.

Counter-argument: This is why the handoff document and checkpoint patterns exist — they reconstruct enough state for a meaningful restart. The principle still applies at the process level (let systemd restart the harness), but application-level recovery requires more than a clean restart.

### Compaction Might Be a Dead End for Multi-Day Runs
**★★☆☆☆ Emerging**

Some practitioners argue that compaction — even when well-implemented — is fundamentally insufficient for multi-day agent runs. The argument: every compaction loses information, and over hours/days, the accumulated losses degrade the agent's effective capability below a useful threshold. The alternative: treat each session as short-lived (2-4 hours max) and invest entirely in handoff quality rather than compaction sophistication.

The van Deth "Context Rot" analysis provides supporting evidence: Claude Code's auto-compaction, despite being a first-party implementation, still causes forgotten context and repeated work. OpenAI's Codex has the same reported issues (GitHub issue #11174).

Counter-argument: Compaction technology is improving rapidly. Server-side compaction (Codex's `/responses/compact`) is more sophisticated than client-side summarization. As models improve at distinguishing important from unimportant context, compaction will get better.

### The Supervisor-as-Agent Pattern May Create More Problems Than It Solves
**★★★☆☆ Moderate**

Using an LLM to supervise another LLM (the VIGIL pattern) introduces a second failure mode: the supervisor itself can hallucinate, misdiagnose, or fail. If your supervisor agent enters a degraded state, it may approve degraded behavior in the supervised agent — a correlated failure that's worse than no supervision at all.

The safer approach: make the supervision layer as deterministic as possible. Use rule-based checks (memory thresholds, latency thresholds, error rate thresholds) for detection, and reserve LLM judgment for ambiguous cases only. Atalay's production implementation does this — the bash watchdog is deterministic, and only the diagnostic step uses an LLM.

### Over-Engineering the Harness Might Matter Less Than Model Improvement
**★★☆☆☆ Emerging**

A minority view holds that the rapid pace of model improvement (longer context windows, better instruction-following, built-in tool use) will make most harness engineering obsolete within 12-18 months. If Claude's context window doubles and its ability to maintain coherence across long conversations improves, many of the workarounds described here become unnecessary.

Counter-argument: History shows that infrastructure concerns shift but never disappear. Even with better models, you still need process supervision, cost controls, and crash recovery. The specific techniques may change, but the architectural layers remain.

---

## 5. Decision Translation — "So What?"

### For the Agent OS Architecture

**Process supervision tree**: Implement a three-level supervision tree. systemd at the base, a lightweight TypeScript/Python coordinator in the middle, and per-agent process management at the top. Each level has independent restart strategies. Do not build a custom supervisor when systemd + application-level checkpointing covers 95% of cases.

**State persistence**: Use SQLite WAL as the primary state store. Log every agent action before execution. Checkpoint conversation state after every meaningful completion. Use git commits for file-state persistence. This gives you crash recovery with zero external infrastructure.

**Context management**: Do not rely on compaction alone for multi-day runs. Implement scheduled session rotation (every 4-8 hours, configurable) with structured handoff documents. The handoff document is the single most important artifact in the system — invest in its quality.

**Health monitoring**: Implement the full metric suite from Section 2.2, exposed as Prometheus-compatible metrics. Use Grafana for dashboarding. Set up PagerDuty/ntfy.sh alerts for critical thresholds. The most important alert is not "agent crashed" (systemd handles that) but "agent is degrading" (rising latency, growing memory, increasing error rate).

**Cost controls**: Implement the four-level budget hierarchy from Tactic 4 before deploying any agent. The global daily cap is non-negotiable. Start strict, loosen based on data. A $20/day global cap with per-agent hourly budgets of $2 is a reasonable starting point for a single-VPS deployment running 4-6 agents.

**Container isolation**: For multi-agent deployment, use Docker containers per agent (NanoClaw's approach) rather than processes per agent (Atalay's approach). The memory overhead is ~50-100MB per container, which is acceptable on a 32GB Hetzner VPS. The security and blast-radius benefits are significant.

### What Would Change This Assessment

- **Model context windows exceeding 2M tokens with maintained coherence**: Would reduce the urgency of context rotation, though not eliminate it.
- **First-party crash recovery from Anthropic or OpenAI**: If the API itself supported session persistence and recovery, much of the harness-level state management becomes unnecessary.
- **A production failure where a cost circuit breaker failed to trigger**: Would increase the priority of defense-in-depth cost controls.

---

## Key Unknowns & Open Questions

1. **Optimal session rotation interval**: No rigorous empirical study compares agent performance across different rotation intervals. The 4-8 hour recommendation is practitioner consensus, not measured.

2. **Handoff document fidelity**: How much context can be successfully transferred via a structured handoff document? At what point does information loss from session rotation exceed information loss from compaction? No one has measured this.

3. **Correlated failures in supervisor-agent systems**: When an LLM supervises another LLM using the same provider, provider-level degradation affects both simultaneously. How should the system handle this? The answer may be using different providers for supervisor and worker.

4. **Long-term memory integration with crash recovery**: How should the ephemeral state recovery system (checkpoints, WAL) interact with the persistent memory system (semantic memory, episodic memory)? These are typically designed independently, but they need to work together for multi-day coherence.

5. **Cost of reliability**: What is the overhead of full durable execution (checkpointing every step, WAL writes, heartbeats) vs. lightweight approaches (periodic checkpoints, git commits)? At what point does reliability infrastructure itself become a bottleneck?

---

## Source Log

| # | Source | Tier | Found Via | Contribution |
|---|--------|------|-----------|-------------|
| 1 | Zylos Research. "AI Agent Self-Healing and Auto-Recovery Patterns." (2026-02-17). [https://zylos.ai/research/2026-02-17-ai-agent-self-healing-auto-recovery] | B | Exa semantic search | Comprehensive survey of self-healing patterns: heartbeat vs. watchdog, circuit breakers, hybrid supervision model, observability metrics. Key claim: 60% downtime reduction, 67% failures from error handling. |
| 2 | Zylos Research. "Durable Execution Patterns for AI Agents." (2026-02-17). [https://zylos.ai/research/2026-02-17-durable-execution-ai-agents] | B | Exa semantic search | Framework comparison (Temporal, Restate, DBOS, LangGraph, Inngest). Temporal $5B valuation context. Saga pattern for AI workflows. Event sourcing for agent state. |
| 3 | Erturk, Mehmet. "Circuit Breakers, Rate Limits, and Cost Ceilings: Production Safety for AI Systems." (2026-02-08). [https://ertyurk.com/posts/circuit-breakers-rate-limits-and-cost-ceilings] | C | Exa semantic search | Detailed implementation of three-layer cost control: circuit breakers (5 failures/60s), rate limiting (token bucket + node-level), cost ceilings (per-workflow budgets). Production-tested patterns. |
| 4 | Atalay, Oguzhan. "Architecting a Multi-Agent AI Fleet on a Single VPS." DEV Community (2026-02-25). [https://dev.to/oguzhanatalay/architecting-a-multi-agent-ai-fleet-on-a-single-vps-3h4c] | C | Exa search | First-person account of running 6 AI agents on a single VPS with systemd, per-agent ports, cron watchdog with LLM diagnostics, oversight agent on cheap model. Practical lessons: treat agents like junior devs, config changes are most dangerous. |
| 5 | Anthropic Engineering. "Effective context engineering for AI agents." (2025-09-29). [https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents] | S | Prior research / Brave search | Primary source on context engineering: compaction, structured note-taking, sub-agent architectures. "Find the smallest possible set of high-signal tokens." |
| 6 | Schmid, Philipp. "Context Engineering for AI Agents: Part 2." (2025-12-04). [https://www.philschmid.de/context-engineering-part-2] | A | Prior research / Brave search | Manus insights: compaction vs. summarization distinction, pre-rot threshold (256K on 1M window), 50% trigger point, 5 rewrites in 6 months. |
| 7 | Cruz, Christopher. "VIGIL: A Reflective Runtime for Self-Healing LLM Agents." arXiv:2512.07094 (2025-12). [https://arxiv.org/abs/2512.07094] | A | Brave search + multiple confirmations | Reflective supervisor runtime that monitors sibling agent behavioral logs. Novel EmoBank + stage-gated diagnosis. No production deployments reported. |
| 8 | White, S.R. et al. "An architectural approach to autonomic computing." ICAC 2004. DOI: 10.1109/icac.2004.1301340 | A | Crossref verification | IBM's MAPE-K loop: Monitor-Analyze-Plan-Execute over shared Knowledge. Foundational framework for self-managing systems. Verified via Crossref. |
| 9 | White, S.R. et al. "Autonomic computing: Architectural approach and prototype." Integrated Computer-Aided Engineering 13(2), 2006. DOI: 10.3233/ica-2006-13206 | A | Crossref verification | Extended version of the MAPE-K architecture with prototype implementation. Verified via Crossref. |
| 10 | Esposito et al. "Autonomic Microservice Management via Agentic AI and MAPE-K Integration." ECSA 2025. | A | Semantic Scholar | Modern application of MAPE-K to agentic AI systems for microservice management. 3 citations. |
| 11 | ClawTank. "Running OpenClaw as a systemd Service: Always-On Linux Setup." (2026-02-18). [https://clawtank.dev/blog/openclaw-systemd-linux-service] | C | Exa search | systemd configuration for OpenClaw: service unit patterns, logging, restart strategies. |
| 12 | OpenClaw GitHub Issue #1731. "Add systemd watchdog support via sd_notify." (2026-01-25). [https://github.com/openclaw/openclaw/issues/1731] | B | Exa search | Documents the gap between HTTP health checks and event loop hang detection. Closed as completed. |
| 13 | NanoClaw GitHub repo (qwibitai/nanoclaw). [https://github.com/qwibitai/nanoclaw] | B | Brave search | Container-per-agent isolation, ~700 lines host orchestrator, Apple Container/Docker, filesystem IPC. |
| 14 | Erlang/OTP Design Principles documentation. [https://www.erlang.org/doc/system/design_principles.html] | S | Brave search | Canonical reference for supervision trees, restart strategies (one-for-one, one-for-all, rest-for-one), OTP behaviors. |
| 15 | Temporal blog. "Of course you can build dynamic AI agents with Temporal." [https://temporal.io/blog/of-course-you-can-build-dynamic-ai-agents-with-temporal] | B | Brave search | Temporal Event History as crash recovery record. Agent uses history for past decisions rather than re-planning. |
| 16 | Temporal blog. "Managing very long-running Workflows with Temporal." [https://temporal.io/blog/very-long-running-workflows] | B | Brave search | Event replay mechanics: Worker replays Event History to resume at appropriate place with appropriate state. |
| 17 | Berger, Eleanor. "Use Git for Automated Checkpointing." (2025-10-11). [https://elite-ai-assisted-coding.dev/p/use-git-for-automated-checkpointing] | C | Exa search | Git commits as universal checkpoint mechanism: detailed history, easy rollback, standard tooling. |
| 18 | Anthropic Claude Agent SDK docs. "Rewind file changes with checkpointing." [https://console.anthropic.com/docs/en/agent-sdk/file-checkpointing] | S | Exa search | First-party file checkpointing via Write/Edit/NotebookEdit tools. Snapshot and restore semantics. Does not capture Bash-made changes. |
| 19 | OpenAI. "Shell + Skills + Compaction: Tips for long-running agents that do real work." [https://developers.openai.com/blog/skills-shell-tips/] | S | Exa search + Brave search | Server-side compaction, skills system, /responses/compact endpoint. "Auto compaction mechanism" that aims to remove need for manual intervention. |
| 20 | van Deth, Vincent. "Context Rot Is Silently Killing Your Claude Code Sessions." [https://vincentvandeth.nl/blog/context-rot-claude-code-automatic-rotation] | C | Exa search | Practitioner diagnosis of context rot: silent quality degradation from lossy compaction. Proposes automatic rotation. |
| 21 | Google Developers Blog. "Architecting efficient context-aware multi-agent framework for production" (ADK). (2025-12-04). | S | Prior research | Tiered storage model: working context, session, memory, artifacts. Context as "compiled view over richer stateful system." |
| 22 | Zylos Research. "SQLite WAL Mode: Patterns and Pitfalls for AI Agent Systems." (2026-02-20). [https://zylos.ai/research/2026-02-20-sqlite-wal-mode-ai-agent-systems] | B | Exa search | WAL failure modes: checkpoint starvation, unbounded growth, lock semantics. Production hardening patterns. |
| 23 | OpenClaw bulletproof-memory skill. [https://playbooks.com/skills/openclaw/skills/bulletproof-memory] | C | Exa search | WAL protocol with SESSION-STATE.md for agent memory persistence across compaction and restarts. |
| 24 | ayushmi/agentstate GitHub repo. [https://github.com/ayushmi/agentstate] | C | Exa search | WAL+snapshots, watch streams, idempotency, leases for AI agent state management. Python/Rust. 55 stars. |
| 25 | Grafana Cloud. "AI Observability." [https://grafana.com/docs/grafana-cloud/monitor-applications/ai-observability/] | B | Tavily search | Performance tracking, safety monitoring, tool analytics, MCP observability, GPU observability for AI stacks. |
| 26 | Zylos Research. "AI Agent Cost Optimization: Token Economics and FinOps in Production." (2026-02-19). [https://zylos.ai/research/2026-02-19-ai-agent-cost-optimization-token-economics] | B | Exa search | Agents make 3-10x more LLM calls than chatbots. $5-8 per task for unconstrained agents. Four pillars: cost landscape, caching, model routing, FinOps tooling. |
| 27 | Athenic. "AI Agent Rate Limiting: Implementing Token Budgets and Usage Quotas." (2025-12-02). [https://getathenic.com/blog/ai-agent-rate-limiting-token-budgets] | C | Exa search | Token sliding windows, multi-level limits (per-request, per-user, per-org, global), pre-flight estimation. |
| 28 | Balditsyn et al. "Uncertainty-Driven Monitoring for ML-Based Autonomic Systems." ACSOS 2025. | A | Semantic Scholar | Modern MAPE-K application with uncertainty quantification for monitoring phase. |
| 29 | Eunomia. "Checkpoint/Restore Systems: Evolution, Techniques, and Applications in AI Agents." (2025-05-11). [https://eunomia.dev/blog/2025/05/11/checkpointrestore-systems-evolution-techniques-and-applications-in-ai-agents/] | B | Brave search | Hybrid approach: stateful checkpointing for fast recovery + stateless model checkpoints for full crash recovery. CRIU for process-level snapshots. |
| 30 | Zylos Research. "AI Agent Session Continuity: Maintaining State Across Restarts and Crashes." (2026-02-18). [https://zylos.ai/research/2026-02-18-ai-agent-session-continuity] | B | Exa search | Layered approach: durable execution for step recovery, memory systems for semantic context, message queues for preventing work loss during gaps. |

---

## Audit Notes

1. **IBM autonomic computing "manifesto"**: The commonly referenced "An Architectural Blueprint for Autonomic Computing" (IBM White Paper, 2003/2006) did not appear as an independent DOI via Crossref. The closely related peer-reviewed papers by White et al. (ICAC 2004, DOI: 10.1109/icac.2004.1301340; ICA-E 2006, DOI: 10.3233/ica-2006-13206) were verified and cited instead. The "manifesto" may be an IBM technical report without a DOI. Claims about the MAPE-K loop are supported by the verified papers.

2. **VIGIL paper**: No DOI found via Crossref for arXiv:2512.07094. It is available on arXiv and cited by multiple secondary sources (ResearchGate, Medium articles, emergentmind.com), but has not been confirmed as peer-reviewed. Confidence capped at ★★★☆☆.

3. **Market size claims**: The $7.92B market figure and 45.82% CAGR from Zylos are attributed to unnamed market research. The 34% budget overrun stat from AgentC2 blog cites "a16z 2025 AI Infrastructure report" but I could not independently verify this specific figure. Both flagged as ★★★☆☆.

4. **Temporal metrics**: The $5B valuation and 9.1 trillion execution count are from Temporal's own blog post about their Series D funding. These are self-reported by the company. Cross-referenced with VentureBeat and TechCrunch reporting on the funding round.

5. **60% downtime reduction**: Claimed by Zylos without linking to a specific study. Treated as practitioner consensus rather than empirically established. ★★★☆☆.

6. **Erlang supervision trees**: The OTP documentation is authoritative. Claims about "40+ years" of telecom use are consistent with Erlang's 1986 origin at Ericsson and OTP's 1998 public release, but the specific claim is a simplification — supervision trees were formalized in OTP, not in Erlang's earliest years. Adjusted to "battle-tested patterns" without specific year claims in the text.

7. **Prior research integration**: This document builds on findings from `/research/outputs/openclaw-architecture-deep-dive.md` (OpenClaw single-process architecture), `/research/outputs/context-window-management-autonomous-agents.md` (compaction strategies), and `/research/outputs/memory-daemon-llm-agents.md` (background memory management). Key findings are referenced rather than duplicated.

8. **Missing perspective**: This research does not cover GPU/TPU supervision patterns (relevant if Agent OS ever runs local models) or multi-node distributed supervision (relevant only if scaling beyond a single VPS). Both are explicitly out of scope for a single-Hetzner-VPS deployment.
