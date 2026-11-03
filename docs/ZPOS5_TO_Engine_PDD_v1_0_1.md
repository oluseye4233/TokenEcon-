---
title: "ZPOS+5 TO ENGINE — ATLAS PromptWare Design Document"
subtitle: "Token Optimization Engine: static prompt compression (ZPOS+5 / ICMS) integrated with experience-driven runtime cost control (TokenEconomizer pattern)"
---

```
JUNGLENOMICS FORGE · ATLAS PROMPTWARE SERIES
════════════════════════════════════════════════════════════
ZPOS+5 TO ENGINE — PROMPTWARE DESIGN DOCUMENT
Production ID:   JNGL-PDD-ZPOSTO-2026-001
Version:         1.0 DRAFT
Date:            2026-11-02 (America/Chicago, CST)
Classification:  Internal design — not certified
Source systems:  ZPOS+5 ICMS Engine 2.0 (ZPOS_ICMS_ENGINE2) · TokenEconomizer (Microsoft Research / UC Berkeley, Oct 2026)
Structure:       ATLAS 4-Part (Cheat Sheet · Executive Summary · Worksheet · Implementation)
════════════════════════════════════════════════════════════
```

# PART 0 — AUTHORSHIP, PROVENANCE AND CLAIM KEY

## 0.1 Authorship and Provenance Block

| Field | Value |
|---|---|
| Commissioned by | Shay (Oluseye Shay Amusa), Junglenomics FORGE |
| Drafted by | Claude (Anthropic), working in the 4J BONSAI PRODUCTION FLOOR project |
| Document title | ZPOS+5 TO Engine — PromptWare Design Document |
| Production ID | JNGL-PDD-ZPOSTO-2026-001 |
| Timestamp | 2026-11-02, America/Chicago (CST) |
| Inputs read | ZPOS_ICMS_ENGINE2 (full text); SPHINX Engine SDD §3.1; ANT QUEEN Features Sheet §7; ATOMIC PROMPT Whitepaper §3.3; SPARTAN MAX ATLAS PDD (structure reference); ATLAS ULTRA SI SPC (4-Part structure); four secondary write-ups of TokenEconomizer |
| Inputs NOT read | The TokenEconomizer paper (arXiv) and GitHub repository; the MarkTechPost page itself failed to load. All TokenEconomizer facts are therefore second-hand |
| Registry check | Searched project knowledge for an existing "ZPOS+5 TO Engine" or "TO Engine" entry: none found. The ZPOS Token Optimization Expert SPC (cited at JCSE 49/50 in ZPOS_ICMS_ENGINE2 §8.3) exists. SPC roster in §4.2 is proposed and has not been checked against Master SPC & Platform Registry v1.0 |
| Naming flag | "TO" is read here as **Token Optimization**. If it means something else, the title, this row and Appendix B are the only places to change |

## 0.2 Claim Key (evidence tags used inline)

| Tag | Meaning |
|---|---|
| [EXT] | Externally reported. For TokenEconomizer this means secondary reporting of the paper, not the paper itself |
| [INT]† | Internally self-scored in project documents, not independently verified |
| [SPEC] | Design intent. Not built, not tested |
| [SYNTH] | Original synthesis in this document, not empirically validated |

## 0.3 Version History

| Version | Date | Change |
|---|---|---|
| 1.0 DRAFT | 2026-11-02 | Initial integration of ZPOS+5 ICMS 2.0 with the TokenEconomizer runtime pattern |

---

# PART 1 — CHEAT SHEET (1 page)

**What it is.** One engine that cuts token cost at two layers. The **Static layer** compresses the text you author (SPCs, PDDs, system prompts) using ZPOS+5. The **Runtime layer** removes waste an agent creates while working (unused tool schemas, rediscovered repo facts, stuck loops) by learning from past sessions. [SPEC]

**Why both.** ZPOS shrinks what you *write*. In agent work, most billed tokens are *accumulated context re-read every turn* and tool definitions re-sent every request, which prompt compression barely touches. TokenEconomizer targets exactly that layer but does nothing for authored prompts. They are complementary. [SYNTH]

**Order of operations (the one rule to remember)** [SPEC]

1. **Prune** what is unused (tool schemas)
2. **Compress** what remains (authored text, once, offline, then freeze)
3. **Inject** only confirmed facts, verbatim, never compressed
4. **Guard** the loop at runtime (advisory only)
5. **Learn** from the finished session

**Hard rules**

| # | Rule |
|---|---|
| R1 | Never recompress mid-session. Compressed prefixes are frozen and hash-pinned so prompt caches are not broken |
| R2 | Injected facts and tool names/parameter schemas are a *verbatim zone*: no lossy compression |
| R3 | Quality gate is **end-task success**, not embedding similarity. Similarity scores are advisory only |
| R4 | Savings are reported **per layer with attribution**. No multiplying of percentages across layers |
| R5 | Nothing is learned from a single session. Confirm before acting (default 3) |
| R6 | The loop guard advises; it never stops a run |
| R7 | Every optimization is reversible with one command and logged |

**Targets** (all [SPEC], to be validated in Phase 4): runtime layer ≥15% mean cost reduction on a fixed agent benchmark; Pass@1 non-inferior within −2 points; static layer holds ZPOS+5's existing thresholds (≥95% semantic preservation) *and* passes a behavioral regression suite.

**Status.** Design only. Not built. Not scored. Not certified.

---

# PART 2 — EXECUTIVE SUMMARY

## 2.1 The problem

Token cost in agentic work has two sources that need different tools.

- **Authored tokens** are the text a human wrote once and the system re-sends constantly: SPCs, PDDs, system prompts, templates. ZPOS+5 addresses this with six compression methods and an ICMS selector. [INT]†
- **Emergent tokens** are produced by the agent's own behavior: full tool-definition lists sent on every request, conversation history re-read each turn, the same repo mistakes repeated every session, and loops of repeated failed attempts. Reporting on one analysis of a popular coding agent found that on a busy day 99% of processed tokens were accumulated input, with 1% generated [EXT, secondary]. Prompt compression cannot reach most of this.

Until now the Junglenomics stack had a strong answer to the first source and none to the second.

## 2.2 What each source system contributes

| | ZPOS+5 ICMS 2.0 | TokenEconomizer |
|---|---|---|
| Layer | Authored text | Agent runtime |
| Timing | Offline / pre-deployment | In-session and between sessions |
| Method | Zipfian analysis + five further methods, selected by ICMS | Learns per repository: fact injection, tool pruning, loop guard |
| Learns over time | No (stateless per prompt) | Yes (per repo, local state) |
| Selection logic | Rules and regex signals | Fixed thresholds |
| Quality measure | Semantic similarity, BERTScore, JCSE | Pass@1 on public benchmarks |
| Reported effect | 35–75% of prompt tokens [INT]† | 18.4–30.9% lower mean cost on 277 SWE-Bench tasks across 5 agent-model settings [EXT]; 16–21% on Web-Bench; 6–20% on two security-auditing benchmarks |
| Evidence base | Internal benchmarks | Public benchmarks; authors' harness published; no independent replication reported at release [EXT] |

TokenEconomizer's three mechanisms, as reported [EXT]:

1. **Micro-skills.** One-line repo facts (e.g. a required build step). A fact is injected only after 3 sessions confirm it; a contradicting session drops it. Off by default.
2. **Tool pruning.** A tool is removed from future sessions if unused across 10 prompts spanning at least 2 sessions, or across 5 sessions. Restore command available; restored tools are not pruned again. Changes are written only after the repo has been idle 60 minutes, to avoid forcing a resumed conversation to rewrite its prompt cache.
3. **Loop guard.** When a turn keeps hitting the same failures, a judge model assesses whether it is stuck. At ≥70% confidence the agent gets one redirect note per turn. It never terminates a turn. Activates only after five turns in the repo have ended well.

## 2.3 Integration thesis

The two systems act on **different cost components**, so they stack rather than overlap, provided three conflicts are designed out. [SYNTH]

| Conflict | Why it matters | Resolution in this PDD |
|---|---|---|
| **Cache invalidation** | Recompressing or editing a prompt prefix mid-session breaks prompt caching, which can raise cost. TokenEconomizer itself waits for idleness before editing settings for this reason | Compress offline only; freeze and hash-pin the compiled prefix (Rule R1); apply runtime changes only at session boundaries or after idle |
| **Lossy compression vs exact strings** | A compressed command, flag or tool parameter can silently break a tool call | Verbatim zone (R2): facts, tool names, parameter schemas are never lossy-compressed; tool descriptions only via PRISM with a tool-selection regression test |
| **Wrong quality metric** | Embedding similarity can stay above 95% while the agent behaves differently | End-task behavioral gate (R3) becomes the pass/fail; similarity is kept as a diagnostic |

Order of operations follows from these: **prune → compress → inject → guard → learn.** Pruning goes first because removing a tool is a larger and safer saving than shrinking its description. Compression comes second so it only works on what survives. [SPEC]

## 2.4 What the integrated engine adds beyond either parent

1. **A Cost Ledger** that attributes every token to a category (static prompt, tool schemas, history re-read, tool output, generated output; cache read vs write). Neither parent publishes this attribution as a first-class artifact. Without it, savings claims cannot be tied to a mechanism. [SPEC]
2. **ICMS-L (learning selector).** ICMS 2.0 picks a methodology by keyword rules. ICMS-L records which methodology actually held up per document class (by behavioral regression results) and shifts its defaults after confirmation, using the same confirm-before-act discipline as micro-skills. [SPEC]
3. **A shared validation harness** testing both layers against end-task outcomes, so the Junglenomics token-economics claim can be checked externally. [SPEC]

## 2.5 Value model (no invented totals)

The engine's total saving is not a sum or product of the parents' headline percentages, because they are measured against different bases (a share of prompt tokens vs a share of total agent cost). The value model is: [SYNTH]

```
Total cost  =  C_static + C_schemas + C_history + C_toolout + C_output
Saving      =  Σ over components of (baseline_i − engine_i)
```

- ZPOS+5 acts mainly on `C_static` (and on `C_schemas` descriptions where permitted).
- Runtime layer acts on `C_schemas` (pruning), `C_history` and `C_toolout` (fewer wasted turns), via facts and loop guard.
- Phase 4 measures each component before and after. Only then can a total be stated.

## 2.6 Disclosed limits and open items (Honesty Gate G3)

1. **Combination is untested.** TokenEconomizer was not evaluated with prompt-compressed instructions; ZPOS+5 was not evaluated on agent trajectories. Every integration claim here is [SYNTH] or [SPEC].
2. **TokenEconomizer facts are second-hand** (see 0.1). Thresholds above should be re-confirmed against the repository before implementation.
3. **TokenEconomizer's reported limits carry over** [EXT]: savings shrink when every task lands in a fresh repo; supported agent versions are narrow; tool pruning varies by agent (single tools in Claude Code, packages in Codex, MCP servers in Copilot CLI, nothing in Pi); results are Python-heavy (SWE-Bench); in at least one setting Pass@1 dipped slightly (74% to 73%) [EXT, secondary].
4. **ZPOS+5 internal inconsistencies** in project documents, to be resolved before they propagate into this build:
   - AEOS is expanded three ways: *Adaptive Evolutionary Optimization System* (ICMS 2.0), *Adaptive Efficiency Optimization System* (SPHINX Engine SDD, MATRIX MARKET SPC Designer), *Adaptive Entropy Optimization System* (CRAFT BOX SDD). NEXUS and QUANTUM also vary in expansion.
   - Per-methodology figures differ across documents (e.g. SPHINX SDD lists AEOS at 35–45% / 96.8% / 180 ms; ICMS 2.0 lists ZPOS at 37.2% / 96.8% / 180 ms and AEOS at 42.9% / 94.3% / 2,340 ms).
   - ICMS 2.0's executive summary states $750K–$3.5M annual savings, while its own §9.1 scenario table tops out at $756,000/yr for 5B tokens/month at an assumed $30/M tokens. The table arithmetic checks out; the headline range is not derived from it.
   - JCSE appears at two incompatible scales (0–50 total; 0–10 per-methodology).
5. **No independent assessor, no SOLVA GAUNTLET, no JCSE re-score.** See §4.8.

---

# PART 3 — WORKSHEET

## 3.1 System architecture

```
┌──────────────────────────── ZPOS+5 TO ENGINE ────────────────────────────┐
│                                                                            │
│  ORCHESTRATOR (TO-ORC)  — order: prune → compress → inject → guard → learn │
│                                                                            │
│  ┌─ STATIC LAYER (offline) ─────────┐   ┌─ RUNTIME LAYER (in/between sessions) ─┐
│  │ ICMS-L selector                  │   │ Tool Pruner        (TO-PRN)           │
│  │ ZPOS · AEOS · NEXUS · PRISM      │   │ Fact Injector      (TO-FCT)           │
│  │ QUANTUM · SYNTHESIS              │   │ Loop Guard         (TO-LPG)           │
│  │ Freeze + hash-pin compiler       │   │ Idle-gated Writer  (TO-WRT)           │
│  └──────────────────────────────────┘   └───────────────────────────────────────┘
│                                                                            │
│  EXPERIENCE STORE (local, per workspace)   COST LEDGER (per-component)     │
│                                                                            │
│  VALIDATION HARNESS: end-task gate · behavioral regression · A/B           │
│  GOVERNANCE: audit log · reversibility · claim tiering                     │
└────────────────────────────────────────────────────────────────────────────┘
```

## 3.2 Components

| ID | Component | Responsibility | Source lineage |
|---|---|---|---|
| TO-ORC | Orchestrator | Enforces order of operations, session lifecycle, freeze rules | New [SPEC] |
| TO-LDG | Cost Ledger | Per-component token and cost attribution, cache read/write split | New [SPEC] |
| TO-XPS | Experience Store | Local per-workspace state: tool usage, failures, candidate facts, methodology outcomes | TokenEconomizer pattern (state kept locally in `~/.tokeneconomizer`) [EXT] |
| TO-CMP | Static Compressor | Runs ZPOS+5 methods through ICMS-L; outputs frozen, hash-pinned compiled prompts | ZPOS+5 ICMS 2.0 [INT]† |
| TO-ICL | ICMS-L selector | Learns methodology defaults per document class from regression outcomes | Extension of ICMS 2.0 §4 [SPEC] |
| TO-PRN | Tool Pruner | Removes unused tool schemas via the host agent's own settings | TokenEconomizer [EXT] |
| TO-FCT | Fact Injector | Injects confirmed one-line workspace facts at session start | TokenEconomizer [EXT] |
| TO-LPG | Loop Guard | Judge-based stuck detection; advisory redirect | TokenEconomizer [EXT] |
| TO-WRT | Idle-gated Writer | Applies setting changes only after idle window; one-command restore | TokenEconomizer [EXT] |
| TO-VAL | Validation Harness | End-task gate, regression suite, A/B, per-layer attribution | New [SPEC] |
| TO-GOV | Governance | Audit log, reversibility, claim-tier labels in all reports | Junglenomics conventions |

## 3.3 Default configuration (parameters carried from parents are marked)

```yaml
to_engine:
  orchestrator:
    order: [prune, compress, inject, guard, learn]
    recompress_mid_session: false           # R1
    freeze_compiled_prefix: true
    prefix_hash: sha256

  static_layer:
    default_methodology: AUTO               # ICMS-L
    preservation_floor: 0.95                # from ZPOS+5 §7.2
    behavioral_regression_required: true    # R3 (new)
    verbatim_zone: [tool_names, tool_param_schemas, injected_facts, code_blocks, commands]
    tool_description_methods: [PRISM]       # only after tool-selection test passes
    hybrid_presets: [enterprise_production, high_volume, segment_based, research]  # ZPOS+5 §6

  runtime_layer:
    tool_prune:
      unused_prompts: 10                    # TokenEconomizer default [EXT]
      min_sessions_for_prompts: 2
      unused_sessions_any_length: 5
      write_after_idle_minutes: 60
      restore_command: "to restore-tools"
      never_reprune_restored: true
    facts:
      enabled: false                        # off by default, as in TokenEconomizer
      confirm_sessions: 3
      review_every_k_sessions: 3
      contradiction_drops_fact: true
      max_injected_tokens: 150              # [SPEC] to be tuned in Phase 4
      mode_no_model_calls: envfacts         # learn from repeated failures only
    loop_guard:
      judge_confidence_min: 0.70
      max_redirects_per_turn: 1
      activate_after_good_turns: 5
      skip_judge_if_turn_exceeds_x_usual: 2.5
      may_terminate_turn: false             # R6

  learning:
    icml_confirm_runs: 3                    # same discipline as facts
    method_default_shift_requires: regression_pass

  governance:
    audit_log: true
    report_claim_tags: true
    attribution_per_component: true         # R4
```

Values marked "[EXT]" are taken from secondary reporting and must be re-confirmed. `max_injected_tokens: 150` and `icml_confirm_runs: 3` are design choices, not sourced.

## 3.4 Data model (Experience Store, minimum viable)

| Table | Key fields | Purpose |
|---|---|---|
| `sessions` | session_id, workspace_id, agent, model, started, ended, outcome | Session boundary and result |
| `tool_usage` | session_id, tool_id, prompts_seen, used_count | Pruning decisions |
| `failure_events` | session_id, turn_id, signature, count | Loop guard and fact candidates |
| `fact_candidates` | fact_text, first_seen, confirmations, contradictions, status | Confirm-before-inject |
| `compiled_prompts` | prompt_id, source_hash, method, output_hash, tokens_in, tokens_out, regression_result | Frozen static layer |
| `methodology_outcomes` | doc_class, method, runs, regression_pass, mean_reduction | ICMS-L learning |
| `ledger` | session_id, component, tokens, cache_read, cache_write, cost | Attribution |
| `audit_log` | ts, action, target, reason, reversible_by | Governance |

## 3.5 API additions (extending ZPOS+5 §7)

| Endpoint | Purpose |
|---|---|
| `POST /api/to/compile` | Static layer: compress and freeze a prompt set; returns output hash and regression result |
| `POST /api/to/session/start` | Applies pruned tool set and confirmed facts; returns the session's frozen prefix hash |
| `POST /api/to/session/end` | Submits outcome; updates Experience Store and Ledger |
| `GET  /api/to/ledger?session=` | Per-component attribution |
| `POST /api/to/restore` | One-command rollback of tool pruning and/or fact injection |
| `GET  /api/to/report` | Claim-tagged savings report with per-layer attribution |

Existing endpoints (`/api/zpos/compress`, `/api/icms/recommend`, `/api/zpos/database`, `/api/zpos/batch`) are retained unchanged.

## 3.6 ATLAS Worksheet — prompt index

Phase colors follow the Atomic Prompt ordering rule: foundation before construction, validation never before construction. Prompt IDs use COMPONENT-TYPE-###. Priority: P0 must, P1 should, P2 could. "Dep" lists prerequisite IDs. Count: **26 prompts across 5 phases.** (This is a design-level worksheet, not the 100+ prompt build-out typical of full ATLAS PDDs; see §4.8.)

### 🔴 Phase 1 — Foundation

| ID | Pri | Component | Task | Dep |
|---|---|---|---|---|
| ORC-SPEC-001 | P0 | Orchestrator | Specify lifecycle states and the prune→compress→inject→guard→learn order, with freeze rules | — |
| LDG-DB-002 | P0 | Ledger / Store | Create Experience Store and Ledger schema (§3.4) | ORC-SPEC-001 |
| LDG-API-003 | P0 | Ledger | Implement token attribution by component with cache read/write split | LDG-DB-002 |
| GOV-LOG-004 | P0 | Governance | Append-only audit log and reversal mechanism | LDG-DB-002 |
| BASE-MEAS-005 | P0 | Validation | Capture **baseline** cost per component on a fixed task set before any optimization | LDG-API-003 |
| REG-CHK-006 | P1 | Registry | Resolve the ZPOS+5 inconsistencies in §2.6 item 4; publish a single reconciled methodology table | — |

### 🟡 Phase 2 — Static layer refit (ZPOS+5 → frozen compiler)

| ID | Pri | Component | Task | Dep |
|---|---|---|---|---|
| CMP-BLD-007 | P0 | Compressor | Wrap existing ZPOS+5 methods behind `compile`; add output hashing and freeze | BASE-MEAS-005, REG-CHK-006 |
| CMP-RULE-008 | P0 | Compressor | Implement verbatim-zone detector (tool names, schemas, code, commands, facts) | CMP-BLD-007 |
| CMP-TST-009 | P0 | Compressor | Build behavioral regression suite: run each compiled SPC/PDD through scripted scenarios and compare outputs to the uncompressed version | CMP-BLD-007 |
| CMP-TOOL-010 | P1 | Compressor | Tool-description compression via PRISM only, gated by tool-selection accuracy test | CMP-RULE-008, CMP-TST-009 |
| CMP-GATE-011 | P0 | Compressor | Replace the 95% similarity floor as sole gate with: regression pass **and** similarity floor | CMP-TST-009 |

### 🟢 Phase 3 — Runtime layer

| ID | Pri | Component | Task | Dep |
|---|---|---|---|---|
| PRN-BLD-012 | P0 | Pruner | Tool-usage tracking and prune decision per §3.3 thresholds | LDG-DB-002 |
| WRT-BLD-013 | P0 | Writer | Idle-gated writer (60 min) and `restore` command; never re-prune restored | PRN-BLD-012, GOV-LOG-004 |
| PRN-ADP-014 | P1 | Pruner | Per-host adapters (Claude Code settings; others as supported). Record what each host can and cannot prune | WRT-BLD-013 |
| FCT-BLD-015 | P1 | Fact Injector | Failure-signature capture; candidate facts; 3-session confirmation; contradiction drop | LDG-DB-002 |
| FCT-INJ-016 | P1 | Fact Injector | Session-start injection within token budget; verbatim zone | FCT-BLD-015, CMP-RULE-008 |
| LPG-BLD-017 | P1 | Loop Guard | Repeated-failure detector, judge call, one advisory redirect per turn, activation gate | LDG-DB-002 |
| LPG-SAFE-018 | P0 | Loop Guard | Prove it cannot terminate a turn; log every redirect | LPG-BLD-017 |
| ORC-BLD-019 | P0 | Orchestrator | Assemble layers in order; reject any mid-session recompression | PRN-BLD-012, CMP-BLD-007, FCT-INJ-016 |

### 🔵 Phase 4 — Validation

| ID | Pri | Component | Task | Dep |
|---|---|---|---|---|
| VAL-AB-020 | P0 | Validation | A/B: baseline vs static-only vs runtime-only vs combined, on the same task set, same agent/model | ORC-BLD-019 |
| VAL-ATTR-021 | P0 | Validation | Per-component attribution report; state savings per layer, no multiplied percentages | VAL-AB-020 |
| VAL-QUAL-022 | P0 | Validation | End-task success (Pass@1 or equivalent) non-inferiority within −2 points | VAL-AB-020 |
| VAL-CACHE-023 | P1 | Validation | Measure cache read/write before and after; confirm freeze rule prevents regressions | VAL-AB-020 |
| VAL-REPL-024 | P1 | Validation | Reproduce TokenEconomizer's published result on its own harness before trusting the runtime layer on ours | PRN-BLD-012 |

### 🟣 Phase 5 — Innovation

| ID | Pri | Component | Task | Dep |
|---|---|---|---|---|
| ICL-BLD-025 | P2 | ICMS-L | Learn methodology defaults per document class from regression outcomes; confirm-before-shift | VAL-QUAL-022 |
| FAB-FUT-026 | P2 | Fleet | Cross-workspace anonymized priors for cold-start (explicit opt-in; local-first by default) | ICL-BLD-025 |

## 3.7 Exemplar atomic prompts (7-element ATLAS format)

### CMP-RULE-008 — Verbatim-zone detector

```
PROMPT ID:     CMP-RULE-008
PHASE:         🟡   PRIORITY: P0
COMPONENT:     Static Compressor
CAMELOT PAIR:  ZPOS Token Optimization Expert × SPARTAN SI   [roster proposed, SPEC]
ZPOS TARGET:   n/a (this prompt defines what is NOT compressed)
JCSE TARGET:   not scored
DEPENDENCIES:  CMP-BLD-007

1. ROLE:        Compression safety engineer. Your job is to prevent silent breakage.
2. INSTRUCTION: Given an input prompt, return a list of character spans that must be
                preserved byte-for-byte. Classify each span as TOOL_NAME, TOOL_PARAM_SCHEMA,
                CODE_BLOCK, SHELL_COMMAND, FILE_PATH, INJECTED_FACT, or QUOTED_LITERAL.
3. EXAMPLE:     Input contains "run `python -m pytest tests/unit` before committing".
                Output spans: SHELL_COMMAND covering the backticked text.
4. CONSTRAINT:  Prefer false positives to false negatives. Never return an overlapping
                span. If uncertain, protect the span. Do not rewrite the text.
5. FORMAT:      JSON array: [{start, end, class, reason}]
6. DATA:        Compressor receives only unprotected spans. Protected spans are re-inserted
                verbatim after compression.
7. SYSTEM:      Deterministic settings; temperature 0; fail closed (protect all) on error.
```

### FCT-BLD-015 — Confirm-before-inject fact capture

```
PROMPT ID:     FCT-BLD-015
PHASE:         🟢   PRIORITY: P1
COMPONENT:     Fact Injector
DEPENDENCIES:  LDG-DB-002

1. ROLE:        Session reviewer that extracts reusable workspace facts.
2. INSTRUCTION: From the last session's failure events and transcript summary, propose
                at most 3 candidate facts. Each must be one line, concrete, and verifiable
                (a command, a required order of steps, a path).
3. EXAMPLE:     "Run the schema migration before the integration tests."
4. CONSTRAINT:  No opinions, no style advice, no secrets, no user data. Do not restate a
                fact already confirmed. A candidate becomes injectable only after 3 distinct
                sessions confirm it; any later contradiction removes it.
5. FORMAT:      JSON: [{fact_text, evidence_event_ids, confidence}]
6. DATA:        Experience Store tables `failure_events`, `fact_candidates`.
7. SYSTEM:      Low temperature; model calls optional (mode envfacts uses none).
```

### LPG-SAFE-018 — Loop guard safety proof

```
PROMPT ID:     LPG-SAFE-018
PHASE:         🟢   PRIORITY: P0
COMPONENT:     Loop Guard
DEPENDENCIES:  LPG-BLD-017

1. ROLE:        Adversarial tester for an advisory-only supervisor.
2. INSTRUCTION: Write tests demonstrating that no code path in Loop Guard can interrupt,
                cancel, or modify a running turn other than appending one redirect note.
3. EXAMPLE:     Test: judge returns 0.99 confidence; assert turn continues and exactly
                one note is emitted.
4. CONSTRAINT:  Cover: repeated high-confidence verdicts, judge timeout, malformed judge
                output, headless runs, and turns over 2.5x usual length (judge skipped).
5. FORMAT:      Test suite plus a one-page report listing each path and its verdict.
6. DATA:        Config §3.3 `loop_guard`.
7. SYSTEM:      Fail open: on any guard error, do nothing.
```

---

# PART 4 — IMPLEMENTATION

## 4.1 Timeline

Estimates are [SPEC] planning figures, not commitments.

| Phase | Weeks | Output |
|---|---|---|
| 1 Foundation | 1–2 | Ledger, store, audit log, **baseline measurement** |
| 2 Static refit | 3–5 | Frozen compiler, verbatim zone, regression suite |
| 3 Runtime layer | 6–9 | Pruner, writer, facts, loop guard, orchestrator |
| 4 Validation | 10–12 | A/B, attribution, replication of TokenEconomizer's result |
| 5 Innovation | 13+ | ICMS-L; optional |

**Decision gate after Phase 1.** If the baseline shows `C_static` is a small share of total cost in your real workloads, the static layer is not where the money is, and Phase 2 effort should shrink accordingly. This is the single most useful early finding. [SYNTH]

## 4.2 Proposed SPC roster [SPEC, not registry-verified]

| Role | Proposed SPC |
|---|---|
| PDD architect and convener | SPHINX ULTRA SI |
| Architecture authority | ADA ULTRA SI |
| Static compression domain | ZPOS Token Optimization Expert SPC; SPARTAN SI |
| Debugging / loop analysis | BUGMXT SI |
| Independent assessor | To be assigned. Must not be the author or the compression SPCs (separation of duties) |

## 4.3 Risks

| # | Risk | Mitigation |
|---|---|---|
| 1 | Compression changes behavior while similarity stays high | R3 behavioral gate; verbatim zone |
| 2 | Cache busting erases savings | R1 freeze; VAL-CACHE-023 |
| 3 | Pruned tool needed later | Restore command; restored tools never re-pruned; idle-gated writes |
| 4 | Injected fact goes stale or is wrong | 3-session confirmation; contradiction drop; token cap; audit log |
| 5 | Loop guard misjudges productive work as stuck | Advisory only; one note per turn; 70% floor; skip on long turns |
| 6 | Host agent updates break adapters | Version-pinned adapters; per-host capability table (PRN-ADP-014) |
| 7 | Learning from too little data | Confirm-before-act everywhere; cold start does nothing |
| 8 | Savings overclaimed | R4 attribution; claim tags; Phase 4 replication |
| 9 | Privacy of stored session data | Local-first state; no cross-workspace sharing without explicit opt-in (FAB-FUT-026) |
| 10 | Savings small when work spans many fresh repos | Acknowledged limit [EXT]; report by workload type |

## 4.4 Validation protocol (Phase 4)

1. Fix a task set, agent, and model. Record baseline per component (BASE-MEAS-005).
2. Run four arms: baseline, static-only, runtime-only, combined. Same tasks, repeated runs.
3. Report per component, with the end-task success rate beside every cost figure.
4. Replicate TokenEconomizer's published number on its own harness first (VAL-REPL-024). If it does not reproduce, treat the runtime layer as unproven.
5. Only after step 3 may any combined claim be written, tagged [INT]† unless an independent assessor reproduces it.

## 4.5 Acceptance criteria

| # | Criterion |
|---|---|
| A1 | Ledger accounts for ≥95% of billed tokens by component |
| A2 | No mid-session recompression occurs (test-proven) |
| A3 | Loop guard cannot terminate a turn (LPG-SAFE-018 passes) |
| A4 | Every optimization is reversible by one command; audit log complete |
| A5 | End-task success non-inferior within −2 points in the combined arm |
| A6 | Runtime layer reaches ≥15% mean cost reduction on the fixed benchmark, or the shortfall is reported plainly |
| A7 | Static layer passes behavioral regression for every compiled prompt shipped |

## 4.6 Relationship to existing systems

| System | Relationship |
|---|---|
| ZPOS+5 ICMS Engine 2.0 | Parent. Its methods become the Static Compressor; its API is retained |
| ARK HARNESS / F9 Machine Floor | Natural host for the runtime layer: tool schemas and session lifecycle are already managed there. Hook points to be mapped in a separate design pass |
| SPHINX Marketplace | ZPOS free-tier testing could expose static compile; runtime layer is a separate paid-tier question, not decided here |
| ATLAS PDD pipeline | Static compile stays at the SDD-to-PDD step; runtime layer sits after deployment |
| 4J.BONSAI token manager | Ledger and report feed the existing analytics |

## 4.7 Expected blocking conditions for any future SOLVA GAUNTLET

These are the conditions I would expect to block a clean verdict today. They are stated as conditions, not as a verdict.

1. No integrated implementation exists.
2. TokenEconomizer behavior has not been verified against its paper or code.
3. ZPOS+5 methodology data is internally inconsistent across project documents (§2.6).
4. No behavioral (end-task) evidence exists for ZPOS+5, only similarity-based and self-scored figures.
5. No independent assessor assigned.

## 4.8 FORGE Certification Block

```
FORGE STATUS:        NOT SUBMITTED
JCSE:                NOT SCORED (no self-assigned score; a new pipeline run is required)
SOLVA GAUNTLET:      NOT RUN
INDEPENDENT ASSESSOR: NONE
DOCUMENT SCOPE:      Design-level PDD. 26 prompts across 5 phases; not a full-depth build
                     worksheet. Expansion to build depth is a follow-on task.
TOKENECONOMIZER FACTS: Second-hand; re-verify against paper and repository before build.
CERTIFICATION CLAIMS: None made by this document.
```

---

# APPENDIX A — Source map

| Topic | Source |
|---|---|
| ZPOS+5 methods, ICMS selection, API, hybrids, ROI, conformance | ZPOS_ICMS_ENGINE2 (project) |
| Alternate ZPOS+5 descriptions | SPHINX_ENGINE_SOFTWARE_DESIGN_DOCUMENT; MARKET_MATRIX_SPC_Designer_ATLAS_Summary; CRAFT_BOX_Software_Design_Document; THE_ANT_QUEEN_Features_Sheet |
| ATLAS worksheet format and phase colors | THE_ATOMIC_PROMPT_WHITEPAPER; ATLAS_ULTRA_SI_SPC_v1.0; SPARTAN_MAX_ATLAS_PDD |
| TokenEconomizer | [MarkTechPost](https://www.marktechpost.com/2026/10/30/microsoft-tokeneconomizer-coding-agent-cost-control/) (headline figures via search snippet); [explainx.ai](https://explainx.ai/blog/microsoft-tokeneconomizer-coding-agent-cost-control-plugin-claude-code-codex-copilot-2026) (mechanisms and thresholds); [AlphaSignal](https://alphasignal.ai/news/microsoft-s-tokeneconomizer-slashes-coding-agent-costs-by-31-without-touching) |
| Agent token composition (99% input) | [Fluid Attacks](https://fluidattacks.com/blog/ai-token-economics-cost-control) (secondary, search snippet) |

# APPENDIX B — Glossary

| Term | Meaning |
|---|---|
| TO | Token Optimization (reading adopted in this PDD; see 0.1) |
| ICMS / ICMS-L | Intelligent Compression Methodology Selector / learning variant (proposed) |
| Verbatim zone | Content never lossy-compressed |
| Freeze | Compiled prompt fixed and hash-pinned for the life of a session |
| Micro-skill | One-line, confirmed workspace fact injected at session start |
| Pass@1 | Share of tasks solved on the first attempt |
| Cost Ledger | Per-component token and cost attribution |

*Junglenomics FORGE · JNGL-PDD-ZPOSTO-2026-001 · v1.0 DRAFT · 2026-11-02*
