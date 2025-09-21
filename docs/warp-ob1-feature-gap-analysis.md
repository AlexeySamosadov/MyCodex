# Warp/OB1 Feature Gap Analysis for Codex

This document captures the outcomes of Discovery workstream Phase 1 from the Warp/OB1 functional parity roadmap. It enumerates high-value capabilities observed in Warp and OB1, compares them with current Codex functionality, defines the MVP parity scope, and seeds a prioritized backlog for future phases.

## 1. Feature Matrix

| Capability Area | Warp Highlights | OB1 Highlights | Codex Today | Gap Assessment | Notes |
|-----------------|-----------------|----------------|-------------|----------------|-------|
| Terminal Rendering | GPU-accelerated renderer, block-based transcript, fast scrolling, inline rich text | Multi-pane layout, inline media (images, charts), adaptive theme | Ratatui-based TUI, linear scrollback, basic styling | **High** – need block model, perf work, better rendering fidelity | Evaluate extending ratatui vs. integrating GPU-backed renderer |
| Command Blocks & Metadata | Each command captured as a block with exit code, duration, context, share link | Structured command cards with env info and retry | Text-based prompt/result pairs, limited metadata | **High** – implement block abstraction, metadata capture, share primitives | Drives later collaboration and telemetry efforts |
| Command Palette / Launcher | Global search for commands/snippets/files, fuzzy search, keyboard-first | Palette includes AI-recommended workflows, project templates | Basic command history, no palette | **Medium** – need palette UX, plugin surface for actions | Could leverage existing `codex exec` automation |
| AI Assistance | Inline suggestions, "Explain this command", natural language command generation | Multi-agent sequences, verification loops, context-aware tool invocation | GPT agent with instruction following, manual tool invocation | **Medium** – need richer reasoning loops, inline surfaces | Pair with agent orchestration upgrades |
| Memory & Context | Cloud sync of sessions, long context retrieval, command annotations | Project timelines, memory inspector, explicit hand-offs between agents | Session history per run, limited persistent memory beyond transcripts | **High** – design memory hierarchy, persistence model | Align with Project Manager agent vision |
| Collaboration | Live share sessions, commenting, team spaces | Multi-user editing, async review workflows | None natively; rely on MCP integrations | **High** – new service & CLI commands required | Later phase but requires early architectural decisions |
| Automation / Workflows | Reusable workflows, CLI automation, schedule tasks | Advanced automation tied to repositories, CI integration | `codex exec` scripts, MCP tooling, but limited templating | **Medium** – expand templates, scheduling, integrations | Coordinate with DevEx workstream |
| Observability | Inline metrics for commands, usage analytics | Command success dashboards, agent performance metrics | Basic logging, manual inspection | **Low-Medium** – instrumentation improvements | Block model prerequisite |
| Security / Governance | Team management, secret storage, policy controls | Audit trails, approval workflows | Sandbox defaults, minimal RBAC | **Medium** – need enterprise-ready controls | Balance with privacy constraints |

### Key Takeaways
* Terminal rendering, memory persistence, and collaboration are critical gaps needing foundational work before advanced UX layers.
* Command blocks and metadata unlock downstream features (sharing, analytics, notebooks) and should lead the MVP scope.
* Codex’s agent strength can be expanded through orchestrated reasoning loops and integration with improved memory systems.

## 2. MVP Parity Definition

The MVP aims to achieve a compelling terminal + agent experience competitive with Warp, while staging more complex OB1-style collaboration for later milestones.

### MVP Goals
1. **Structured Command Transcript** with block metadata (command, output, exit code, duration, working directory) and inline AI annotations.
2. **Enhanced Agent Reasoning Loop** featuring Project Manager supervision, tool selection guidance, and automatic summarization of progress.
3. **Persistent Memory Layer** enabling retrieval of prior decisions, TODOs, and context beyond a single session.
4. **Command Palette + Workflow Launcher** for fuzzy searching commands, saved snippets, and Codex tasks.
5. **Performance Baseline** achieving sub-100ms perceived latency for command rendering and <2s response time for agent reasoning in common tasks.

### Out of Scope for MVP
* Real-time collaborative editing or live share.
* Cloud workspace provisioning and secret management.
* Full enterprise governance (approvals, auditing dashboards).
* GPU-accelerated renderer beyond what is required to meet MVP performance targets.

## 3. Prioritized Backlog Seeds

The following backlog items are grouped by theme and priority (P0 highest). They should be refined into actionable issues/epics.

### Terminal & Transcript (Owner: Terminal UX pod)
- **P0:** Draft architecture RFC for command block model (data schema, storage, rendering, telemetry hooks).
- **P0:** Prototype block rendering within current ratatui stack, including basic diffing and metadata display.
- **P1:** Implement scrollback persistence with lazy loading for long sessions.
- **P1:** Integrate inline AI annotations (e.g., explain output, suggest next steps) anchored to blocks.
- **P2:** Explore GPU-backed rendering spike for high-latency environments.

### Memory & Project Management (Owner: Memory pod)
- **P0:** Design memory hierarchy (short/mid/long-term) and select storage backend(s) with local-first strategy.
- **P0:** Build summarization + vector store prototype for session memory rejuvenation.
- **P1:** Define schema and API for Project Manager agent to track plan-of-record, tasks, and risks.
- **P1:** Implement evaluation harness measuring retention over 1K/5K/10K token windows.
- **P2:** Investigate optional cloud sync with encryption at rest and RBAC.

### Agent Orchestration (Owner: Agent Intelligence pod)
- **P0:** Create prompt library and protocols for Project Manager, Implementer, Reviewer agents.
- **P0:** Implement self-critique loop leveraging command/test feedback.
- **P1:** Establish multi-agent message schema + error handling contract (integrated with MCP).
- **P1:** Benchmark agents on Terminal Bench and Codex custom suites; compare vs Warp/OB1 baselines.
- **P2:** Provide configurable agent personas/templates for different workflows.

### Command Palette & Automation (Owner: Productivity pod)
- **P0:** Specify command palette UX (keyboard shortcuts, fuzzy search, integration points).
- **P1:** Integrate saved snippets and `codex exec` scripts into palette results.
- **P1:** Add workflow launcher for curated templates (scaffold, deploy, troubleshoot).
- **P2:** Schedule or trigger workflows via CLI/cron integration.

### Observability & Quality (Owner: Platform pod)
- **P0:** Define telemetry events for command blocks, agent outcomes, latency metrics (opt-in controls).
- **P1:** Expand snapshot/UI tests to cover new block rendering and agent transcripts.
- **P1:** Set up dashboards tracking MVP success metrics (latency, success rate, retention).

## 4. Discovery Follow-ups & Research Plan

* **User Interviews:** Recruit 8–10 power users across Warp, OB1, and current Codex cohorts. Focus on terminal pain points, collaboration expectations, and memory requirements. Deliver summary report with opportunity sizing.
* **Competitive Teardowns:** Produce detailed walkthroughs of Warp and OB1 flows (command execution, sharing, AI assist) with annotated screenshots for reference during design reviews.
* **Technical Spikes:**
  - Benchmark ratatui rendering performance under heavy output scenarios vs. Warp’s GPU renderer.
  - Evaluate existing open-source terminal engines (e.g., Alacritty, Zed terminal) for potential integration.
* **Success Metric Baseline:** Instrument current Codex to capture command latency, agent success rate, and context retention to establish pre-change benchmarks.

## 5. Risks Identified During Discovery

1. **Rendering Rewrite Complexity:** Introducing a block model may require large-scale refactors of the TUI. Mitigate via incremental layering and early prototyping.
2. **Memory Storage Costs:** Long-lived context may demand significant storage; plan tiered retention and user-configurable pruning.
3. **Agent Reliability:** Multi-agent orchestration could degrade UX if agents disagree or loop; design arbitration and fallback strategies.
4. **Privacy Concerns:** Persistent memory and telemetry must respect sandboxed environments and user consent; ensure data isolation.

---
_Authored by: Codex Discovery Team (Week 2)_
