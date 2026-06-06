---
title: "WeaverTools — Current-State Snapshot (2026-05-14)"
date: 2026-05-14
draft: false
type: "report"
status: "published"
weight: 20260514
tags: ["weavertools", "ai-architecture", "agents", "technical-report", "rust", "memory-system"]
abstract: |
  A code-grounded snapshot of WeaverTools — a Rust-native harness running local
  language models as security-isolated agents backed by per-agent ArangoDB memory graphs and a
  shared GPU-resident inference server. Static-analysis review of crate structure, the agent binary
  surface, the tool registry, end-to-end paths, schema reality, and the gap between architectural
  claims and code at HEAD.
---

*Methodology:* Static analysis only — no tests run, no services hit. Every claim cites a file path. The intent is to ground the accompanying `weavertools-PRD.md` (delivered in the same PR as this snapshot) in code reality, not in PRD-claim reality.

## 0. TL;DR

WeaverTools is a Rust-native harness that runs local language models as security-isolated agents (one OS user per agent) backed by per-agent ArangoDB memory graphs and a shared GPU-resident inference server. At HEAD the system can provision OS users + databases, spawn agents driven by `agents/*.yaml`, and dispatch HeroBench benchmark tasks via `weaver agent start <task.yaml>`. The nap (context-pressure memory offload) path fires on KV budget exhaustion and writes summaries to `belief_nodes` with embed-on-write stamping (Pen + engine-nap both wired). The in-process Rust candle encoder (`EmbedderClient`) is the sole embedder backend; the Python gRPC service is retired. OpenInference JSONL traces are emitted per run with `llm.sampling`, `llm.weights_blake3`, `encoder.weights_blake3`, and `agent.prompt_stack` attributes. Stubs: `kind: chat` returns an error, `kind: chatroom` and `kind: training-loop` both hit `UnimplementedHandler`. The HNSW vector index on `belief_nodes.embedding` is defined in code but creation at collection-bootstrap time has not been verified by a passing integration test; the preseed materializer (batch embed-on-write at agent-load) is wired and tested. The decoder-side candle stack in `weaver-spu` is present but the live inference path in production still uses the llama.cpp GGUF backend; the pure-candle decoder is Phase 2 work.

## 1. Crate structure

**weaver-core** (`crates/weaver-core/src/`) — The harness kernel. Exports `Tool`/`Provider`/`Channel`/`Embedder` traits; the `QueryEngine` agentic loop with three-zone context budget manager; `AgentRuntime`/`AgentHandle`; tool registry and all built-in tool implementations (filesystem, shell, search, memory, Pen/Notepad); spec parsing (`AgentState`); OS/DB provisioning; GPU pool/claim/grant; embed-on-write helpers (`embed_write.rs`); BLAKE3 weights-hash module; autonomic writer and detector; reconciliation pipeline. 714 `#[test]` annotations across 54 files.

**weaver-spu** (`crates/weaver-spu/`) — Semantic Processing Unit. Renamed from `weaver-inference` in PR-0.5.C. Houses decoder side (GpuOrchestrator, GGUF backend, OpenAI-compatible HTTP server, model loaders — currently the live production path) and encoder side (Jina V4 candle forward: `lora_dispatch`, `pooling`, `matryoshka`, `qwen25vl_loraed`, `jina_v4`, `client::EmbedderClient` — all gated on `--features candle`, all landed in Phase 1 PRs). Legacy gRPC client (`grpc_client_legacy`) is retired per PR-1.J; the module comment in `encoder/mod.rs` confirms deletion. Feature flags: `gguf` pulls llama-cpp-2 for GGUF encoder (`llama_cpp_client`, `gguf_backend`); `candle` pulls the candle stack for Jina V4; `cuda` for CUDA kernels. 414 `#[test]` annotations across 29 files.

**weaver-database** (`crates/weaver-database/`) — ArangoDB client. Exports graph schema (`belief_nodes`, `episodic_memory`, `hanging_chad`, `meta_chad`, 24 edge collections, 6 named graphs), AQL query/crud/batch primitives, code-analysis ingestors (Rust via `syn`, Python via `rustpython-parser`), Persephone gRPC embedder client (transitional — embedder is now in-process). The constant `JINA_DIM = 2048` is hardcoded in `schema.rs:733` and used in `sleep_stage_a.rs` for zero-padding preseed summaries; this is a known deviation from the per-agent-configurability design. 252 `#[test]` annotations across 31 files.

**weaver-interface** (`crates/weaver-interface/`) — Binary crate producing `weaver`. Subcommand tree plus the Unix-socket daemon server (`server.rs`), protocol (`protocol.rs`), task YAML dispatcher (`kinds/`), `SO_PEERCRED` auth. Feature-gated: `inference` enables `weaver serve`; `embedder-rust` enables the `load_per_agent_embedder` candle path at daemon load time. 216 `#[test]` annotations across 18 files.

**weaver-trace** (`crates/weaver-trace/`) — OpenInference-shaped OTLP/JSON span emitter. `init()` installs a global `tracing_subscriber` layer; spans write to a line-delimited JSONL file. All attribute name constants are in `attributes.rs`. 16 `#[test]` annotations across 5 files.

**weaver-analysis** (`crates/weaver-analysis/`) — Offline Phase-1a attribution instrument. Consumes JSONL trace files; runs `H_t > μ_w + k·σ_w` decision-point detection (`descriptor`), matched-pair contrast analyzer with Mann-Whitney U + bootstrap CIs + Bonferroni (`analyzer`), verdict + power reports. No live harness dependency. 226 `#[test]` annotations across 16 files.

**weaver-demo** (`crates/weaver-demo/`) — Research and benchmark drivers. Binaries: `hanoi_bench`, `trajectory_runner`, `m6_validator`, `herobench_bench` (deprecated, retained). Most benchmark logic has migrated into `weaver-interface/kinds/herobench.rs` driven by `weaver agent start`. 414 `#[test]` annotations across 23 files.

## 2. Binary surface

The `weaver` binary (`crates/weaver-interface/src/main.rs`) exposes these subcommands:

**Working:**
- `weaver run <state_file.yaml>` — headless agent from YAML, stdio channel
- `weaver repl` — interactive REPL; resolves transport: socket → base_url → config → default socket
- `weaver agent create` — OS user + filesystem + optional ArangoDB provisioning
- `weaver agent list`, `show`, `load`, `unload`, `start`, `destroy`, `grant-db`, `revoke-db`
- `weaver agent start <task.yaml> [--agent <name>]` — dispatches to daemon; `kind: benchmark/suite: herobench` is the only fully working kind
- `weaver model load/unload/list` — diagnostic wrappers over the inference server REST API
- `weaver codebase ingest/stats` — ArangoDB codebase graph ingestion
- `weaver harness embedder show/reset` — cohort-pin inspection
- `weaver daemon` — Unix-socket daemon server at `/run/weaver/daemon.sock`
- `weaver trace view` — TUI viewer for JSONL trace files + joined memory state

**Stubbed / error-returning:**
- `weaver serve` — only compiled with `--features inference`; the systemd unit calls `/opt/weaver/bin/weaver serve --config /opt/weaver/server.toml`
- `kind: chat` — `ChatStubHandler` returns a hard error per `crates/weaver-interface/src/kinds/chat_stub.rs`
- `kind: chatroom`, `kind: training-loop` — `UnimplementedHandler`, hard errors

**Systemd surface:** Three units in `services/`: `weaver-infer.service` (inference, `Type=simple`, `User=weaver-admin`, `WorkingDirectory=/opt/weaver`, `PartOf=weaver.target`); `weaver-daemon.service` (daemon, `BindsTo=weaver-infer.service`, `PartOf=weaver.target`); `weaver.target` (synthetic harness target, `Wants=` + `After=` both services, added 2026-05-08 in PR-1.J alongside the Python-embedder retirement). No `weaver-embedder.service` present — the Python embedder service is retired. Operators start the chain with `systemctl start weaver.target`.

**Deprecated paths:** `cargo run -p weaver-demo --bin herobench_bench` is the old entry point; `CLAUDE.md` marks it deprecated. The binary is still compiled; `weaver agent start` is the canonical replacement.

## 3. Tool registry

All implementations are in `crates/weaver-core/src/tools/`. The `with_builtins()` factory (`mod.rs:134`) and `all_builtins()` (`mod.rs:173`) define the default registry. DB-dependent tools are gated on `ctx.db_pool.is_some()` and `ctx.codebase_pool.is_some()`.

**Filesystem:**
- `crates/weaver-core/src/tools/file_read.rs` — `FileReadTool`. In default registry: yes. Tests: yes (11 via `tool_tests.rs`). Reads files with offset/limit.
- `crates/weaver-core/src/tools/file_write.rs` — `FileWriteTool`. Default: yes. Tests: yes. Atomic write via `.tmp` + rename.
- `crates/weaver-core/src/tools/file_edit.rs` — `FileEditTool`. Default: yes. Tests: yes (11). Diff-based patch with similar crate.

**Shell:**
- `crates/weaver-core/src/tools/bash.rs` — `BashTool`. Default: yes. Tests: yes (23). Per-invocation safety classification (`invocation_properties` inspects the command string).

**Search:**
- `crates/weaver-core/src/tools/glob_tool.rs` — `GlobTool`. Default: yes. Tests: none in this file.
- `crates/weaver-core/src/tools/grep_tool.rs` — `GrepTool`. Default: yes. Tests: none in this file.

**Memory (db_pool required):**
- `crates/weaver-core/src/tools/memory/memory_read.rs` — `MemoryReadTool`. Default: conditional. Tests: yes (20). Reads from `belief_nodes`.
- `crates/weaver-core/src/tools/memory/memory_write.rs` — `MemoryWriteTool`. Default: conditional. Tests: yes (12). Writes to `belief_nodes`.

**Pen/Notepad (db_pool required):**
- `crates/weaver-core/src/tools/notes.rs` — `PenTool` + `NotepadTool`. Default: conditional. Tests: yes (41). Pen calls `try_embed_and_stamp_belief_node` from `embed_write.rs` before insert — embed-on-write is live for this tool.

**Codebase (codebase_pool required):**
- `crates/weaver-core/src/tools/memory/codebase_stats.rs` — `CodebaseStatsTool`. Conditional.
- `crates/weaver-core/src/tools/memory/codebase_symbol.rs` — `CodebaseSymbolTool`. Conditional.
- `crates/weaver-core/src/tools/memory/codebase_traverse.rs` — `CodebaseTraverseTool`. Conditional.
- `crates/weaver-core/src/tools/memory/codebase_search.rs` — `CodebaseSearchTool`. Conditional. Vector similarity search against `belief_nodes.embedding`.
- `crates/weaver-core/src/tools/memory/codebase_ingest.rs` — `CodebaseIngestTool`. Conditional.

**Bench-specific (registered by HeroBench kind handler directly, not via built-in registry):**
- `crates/weaver-demo/src/herobench/tools.rs` — `HeroBenchActTool`, `HeroBenchObserveTool`, `HeroBenchPlanTool`. 66 `#[test]` annotations.

`memory_format.rs` — shared formatting helper for memory tool outputs. Not a Tool impl; no registration.

## 4. End-to-end paths

**Can `weaver agent create <name>` provision a per-OS-user agent with DB?**
VERIFIED by code trace. `crates/weaver-interface/src/agent.rs` calls `provision_agent()` (OS user via `SudoProvisioner`) then `provision_hades_database()` from `weaver-core/src/spec/hades_provision.rs`. The `--no-hades` flag skips DB creation. Validation in `validate_agent_name()` (`provision.rs:46`) enforces lowercase alphanumeric + hyphens, start-with-letter, no trailing hyphen, not "admin".

**Can `weaver agent start <task.yaml> --agent <name>` dispatch a benchmark run?**
VERIFIED. The daemon server (`server.rs`) accepts `POST /v1/tasks`, resolves the agent from its registry, builds an `AgentHandle` vec, and calls `kinds::dispatch(spec)` → `HeroBenchHandler::run()` (`kinds/herobench.rs`). The `--agent` flag at the CLI level maps to `AgentAction::Start { task, agent }` in `agent.rs`.

**Does the nap actually fire on KV pressure during a run?**
VERIFIED by code. `QueryEngine::run()` (`engine/query.rs`) uses `ContextBudgetManager` (three zones: Green / Yellow / Red; defaults 60%/80% of context size). When zone enters Red or growth-rate threshold exceeded, it emits `QueryEvent::ContextNap`, calls `ContextNapCallback::save_summary()` → `HadesNapCallback::save_summary()` (implemented in `engine/runtime.rs`), which writes the nap document to `belief_nodes`. The `QueryEvent::NapComplete` is confirmed handled in `render_query_event()` in `main.rs`.

**Does the nap write to `belief_nodes` with embed-on-write?**
VERIFIED. `HadesNapCallback::save_summary()` in `crates/weaver-core/src/engine/runtime.rs` calls `try_embed_and_stamp_belief_node()` from `embed_write.rs` before the ArangoDB insert. The four fields (`embedding`, `embedding_model`, `embedding_dim`, `embedding_task`) are stamped if an embedder is present; the failure-mode escape (embedder absent or error → proceed without embedding, sleep stage A backfills) is implemented and unit-tested. Pen (`tools/notes.rs`) calls the same helper on every `Pen` write. Both producers are live at HEAD.

**Does the next session read accumulated memories on boot (preseed/prompt assembly)?**
PARTIAL. The preseed materializer (`crates/weaver-core/src/spec/preseed/`) produces recipe-book, world-knowledge, notes, and prompt-fragment documents and writes them to `belief_nodes` via `try_batch_embed_and_stamp_belief_nodes()`. The agent-load backfill (`embed_unembedded_belief_nodes()` in `embed_write.rs`) runs at daemon `load` time to embed any docs that landed without embeddings. The prompt-assembly side (reading memories into context on session boot) is referenced in the spec but the exact hook in `AgentRuntime::spawn*` was not directly traced to a tested integration path at HEAD. The preseed write-path is tested; the retrieval-into-system-prompt path is unverified here.

**Are OpenInference traces actually emitted with `llm.sampling`, `llm.weights_blake3`, `encoder.weights_blake3`?**
VERIFIED for attribute definitions. All three constants are declared in `crates/weaver-trace/src/attributes.rs` (`LLM_SAMPLING`, `LLM_WEIGHTS_BLAKE3`, `ENCODER_WEIGHTS_BLAKE3`). The `weights_hash.rs` module provides `blake3_of_file()` and `blake3_of_directory_manifest()` with sidecar-cache and tested. Direct emission call sites in `HeroBenchHandler` were not fully traced in this analysis — the attribute infrastructure is present and the spec is implemented, but call-site wiring in the benchmark loop is UNVERIFIED at this snapshot.

**Does the in-process embedder cutover (Phase 1) actually work in production agent runs?**
PARTIAL. The `load_per_agent_embedder()` function in `server.rs` dispatches on `encoder_model_id`: `qwen3-embedding-*` → `LlamaCppEmbedderClient::load()` (GGUF backend, feature-gated `gguf`); everything else (including Jina V4) → `EmbedderClient::from_snapshot()` (candle backend, feature-gated `embedder-rust`). Both paths are implemented. Production use depends on `--features embedder-rust` being set at build time and `[embedder].snapshot` being configured in `server.toml`. The Python gRPC client is confirmed retired in the module comments.

## 5. Schema reality

**`belief_nodes` collection — fields actually written:**

Verified from `embed_write.rs`, `tools/notes.rs`, `engine/runtime.rs`:
- `_key` — written by Pen and engine-nap producers
- `node_type` — written (e.g., `"note"`, `NOTE_NODE_TYPE`)
- `consolidation_status` — written (`"fresh"` on Pen writes per `notes.rs`)
- `topic`, `content` — written by Pen and engine-nap
- `created_at` — written
- `embedding` — written when embedder available (4-field stamp: `embedding`, `embedding_model`, `embedding_dim`, `embedding_task`)
- `embedding_model` — provenance: actual backend identifier from `EmbedResult.model`, not a hardcoded constant
- `embedding_dim`, `embedding_task` — written alongside `embedding`

**Fields defined but potentially unwritten in current code:**
- Several fields from the pre-unification schema (`summary`, `timestamp`) were renamed; docs that predate the migration carry the old names.

**HNSW index status:** The index definition exists as `BELIEF_EMBEDDING_INDEX_NAME = "embedding_hnsw"` in `crates/weaver-database/src/graph/belief.rs` with a `belief_embedding_index_for_dim(dimension)` constructor. Code comment in `belief.rs` states the index is created "at per-agent collection-bootstrap time (idempotent)." No passing integration test for index creation was found in this static analysis pass. The hardcoded `JINA_DIM = 2048` constant in `schema.rs:733` is used by `sleep_stage_a.rs` for zero-filling embeddings — this departs from the per-agent configurability contract.

**Trace span attributes actually emitted vs spec:**
Attribute constants in `attributes.rs` cover: standard OpenInference LLM span attributes, `llm.token_ids`, `llm.token_entropies`, `llm.prompt_blocks`, `llm.sampling`, `llm.sampling_version`, `llm.weights_blake3`, `encoder.weights_blake3`, `agent.prompt_stack`, staged sleep attributes (`stage_a.*` through `stage_d.*`), reconciliation attributes, decision-point attributes, database query/mutation attributes. The attribute namespace is fully spec-compliant. Emission at actual call sites in `kinds/herobench.rs` was not fully traced in this snapshot.

## 6. Architectural claims vs code reality

**"Only the harness hits the database; model emits signals"** — PARTIAL. True for nap writes: `HadesNapCallback` is harness-side. Pen and Notepad are still model-invocable tools (the agent calls them explicitly). The MEMORY.md note flags these as "transitional surfaces." The model-invokes-memory-tool path exists in the tool registry and is accessible to any agent that has `db_pool`.

**"Encoder + decoder co-resident on same GPU per agent"** — PROJECTED for the unified candle SPU. `load_per_agent_embedder()` reads `encoder_gpu` from `spu.encoder.gpu` in the agent YAML and constructs the encoder on that GPU device. The decoder is loaded via the inference server on its own GPU. Per-agent GPU separation (encoder on GPU1, decoder on GPU0) is configurable via YAML; the code supports it but the "split-GPU default" claim from MEMORY.md topology bench is configuration, not a hardcoded default.

**"Per-agent OS user isolation via runuser"** — VERIFIED. `provision_agent()` calls `useradd weaver-<name>`, home directory created under `/home/weaver-<name>`, agent state at `~/.config/weaver/agent.yaml`. The daemon reads agent files group-bit (0640 `weaver-<name>:weaver-admin`).

**"Nap fires on engine pressure; not model-invoked"** — VERIFIED. `ContextBudgetManager::observe()` is called by the query engine loop on every iteration. The nap trigger is purely engine-side; the model is not involved in the decision.

**"Embed-on-write is universal for `belief_nodes`"** — PARTIAL. Pen (live, tested) and engine-nap (live, tested) are wired. The preseed materializer batch path is wired (`try_batch_embed_and_stamp_belief_nodes`). The "sleep stage A retroactive backfill" path is described in code comments and handled by `embed_unembedded_belief_nodes()` but not confirmed as a scheduled, always-running job — it's called at agent-load time. The statement "universal" is aspirational; pre-spec docs in existing collections lack embeddings until backfill runs.

**"In-process embedder (Phase 1 cutover)"** — VERIFIED. `grpc_client_legacy` module is absent from `encoder/mod.rs` at HEAD (only referenced in historical comments as "retired in PR-1.J"). `EmbedderClient` (`encoder/client.rs`) is the production path under `--features candle` + `--features embedder-rust`.

**"Trace pipeline emits OpenInference spans"** — VERIFIED. `weaver-trace` crate initializes a `WeaverTraceLayer` writing OTLP/JSON. Constants follow the OpenInference spec. The `TraceGuard` flushes on drop.

**"Unix-socket-only for harness-internal traffic"** — VERIFIED. Daemon at `/run/weaver/daemon.sock`; inference server at `/run/weaver/inference.sock` (`DEFAULT_INFERENCE_SOCKET`). `OpenAiProvider::unix()` uses `hyperlocal` over `tokio::net::UnixStream`. External API access via TCP is the REPL fallback path only.

**"weaver-spu crate rename from weaver-inference"** — VERIFIED. `Cargo.toml` workspace member is `weaver-spu`. The `lib.rs` doc comment explicitly records the PR-0.5.C fold of `weaver-inference` content. `weaver-embedding` is also folded (PR-0.5.D mentioned in comments).

**"Hardcoded-instantiation absence (configurability)"** — PARTIAL. The `Embedder` trait and embedding field stamping correctly avoid hardcoding model identifiers (provenance comes from `EmbedResult.model`). However, `JINA_DIM = 2048` in `weaver-database/src/graph/schema.rs:733` is an instantiation constant used in `sleep_stage_a.rs` for zero-filling, which departs from the per-profile pluggability commitment.

**"Per-task specialists, agent-as-config"** — PARTIAL. The two-YAML design (agent identity vs. task work) is implemented. Multiple agent names can appear in task YAML via `participants:`. Only one benchmark kind (`herobench`) and one bench suite are fully wired.

**"Three-bucket doc convention (just landed)"** — VERIFIED at the filesystem level (`docs/prds/{implemented,in-progress,proposed}/`, `docs/specs/{implemented,in-progress,proposed}/`) but not enforced by any code constraint. It is a documentation convention, not a build/test gate.

**"Agent lifecycle: create/load/task/unload"** — VERIFIED. `AgentAction` enum in `agent.rs` has `Create`, `Load`, `Start` (task dispatch), `Unload`, `Destroy`. The daemon server maintains an agent registry with load/unload endpoints.

**"Same-GPU configurable, split-GPU default"** — PROJECTED as default. The code supports per-agent `spu.encoder.gpu` config; what the benchmark agent YAMLs actually set was not read in this analysis.

## 7. Deprecated but still present

- **`crates/weaver-demo/src/bin/herobench_bench.rs`** — deprecated per CLAUDE.md. Still compiled as the `herobench_bench` binary in `weaver-demo/Cargo.toml:57-58`. Recommendation: the binary entry point can be removed from `Cargo.toml`; the library modules it calls (`herobench/benchmark.rs` etc.) are still actively used by the `kinds/herobench.rs` handler.
- **`weaver-demo` trajectory_runner and m6_validator binaries** — retained research tooling, not deprecated per CLAUDE.md but not part of the primary operator workflow.

## 8. Stubs and unwired surfaces

- **`kind: chat`** — `ChatStubHandler` in `crates/weaver-interface/src/kinds/chat_stub.rs` returns an error on invocation. Parseable and dispatchable; not runnable.
- **`kind: chatroom`**, **`kind: training-loop`** — `UnimplementedHandler` with `anyhow::bail!`. Both confirmed in `kinds/mod.rs:109-114`.
- **Sleep stages B/C/D in the live harness** — implemented in `crates/weaver-demo/src/herobench/sleep_stage_{b,c,d}.rs` and tested via four integration tests (`tests/sleep_stage_{b,c,d}_live.rs` + `tests/sleep_cycle_tx_atomicity.rs`). Audited 2026-05-14: **not invoked from `kinds/herobench.rs`** (the live `weaver agent start <task.yaml>` HeroBench dispatch path has zero sleep-stage references) and **not invoked from `herobench_bench.rs`** (the deprecated binary also has zero references). The active consumer is `crates/weaver-demo/src/bin/trajectory_runner.rs` (562 lines), a separate binary invoked by `scripts/experiments/launch_trajectory_run.sh` *between* benchmark iterations to run pre-sleep + Stages A/B/C/D via `sleep_cycle::run_with_components`. Sleep is therefore a between-iteration apparatus, not an in-run one. Per the platform PRD's three-tier memory architecture §5, the in-run sleep cycle that consolidates Tier 1 → Tier 2 is Tier 3a/3b work; the existing demo-library stages are the closest in-repo proxy for that mechanism and likely the right starting point when Tier 3a wiring begins.
- ~~**`weaver.target` systemd unit** — referenced in `WantedBy=` of both service files but no `weaver.target` file is present in `services/`; operators must create it or it was installed separately.~~ **Resolved 2026-05-14 (audit error):** `services/weaver.target` does exist — it was added on 2026-05-08 (PR-1.J Python-embedder retirement) with proper `Wants=` + `After=` for both service units. The original snapshot's claim was a false negative.
- **`weaver-spu` decoder candle backend** — modules exist (`decoder/engine.rs`, `models/qwen2_5_vl.rs`) but the live production inference path still uses `llama_cpp_backend.rs` + GGUF. Phase 2 PRs are required for candle-backed autoregressive decoding.
- **`CodebaseSearch` vector similarity** — the tool exists and calls AQL COSINE_SIMILARITY, but this only works if the HNSW index is actually created at collection-bootstrap time. That bootstrap call was not traced to a verified code path in this snapshot.
- **`encoder/client.rs` (`EmbedderClient`)** — listed as "PR-1.F" in the module comment; the file exists at `crates/weaver-spu/src/encoder/client.rs` and is gated behind `#[cfg(feature = "candle")]`. Whether the feature is enabled in the production build depends on `Cargo.toml` feature flags not examined here.

## 9. Numbers

**Test annotations by crate (`#[test]`, proxy for size):**
- `weaver-core`: 714 across 54 files (largest by module count)
- `weaver-spu`: 414 across 29 files
- `weaver-demo`: 414 across 23 files
- `weaver-database`: 252 across 31 files
- `weaver-analysis`: 226 across 16 files
- `weaver-interface`: 216 across 18 files
- `weaver-trace`: 16 across 5 files
- **Workspace total: ~2252 test functions across 176 files**

**Agent YAMLs in `agents/`:** 25 files (20 `herobench-benchero-{0-19}`, plus `herobench-q3e-smoke`, `herobench-qwen3-embed-smoke`, `herobench-gemma-stack`, `herobench-cohort-arm-b`, `hb-qwen3-tense-probe`).

**Task YAMLs in `tasks/`:** 4 files (`herobench-port8000.yaml`, `herobench-port8001.yaml`, `herobench-port8001-preseed-smoke.yaml`, `herobench-postfix-port8001.yaml`).

**Systemd unit files in `services/`:** 3 (`weaver-daemon.service`, `weaver-infer.service`, `weaver.target`). No `weaver-embedder.service` (retired).

**Key hardcoded constant of concern:** `JINA_DIM = 2048` at `crates/weaver-database/src/graph/schema.rs:733`.

---

## Essential reading map for understanding HEAD

- `Cargo.toml` — workspace layout, crate rename from weaver-inference to weaver-spu confirmed
- `crates/weaver-core/src/lib.rs` — public API surface
- `crates/weaver-core/src/embed_write.rs` — embed-on-write contract, all producers
- `crates/weaver-core/src/embedder.rs` — `Embedder` trait
- `crates/weaver-core/src/engine/query.rs` — nap trigger, `ContextNapCallback`
- `crates/weaver-core/src/engine/runtime.rs` — `HadesNapCallback`, `AgentRuntime`
- `crates/weaver-core/src/engine/budget.rs` — three-zone context manager
- `crates/weaver-core/src/tools/mod.rs` — tool registry, `all_builtins()`
- `crates/weaver-core/src/tools/notes.rs` — Pen embed-on-write
- `crates/weaver-core/src/weights_hash.rs` — BLAKE3 sidecar cache
- `crates/weaver-interface/src/main.rs` — subcommand tree, daemon wiring
- `crates/weaver-interface/src/server.rs` — daemon server, `load_per_agent_embedder()`
- `crates/weaver-interface/src/kinds/mod.rs` — task dispatch, `UnimplementedHandler`
- `crates/weaver-interface/src/kinds/chat_stub.rs` — chat stub
- `crates/weaver-interface/src/agent.rs` — agent lifecycle subcommands
- `crates/weaver-spu/src/lib.rs` — SPU module map
- `crates/weaver-spu/src/encoder/mod.rs` — encoder feature gates, gRPC retirement
- `crates/weaver-database/src/graph/belief.rs` — `belief_nodes` field constants, HNSW index def
- `crates/weaver-database/src/graph/schema.rs` — `JINA_DIM = 2048` (hardcoded instantiation)
- `crates/weaver-trace/src/attributes.rs` — all OpenInference attribute constants
- `services/weaver-daemon.service`, `services/weaver-infer.service` — systemd surface
