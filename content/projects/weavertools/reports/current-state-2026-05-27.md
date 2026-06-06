---
title: "WeaverTools — Current-State Snapshot (2026-05-27)"
date: 2026-05-27
draft: false
type: "report"
status: "published"
weight: 20260527
tags: ["weavertools", "ai-architecture", "agents", "technical-report", "rust", "memory-system"]
abstract: |
  A static-analysis-plus-live-verification snapshot. The clone-on-create
  memory-substrate apparatus now runs end-to-end (a 4-member cohort completed 4 ok, 0 failed);
  getting there fixed a chain of substrate bugs and surfaced the runtime-events-versus-beliefs
  question that frames an imminent direction decision.
---

Predecessor: [`current-state-2026-05-24`]({{< relref "/projects/weavertools/reports/current-state-2026-05-24.md" >}}).

*Methodology — different from the prior two snapshots.* 05-14 and 05-24 were **static analysis only** ("no tests run, no services hit"). **This one is static analysis + live verification.** Over this session the daemon was rebuilt/reinstalled and restarted, a 4-fresh-agent cohort was run end-to-end, per-agent ArangoDB databases were queried directly, and the seed-deletion bug was isolated by controlled create→load→inspect. Claims tagged **[live]** were observed on the running system; **[code]** claims are file-grounded; **[proposed]** are PRD/decision items not yet built. This snapshot exists to ground an imminent **direction decision** — read §9 first if that's why you're here.

---

## 0. TL;DR

The clone-on-create memory-substrate apparatus now **runs end-to-end**: `weaver experiment cohort run` bakes/clones a vanilla belief graph into N identical fresh agents, runs a HeroBench task on each, and `cohort diff` reads per-member graph-formation readouts. A 4-member cartographer-style run completed **4 ok, 0 failed** [live]. Getting there fixed a chain of substrate bugs (#413/#420/#421/#422) and surfaced two findings that drive the pending decision:

1. **The per-agent encoder only loads if the daemon is built with `--features embedder-rust`.** The documented install recipe (`inference,viewer`) **omits it**, producing an encoder-less daemon that forms **no memory** (no embed-on-write possible). The installed binary now includes it [live]; the install recipe is a real gap to fix [code].
2. **The runtime autonomic writer writes events into `belief_nodes`** (`autonomic/writer.rs` → `BELIEF_NODES_COLLECTION`), deviating from the architecture's "beliefs mutate only at sleep" principle [live, code]. Runtime should produce *events*; sleep reconciles them into beliefs. This is the seam the next direction decision sits on.

Two new PRDs landed on `main` (proposed) framing a possible pivot in **how event-triggers are recognized** — measure the decoder's natural phrasing distribution rather than force a grammar (`event-trigger-phrasing-profile-PRD`), and develop that measurement into a standing evaluation tool (`phrasing-profile-eval-tool-PRD`).

---

## 1. What changed since 2026-05-24

Grouped by theme; all merged to `main`.

**Working-memory event substrate (§6.1):**
- `#410` register `engagement_edge` + `conversation_nodes` collections.
- `#411`/`#412` DB write API + write engagement edges (`conversation_node → belief_node`) at surfacing time.

**Associative surfacing:**
- `#407` deterministic structural-expansion axis (assoc-surfacing §10); `#408` resolve structural target from action-node markers; `#409` specific craft-target + objective injection.

**Memory-substrate images (clone-on-create):**
- `#405` `weaver harness image bake` verb; `#406` clone-on-create. Catalog at `/opt/weaver/images/`, keyed by `(domain, encoder, gnn, dim)`. Bake order: provision → materialize seed → embed → **create HNSW index last** → structural-embed → arangodump snapshot [code].

**Vector-index correctness:**
- `#413` build the belief vector index at the **encoder's real dim** (resolved from `embedder.info()`), not a hardcoded constant. The `JINA_DIM = 2048` hardcode flagged in the 05-14 snapshot is **gone** from `weaver-database` [verified — grep empty].

**Secrets:**
- `#415` agent `db_password` resolves from `~/.weaver_env` (minted at provision), not committed yaml; `KnowledgeState::resolved_db_password(name)`.

**Cohort experiment tool:**
- `#414` experiment spec + `graph_formation_readout` (T3); `#416/#417/#418` run/diff/list/show/teardown; `#419` restore after a history rewrite. `weaver experiment cohort …`; registry at `/opt/weaver/cohorts/`.

**Embed-before-insert chain (this session, the substrate-correctness arc):**
- `#420` defer per-agent prompt materialization to **load** and embed-before-insert (the non-sparse vector index rejects null-embedding inserts).
- `#421` spawn bootstraps the system prompt from `prompts_dir` files on first load (sequencing fix) + cohort tolerates already-loaded member + diff reads member DB with member credentials.
- `#422` use per-key `upsert` not import `overwrite=true` (which **truncates** the collection — the seed-killer).

**Docs:**
- `event-trigger-phrasing-profile-PRD` + `phrasing-profile-eval-tool-PRD` filed (proposed, on `main`).

## 2. Crate structure (8 crates)

Unchanged from 05-24 except as noted. `weaver-core`, `weaver-interface` (binary `weaver`), `weaver-database`, `weaver-spu`, `weaver-trace`, `weaver-analysis`, `weaver-demo`, **`weaver-experimental`**.

`weaver-experimental` houses the swappable-hypothesis impls: the **memory-peer GNN** (`memory_peer/` — `mean-pool-v0`, `relational-pool-v0`, both *deterministic, untrained* aggregators) and the **sleep** stages (`sleep/` — `FreudianSleepV1`, presleep + Stage A/B/C/D, `live_contexts`). The canonical `weaver-spu/src/structural/` home for the GNN (sprint task #219) is **not built**; the GNN lives in `weaver-experimental` as the v0 hypothesis [code].

## 3. The substrate facts established live this session (load-bearing)

These are new, verified, and bear directly on the decision:

- **The `belief_nodes` HNSW vector index is non-sparse and total** [live]. It rejects any insert whose `embedding` is missing/null (`ArangoError 10: Expecting type Array`); `sparse:true` does **not** relax it; index *creation* also requires every doc to carry a valid embedding. Consequence: the "land with `embedding: null`, backfill later" escape (`embed_write.rs`, `belief-nodes-embedding-Spec`) is **only valid before the index exists**. Once present (clone-on-create, or post-bench-run index), embed-before-insert is mandatory. Documented in `belief-nodes-embedding-Spec §4.1.3`.
- **Per-agent encoder load is gated on `--features embedder-rust`** [live]. Without it `load_per_agent_embedder` compiles to a no-op → `handle.embedder() == None` → prompt materialization, semantic backfill, and structural-embed all skip → **no runtime memory forms**. `inference = ["weaver-spu/cuda","weaver-spu/gguf"]` does **not** include `embedder-rust`; `scripts/install.sh`'s die-message recipe is `inference,viewer` — i.e. the documented build produces an encoder-less daemon. **Gap to fix** (have `inference` imply `embedder-rust`, or correct the recipe + docs).
- **`insert_documents(overwrite=true)` truncates the whole collection** (ArangoDB import semantics), not per-key replace [live]. This was the seed-deletion bug (load wiped all 627 cloned `world_graph` beliefs, left 6 prompts). Fixed in #422 via `upsert_documents` (`onDuplicate=replace`). Post-fix verified: a fresh agent loads to **633 = 627 seed + 6 prompt** [live].
- **Sleep reconciles from `episodic_memory`** (`sleep/presleep.rs:126 FOR e IN episodic_memory`), plus `hanging_chad`/`meta_chad`. It does **not** read `conversation_nodes`. And `episodic_memory` is **not written at runtime today** (a cohort member showed `episodic_memory=0` after a run) — so the runtime→event-store→sleep pipeline is currently **unfed** [live, code].
- **The autonomic writer writes to `belief_nodes`** (`autonomic/writer.rs` uses `memory_write(BELIEF_NODES_COLLECTION, …)`). In a live run a member's `belief_nodes` accumulated `decision`/`observation` note-shaped docs (27 + 5) while the working-memory path correctly wrote `conversation_nodes` (42) + `engagement_edge` (78) [live]. This is the architectural deviation §9 addresses.

## 4. Apparatus status — cohort + images (VERIFIED live)

- `weaver harness image bake --domain --seed --encoder --gnn` produces a vanilla graph image (627 belief_nodes + edges + embeddings + HNSW index + structural embeddings), manifest at `/opt/weaver/images/<key>/manifest.json` [live].
- `weaver experiment cohort run --agent-yaml --task --members N` clones the resolved image into N fresh agents, runs the task on each serially, retains all [live, 4 ok].
- `weaver experiment cohort diff <name>` reads each member's `graph_formation_readout` (seed vs formed partition) — **requires the member's own DB credentials** (fixed #421); reads cleanly now (`seed_node_count: 627`) [live].
- **Bake idempotency caveat:** a *failed* bake leaves a partial indexed image DB that breaks subsequent re-bakes (the same null-insert-into-indexed-collection failure). Workaround applied this session: drop the stale image DB before re-bake. Not yet hardened into the bake verb [live].

## 5. GNN reality

- Registered impls: `mean-pool-v0`, `relational-pool-v0` (`weaver-experimental/src/memory_peer/registry.rs`) — deterministic, **untrained** aggregators [code].
- `herobench-fullmem.yaml` binds `relational-pool-v0`; bake + load run a structural-embed pass producing `structural_embedding` on belief_nodes [live — image had 627 structural-embedded].
- **No trained GraphSAGE**; `#219` (the `weaver-spu/src/structural/` scaffolding) and `#225` (training) are pending. Engagement-contrastive *training* is density-gated (threshold ~50); the deterministic typed-edge walk + structural pool are the pre-GNN path and work now.

## 6. Schema reality

- `belief_nodes`: 4-field embedding stamp (`embedding`/`embedding_model`/`embedding_dim`/`embedding_task`); encoder is `qwen3-embedding-0.6b-gguf-q8`, **dim 1024** (the 2048 hardcode is resolved). HNSW index `embedding_hnsw` created at the encoder's real dim — at bench-run (`ensure_belief_vector_index`, #413) and baked into images.
- Event collections exist: `conversation_nodes` (working-memory §6.1; the proposed `today_` rename target), `engagement_edge` (`conversation_node → belief_node`), `episodic_memory` (sleep's event source), `hanging_chad`/`meta_chad` (curiosity/contradiction).
- `prompt`-kind belief_nodes are legitimate constitutive memory (materialized at load); runtime `observation`/`decision` belief_nodes are **not** legitimate (the §9 issue).

## 7. Decision context — the two new PRDs

- **`event-trigger-phrasing-profile-PRD`** (methodology, proposed, may still shift): measure the decoder's *natural* event-signaling phrasing distribution, frequency-gate a recognizer, score per-(decoder, kind) **coverage × collision** against a closed-world oracle — before forcing a grammar or SFT. Borrows the templating premise from `frequency-gated-recall-PRD`.
- **`phrasing-profile-eval-tool-PRD`**: a standalone CLI that runs the methodology on arbitrary HuggingFace models (download → per-model adapter → bundled ground-truth task → per-(model,kind) report), publishing a HuggingFace dataset + writeup.

## 8. Numbers

- Crates: 8. Agent YAMLs in `agents/`: 39. Task YAMLs in `tasks/`: 26. `services/`: `weaver-daemon.service`, `weaver-infer.service`, `weaver.target` (+ `report-charts/`). No `weaver-embedder.service` (in-process embedder).
- Installed binary `/opt/weaver/bin/weaver` built `--features inference,viewer,embedder-rust` this session (the embedder-rust addition is the live fix; **not yet reflected in `install.sh`'s recipe**).
- Decoder GGUFs on disk include a size ladder: `qwen2.5-0.5b`, `qwen2.5-1.5b`, `gemma-3-1b` (small) + `gemma-4-26B-A4B`, `qwen3-coder-30B-A3B`, `gemma4-31b` (large). Mid sweet-spot (Qwen3-4B/Gemma-3-4B) not local.

## 9. Open decisions (why this snapshot exists)

The pending direction call centers on **runtime events vs beliefs**, and possibly a **methodology pivot**. The decisions, in dependency order:

1. **`today_` event container + stop writing beliefs at runtime.** Rename `conversation_nodes` → `today_` (one container for all between-sleep events), redirect the autonomic writer there with an `event_type` discriminator, keep `engagement_edge: today_node → belief`. Beliefs mutate only at sleep. (Task #303.) **Prerequisite plumbing — likely lands regardless of the methodology pivot.**
2. **How event-triggers are recognized** (the possible pivot): force a structured grammar (`OBSERVE:`) + SFT, **or** measure the natural phrasing distribution and frequency-gate a recognizer (the phrasing-profile PRD). Measurement-first is the stated lean.
3. **Smaller decoders.** Strong rationale (latency, faster iteration, *more* templated → easier recognition). Gated by the instruction-following floor; the cohort/image apparatus already supports decoder-size sweeps against a shared image (decoder is yaml, not in the image key).
4. **Sleep cycle as first real test.** Feed `episodic_memory`/`today_` from runtime; a cartographer corpus (verified observations vs the map) becomes sleep's first reconciliation test case.
5. **Eval-tool build:** standalone binary vs `weaver experiment` verb, and which ground-truth task to bundle.

## Essential reading map (delta from 2026-05-24)

- `docs/prds/proposed/event-trigger-phrasing-profile-PRD.md`, `phrasing-profile-eval-tool-PRD.md` — the pending-direction PRDs.
- `docs/specs/in-progress/belief-nodes-embedding-Spec.md` §4.1.3 — embed-before-insert under a non-sparse index (the substrate constraint).
- `crates/weaver-core/src/embed_write.rs` — `materialize_embedded_belief_nodes` (embed-before-insert, upsert), `embed_unembedded_belief_nodes` (load backfill).
- `crates/weaver-core/src/autonomic/writer.rs` — the runtime→`belief_nodes` write (the §9.1 issue).
- `crates/weaver-interface/src/server.rs` — `load_per_agent_embedder` (the `embedder-rust` gate), load-time prompt materialization + spawn bootstrap.
- `crates/weaver-interface/src/cohort.rs` — cohort run/diff; `member_read_pool` (member-cred DB read).
- `crates/weaver-core/src/spec/image_bake.rs` — bake order (embed → index last) + `clone_image_into_db`.
- `crates/weaver-experimental/src/{memory_peer,sleep}/` — untrained GNN aggregators + Freudian sleep stages.
- `crates/weaver-experimental/src/sleep/presleep.rs` — sleep's event source (`episodic_memory`).
- `crates/weaver-database/src/graph/schema.rs` — `CONVERSATION_NODES_COLLECTION` (the `today_` rename target), engagement-edge endpoints.
- `scripts/install.sh` — the `inference,viewer` recipe gap (missing `embedder-rust`).
