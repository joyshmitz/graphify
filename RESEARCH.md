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

## Next Steps (research, not implementation)

- [ ] Map concrete data flows: what exact JSON fields would an overlay need from graph.json + issues.jsonl
- [ ] Evaluate: does CASS already have enough outcome data to serve as fitness function for fast_cmaes?
- [ ] Prototype: run bvr triage with different weight presets on a real project, compare against actual closure order
- [ ] Explore: can graphify's GRAPH_REPORT.md be a CASS knowledge source (episodic layer)?
- [ ] Assess: what would a minimal "project lens" query API look like?
