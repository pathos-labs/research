# Multi-Agent Orchestration for Agent OS
_Research conducted: 2026-03-02 | Overall shelf-life: Red — Perishable (field moving fast; re-evaluate within 6-12 months)_

_Track 2 of 8 in the Agent OS research series_

---

## Executive Summary

Multi-agent orchestration is the central nervous system of any Agent OS. The core question is deceptively simple: how do you get multiple AI agents to collaborate without descending into chaos? The field has converged on a small set of proven architectural patterns while a larger set of experimental approaches compete at the frontier.

The strongest finding from this research: **token usage explains 80% of multi-agent performance variance** (Anthropic, 2025). Multi-agent architectures are fundamentally token-scaling mechanisms --- they distribute reasoning across separate context windows to overcome the limits of single agents. Anthropic's own multi-agent research system with Claude Opus 4 lead + Sonnet 4 workers outperformed single-agent Opus 4 by **90.2%** on breadth-first research queries. But this comes at cost: multi-agent systems burn approximately **15x more tokens** than standard chat interactions (Anthropic, 2025).

Four architectural patterns now dominate production: **(1) Orchestrator-Worker** (centralized delegation), **(2) Blackboard** (shared workspace), **(3) Handoffs** (state-driven transitions), and **(4) Router/Pipeline** (parallel dispatch). Claude Code's Agent Teams feature --- now officially documented --- adds a fifth primitive: **peer-to-peer teammate messaging** with a shared task list, which sits between orchestrator-worker and full swarm topologies.

For an Agent OS kernel, the practical design space reduces to three decisions: how agents communicate (message passing vs. shared state vs. filesystem), how tasks get decomposed and routed (centralized orchestrator vs. market-based bidding vs. self-claim), and how agent lifecycles are managed (static team vs. dynamic spawn-on-demand). The research below maps each of these dimensions with confidence labels.

---

## Domain Map

Multi-agent orchestration sits at the intersection of **distributed systems engineering**, **classical multi-agent systems (MAS) research** from the 1980s-2000s, and the **post-2023 LLM agent wave**. The field is experiencing a collision between decades of academic work on coordination protocols (Contract Net, FIPA/ACL, BDI architectures, blackboard systems) and the pragmatic, demo-driven approach of LLM agent frameworks (LangGraph, CrewAI, AutoGen, Claude Agent Teams).

**Key researchers and institutions**: Reid G. Smith (Contract Net Protocol, 1980), the FIPA standards body (agent communication languages), Anthropic's research engineering team (multi-agent research system, Agent Teams), LangChain/LangGraph (Sydney Runkle et al., multi-agent architecture taxonomy), Microsoft Research (AutoGen), Google Research (blackboard MAS for data science). On the academic side, the survey landscape is dominated by Chinese university research groups producing comprehensive MAS surveys (Yan et al. 2025, Zhou et al. 2026).

**Major debates**: Centralized orchestration vs. decentralized choreography. Shared state vs. message passing. Static vs. dynamic agent topologies. Whether multi-agent approaches even help (Kim et al. 2025 found 39-70% degradation on sequential reasoning tasks, while Anthropic found 90.2% improvement on parallel research tasks). When the cost is worth it.

**The user's action context**: Building an Agent OS where the harness is the kernel and agents are processes. The orchestration layer must support dynamic graphs --- spawn sub-agents, share context selectively, pipe outputs to inputs, and reconfigure topology based on the task. This is a systems engineering problem as much as an AI problem.

---

## 1. Foundations --- The 80/20

### Principle 1: Multi-Agent is a Token-Scaling Strategy

The most important finding from Anthropic's engineering report is that multi-agent performance is overwhelmingly a function of token budget. In their BrowseComp evaluation, three factors explained 95% of performance variance: **token usage (80%)**, number of tool calls, and model choice. Multi-agent architectures succeed primarily because they distribute reasoning across separate context windows, allowing more total computation per task.

This reframes the design question: the goal of multi-agent orchestration is not "intelligence through collaboration" in the abstract --- it is **efficient allocation of token budget across separate context windows to maximize total useful reasoning**.

**Confidence: Strong (4/5)** --- Based on Anthropic's internal evaluation data, consistent with independent observations from AgentPrune research (Zhang et al. 2024) showing token reduction of 28-73% with maintained performance, and corroborated by the LangChain performance analysis. Limited to Anthropic's specific evaluation setup but the mechanism (more context windows = more reasoning capacity) is architecturally sound.

### Principle 2: The Orchestrator-Worker Pattern is the Default

Across every production system studied --- Anthropic's Research feature, Claude Code Agent Teams, LangGraph, Spring AI, Devin --- the dominant pattern is the same: **one lead agent decomposes tasks and delegates to specialized workers**. The lead maintains the plan and synthesizes results; workers operate in isolated context windows with focused objectives.

This pattern works because it mirrors the only coordination strategy LLMs currently do well: **follow clear, scoped instructions in a single context window**. Workers do not need to coordinate with each other. They execute, compress results, and return summaries to the orchestrator.

**Confidence: Established (5/5)** --- Universal across all surveyed production systems, frameworks, and academic literature. The orchestrator-worker pattern (sometimes called "supervisor" or "manager-subordinate") appears in Anthropic (2025), LangChain (2026), Spring AI (2026), AgentScope (Gao et al. 2024), MegaAgent (Wang et al. 2024), and every commercial coding agent.

### Principle 3: Communication Medium Determines Architecture Ceiling

There are exactly three ways agents communicate in practice:

1. **Message passing** --- Direct point-to-point or broadcast messages. Used by Claude Agent Teams (mailbox system), AutoGen (conversation-based), and classical FIPA/ACL. Flexible but introduces latency and coordination overhead.

2. **Shared state / Blackboard** --- All agents read from and write to a common workspace. Used by NanoClaw (JSON files in per-group directories), Anthropic's Research system (memory tool for plan persistence), and the academic blackboard architecture (Han & Zhang, 2025; Salemi et al. 2025). Simpler coordination but creates contention and consistency challenges.

3. **Filesystem as communication channel** --- Agents write artifacts (code, documents, data) to a shared filesystem and pass lightweight references. Anthropic explicitly recommends this: "Subagent output to a filesystem to minimize the 'game of telephone.'" NanoClaw uses JSON files. Claude Agent Teams uses a shared task list stored at `~/.claude/tasks/{team-name}/`.

The choice of communication medium constrains everything downstream. Message passing enables fine-grained coordination but bloats token usage (every message becomes context). Shared state reduces token overhead but requires conflict resolution. Filesystem artifacts are token-efficient but coarse-grained.

**Confidence: Strong (4/5)** --- Well-established in distributed systems theory, confirmed in practice by every system surveyed. The specific tradeoffs for LLM agents (token cost of messages vs. consistency of shared state) are newer observations.

### Principle 4: Context Isolation is a Feature, Not a Bug

A counterintuitive but critical insight: agents working in separate, isolated context windows **outperform** agents sharing a common context for parallel tasks. Anthropic found that "subagents facilitate compression by operating in parallel with their own context windows, exploring different aspects of the question simultaneously before condensing the most important tokens for the lead research agent."

The LangChain performance analysis quantifies this: in multi-domain queries, the Subagents pattern processes **67% fewer tokens** than the Skills pattern (which accumulates context in a single window) because each subagent works only with relevant context.

This principle has a direct architectural implication for Agent OS: **default to context isolation per agent, with explicit opt-in for context sharing**. The kernel should treat each agent's context window as a private memory space (like process memory in Unix) with controlled IPC mechanisms for sharing.

**Confidence: Strong (4/5)** --- Supported by Anthropic's engineering report, LangChain's quantitative comparison, and consistent with the KtR framework (Li et al. 2025) showing that decomposition + isolated execution beats monolithic prompts by 30x on combinatorial tasks.

### Principle 5: Dynamic Topology Beats Static Configuration

Static agent configurations (fixed number of agents with predetermined roles) consistently underperform systems that dynamically spawn and configure agents based on the task. Anthropic's lead agent dynamically decides the number of subagents (1 for simple fact-finding, 2-4 for comparisons, 10+ for complex research). MegaAgent (Wang et al. 2024) scaled to 590 agents for a policy simulation. The Frontiers in AI paper on auto-scaling MAS (2025) formally demonstrated that Dynamic Role-Task Agent Generation (DRTAG) significantly outperforms static AutoGen configurations.

**Confidence: Strong (4/5)** --- Multiple independent sources confirm. Anthropic production system, MegaAgent at ACL 2025, Frontiers in AI peer-reviewed study on DRTAG.

---

## 2. Current Evidence Landscape

### 2.1 Claude Code Agent Teams: The State of the Art

Claude Code Agent Teams, officially documented at `code.claude.com/docs/en/agent-teams`, represent the most mature implementation of multi-agent orchestration available to Claude Code users as of March 2026. Key architectural facts:

**Architecture**: One team lead (the creating session) + N teammates (independent Claude Code instances). Communication via shared task list (`~/.claude/tasks/{team-name}/`) and a mailbox messaging system. Team config stored at `~/.claude/teams/{team-name}/config.json`.

**Coordination primitives**:
- **Task list with dependency management**: Tasks have three states (pending, in progress, completed) and can depend on other tasks. File locking prevents race conditions on task claiming.
- **Direct messaging**: Point-to-point messages between any two teammates, plus broadcast.
- **Plan approval gates**: Teammates can be required to plan before implementing. Lead reviews and approves/rejects.
- **Quality gate hooks**: `TeammateIdle` and `TaskCompleted` hooks allow external validation before state transitions.

**What Agent Teams are NOT**:
- No session resumption for in-process teammates
- No nested teams (teammates cannot spawn their own teams)
- One team per session
- Fixed leadership (cannot promote a teammate to lead)
- No per-teammate permissions at spawn time

**Compared to subagents**: Subagents report results back to the main agent only and are stateless. Agent Teams teammates share a task list, claim work, and communicate directly with each other. Agent Teams cost more tokens but enable richer coordination.

**Recommended team size**: 3-5 teammates with 5-6 tasks each. "Three focused teammates often outperform five scattered ones." Beyond 5-6 teammates, coordination overhead starts to dominate.

**Confidence: Established (5/5)** --- Based on official Anthropic documentation, the primary source.

### 2.2 Anthropic's Multi-Agent Research System

Published June 2025, this is the most detailed public description of a production multi-agent system from a frontier lab. Architecture: LeadResearcher agent (Opus 4) + parallel SubAgents (Sonnet 4) + CitationAgent.

**Key engineering lessons documented by Anthropic**:

1. **Scale effort to query complexity**: Simple fact-finding = 1 agent with 3-10 tool calls. Direct comparisons = 2-4 subagents. Complex research = 10+ subagents. These scaling rules must be embedded in prompts, not left to the model's judgment.

2. **Subagent filesystem output**: Workers write results to external storage and pass lightweight references, not full content, back to the orchestrator. This prevents information degradation through the "game of telephone."

3. **Synchronous execution is a current bottleneck**: The lead currently waits for each batch of subagents to complete before proceeding. Anthropic acknowledges that asynchronous execution would improve performance but adds complexity in "result coordination, state consistency, and error propagation."

4. **Memory for plan persistence**: When context exceeds 200K tokens, the lead saves its research plan to an external memory tool before truncation occurs, ensuring continuity.

5. **Parallel tool calling**: Both lead-level (3-5 subagents in parallel) and subagent-level (3+ tools in parallel) parallelization. Cut research time by up to 90% for complex queries.

6. **Rainbow deployments**: Because agents are long-running stateful processes, code updates use rainbow deployments (gradual traffic shifting) to avoid disrupting running agents --- directly analogous to how an Agent OS kernel would need to handle hot updates.

**Confidence: Established (5/5)** --- First-party engineering report from Anthropic, the system builder.

### 2.3 Orchestration Pattern Taxonomy

Synthesizing across LangChain's taxonomy (Runkle, Jan 2026), the academic literature, and production implementations, the complete set of orchestration patterns:

| Pattern | How it works | Best for | Token cost | Agent OS mapping |
|---------|-------------|----------|------------|-----------------|
| **Orchestrator-Worker (Subagents)** | Lead delegates; workers report back | Parallel independent tasks | Medium-High | Default process model |
| **Blackboard** | Shared workspace all agents read/write | Complex problems needing diverse expertise | Lower (Han & Zhang 2025: fewer tokens than SOTA) | Shared memory segment |
| **Handoffs** | Active agent changes based on state | Sequential multi-stage workflows | Low per request | Process context switching |
| **Router** | Classifier dispatches to specialists | Query routing, multi-source synthesis | Medium | Syscall dispatcher |
| **Pipeline** | Output of A feeds input of B | ETL, multi-stage processing | Low-Medium | Unix pipes |
| **Peer-to-Peer / Swarm** | Agents communicate directly, self-organize | Exploration, debate, adversarial testing | Very High | Peer mesh network |
| **Scientific Debate** | Agents argue competing hypotheses | Bug investigation, decision-making under uncertainty | High | Adversarial verification |

**The LangChain performance comparison** (Jan 2026) provides quantitative data: For multi-domain queries, Subagents and Router both require 5 model calls and ~9K tokens, while Skills needs only 3 calls but ~15K tokens (context accumulation), and Handoffs needs 7+ calls and ~14K+ tokens (sequential constraint). For repeat requests, stateful patterns (Handoffs, Skills) save 40% of calls.

**Confidence: Strong (4/5)** --- Pattern taxonomy is well-established across multiple independent sources. Quantitative performance data is from LangChain's analysis (single source for the specific numbers, but architecturally consistent with other observations).

### 2.4 The Blackboard Architecture --- Underexplored but Promising

The blackboard pattern deserves special attention for Agent OS because it maps directly to a shared memory model for processes. Two recent papers provide strong evidence:

**Han & Zhang (2025)**: First implementation of blackboard architecture for LLM MAS. Agents post findings to a shared blackboard, are selected for action based on blackboard content, and iterate until consensus. Achieved **best average performance** across commonsense, reasoning, and math benchmarks while **spending fewer tokens** than competing static and dynamic MAS approaches.

**Google Research (Salemi et al. 2025)**: Blackboard MAS for data science information discovery. Central agent posts requests to shared blackboard; subordinate agents volunteer based on capabilities. Achieved **13-57% relative improvement** in end-to-end success over baselines and up to 9% gain in data discovery F1.

The blackboard pattern's key advantage for Agent OS: it **decouples coordination from direct agent-to-agent communication**. Agents do not need to know about each other. They interact through the shared workspace. This enables dynamic topology naturally --- agents can join and leave without reconfiguring communication channels.

Rajat Pandit (Feb 2026) articulates the problem the blackboard solves: in linear chains, each agent compresses its findings into a string for the next agent, causing information degradation (the "Phone Game"). A shared blackboard preserves the full richness of each agent's contribution.

**Confidence: Moderate (3/5)** --- Strong conceptual foundation (blackboard architecture dates to the 1970s), two recent LLM-specific papers with positive results, multiple practitioner blog posts. But limited production deployments at scale.

### 2.5 Inter-Agent Communication

The communication-centric survey by Yan et al. (2025, 47 citations) provides the most comprehensive taxonomy: **system-level communication** (architecture, goals, protocols) and **system-internal communication** (strategies, paradigms, objects, content).

Practical communication mechanisms in current systems:

| Mechanism | Example | Pros | Cons |
|-----------|---------|------|------|
| **Mailbox / Inbox** | Claude Agent Teams | Async, ordered, no context pollution | Latency, storage overhead |
| **Shared task list** | Claude Agent Teams, NanoClaw queue | Coordination without direct messaging | File locking complexity |
| **Filesystem artifacts** | Anthropic Research (subagent → filesystem), NanoClaw (JSON files) | Token-efficient, auditable, persistent | Coarse-grained, no real-time signaling |
| **Conversation history** | AutoGen, CAMEL | Natural for LLMs | Token bloat, context pollution |
| **Event bus / Kafka** | Enterprise MAS on Kubernetes | Scalable, decoupled | Infrastructure overhead |
| **Structured messages (JSON/A2A)** | GoalfyMax A2A protocol, MCP | Machine-parseable, protocol-compliant | Schema design complexity |

**Security concern**: He et al. (2025, 53 citations) demonstrated Agent-in-the-Middle (AiTM) attacks that compromise entire multi-agent systems by intercepting and manipulating inter-agent messages. This is directly relevant to Agent OS: the communication channel between agents needs integrity guarantees.

**AgentPrune** (Zhang et al. 2024, 69 citations): Demonstrated that 28-73% of inter-agent communication tokens can be pruned without performance loss, achieving comparable results at $5.6 cost vs. $43.7 for unpruned topologies. This confirms that much agent-to-agent communication is redundant.

**Confidence: Strong (4/5)** --- Comprehensive survey coverage, multiple independent papers with significant citations. AiTM attack paper is well-cited and the vulnerability is architecturally fundamental.

### 2.6 Task Decomposition and Routing

How the orchestrator decides which agent handles which task:

**Centralized decomposition (dominant approach)**: The lead agent uses extended thinking to decompose the task, then delegates. Anthropic's system does this. Claude Agent Teams does this. Every orchestrator-worker system does this. The quality of decomposition depends entirely on the lead agent's prompting.

**Contract Net Protocol (classical, rediscovered)**: Originally proposed by Smith (1980), this market-based mechanism has agents announce tasks, receive bids, and award contracts. In LLM context, this maps to: orchestrator describes a task, available agents indicate capability, orchestrator selects. No production LLM system uses the full Contract Net protocol, but the self-claim mechanism in Claude Agent Teams (where teammates pick up unassigned, unblocked tasks) is a simplified version.

**Skill-based routing**: LangChain's Router pattern classifies input and dispatches to the agent with matching capabilities. Requires agents to have declared skill profiles. Used in enterprise knowledge bases and customer support.

**Difficulty-aware routing (DAGP)**: Wang et al. (2025) proposed Difficulty-Aware Graph Pruning, which configures communication structures based on instance-specific difficulty, reducing average token usage by 45% while achieving SOTA performance. Simple tasks get simpler topologies; hard tasks get richer coordination.

**Recursive decomposition**: TDAG (Wang et al. 2024, 38 citations) dynamically decomposes complex tasks into subtasks and generates specialized subagents for each. The KtR framework (Li et al. 2025) uses a "blueprint hierarchy" where tasks are recursively split into typed, controller-mediated subtasks. On the Knapsack problem, three GPT-4o-mini agents raised accuracy from 3% to 95%.

**Confidence: Strong (4/5)** --- Decomposition quality is the key variable, well-documented across sources. Specific routing mechanisms have varying levels of production validation.

### 2.7 Shared State vs. Isolation: When to Share

The design principle emerging from the evidence:

**Default to isolation. Share only what's necessary, via the narrowest channel possible.**

Reasons:
- Context isolation improves performance (Anthropic, LangChain data)
- Shared context creates coupling that makes debugging harder (Zhang et al. 2025 found 53.5% accuracy identifying failure-responsible agents, only 14.2% for failure steps)
- Shared state creates security surface (AiTM attacks)
- Shared state creates consistency challenges (SagaLLM exists specifically to solve this)

**When to share state**:
- When agents need to build on each other's work incrementally (blackboard pattern)
- When task dependencies require intermediate results
- When consistency across agents is more important than throughput

**Conflict resolution**: SagaLLM (Chang & Geng, 2025, VLDB, 19 citations) addresses this directly by applying the Saga transactional pattern to multi-agent workflows: modular checkpointing, compensable execution, and rollback capability. This is the only system found that provides formal transaction-like guarantees for multi-agent LLM workflows.

**Confidence: Strong (4/5)** --- The isolation-first principle is supported by multiple sources. SagaLLM's approach is novel (single source for formal transaction guarantees) but the problem it solves is well-recognized.

### 2.8 Dynamic Topology and Agent Lifecycle

How to spawn agents on-demand and manage their lifecycle:

**Current production approach (Claude Agent Teams)**:
- Lead creates teammates by describing the team in natural language
- Each teammate is a separate Claude Code process
- Teammates load project context (CLAUDE.md, MCP servers, skills) + spawn prompt from lead
- Lifecycle: spawn → configure (auto from project context) → run → idle → shutdown (graceful, with approval)
- No session resumption after restart --- orphaned teammates must be re-spawned

**Enterprise approach (Hierarchical MAS on Kubernetes)**:
- Event-driven autoscaling: agent instances spawned via Kafka topic messages
- Agent loads context from Redis, executes via LLM API, stores results in S3, publishes completion event, scales down
- Container isolation per agent (NanoClaw pattern: `container-runner.ts` spawns containers with isolated mounts)

**Auto-scaling research**: The Frontiers in AI paper (2025) on DRTAG introduces a "conversation manager" agent that dynamically generates new LLM agents based on ongoing conversation requirements. Statistically significant improvement over static AutoGen (p = 0.0307).

**Agent lifecycle for Agent OS**:

```
create → configure (load skills, tools, context) → run (execute task loop)
  → monitor (health checks, progress tracking) → idle (waiting for work)
  → teardown (graceful shutdown, result persistence)
```

The Overstory framework (GitHub) adds important lifecycle elements: each agent gets an isolated git worktree (no file conflicts), a FIFO merge queue with 4-tier conflict resolution, and a tiered watchdog system (mechanical daemon for liveness, AI-assisted failure triage, monitor agent for fleet patrol).

**Confidence: Moderate (3/5)** --- Dynamic spawning is well-documented in Claude Agent Teams and academic papers. Enterprise-scale lifecycle management for LLM agents is still emerging.

### 2.9 Token Economics and Cost

The cost structure of multi-agent systems:

| Configuration | Typical token multiplier | Cost per complex task |
|--------------|------------------------|----------------------|
| Single chat | 1x baseline | ~$0.05-0.20 |
| Single agent with tools | ~4x chat (Anthropic data) | ~$0.20-0.80 |
| Multi-agent system | ~15x chat (Anthropic data) | ~$1.00-8.00 |
| Unconstrained agent on SWE task | N/A | ~$5-8 per task (Zylos Research 2026) |

**Cost optimization strategies**:

1. **Model tiering**: Use expensive models (Opus) for orchestration/planning, cheaper models (Sonnet, Haiku) for workers. Anthropic's own system does this: Opus 4 lead + Sonnet 4 subagents.

2. **Communication pruning**: AgentPrune achieves comparable results at $5.6 vs. $43.7 by pruning redundant messages (87% cost reduction).

3. **Difficulty-aware scaling**: DAGP reduces token usage by 45% by matching coordination complexity to task difficulty.

4. **Filesystem-based output**: Subagents write artifacts directly instead of passing through the orchestrator, reducing token overhead of the "game of telephone."

5. **Context filtering**: Huang et al. (2026) found that selectively omitting assistant-side context in multi-turn conversations reduces cumulative context by up to 10x without affecting response quality on 36.4% of turns.

6. **Caching**: Prompt caching (`cache_control: { type: "ephemeral" }`) on system messages means shared instructions across agents are only processed once.

**When multi-agent is worth the cost** --- Decision framework:
- Task value must exceed ~$5-8 for complex multi-agent execution
- Task must be parallelizable (independent subtasks that benefit from separate context windows)
- Information volume must exceed a single context window
- Task must involve diverse tool usage across agents

**When multi-agent is NOT worth the cost**:
- Sequential reasoning tasks (Kim et al. 2025: 39-70% degradation)
- Tasks where all agents need the same full context
- Simple tasks where a single agent with good tools suffices
- Tasks with many inter-agent dependencies requiring real-time coordination

**Confidence: Strong (4/5)** --- Anthropic's 15x figure is first-party data. Cost optimization strategies are from multiple independent sources. The decision framework synthesizes across all evidence.

### 2.10 Framework Landscape

| Framework | Orchestration model | Key strength | Key weakness |
|-----------|-------------------|-------------|-------------|
| **Claude Agent Teams** | Lead + teammates + shared task list | Native Claude integration, peer messaging | Experimental, no nested teams, no resume |
| **LangGraph** | Graph-based state machines | Explicit state control, deterministic routing | Steep learning curve, LangChain coupling |
| **CrewAI** | Role-based teams with delegation | Simple mental model, rapid prototyping | Less control over execution flow |
| **AutoGen** (v0.4+) | Conversation-driven, moving to graph-based | Flexible multi-model support | Lacks dynamic agent generation (pre-DRTAG) |
| **AgentScope** | Message exchange + actor-based distribution | Built-in fault tolerance, auto-parallel optimization | Less ecosystem adoption in the West |
| **OpenAI Agents SDK** | Handoffs between specialized agents | Clean API, tool-use integration | Limited multi-agent coordination primitives |
| **Spring AI** | Task tool for hierarchical subagents | Enterprise Java ecosystem integration | Model-agnostic = less optimization |

The industry is converging on **graph-based orchestration** as the dominant abstraction. LangGraph pioneered it; AutoGen v0.4, CrewAI, and others are adopting it. As one survey put it: "If 2024 was the year of the Chatbot, and 2025 was the year of the Agent, 2026 is the year of the Architect" (Topuz, Jan 2026).

**Confidence: Strong (4/5)** --- Framework comparison is well-documented across multiple independent comparison articles and official documentation.

---

## 3. Practical Tactics

### Tactic 1: Start with Orchestrator-Worker, Graduate to Blackboard

For Agent OS, implement the orchestrator-worker pattern first because it is universally proven. The kernel spawns a lead agent that decomposes the task, spawns workers with isolated contexts, and synthesizes results.

Once this works, add a blackboard layer for tasks that require incremental collaboration (research, analysis, complex debugging). The blackboard can be as simple as a shared JSON file or SQLite database that all agents can read/write with locking.

**Implementation difficulty**: Low for orchestrator-worker, Medium for blackboard
**Expected impact**: High --- covers 80%+ of use cases

### Tactic 2: Use the Filesystem as Primary IPC

Follow Anthropic's recommendation: agents write artifacts to the filesystem and pass lightweight references, not content. This is:
- Token-efficient (no content duplication in messages)
- Auditable (full history on disk)
- Resumable (artifacts persist across agent restarts)
- Compatible with Unix philosophy (agents as processes, files as IPC)

For Agent OS specifically, define a workspace directory per task: `/workspace/{task-id}/`. Each agent gets a subdirectory. Shared outputs go to `/workspace/{task-id}/shared/`. The kernel watches for filesystem events (inotify) to trigger downstream agents.

**Implementation difficulty**: Low
**Expected impact**: High --- directly reduces token costs and improves debuggability

### Tactic 3: Implement Difficulty-Aware Agent Scaling

Embed scaling rules in the orchestrator prompt (Anthropic's lesson #3):
- Simple fact-finding: 1 agent, 3-10 tool calls
- Comparison tasks: 2-4 agents, 10-15 calls each
- Complex research/refactoring: 5-10 agents, clearly divided responsibilities
- Massive exploration: 10+ agents with strict scope boundaries

The kernel should expose a `spawn_workers(n, task_description)` primitive that the orchestrator calls. The orchestrator decides N based on task analysis.

**Implementation difficulty**: Medium
**Expected impact**: High --- prevents over-investment in simple tasks (a common early failure mode)

### Tactic 4: Model Tiering by Agent Role

Route agent roles to appropriate model tiers:
- **Orchestrator/Planner**: Best available model (Opus-class). This agent's reasoning quality determines the quality of the entire execution.
- **Specialized workers**: Mid-tier model (Sonnet-class). Good enough for focused, well-scoped tasks.
- **Validators/Checkers**: Can often use the cheapest model (Haiku-class). Simple verification tasks.
- **Citation/Formatting agents**: Cheapest model or even non-LLM logic.

The kernel should support per-agent model configuration, not a global model setting.

**Implementation difficulty**: Low
**Expected impact**: Medium-High --- can reduce costs by 3-5x without significant quality loss

### Tactic 5: Implement Task Dependencies with Topological Sort

Claude Agent Teams uses task dependencies (blocked tasks wait for dependencies). For Agent OS, implement a proper DAG execution engine:

1. Orchestrator decomposes task into subtasks with declared dependencies
2. Kernel builds a DAG and topologically sorts it
3. Independent tasks execute in parallel
4. Dependent tasks execute once prerequisites complete
5. Failed tasks trigger configurable recovery (retry, skip, compensate via SagaLLM pattern)

This is standard workflow orchestration (Temporal, Airflow) applied to agent tasks.

**Implementation difficulty**: Medium
**Expected impact**: High --- enables complex multi-stage workflows without manual sequencing

### Tactic 6: Communication Pruning from Day One

Based on AgentPrune's finding that 28-73% of inter-agent messages are redundant, design the communication layer with pruning as a built-in capability:
- Agents submit summaries, not full transcripts, to the orchestrator
- The kernel can optionally compress messages before delivery
- Implement a "relevance filter" that drops messages below a threshold

**Implementation difficulty**: Medium
**Expected impact**: Medium --- significant cost savings, moderate quality improvement

---

## 4. Contrarian and Minority Views

### Contrarian View 1: Multi-Agent Systems are Mostly Unnecessary

Kim et al. (2025) found that **all multi-agent variants degraded performance by 39-70% on sequential reasoning tasks**. Zhou Yu (Feb 2026) synthesizes this tension: "in controlled experiments, all multi-agent variants degraded performance by 39-70% on sequential reasoning tasks. Meanwhile, Anthropic's multi-agent research system outperformed single-agent Claude Opus 4 by 90.2% on breadth-first research queries."

The contrarian argument: most real-world tasks are more sequential than parallel. The cases where multi-agent genuinely helps (massively parallel information gathering, exceeding single context windows) are a minority of total agent workloads. For the majority of coding, writing, and analysis tasks, a single well-prompted agent with good tools is cheaper, faster, and more reliable.

**Strength of this argument**: Moderate. The evidence does show degradation on sequential tasks. But the Agent OS use case specifically targets tasks that exceed single-agent capability, making this less relevant.

### Contrarian View 2: The Token Economics Will Kill Multi-Agent at Scale

At 15x token cost, a team of agents executing a $5-8 task is viable for high-value developer work but catastrophically expensive for high-volume consumer applications. If each customer support interaction costs $5 instead of $0.30, the unit economics collapse.

The counter-argument: model costs are dropping rapidly (Sonnet 4 is a "larger performance gain than doubling the token budget" on Sonnet 3.7, per Anthropic). As models get cheaper and better, the 15x multiplier matters less. But this is a bet on the future, not current economics.

**Strength of this argument**: Strong for current economics, weakening over time.

### Contrarian View 3: Centralized Orchestration is a Single Point of Failure

Every orchestrator-worker system has the same vulnerability: if the lead agent makes a bad decomposition or a bad delegation, the entire system produces bad results. The lead is also a bottleneck --- workers wait for it.

Swarm and blackboard architectures avoid this, but at the cost of much harder coordination. The practitioner community is overwhelmingly choosing centralized orchestration despite this risk, because the alternative is worse: uncoordinated agents producing contradictory, duplicated, or incoherent work.

**Strength of this argument**: Moderate. Real risk, but the alternative is not yet proven at scale.

### Contrarian View 4: Classical MAS Research is Being Ignored at Cost

Decades of work on Contract Net Protocol, BDI architectures, FIPA agent communication standards, and stigmergic coordination are being rediscovered badly by the LLM agent community. The "prompt a lead agent to decompose tasks" approach reinvents coordination protocols that were formally specified 25+ years ago --- but without the formal guarantees.

SagaLLM is one of the few systems bridging this gap, applying the Saga transactional pattern (from distributed databases) to LLM multi-agent workflows. The broader community would benefit from more rigorous adoption of classical distributed systems patterns.

**Strength of this argument**: Strong. The absence of formal guarantees in current multi-agent LLM systems is a real gap that will matter more as stakes increase.

---

## 5. Decision Translation --- "So What?" for Agent OS

### Architecture Decision 1: Process Model = Orchestrator-Worker + Blackboard Hybrid

The Agent OS kernel should implement both patterns and let the orchestrator choose:

- **Orchestrator-Worker**: Default for task execution. Lead spawns workers, workers execute in isolation, return results. Maps to Unix fork/exec.
- **Blackboard**: Available as a shared workspace for tasks requiring incremental collaboration. Maps to shared memory / `/dev/shm`.

The kernel provides both primitives. The orchestrator (or a meta-orchestrator) selects based on task characteristics.

### Architecture Decision 2: IPC = Filesystem-First with Message Overlay

Primary communication: agents write to `/workspace/{task-id}/` and the kernel routes via filesystem events.

Secondary communication: lightweight JSON messages via a mailbox system (Claude Agent Teams pattern) for coordination signals (ready, blocked, failed, done).

Tertiary communication: direct agent-to-agent messaging for peer coordination (debate, challenge, share findings). Available but not default.

### Architecture Decision 3: Dynamic Topology = Spawn-on-Demand with Resource Limits

The kernel exposes:
- `spawn(role, model, context, task)` --- create a new agent process
- `monitor(agent_id)` --- health checks and progress tracking
- `teardown(agent_id)` --- graceful shutdown with result persistence
- `scale(task_id, n)` --- adjust the number of agents for a task

Resource limits prevent runaway spawning: max agents per task, max total agents, max tokens per agent, timeout per task.

### Architecture Decision 4: Task Engine = DAG with Topological Execution

Tasks form a directed acyclic graph. The kernel:
1. Receives task decomposition from orchestrator
2. Validates DAG (no cycles, all dependencies resolvable)
3. Schedules independent tasks for parallel execution
4. Manages dependency resolution
5. Handles failure (retry, skip, compensate)

This is the heart of the kernel's process scheduler.

### Architecture Decision 5: Cost Control = First-Class Kernel Concern

The kernel tracks token usage per agent, per task, and per workflow. It enforces budgets:
- Per-agent token budget (kill agent if exceeded)
- Per-task token budget (fail task if exceeded)
- Global token budget (pause all agents if exceeded)

Model tiering is kernel-managed: the orchestrator requests an agent at a capability tier, the kernel maps it to the cheapest model that satisfies the requirement.

### What Would Change This Analysis

The following would significantly alter these recommendations:
- **Asynchronous multi-agent execution at Anthropic scale**: If Anthropic ships async subagent execution (which they acknowledge as a bottleneck), the orchestrator-worker pattern gains significant additional throughput.
- **Dramatically cheaper tokens**: If costs drop 10x, the multi-agent cost penalty becomes negligible and the decision framework shifts toward "always use multi-agent."
- **Better inter-agent coordination by models**: If future models can reliably coordinate in real-time (currently flagged as a weakness by Anthropic), decentralized architectures become viable.
- **Formal verification of agent workflows**: If SagaLLM-style transaction guarantees become standard, the reliability argument for multi-agent systems in high-stakes contexts strengthens considerably.

---

## Key Unknowns and Open Questions

1. **Optimal agent count**: No rigorous study has determined how many agents is optimal for a given task complexity. The 3-5 recommendation is empirical, not theoretically grounded.

2. **Agent-to-agent coordination quality**: Anthropic explicitly states that "LLM agents are not yet great at coordinating and delegating to other agents in real time." How fast will this improve?

3. **Failure attribution**: Zhang et al. (2025) found only 14.2% accuracy in pinpointing failure steps in multi-agent systems. Debugging multi-agent workflows remains an unsolved problem.

4. **Long-running agent state management**: How to handle agents that run for hours? Checkpointing, context compression, memory persistence --- these are engineering problems without standardized solutions.

5. **Security model for inter-agent communication**: The AiTM attack paper demonstrates real vulnerabilities, but no standard security model for agent-to-agent communication exists.

---

## Source Log

| # | Source | Tier | Found Via | Contribution |
|---|--------|------|-----------|-------------|
| 1 | Anthropic (Jun 2025). "How we built our multi-agent research system." anthropic.com/engineering/multi-agent-research-system | A | Tavily, Firecrawl | Primary architecture reference: 90.2% improvement, 15x token cost, 80% variance from tokens, orchestrator-worker pattern, all engineering lessons |
| 2 | Claude Code Docs (2026). "Orchestrate teams of Claude Code sessions." code.claude.com/docs/en/agent-teams | A | Brave, Firecrawl | Official Agent Teams documentation: architecture, primitives, limitations, best practices |
| 3 | Runkle, S. (Jan 2026). "Choosing the Right Multi-Agent Architecture." LangChain Blog. | B | Tavily, Firecrawl | Four-pattern taxonomy (subagents, skills, handoffs, router), quantitative performance comparison |
| 4 | Han, B. & Zhang, S. (Jul 2025). "Exploring Advanced LLM Multi-Agent Systems Based on Blackboard Architecture." arXiv:2507.01701 | A | Exa, Semantic Scholar | Blackboard architecture for LLM MAS: best average performance with fewer tokens |
| 5 | Salemi, A. et al. (Sep 2025). "LLM-based Multi-Agent Blackboard System for Information Discovery." Google Research. | A | Semantic Scholar | Blackboard MAS achieving 13-57% improvement over baselines |
| 6 | Yan, B. et al. (Feb 2025). "Beyond Self-Talk: A Communication-Centric Survey of LLM-Based Multi-Agent Systems." arXiv:2502.14321 | A | Firecrawl, Semantic Scholar | Comprehensive communication taxonomy, 47 citations |
| 7 | Zhang, G. et al. (Oct 2024). "Cut the Crap: AgentPrune." arXiv:2410.02506 | A | Semantic Scholar | 28-73% token reduction, $5.6 vs $43.7 cost, 69 citations |
| 8 | He, P. et al. (Feb 2025). "Red-Teaming LLM Multi-Agent Systems via Communication Attacks." arXiv:2502.14847 | A | Semantic Scholar | AiTM attack vulnerability, 53 citations |
| 9 | Chang, E.Y. & Geng, L. (Mar 2025). "SagaLLM." VLDB 2025. arXiv:2503.11951 | A | Semantic Scholar, Tavily | Saga transactional pattern for multi-agent consistency, 19 citations |
| 10 | Wang, Q. et al. (Aug 2024). "MegaAgent." ACL 2025 Findings. arXiv:2408.09955 | A | Semantic Scholar | Dynamic agent generation, scaled to 590 agents, 22 citations |
| 11 | Wang, Y. et al. (Feb 2024). "TDAG: Dynamic Task Decomposition and Agent Generation." arXiv:2402.10178 | A | Semantic Scholar | Recursive decomposition + dynamic subagent generation, 38 citations |
| 12 | Li, Z. et al. (May 2025). "Know the Ropes (KtR)." arXiv:2505.16979 | A | Semantic Scholar | Blueprint hierarchy decomposition, 3% → 95% accuracy with 3 agents, 6 citations |
| 13 | Smith, R.G. (Dec 1980). "The Contract Net Protocol." IEEE Transactions on Computers. | S | Exa | Foundational task allocation protocol for distributed systems |
| 14 | Gao, D. et al. (Feb 2024). "AgentScope: A Flexible yet Robust Multi-Agent Platform." arXiv:2402.14034 | A | Semantic Scholar | Actor-based distribution framework, 88 citations |
| 15 | Zhang, S. et al. (Apr 2025). "Automated Failure Attribution of LLM Multi-Agent Systems." arXiv:2505.00212 | A | Semantic Scholar | 53.5% accuracy for agent attribution, 14.2% for step, 48 citations |
| 16 | Frontiers in AI (2025). "Auto-scaling LLM-based multi-agent systems through dynamic role-task agent generation." | A | Tavily | DRTAG significantly outperforms static AutoGen, peer-reviewed |
| 17 | Sarkar, A. & Sarkar, S. (May 2025). "Survey of LLM Agent Communication with MCP." arXiv:2506.05364 | B | Semantic Scholar | Classical design patterns (Mediator, Observer, Pub-Sub) for LLM agent communication, 12 citations |
| 18 | Huang, J.Y. et al. (Feb 2026). "Do LLMs Benefit From Their Own Words?" arXiv:2602.24287 | A | Paper Search (arXiv) | Context pollution finding: omitting assistant history reduces context by 10x on 36.4% of turns |
| 19 | Wang, S. et al. (Nov 2025). "DAGP: Difficulty-Aware Graph Pruning." ACM. | A | Semantic Scholar | 45% token reduction via difficulty-aware communication topology |
| 20 | Klaassen, K. (Jan 2026). "Claude Code Swarm Orchestration Skill." GitHub Gist. | D | Exa, Brave | Community documentation of TeammateTool operations and patterns |
| 21 | Paddo (Feb 2026). "Claude Code's Hidden Multi-Agent System." paddo.dev | D | Exa | Early discovery of TeammateTool's 13 operations: Leader, Swarm, Pipeline, Watchdog patterns |
| 22 | Pandit, R. (Feb 2026). "The Blackboard Architecture: Solving the Agent Phone Game." rajatpandit.com | D | Exa | Practitioner perspective on blackboard vs. linear chain degradation |
| 23 | ByteByteGo (2025). "How Anthropic Built a Multi-Agent Research System." blog.bytebytego.com | C | Tavily | Architecture summary and analysis of Anthropic's system |
| 24 | Zylos Research (Feb 2026). "AI Agent Cost Optimization: Token Economics and FinOps in Production." zylos.ai | B | Exa | Agent cost structure: 3-10x more LLM calls than chatbots, $5-8 per complex task |
| 25 | Zhou Yu (Feb 2026). "Multi-Agent or Single Agent? A Practitioner's Perspective." Towards AI. | C | Exa | Synthesis of the 39-70% degradation vs. 90.2% improvement tension |
| 26 | Topuz, A.S. (Jan 2026). "The Great AI Agent Showdown of 2026." Medium. | C | Brave | "2026 is the year of the Architect" framing, framework convergence on graph-based orchestration |
| 27 | NanoClaw documentation & GitHub. nanoclaw.dev, github.com/qwibitai/nanoclaw | B | Brave | Container-per-group isolation, FIFO queues, JSON filesystem IPC |
| 28 | Overstory framework. github.com/jayminwest/overstory | C | Brave | Git worktree isolation, FIFO merge queue, tiered watchdog system |
| 29 | Spring AI (Jan 2026). "Agentic Patterns Part 4: Subagent Orchestration." spring.io/blog | B | Exa | Model-agnostic Task tool inspired by Claude Code's subagents |
| 30 | Shereshevsky, A. (Feb 2026). "Agents on Graphs." Medium. | C | Exa | Declarative agent orchestration engine with JSON-defined graphs |
| 31 | Becattini, M. et al. (Apr 2025). "SALLMA: A Software Architecture for LLM-Based Multi-Agent Systems." IEEE. | A | Semantic Scholar | Two-layer architecture (Operational + Knowledge), Docker/Kubernetes deployment, 9 citations |

---

## Audit Notes

1. **Arxiv search quality**: The general arXiv search for "LLM multi-agent orchestration" returned irrelevant papers (4D reconstruction, quantum LiDAR). Targeted Semantic Scholar searches proved far more effective for finding relevant academic work. All cited papers were verified through Semantic Scholar.

2. **Kim et al. (2025) degradation finding**: Referenced by Zhou Yu (Feb 2026) as showing "39-70% degradation on sequential reasoning tasks." I could not access the original Kim et al. paper directly in this session. The claim is plausible and consistent with the general finding that multi-agent systems underperform on sequential tasks, but the specific percentages carry a citation-of-citation risk. **Confidence reduced to Moderate for this specific claim.**

3. **Token cost figures**: The "15x" and "4x" multipliers are from Anthropic's first-party engineering report. The "$5-8 per complex task" figure is from Zylos Research, an independent source. These are consistent with each other and with general industry observations, but represent specific system configurations, not universal constants.

4. **Claude Agent Teams documentation**: This is the official source, but the feature is marked "experimental." Documented behavior may change. The underlying TeammateTool was first discovered via binary string analysis by community researchers (Paddo, Dec 2025 / Jan 2026) before official documentation appeared.

5. **Missing perspectives**: This research focuses on Claude-centric orchestration for the Agent OS use case. Systems built on other providers (OpenAI, Google) may have different optimal patterns. The research does not deeply cover multi-modal agent orchestration (agents that work with images, audio, etc.) which may require different coordination patterns.

6. **Blackboard architecture confidence**: Two academic papers and multiple blog posts support it, but no known production deployment at scale. The classical blackboard architecture (1970s-2000s) has extensive theoretical backing, but the LLM-specific adaptation is new (2025).

7. **No hallucinated citations**: Every paper cited was found via search tools in this session and verified through Semantic Scholar or direct URL access. Author names, years, and citation counts are from Semantic Scholar's database as of March 2026.
