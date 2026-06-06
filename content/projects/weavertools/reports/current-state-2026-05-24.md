---
title: "WeaverTools — Current-State Snapshot (2026-05-24)"
date: 2026-05-24
draft: false
type: "report"
status: "published"
weight: 20260524
tags: ["weavertools", "ai-architecture", "agents", "technical-report", "rust", "memory-system"]
abstract: |
  Experiment-readiness snapshot following the 2026-05-14 review. The
  attribution instrument is code-complete and tested; the remaining gap to a Phase-1a verdict is
  largely operational. Reviews the changes since that review, the attribution gate, apparatus status, and the
  one load-bearing apparatus risk: cold-start vector retrieval over belief_nodes.
---

Predecessor snapshot: [`current-state-2026-05-14`]({{< relref "/projects/weavertools/reports/current-state-2026-05-14.md" >}}).

*Methodology:* Static analysis + targeted `grep`/file reads; two code-grounded subagent passes (apparatus + attribution instrument). **No tests run, no services hit, no sweep executed.** Every claim cites a file path or PR. This snapshot exists to refresh the `weavertools-PRD.md` anchor, whose "verified commitments" table is now significantly out of date, and to answer one question: *how far are we from running proper experiments?*

**Scope caveat — the memory/GNN cluster is a moving target.** A parallel session is actively pushing on the memory system; the inductive GNN build-out (GraphSAGE-class `graphsage-v1`, replacing the `mean-pool-v0` scaffold) begins ~2026-05-25. Status claims about surfacing / GNN / working-memory / engagement-edge code are bracketed as **in-flux** throughout and should not be treated as stable.

## 0. TL;DR — experiment readiness

The headline: **we are much closer to a Phase 1a attribution verdict run than the stale PRD suggests, and the remaining gap is mostly operational, not engineering.** The attribution instrument is code-complete and tested; all four Phase-1a PRs (#162/#163/#164/#172 + driver follow-ons) are merged; per-token entropy capture feeding it is wired. The distance to a verdict is: de-risk vector retrieval, confirm prompt-block eligibility, add live-server integration tests, then define + run the matched-pair sweep and invoke the driver.

The one real apparatus *risk* on that path: **vector retrieval over `belief_nodes` is conditionally operational, not robust** — the cosine index is created on the image-bake and HeroBench paths but not in the general `weaver agent create`/load lifecycle, ArangoDB 3.12.4's Faiss-IVF rule forbids index creation until ≥`n_lists` (64) docs exist, and the `belief_nodes` `APPROX_NEAR_COSINE` retrieval path has no linear-scan fallback. This produces a cold-start window where "memory-on" turns can be silently non-retrieving — directly threatening the integrity of the memory-on vs memory-off contrast the experiment measures.

## 1. What changed since 2026-05-14

**Tier 0 — fully closed.** All five PRD Tier-0 items resolved (#348–#351): `weaver.target` confirmed present, emit-side roundtrip test confirmed extant (CI gate deferred — no CI infra), sleep-stage B/C/D dispatch audited, JINA_DIM relocated to Tier 1 with a proper spec.

**Embedder-model selection cleanup — landed** (#353). `weaver-spu/.../encoder/registry.rs` exposes `SUPPORTED_EMBEDDERS` (jina-v4-fp16-gguf, qwen3-embedding-0.6b-gguf-q8) with `lookup`/`supported_model_ids`; `embed_write.rs` sources dim + model from `EmbedResult` (no write-path `JINA_DIM` hardcode). Residual `2048` occurrences are test fixtures, SPU catalog metadata, and a `.unwrap_or(2048)` fallback — not the old instantiation constant.

**Nap/Sleep algorithm trait extraction — landed** (#361–#380). `NapAlgorithm`/`SleepAlgorithm` substrate traits + cycle-history collections; `DefaultNapV1` and `FreudianSleepV1` lifted into `weaver-experimental`; yaml-driven resolution at agent-load (`sleep.algorithm:`), admin verb, `PassthroughNap`/`passthrough-sleep-v1` ablation no-ops. `fire_nap_flow` now dispatches through the trait. This is substantial apparatus the prior snapshot did not have.

**Embed-on-write universality — advanced** (#390/#391, closing #150/#151). `weaver agent backfill-embeddings` admin verb (`agent.rs:~1512`); Stage A `retroactive_embed_pass` (`sleep/stage_a.rs:~241`) shares the embed path. Backfill is now an available operation, not agent-load-only.

**Prompt-assembly read path — landed.** `runtime.rs:~1620` `load_system_prompt` queries `belief_nodes` for `provenance.kind=='prompt' AND provenance.source=='preseed'`, sorts by topic, joins as highest-priority prompt source with graceful fallback. The PRD/2026-05-14 "PARTIAL — unverified" status is closed.

**Vector-index error 1554 — fixed, but see §5.** #396 reordered the AQL so `APPROX_NEAR_COSINE/SORT/LIMIT` precede provenance filters (regression-guarded); #389/#153 dropped the phantom InnerProduct metric and surfaced the IVF training precondition.

**Surfacing reader — landed (in-flux).** R1–R5 (#392–#395): harness-mediated memory-push reader wired through agent yaml, per-turn `agent.surfacing.*` telemetry, skip-under-Red-pressure. Deterministic structural-expansion axis + target resolution (#407/#408/#409). *In-flux — owned by parallel session.*

**Memory-peer GNN — scaffold landed (in-flux).** `GNNInference` substrate + `mean-pool-v0` (#398/#219), inference pipeline + full-memory run (#399/#220). `engagement_edge` + `conversation_nodes` schema registered (#410, working-memory §6.1) — prerequisite for the engagement-contrastive training objective. The *proper inductive GNN* is the imminent build-out. *In-flux.*

**Read-side daemon + SSE — landed.** `nap.history`/`sleep.history`/`memory.query`/`memory.graph.neighbors` endpoints (#381–#384); `SpanHub` broadcast + `trace.subscribe`/`signals.subscribe` SSE (#385–#388). Apparatus for the GUI/readside track.

**Memory-substrate images — landed** (#402–#406). `belief_graph` named-graph helper + structural-pass self-heal, catalog/manifest module, `weaver harness image bake` verb, clone-on-create. *(This is also where the `belief_nodes` cosine index gets reliably created — see §5.)*

**Docs/program reframing.** HAH established as the founding hypothesis; four-property memory commitment codified (#364); NL paper schema relocation (#356).

## 2. Crate structure (8 crates)

Unchanged in shape from 2026-05-14; `weaver-spu` and `weaver-experimental` are the two beyond CLAUDE.md's stale "7 crates" list. `weaver-experimental` now houses the lifted sleep cycle (`sleep/`), nap (`nap.rs`), and memory-peer impls (`memory_peer/` — `mean-pool-v0`, `relational-pool-v0` + registry). Workspace `#[test]`-bearing files: 212.

## 3. Attribution instrument — the experiment gate (READY, modulo a run)

**The instrument is code-complete and tested; the gap to a verdict is operational.** Verified in `crates/weaver-analysis/`:

- **Decision-point detection** (`descriptor.rs`): rolling-window z-score `H_t > μ_w + k·σ_w` (window 32, k 2.0), spike-recovery tested.
- **Matched-pair analyzer** (`analyzer.rs`): Mann-Whitney U (tie-corrected, continuity-corrected) + seed-locked bootstrap 95% CI on median diff + Bonferroni over a locked `Family`; byte-deterministic.
- **Verdict + power** (`verdict.rs`, `power.rs`): per-contrast `SignalPresent/Absent/Inconclusive/TuningFailure` via the 3-conjunct rule.
- **End-to-end driver** (`bin/weaver-attribute.rs`): experiment.yaml → descriptor → analyzer → verdicts → `contrast_report.json` + `verdicts.json` + `verdict.md`; refuse-to-overwrite, exit-code matrix, pinnable timestamp.
- **Trace capture feeding it is real and wired:** per-token Shannon entropy in the decoder (`weaver-spu/.../gguf.rs` `shannon_entropy_bits`, pre-sampler logits) → recorded on the LLM span at `engine/query.rs:~1985`. `memory_bypass` session flag exists (`config.rs:~74`, default-false). HeroBench `ConditionFlags` is the apparatus.

**Phase-1a "4 PRs" — all merged:** PR-Att-1a-1 (#162, entropy + memory-bypass), 1a-2/2b/2c (#164/#170/#171, addressable prompt blocks), 1a-3 (#163, analysis crate), 1a-4 (#172, power + verdict + condition tags); plus driver track PR-Att-D-1..4 (#198/#201) and PR-Ext-4 (#193).

**ΔH_residual replay-ablation is specified-only, by design** — deferred to Phase 1b, gated on Phase 1a finding signal. No replay/ablation/counterfactual mechanism, no `REPLAY` span kind, no ΔH_residual computation in code. Feeder specs `residual-capture-Spec` + `rich-metrics-capture-Spec` are both "begins after benchero-4/5." So the paper's *quantitative core* is downstream of the Phase 1a verdict, not part of it.

**Gap to a verdict run:** blocker is **(d) running it + producing the trace bundle**, not instrument code or core trace capture. No committed `experiment.yaml` and no `docs/experiments/attribution-phase1a-<date>/` artifact dir exist. One unverified eligibility item: `prompt_blocks` must be well-partitioned for the target chat templates or runs fall back to "not Phase-1a-eligible" (plumbing exists; real-run partitioning unconfirmed from code alone).

## 4. Apparatus status (Tier 1) — code-grounded

| Item | Status | Evidence |
|---|---|---|
| Attribution instrument | **Done, tested** | §3 |
| Per-token entropy capture | **Done, wired** | `gguf.rs` → `query.rs:~1985` |
| Embed-on-write + backfill (#150/#151) | **Landed** | `embed_write.rs`; `backfill-embeddings`; Stage A retro-embed |
| Prompt-assembly read path | **Landed** | `runtime.rs:~1620` `load_system_prompt` |
| Embedder-model cleanup (#353) | **Landed** | `encoder/registry.rs` `SUPPORTED_EMBEDDERS` |
| Vector retrieval over `belief_nodes` | **Conditional / fragile** | §5 — the load-bearing risk |
| Trace emit-site coverage | **Partial** | live `agent.surfacing.*` + herobench attrs emit; 5 forward-declared namespaces (`surprise.*`, `memory.query.*`, `belief.*`, `structural.*`, `decision.*`, `stage_a.decisions`) declared-not-emitted (intentional, PR-C #360) |
| Nap/Sleep trait extraction | **Landed** | #361–#380 |
| Surfacing reader R1–R5 + structural | **Landed (in-flux)** | #392–#395, #407–#409 |
| Memory-peer GNN | **Scaffold landed (in-flux)** | `mean-pool-v0` #398/#399; inductive GNN imminent |

## 5. Schema reality — the vector-retrieval cold-start

`belief_nodes.embedding` cosine index is `embedding_hnsw` (`graph/belief.rs:228`, `belief_embedding_index_for_dim`, `n_lists = 64`). `create_vector_index` (`db/index.rs:121`) is invoked in exactly two production sites:

1. **`weaver-core/src/spec/image_bake.rs:179`** — the image-bake path (reliable; index exists on baked DBs).
2. **`weaver-demo/src/herobench/belief.rs:1007`** — best-effort on the HeroBench path; **degrades to linear-scan dedup on failure** and explicitly warns. The IVF precondition (≥64 docs at creation time) means a fresh empty `belief_nodes` *cannot* get the index; the code comment names `backfill-embeddings` (#151) as the intended re-attempt site once the collection is populated.

**There is no `create_vector_index` call in the general `weaver agent create`/load provisioning path**, and the `belief_nodes` `APPROX_NEAR_COSINE` retrieval path (`graph/memory_query.rs:122`, used by the live surfacing reader and the daemon `memory.query` endpoint) has **no linear-scan fallback** — unlike the codebase/arxiv chunk path in `db/vector.rs`. Consequence:

- **Image-baked agent** → retrieval works.
- **HeroBench run** → retrieval comes online once docs accumulate and index (re-)creation succeeds; **cold-start window before that errors 1554 with no graceful degradation** on the retrieval path.
- **Fresh ordinary agent (no bake)** → has embeddings, no index → surfacing/memory.query 1554s.

This is the single finding most worth confirming empirically before trusting a verdict, because it can silently invalidate the "memory-on" arm.

## 6. Roadmap to "proper experiments" (ordered, no calendar)

**Gate A — Phase 1a attribution verdict (the near experiment):**
1. **De-risk `belief_nodes` vector retrieval** — add a linear-scan fallback on the retrieval path, or guarantee the index exists before scored turns (pre-seed ≥64 docs / create-at-load). *Only real engineering item on the critical path.*
2. **Confirm `prompt_blocks` partitioning eligibility** for the target chat templates.
3. **Add live integration tests against a real ArangoDB 3.12.4 server** — embed-on-write + retrieval are unit-tested only; that's exactly how 1554 slipped.
4. **Operator: define `experiment.yaml`, run the matched-pair sweep** (tasks × {memory_on/off, harness_full/minimal} × N on Qwen, then Gemma replication) → invoke `weaver-attribute` → `verdict.md` + artifact dir.

**Gate B — Phase 1b (ΔH_residual replay-ablation):** deliberately downstream of Gate A's finding signal; `residual-capture` + `rich-metrics-capture` begin after benchero-4/5.

**Parallel / non-gating:** Tier 2 (GUI #352, SPU API, GGUF cleanup), memory-substrate images, and the inductive-GNN / working-memory / engagement-edge track owned by the parallel session.

## 7. Carryover stubs (unchanged)

`kind: chat` (`ChatStubHandler`, error), `kind: chatroom` + `kind: training-loop` (`UnimplementedHandler`); `weaver-spu` candle decoder present but production stays on GGUF; `herobench_bench` binary deprecated but library modules live. Sleep stages B/C/D remain a between-iteration apparatus (trajectory_runner), not in-run.

## 8. Numbers

- 8 workspace crates; 212 `#[test]`-bearing files.
- Agent YAMLs in `agents/`: 39. Task YAMLs in `tasks/`: 20.
- Docs: PRDs 6 in-progress / 14 proposed; specs 18 in-progress / 33 proposed.

---

## Essential reading map (delta from 2026-05-14)

- `crates/weaver-analysis/` + `bin/weaver-attribute.rs` — the attribution instrument (the experiment gate)
- `crates/weaver-database/src/graph/memory_query.rs` — `APPROX_NEAR_COSINE` retrieval path (no fallback)
- `crates/weaver-database/src/graph/belief.rs` + `db/index.rs` — index def + creation
- `crates/weaver-core/src/spec/image_bake.rs` — where the cosine index reliably gets created
- `crates/weaver-experimental/src/{nap.rs,sleep/,memory_peer/}` — lifted nap/sleep + GNN scaffold
- `crates/weaver-spu/src/encoder/registry.rs` — `SUPPORTED_EMBEDDERS` (no-default-embedder)
- `crates/weaver-core/src/engine/runtime.rs:~1620` — `load_system_prompt` read path
- `docs/specs/proposed/attribution-instrument-phase-1.md` — Phase 1a/1b boundary + 4-PR log
