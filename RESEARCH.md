# Cross-Tool Synergy Research

**Branch:** `research/cross-tool-synergy`
**Date:** 2026-04-09
**Status:** exploration

---

## Context

Five tools that independently cover different phases of AI-agent-assisted development. This document captures the exploration of how they interact, where the gaps are, and what synergies exist.

| Tool | Repo | Language | LOC | Phase |
|------|------|----------|-----|-------|
| **graphify** | safishamsi/graphify | Python | ~7.3K | Understanding (code → knowledge graph) |
| **beads_rust (br)** | local | Rust | ~75K | Planning (dependency-aware issue tracker) |
| **beads_viewer_rust (bvr)** | local | Rust | ~219K | Prioritization (graph-aware triage engine) |
| **fast_cmaes** | Dicklesworthstone/fast_cmaes | Rust+PyO3 | — | Optimization (derivative-free parameter tuning) |
| **cass_memory_system (cm)** | Dicklesworthstone/cass_memory_system | TypeScript/Bun | — | Learning (procedural memory with confidence decay) |

---

## Tool Profiles

### graphify

Multimodal knowledge graph builder. Two-pass extraction: deterministic tree-sitter AST (20 languages, 0 tokens) + Claude subagents for docs/images/papers. Leiden community detection. Interactive HTML visualization (vis.js). SHA256 cache for incremental rebuilds. Integrates as `/graphify` skill across 7 platforms (Claude Code, Codex, OpenCode, OpenClaw, Factory Droid, Trae, Trae CN).

**Core pipeline:**
```
detect → extract (AST + LLM) → build (NetworkX) → cluster (Leiden) → analyze → report + export
```

**Key outputs:** graph.json, graph.html, GRAPH_REPORT.md, Obsidian wiki, cache/

### beads_rust (br)

Local-first, dependency-aware task tracker. SQLite primary storage + JSONL for git-friendly collaboration. Non-invasive (zero automatic git operations). 35+ CLI subcommands. Agent-first: every command supports `--json`. Hash-based short IDs (e.g., `bd-7f3a2c`). Append-only event audit log.

**Key data:** Issues (status, priority P0-P4, type, labels) + Dependencies (blocks, parent-child, conditional-blocks, waits-for, related)

### beads_viewer_rust (bvr)

Read-only analytics layer over br data. Builds in-memory petgraph DiGraph. 9 graph algorithms. 8-component transparent ImpactScore with weight presets. 39 `--robot-*` commands. 12-mode interactive TUI. Static HTML+SQLite dashboard export. TOON format output (60% smaller than JSON).

**Algorithms:** PageRank, Betweenness, Eigenvector, HITS, Kosaraju SCC, Critical Path, K-Core, Articulation Points, Slack

**Scoring:**
```
pagerank(0.22) + betweenness(0.20) + blocker_ratio(0.13) + 
time_to_impact(0.10) + priority(0.10) + urgency(0.10) + 
risk(0.10) + staleness(0.05) = 1.0
```

### fast_cmaes

Rust CMA-ES with Python bindings. Derivative-free optimizer for continuous parameters in black-box/noisy/non-differentiable objectives. SIMD (AVX 3-4x), Rayon parallelism, lazy eigen decomposition (5-10x fewer O(n³) calls). Ask-tell pattern. Box constraints + mirroring + rejection + repair + penalty pipeline.

### cass_memory_system (cm)

Three-layer cognitive architecture for AI agent memory:
- **Episodic:** raw session logs (JSONL)
- **Working:** diary summaries
- **Procedural:** playbook bullets with scored rules

90-day confidence half-life. 4x harmful multiplier. Maturity progression: candidate → established → proven. Anti-pattern inversion. Cross-agent knowledge transfer (Claude Code ↔ Cursor ↔ Codex).

---

## The Workflow Cycle

```
Understand → Plan → Prioritize → Execute → Learn
 graphify      br       bvr       agents    CASS
                                              │
              ◄────────── feedback loop ◄─────┘
```

Each tool covers one phase. The gaps are in the transitions.

---

## Identified Gaps

### Gap 1: Understanding ↛ Planning

graphify builds a knowledge graph of the codebase. br has issues. There is no link between them. When an agent creates "refactor auth module" — it doesn't know auth is a god node with 23 connections across 4 communities. The information exists, but in a different tool.

### Gap 2: Prioritization ↛ Architectural Context

bvr recommends "do bd-42 first" based on task graph metrics. But bvr doesn't know that bd-42 touches a module with lowest cohesion and 6 ambiguous edges — a risky zone in the knowledge graph. Architectural risk and task priority live in separate worlds.

### Gap 3: Execution ↛ Learning (closed by CASS)

Agent closes an issue. Code changed. But without CASS:
- graphify graph is stale (manual re-run needed)
- bvr weights are never calibrated against real outcomes
- No one knows if bvr's recommendation was good
- Next agent starts from zero

CASS closes this: outcomes → playbook bullets → context retrieval → better decisions.

---

## CASS as the Feedback Backbone

CASS doesn't just close Gap 3 — it feeds back into every phase:

| CASS remembers... | Feeds back to... |
|---|---|
| "auth module always breaks during refactor" | graphify: raise risk score for auth community |
| "bvr recommended bd-42 first, but it took 3x longer" | fast_cmaes: fitness data for weight calibration |
| "cross-file changes in cluster.py caused regressions" | bvr: additional risk signal during triage |
| "Claude handles Rust better, Codex handles Python" | planning: agent routing by language |

### CASS → fast_cmaes → bvr loop

```
CASS outcomes (success/harmful marks)
        ↓
fitness function: do bvr recommendations match actual outcomes?
        ↓
fast_cmaes tunes bvr weights (8 continuous params, sum=1)
        ↓
bvr gives better recommendations
        ↓
CASS records new outcomes
        ↓
cycle closes
```

This is the most concrete integration point: CASS provides the historical data that fast_cmaes needs as a fitness function to optimize bvr's ImpactScore weights.

---

## Synergy Directions

### Direction A: Enrich existing tools independently

Each tool improves on its own, no new tool needed.

- graphify gets algorithms from bvr (PageRank, SCC, critical path via NetworkX)
- bvr gets architectural context from graphify (god nodes, cohesion, communities)
- fast_cmaes tunes weights of both via `graphify tune` / `bvr --auto-tune`
- CASS feeds historical data to all

**Pro:** minimal scope, each tool stays independent
**Con:** inter-tool connection remains manual

### Direction B: Integration layer (thin bridge)

Not a new tool, but a contract between existing ones. Shared format for overlaying two graphs.

```
graphify-out/graph.json  ──┐
                           ├──→  overlay  ──→ enriched triage
.beads/issues.jsonl     ──┘
```

Overlay rules:
- Issue mentions file path → edge to corresponding graphify node
- Issue description contains keyword → semantic match to graphify community
- graphify community with many issues = "hot zone"
- graphify god node with zero issues = "unmonitored risk"

**Pro:** connects understanding with planning, both tools unchanged
**Con:** another artifact to maintain

### Direction C: New tool — "project lens"

Meta-tool that reads output of both graphs and answers questions neither can answer alone.

| Question | graphify alone | br/bvr alone | Combined |
|---|---|---|---|
| Blast radius of this issue? | — | Knows blockers | Blockers + which modules + their connections |
| Where is the biggest risk? | Knows low-cohesion zones | Knows stale issues | Low-cohesion + stale = highest risk |
| Do these two issues conflict? | — | Knows dep conflicts | + knows both touch same community |
| What's not covered by issues? | — | — | God nodes without issues = blind spots |
| Planning quality? | — | — | % of architecture covered by issues |

**Pro:** closes all three gaps, new quality of information
**Con:** new tool = new maintenance burden

---

## graphify: Identified Weaknesses

Found during code analysis of extract.py, build.py, cluster.py, analyze.py.

| Problem | Location | Impact |
|---------|----------|--------|
| God nodes = degree only | analyze.py:39-58 | Misses bridging bottlenecks |
| Betweenness O(V³), computed twice without cache | analyze.py:357,263 | Slow on >1K nodes |
| Surprise score — sum of bonuses, no normalization | analyze.py:134-187 | Large graphs overshadow small |
| Cross-file resolution — Python only | extract.py:2039-2169 | 19 languages without cross-file edges |
| Hyperedges stored but never analyzed | build.py:49-51 | Dead data |
| Graph is undirected, direction in _src/_tgt attrs | build.py:43-48 | Loses directed analysis |
| No cycle detection | — | Can't see circular dependencies |
| Hard-coded thresholds (peripheral=2, hub=5) | analyze.py:179-185 | Not adaptive to graph size |
| Call graph — direct calls only | extract.py:844-969 | Misses dynamic dispatch |
| Concept node detection — fragile heuristic | analyze.py:93-109 | source_file dot check is brittle |

### Proposed graphify improvements (from bvr algorithms)

| # | What | How | Effort | Value |
|---|---|---|---|---|
| 1 | Directed graph | nx.DiGraph() instead of nx.Graph() | Medium | High |
| 2 | PageRank god nodes | nx.pagerank() replaces degree() | Low | High |
| 3 | Cycle detection | nx.strongly_connected_components() | Low | Medium |
| 4 | Cache betweenness | Compute once, pass as parameter | Low | Medium |
| 5 | Normalized surprise | Divide score by log(graph_size) | Low | Medium |
| 6 | Cross-file for Go/Rust/TS | Extend _resolve_cross_file_imports | High | High |
| 7 | Articulation points | nx.articulation_points() → report section | Low | Medium |
| 8 | Adaptive thresholds | percentile(degrees, 25/75) | Low | Low |
| 9 | TOON output | --format toon for agent-friendly output | Medium | Medium |

---

## fast_cmaes: Where It Fits

CMA-ES optimizes continuous parameters without gradients. Relevant targets:

| Target | Params | Fitness function | Data source |
|---|---|---|---|
| bvr ImpactScore weights | 8 (sum=1) | Rank correlation: triage order vs actual closure | CASS outcomes + br events |
| graphify surprise score | 6 bonuses | User relevance of flagged connections | CASS feedback or implicit (Claude usage) |
| graphify Leiden params | 3 (resolution, max_frac, min_split) | Modularity / avg cohesion | Graph-internal metric |
| graphify NodeImportanceScore | 7 (if implemented) | Cross-project stability | Multi-repo benchmark |
| vis.js physics | 5 layout params | Edge crossing minimization | Graph-internal metric |

**Key dependency:** most fitness functions require CASS outcome data. Without historical feedback, only graph-internal metrics (modularity, cohesion) are available for optimization.

---

## Three Knowledge Layers

```
graphify  = "what exists"     (structure now)
br + bvr  = "what to do"     (tasks and priorities)
CASS      = "what happened"  (experience and patterns)
fast_cmaes = "how to improve" (optimization from experience)
```

No single tool provides the full picture. Together they form a closed loop:
**understand → plan → act → learn → improve → understand better**

---

## Open Questions

1. **Is Direction B (overlay) enough, or does Direction C (project lens) justify a new tool?**
   The answer depends on whether the overlay data produces questions that are asked frequently enough to warrant dedicated tooling.

2. **What is the minimum viable feedback loop?**
   CASS outcomes → bvr weight tuning via fast_cmaes is the simplest closed loop. Does it produce measurably better triage recommendations?

3. **Should graphify consume .beads data directly?**
   Adding issues as nodes in the knowledge graph connects "what exists" with "what needs work". But it mixes two very different data lifecycles (static code vs dynamic tasks).

4. **Is cross-agent memory (CASS) more valuable than cross-tool integration?**
   An agent that remembers past sessions may produce more value than tools that share data — because the agent is the one making decisions.

5. **Where does the human fit?**
   All five tools are agent-friendly (--json, --robot-*, MCP). But the cycle Understand → Plan → Prioritize → Execute → Learn can run without human intervention. Is that desirable? Where should the human checkpoint be?

---

## Gap 1 Verification: frankensqlite Case Study

**Test subject:** `/Users/sd/projects/frankensqlite/` — ground-up Rust reimplementation of SQLite with MVCC concurrent writers. 750K LOC, 26 crates, 644 .rs files. 1757 beads issues (314 open, 1398 closed, 277 blocked). No graphify graph exists.

### Six Sources of Knowledge, Six Separate Views

| Source | What it knows | What it doesn't know |
|---|---|---|
| **br (1757 issues)** | Task dependencies, priorities, blockers, statuses | Which code modules an issue touches, architectural risk |
| **bv (triage)** | PageRank/betweenness in task graph, ImpactScore | connection.rs = 88K LOC, fsqlite-mvcc = 65K LOC, crate dep depth |
| **AGENTS.md (842 lines)** | Workflow rules, toolchain, trauma rules, multi-agent protocol | Doesn't codify architectural risk zones or god nodes |
| **COMPREHENSIVE_SPEC (18K lines)** | Full specification: MVCC, RaptorQ, SSI, ECS, all subsystems | Not linked to issues — "what's specified" vs "what's built" vs "what's tracked" |
| **tasks.md (phases 1-9)** | Original phased plan with checkboxes | Stale — says Phase 2 "IN PROGRESS" but code is at Phase 6+ |
| **Code (750K LOC)** | Actual state — what compiles and passes tests | No context for "why" or "what's left" |

No single source provides the full picture. Each answers a different question but none answers: "what is architecturally important, how well is it covered by issues, and what's the risk?"

### Finding 1: Issue-to-Architecture Mapping is Absent

Open issues by crate mention (text search in title + description, 314 open issues):

| Crate | LOC | Open issues mentioning it | LOC per issue |
|---|---|---|---|
| fsqlite-core | 132K | 58 | 2,286 |
| fsqlite-harness | 106K | 24 | 4,417 |
| fsqlite-mvcc | 65K | 50 | 1,300 |
| fsqlite-e2e | 54K | 0 | ∞ |
| fsqlite-vdbe | 45K | 24 | 1,875 |
| fsqlite-pager | 21K | 21 | 1,000 |
| fsqlite-wal | 19K | 34 | 559 |
| fsqlite-types | 17K | 8 | 2,125 |
| fsqlite-btree | 15K | 6 | 2,500 |
| fsqlite-parser | 15K | 5 | 3,000 |
| fsqlite-wasm | — | 69 | over-planned (crate doesn't exist yet) |

82 of 314 open issues (26%) mention no crate at all. bv ranks these alongside crate-specific issues without knowing which code they affect.

### Finding 2: connection.rs (88K LOC) is Invisible to Task Graph

The single largest file in the project — `crates/fsqlite-core/src/connection.rs` at 88,683 lines — is larger than the entire graphify codebase (7.3K LOC). It contains the parse cache, compiled cache, concurrent_mode_default, and the full DDL/DML executor.

bv's top recommendation `bd-db300.1.3` ("automate perf") scores 0.47. PERFORMANCE_OPTIMIZATION_PLAN.md specifically identifies `connection.rs ~lines 1448-1456` and `~lines 2267-2383` as bottleneck locations. But bv doesn't know this. The performance plan and the triage engine live in separate worlds.

### Finding 3: AGENTS.md Trauma Rules = Manual CASS

AGENTS.md lines 263-285 contain a hand-written trauma rule:

> "On Feb 10 2026, an agent set concurrent_mode_default to false and implemented serialized file locking in MemoryVfs, completely defeating the project's core innovation."

This is exactly what CASS procedural memory does — recording a harmful pattern with context. But AGENTS.md does this manually for one incident. It doesn't scale to 750K LOC across 26 crates. CASS would systematically capture, score, and surface such patterns with confidence decay.

### Finding 4: Spec→Issues→Code→Tests — No Traceability Chain

BEAD_AUDIT_REPORT.md shows 149 beads mapped to spec sections, with overlap issues in §4 (Asupersync) and §5.10 (Write Merging). UNIT_INVARIANT_MATRIX.md shows 5,016 tests with P0 gaps in "MVCC concurrent tests, RaptorQ repair, Pager integration."

But nobody tracks the full chain: spec section X → issue bd-YYY → crate fsqlite-Z → file F.rs → tests T1,T2. Each link is tracked separately.

### Finding 5: Crate Dependency Depth Amplifies Risk

PROPOSED_ARCHITECTURE.md documents the dependency tree:
```
fsqlite-core → vdbe → btree → pager → vfs → types
                                             → error
             → mvcc → wal → pager → vfs
             → planner → ast → types
             → parser → ast → types
```

A change in fsqlite-types propagates through the entire stack. 8 open issues mention fsqlite-types — none is tagged as "high blast radius." bv can't compute this because it only sees task-level dependencies (issue blocks issue), not code-level dependencies (crate depends on crate).

### Finding 6: Multi-Agent Risk Without Architectural Awareness

AGENTS.md line 835: "potentially dozen of other agents working on the project at the same time." Agent Mail provides file reservations but not architectural awareness. An agent reserving `crates/fsqlite-types/src/lib.rs` doesn't know it affects every other crate. Multiple agents may have issues touching connection.rs (88K LOC) simultaneously without architectural context.

### What graphify Would Provide Here

| Scenario | Without graphify | With graphify overlay |
|---|---|---|
| Agent takes `bd-db300.5.1` (per-core txn pipeline) | bv says score=0.30. No file/crate context | graphify data shows: fsqlite-core has 6,084 nodes, connection.rs is articulation point (degree 2,127). This context is currently unavailable to bv |
| Agent takes `bd-35lpy` (WASM bindgen API) | bv says score=0.31 | graphify data shows: fsqlite-wasm has 76 nodes (smallest crate by code). bv scores it similarly to issues in larger crates |
| New agent starts session | Reads 842-line AGENTS.md | graphify GRAPH_REPORT.md would show god nodes and communities, but doesn't exist for this project yet |

### Empirical Verification (graphify AST extraction on frankensqlite)

graphify extracted 31,656 nodes, 74,036 edges, 351 communities, and 1,696 articulation points from 644 Rust files (tree-sitter, 0 LLM tokens). Overlaying with beads issue data (full text search in titles + descriptions):

| Assessment | Crates | Example |
|---|---|---|
| HIGH RISK (many nodes, few issues) | fsqlite-harness, fsqlite-core, fsqlite-func | harness: 8,897 nodes / 24 issues = 371 nodes/issue |
| BLIND SPOT (code exists, zero issues) | fsqlite, fsqlite-ext-json, fsqlite-ext-fts5, fsqlite-ext-icu, fsqlite-ext-misc, fsqlite-ext-fts3 | fsqlite: 757 nodes, 0 issues |
| UNDER-TRACKED | fsqlite-types (1,068 nodes / 8 issues), fsqlite-parser (740 / 5), fsqlite-btree (680 / 6) | parser.rs is articulation point with degree 387 |
| OVER-PLANNED (issues ahead of code) | fsqlite-wasm | 76 nodes / 69 issues — crate barely exists |

Key god node finding: `parse_one()` has 274 edges and `Parser` has 86 edges — both in parser.rs (articulation point, degree 387). But fsqlite-parser has only 5 open issues. bv doesn't recommend any parser work because the task graph shows no blocked dependencies there.

Meanwhile, bv puts 3 WASM issues in its top 10 recommendations. graphify shows fsqlite-wasm has only 76 code nodes — the crate barely exists. bv over-prioritizes it because WASM issues have high PageRank in the task dependency graph, not because the code needs work.

Full data: frankensqlite branch `research/graphify-gap-verification`, file `GAP1_VERIFICATION.md`.

### Verdict

**Gap 1 is confirmed and deeper than initially theorized.** It is not simply "graphify isn't linked to br." It is:

1. **Six knowledge sources, none complete** — spec, issues, code, tests, perf plan, and trauma rules each cover a fragment
2. **Architectural risk is invisible to task graph** — bv ranks issues without LOC, dependency depth, or test coverage data
3. **Trauma rules don't scale** — AGENTS.md captures one incident manually; CASS would systematize this
4. **No traceability chain** — spec→issue→crate→file→test links are not tracked end-to-end
5. **Multi-agent workflows amplify the gap** — 12 agents without architectural awareness multiply the risk of unintended damage

---

## Next Steps (research, not implementation)

- [ ] Map concrete data flows: what exact JSON fields would an overlay need from graph.json + issues.jsonl
- [ ] Evaluate: does CASS already have enough outcome data to serve as fitness function for fast_cmaes?
- [ ] Prototype: run bvr triage with different weight presets on a real project, compare against actual closure order
- [ ] Explore: can graphify's GRAPH_REPORT.md be a CASS knowledge source (episodic layer)?
- [ ] Assess: what would a minimal "project lens" query API look like?
