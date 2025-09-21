# Warp/OB1 Architecture Foundations Plan for Codex

This document expands Roadmap Phase 2 ("Architecture Foundations") with concrete design decisions, interfaces, and sequencing. It bridges the discovery outputs with engineering execution so the Terminal UX, Memory, and Agent Intelligence pods can implement Warp/OB1 parity features in a coordinated way.

## 1. Goals and Non-Goals

**Goals**
- Provide an incrementally adoptable architecture that introduces a command block model, richer rendering, and metadata capture without disrupting existing Codex workflows.
- Define clear boundaries between the CLI, daemon, and emerging services (memory, collaboration) to enable parallel development.
- Expose extension points (via MCP-compatible APIs) for future palettes, workflows, and third-party integrations.
- Establish performance and observability baselines that guarantee <16 ms render latency and sub-2 s perceived agent response under typical loads.

**Non-goals**
- Deliver GPU-accelerated rendering or cloud workspace orchestration (deferred to later phases).
- Ship real-time collaboration; this document only ensures the architecture leaves room for it.
- Redesign security/governance policies beyond establishing integration hooks.

## 2. Terminal Engine Strategy

### 2.1 Architectural Direction
- **Baseline renderer:** Continue building on the ratatui stack for the MVP, encapsulating it behind a `TerminalSurface` trait that can later be implemented by alternative renderers (GPU, WebGL).
- **Block abstraction:** Introduce a `CommandBlock` data structure captured by the daemon, persisted locally, and rendered by the TUI. Blocks include metadata (command text, directory, env fingerprint, exit code, duration, timestamps, tags) and a stream of output frames.
- **Incremental adoption:** Existing linear transcript views wrap each prompt/response in a synthetic block, allowing hybrid sessions during rollout.

### 2.2 Data Flow
1. CLI issues a command to the daemon through the existing IPC channel (Unix socket / named pipe).
2. Daemon spawns the process, streaming stdout/stderr through a block-aware multiplexer that:
   - Captures metadata at start/end events.
   - Annotates frames with ANSI parsing results and structural markers (sections, tables).
3. The daemon writes `CommandBlock` records to the local block store (SQLite or sled) and broadcasts incremental updates over a pub/sub channel.
4. The TUI subscribes to block updates, performs diffing against its current render tree, and requests lazy loading for older blocks to control memory usage.

### 2.3 Rendering Model
- **Virtualized scrollback:** Maintain only visible blocks plus N cached blocks in memory, fetching older content on demand.
- **Styling pipeline:** Apply stylized spans via ratatui `Stylize` helpers, using theming descriptors for backgrounds/foregrounds to ensure future renderer parity.
- **Inline annotations:** Reserve per-block side panels to display AI explanations, follow-up suggestions, or metrics. These are represented as `BlockAttachment` records so they can be rendered or exported uniformly.
- **Accessibility:** Support dynamic font scaling (through terminal capabilities), high-contrast themes, and a simplified text-only fallback for headless environments.

### 2.4 Performance & Observability
- Measure render latency via instrumentation in the TUI (`codex_tui_render_latency_ms`) and daemon publish latency (`codex_block_publish_ms`).
- Implement back-pressure signals: if the renderer falls behind, the daemon batches frame updates with diff hints to avoid UI thrash.
- Persist block indices with checksum hashes to detect data corruption and enable deterministic replay for testing.

## 3. Command Block Schema

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Stable identifier shared across CLI, daemon, and services |
| `sequence` | Integer | Monotonic order of execution in the session |
| `command_text` | String | Raw user-entered command or agent-synthesized action |
| `working_dir` | Path | Directory where the command executed |
| `env_fingerprint` | Hash | Hash of relevant environment variables / profile |
| `exit_code` | Integer | Process exit status |
| `duration_ms` | Integer | Wall-clock duration |
| `started_at` / `finished_at` | Timestamp | Precise timing for analytics and ordering |
| `attachments` | Array<BlockAttachment> | AI annotations, metrics, shared links |
| `outputs` | Array<OutputFrame> | Streamed output with ANSI spans and structural hints |
| `tags` | Array<String> | User/agent-provided labels (e.g., `build`, `deploy`, `error`) |

`OutputFrame` encapsulates a chunk of stdout/stderr with metadata:
- `stream`: enum (`stdout`, `stderr`, `system`)
- `content`: UTF-8 string with normalized newline handling
- `decorations`: list of semantic tokens (table, code block, markdown heading)
- `timestamp_offset_ms`: relative to `started_at`

`BlockAttachment` types cover inline explanation markdown, follow-up commands, telemetry snapshots, and shareable URLs. Attachments include `visibility` flags for private vs. shareable data.

## 4. Service Topology & Interfaces

### 4.1 Components
- **CLI (codex-cli):** Presents the TUI, handles user input, and renders blocks.
- **Daemon (`codexd`):** Executes commands, manages block storage, brokers agent interactions.
- **Memory service (future local service):** Provides retrieval APIs for mid/long-term memory; initially runs embedded in the daemon process.
- **Collaboration/telemetry gateway:** Optional process for syncing blocks or metrics to cloud endpoints when enabled.

### 4.2 IPC and API Contracts
- Adopt a gRPC-over-uds (Unix domain socket) interface for high-volume block streaming with flow control, while keeping existing lightweight JSON IPC for commands to preserve backward compatibility.
- Define proto schemas:
  - `ExecuteCommandRequest` / `ExecuteCommandResponse` (includes block ID, status updates).
  - `SubscribeBlocksRequest` / `BlockUpdateStream` for real-time UI updates.
  - `AttachmentRequest` for adding AI annotations or telemetry records.
- Provide HTTP/WebSocket adapters for eventual remote clients; these wrap the same proto messages to maintain parity.

### 4.3 Storage Layout
- Local block store: `~/.codex/blocks/<session-id>.sqlite` with normalized tables (`blocks`, `frames`, `attachments`).
- Indexed views to support filters (by tag, time range, exit status).
- Garbage collection policy configurable via CLI (retain by age, count, or storage quota).

### 4.4 Agent & Memory Integration
- When an agent generates a command, the daemon links the resulting block with the agent conversation ID for traceability.
- Summaries from the memory system are stored as `BlockAttachment::Summary` entries with embeddings persisted to the memory vector store.
- Retrieval calls provide the TUI with context chips that can be displayed alongside relevant blocks.

## 5. Extensibility & Plugin Model

- Extend the MCP tool registry with **block hooks**: third-party tools can observe block lifecycle events (`on_block_created`, `on_block_completed`) to trigger automations.
- Introduce **Palette Providers**: plugins expose searchable commands/snippets via a simple trait (`fn results(query, context) -> Vec<PaletteItem>`). Providers declare required capabilities and scopes to enforce security boundaries.
- Allow **Attachment Injectors**: integrations can append attachments (e.g., CI status, ticket links) after validation. Attachments must pass sanitization to prevent malicious rendering.
- Provide sandbox-safe scripting by executing plugins in WASM or separate processes communicating over MCP channels.

Security considerations:
- All plugins run with least privilege; they only receive normalized block metadata unless explicitly granted access to raw output.
- Audit logs capture plugin activity (block IDs touched, actions taken) and feed into observability dashboards.

## 6. Milestones (Weeks 2–6)

| Week | Milestone | Description | Owner |
|------|-----------|-------------|-------|
| 2 | Architecture RFC Draft | Publish RFC summarizing block model, IPC changes, and storage design. Circulate for review. | Terminal UX + Platform |
| 3 | Prototype Block Capture | Implement daemon-side block struct, local SQLite persistence, and basic streaming to a CLI debug view. | Terminal UX |
| 4 | Rendering Spike | Integrate block rendering into ratatui behind feature flag, including virtualized scrollback and attachment placeholders. | Terminal UX |
| 5 | IPC Consolidation | Transition CLI ↔ daemon streaming to gRPC-over-uds, maintain compatibility shims. | Platform |
| 6 | Extensibility Spec | Define MCP plugin extensions (block hooks, palette providers) and produce sample plugin scaffolds. | Productivity + Platform |

Exit criteria:
- RFC approved with consensus across pods.
- Prototype demonstrates capturing at least three sequential commands with metadata and streaming updates without regressions.
- Performance instrumentation in place with baseline metrics recorded.

## 7. Open Questions & Risks

1. **Renderer swap readiness:** If ratatui cannot sustain performance targets, evaluate embedding a GPU-based renderer. Decision checkpoint at Week 6 with profiling data.
2. **Storage footprint:** Need empirical data on block store growth for long sessions; may require compression or deduplicated frame storage.
3. **Cross-platform IPC:** Windows support for Unix sockets is limited; fallback to named pipes with equivalent flow control must be validated.
4. **Security review:** Plugin hooks introduce new surfaces; plan security review before exposing to third parties.
5. **Migration path:** Determine how to migrate legacy transcripts into block format without data loss.

---
_Prepared by: Architecture Foundations Working Group_
