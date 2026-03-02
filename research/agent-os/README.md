# Agent OS — Research Package

**Date:** 2026-03-02
**Total:** ~50,000 words | 270+ sources | 8 research tracks

Deep research on building a production-grade Claude Code CLI harness — an operating system for AI agents. Always-on, self-healing, multi-agent, MCP-native, secure.

## Research Tracks

| # | Track | Focus | Words | Sources |
|---|-------|-------|-------|---------|
| 1 | [Process Architecture & Self-Healing](agent-os-process-architecture.md) | Supervision trees, crash recovery, session rotation, cost circuit breakers | 7,446 | 30 |
| 2 | [Multi-Agent Orchestration](agent-os-multi-agent-orchestration.md) | Orchestrator-Worker + Blackboard, Claude Agent Teams, token economics | 6,865 | 31 |
| 3 | [Interface & Control Plane](agent-os-interface-control-plane.md) | Protocol-first (JSON-RPC), Discord, Warp Oz, approval workflows | ~5,000 | 28 |
| 4 | [Security & Production Hardening](agent-os-security-hardening.md) | 5-layer defense-in-depth, container isolation, cost as security | 7,900 | 55 |
| 5 | [MCP Architecture](agent-os-mcp-architecture.md) | Gateway pattern, lazy loading, read/write pools, tool scoping | 5,854 | 38 |
| 6 | [Developer Experience](agent-os-developer-experience.md) | Convention-over-config, inner loop, observability, daily ops | ~5,000 | 32 |
| 7 | [Ecosystem Landscape](agent-os-ecosystem-landscape.md) | *Claw ecosystem, enterprise platforms, trends, predictions | 6,800 | 56 |
| 8 | [Architecture Decision Record](agent-os-architecture-decision-record.md) | 11 decisions, 6-phase build order, risk register | ~5,000 | Synthesis |

## Architecture Summary

- **Runtime:** TypeScript + Claude Agent SDK
- **Process model:** Container-per-agent (Docker), 3-level supervision tree
- **Control daemon:** JSON-RPC 2.0 over Unix socket + WebSocket
- **Agent lifecycle:** Convention-over-config (directory + CLAUDE.md)
- **Multi-agent:** Orchestrator-Worker + Blackboard hybrid
- **Interface:** Protocol-first, Discord bot as first client
- **MCP:** Gateway with lazy loading, read/write pool separation
- **Security:** 5-layer defense-in-depth, default-deny
- **Deployment:** Single Hetzner VPS, Docker Compose, systemd

## Build Order

1. Skeleton — Daemon that spawns one agent in a container
2. Safety — Cost circuit breakers, command allowlists, audit logging
3. Resilience — Supervision tree, session rotation, handoff documents
4. Discord — Bot client speaking the JSON-RPC protocol
5. MCP + Multi-Agent — Gateway, tool scoping, orchestrator-worker
6. Polish — Stress testing, monitoring, documentation
