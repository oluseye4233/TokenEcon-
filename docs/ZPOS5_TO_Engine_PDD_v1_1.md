---
title: "ZPOS+5 TO ENGINE — ATLAS PromptWare Design Document"
subtitle: "Token Optimization Engine: static prompt compression (ZPOS+5 / ICMS) integrated with TokenEconomizer as the runtime cost-control layer"
---

```
JUNGLENOMICS FORGE · ATLAS PROMPTWARE SERIES
════════════════════════════════════════════════════════════
ZPOS+5 TO ENGINE — PROMPTWARE DESIGN DOCUMENT
Production ID:   JNGL-PDD-ZPOSTO-2026-001
Version:         1.1 DRAFT (supersedes 1.0 DRAFT / 1.0.1)
Date:            2026-11-03 (America/Chicago, CST)
Classification:  Internal design — not certified
Source systems:  ZPOS+5 ICMS Engine 2.0 (ZPOS_ICMS_ENGINE2)
                 microsoft/TokenEconomizer @ d58755d (2026-11-02), Apache-2.0
                 arXiv 2610.22732 (Li et al., UC Berkeley / Microsoft Research / Microsoft Security)
Structure:       ATLAS 4-Part (Cheat Sheet · Executive Summary · Worksheet · Implementation)
════════════════════════════════════════════════════════════
```

# PART 0 — AUTHORSHIP, PROVENANCE AND CLAIM KEY

## 0.1 Authorship and Provenance Block

| Field | Value |
|---|---|
| Commissioned by | Shay (Oluseye Shay Amusa), Junglenomics FORGE |
| Drafted by | v1.0: Claude (Anthropic), 4J BONSAI PRODUCTION FLOOR project. v1.1: Claude (Anthropic), Claude Code session, reworked against the TokenEconomizer source |
| Document title | ZPOS+5 TO Engine — PromptWare Design Document |
| Production ID | JNGL-PDD-ZPOSTO-2026-001 |
| Timestamp | 2026-11-03, America/Chicago (CST) |
| Inputs read (v1.1) | **`github.com/microsoft/TokenEconomizer`**, default branch, commit `d58755d849cd9157d29aef539f2beab3d5fee0fc` (2026-11-02): `README.md`, `CLAUDE.md`, `docs/PLUGIN.md`, `docs/ARCHITECTURE.md`, `plugin/README.md`, `experiments/README.md`, the module READMEs (`microskills`, `toolprune`, `loopguard`, `plugin`, `trace`), and the source of `ledger.py`, `pricing.py`, `stage.py`, `domain.py`, `microskills/lifecycle.py`, `microskills/envfacts.py` (caps), `toolprune/tools.py`, `loopguard/rules.py` (presets), `plugin/config.py` (defaults), `docs/assets/results_overview.py` (Table 1 rows) |
| Inputs carried from v1.0 | ZPOS_ICMS_ENGINE2 (full text); SPHINX Engine SDD §3.1; ANT QUEEN Features Sheet §7; ATOMIC PROMPT Whitepaper §3.3; SPARTAN MAX ATLAS PDD; ATLAS ULTRA SI SPC. These were **not** re-read for v1.1; ZPOS+5 content is carried over unchanged |
| Inputs NOT read | The arXiv paper itself (2610.22732). Paper figures below are the ones the repository quotes from its Table 1 (`README.md`, `docs/assets/results_overview.py`), not read from the PDF. The repository's test suite was **not executed** in this session |
| Repository search | GitHub search for "TokenEconomizer" (2026-11-03): the canonical repo is `microsoft/TokenEconomizer` (~4.5k stars). Others found are forks, pinned copies or third-party variants (`pr9nav/tokeneconomizer`, `mmango7474/tokeneconomizer-linux`, `Envera/tokeneconomizer-lite`, `Joe05520/codex-te-quota-gate`, …) and are not used as sources |
| Registry check | Unchanged from v1.0: no existing "TO Engine" entry found; SPC roster in §4.2 is proposed and not registry-verified |
| Naming flag | "TO" = **Token Optimization**. If it means something else, the title, this row and Appendix B are the only places to change |

## 0.2 Claim Key (evidence tags used inline)

| Tag | Meaning |
|---|---|
| [SRC] | **New in v1.1.** Read directly in the TokenEconomizer repository at commit `d58755d`. A file path is given where useful |
| [PAPER] | A figure from the TokenEconomizer paper, as quoted by its repository. Not independently reproduced |
| [EXT] | Externally reported by secondary sources (v1.0 web write-ups). Retained only where the repository is silent |
| [INT]† | Internally self-scored in Junglenomics project documents, not independently verified |
| [SPEC] | Design intent. Not built, not tested |
| [SYNTH] | Original synthesis in this document, not empirically validated |

## 0.3 Version History

| Version | Date | Change |
|---|---|---|
| 1.0 DRAFT | 2026-11-02 | Initial integration of ZPOS+5 ICMS 2.0 with the TokenEconomizer runtime pattern, from second-hand sources |
| **1.1 DRAFT** | 2026-11-03 | Reworked against the TokenEconomizer repository. Every runtime-layer fact re-verified ([EXT] → [SRC]); 14 corrections (Appendix C). **Architecture decision changed:** the runtime layer is no longer re-implemented — TokenEconomizer is adopted as a pinned dependency and extended only through its published seams (`Stage`, `DomainAdapter`, `ledger`/`pricing`). Worksheet re-cut from 26 to 30 prompts; config, data model, API, validation protocol and risks rewritten to match the real system |

---

# PART 1 — CHEAT SHEET (1 page)

**What it is.** One engine that cuts token cost at two layers. The **Static layer** compresses the text you author (SPCs, PDDs, system prompts, instruction files) using ZPOS+5, offline, then freezes it. The **Runtime layer** *is* TokenEconomizer: it learns from finished sessions in a repository and removes waste the agent creates while working — unused tool schemas, rediscovered repo facts, stuck loops. [SPEC]

**Why both.** ZPOS shrinks what you *write*. In agent work, most billed input is the conversation and tool definitions re-sent every turn: TokenEconomizer's own pricing module notes that "~90%+ of `tokens_total` is `cache_read_input_tokens`" in an agent loop [SRC `pricing.py`]. Prompt compression barely touches that layer; TokenEconomizer targets it and does nothing for authored prompts. They are complementary. [SYNTH]

**Build vs adopt.** TokenEconomizer is Apache-2.0, standard-library-only Python, ships as a plugin for Claude Code, Codex, Copilot CLI and Pi (opencode and Cursor experimental), and exposes stable seams. **We adopt it, pin it, and extend it; we do not fork its mechanisms.** [SPEC]

**Order of operations (the one rule to remember)** [SPEC]

1. **Prune** what is unused (TokenEconomizer tool pruning, at session boundaries)
2. **Compress** what remains (ZPOS+5, authored text only, once, offline, then freeze)
3. **Inject** only confirmed facts, verbatim, never compressed (TokenEconomizer micro-skills)
4. **Guard** the loop at runtime (TokenEconomizer loop guard, advisory only)
5. **Learn** from the finished session (TokenEconomizer fold + TO ledger + ICMS-L)

**Hard rules**

| # | Rule |
|---|---|
| R1 | Never recompress mid-session. Compiled prefixes are frozen and hash-pinned; a resumed conversation gets the **byte-identical** text it started with (TokenEconomizer does the same for facts [SRC `docs/PLUGIN.md` §3, "Prompt caching"]) |
| R2 | Verbatim zone: injected facts, tool names, parameter schemas, code, commands and paths are never lossy-compressed |
| R3 | Quality gate is **end-task success**, not embedding similarity. Similarity is diagnostic only |
| R4 | Cost is counted **as billed** (cache-aware effective tokens, TokenEconomizer `pricing.py`), **including every optimizer model call**, and reported per layer with attribution. No multiplying percentages across layers |
| R5 | Nothing is learned from a single session. Confirm before acting (TokenEconomizer: 3 confirming sessions for facts; ≥2 sessions for pruning) |
| R6 | The loop guard advises; it never stops a turn |
| R7 | Every optimization is reversible with one command and logged (`te toolprune restore`, `to restore`) |
| R8 | **New.** Hooks fail open and never wait on a model call. Any TO error leaves the session as if TO were not installed (TokenEconomizer convention [SRC `CLAUDE.md`]) |
| R9 | **New.** Shadow first. Every new mechanism runs in `shadow` mode (decide and log, act on nothing) before it is allowed to act [SRC `plugin/config.py`] |

**Targets** (all [SPEC], validated in Phase 4): runtime layer ≥15% mean cost reduction on our fixed benchmark *with TokenEconomizer's own accounting*; end-task success non-inferior within −2 points; static layer holds ZPOS+5's existing ≥95% semantic floor *and* passes the behavioral regression suite.

**Status.** Design only. TokenEconomizer exists and is public; the integrated TO Engine is not built, scored or certified.

---

# PART 2 — EXECUTIVE SUMMARY

## 2.1 The problem

Token cost in agentic work has two sources that need different tools.

- **Authored tokens** are text a human wrote once and the system re-sends constantly: SPCs, PDDs, system prompts, `CLAUDE.md`/`AGENTS.md`-style instruction files, templates. ZPOS+5 addresses this with six compression methods and an ICMS selector. [INT]†
- **Emergent tokens** come from the agent's own behavior: full tool-definition lists sent on every request, conversation history re-read each turn, the same repo mistakes repeated every session, and loops of failed attempts. TokenEconomizer's framing: "Every turn pays again for the agent's instructions, every loaded tool, and the conversation so far" [SRC `README.md`]. Prompt compression cannot reach most of this.

The Junglenomics stack had a strong answer to the first source and none to the second. A production-grade answer to the second now exists in public.

## 2.2 What each source system contributes

| | ZPOS+5 ICMS 2.0 | TokenEconomizer |
|---|---|---|
| Layer | Authored text | Agent runtime |
| Timing | Offline / pre-deployment | Hooks during a session; learning at session end (`te fold`) [SRC] |
| Method | Zipfian analysis + five further methods, selected by ICMS | Three mechanisms (micro-skills, tool pruning, loop guard) + an opt-in repository map (aider) [SRC] |
| Agent coupling | None (operates on text) | Black-box: never modifies the agent, its model or its tool descriptions; acts only through the agent's own hooks and settings files [SRC] |
| Learns over time | No (stateless per prompt) | Yes, per repository *and per agent*, local state under `~/.tokeneconomizer` (`0700`/`0600`) [SRC] |
| Selection logic | Rules and regex signals | Evidence thresholds + a judge model (loop guard) + an optional LLM fact proposer [SRC] |
| Cost accounting | Token counts | Cache-aware effective tokens per billed model id; plugin's own model calls charged to the same ledger [SRC `ledger.py`, `pricing.py`] |
| Quality measure | Semantic similarity, BERTScore, JCSE | Official SWE-Bench grader (Pass@1); Web-Bench Pass@2 [SRC `experiments/README.md`] |
| Reported effect | 35–75% of prompt tokens [INT]† | SWE-Bench, 277 tasks, 6 repos, 5 agent–model combinations: **18.4–30.9% lower mean cost**, cheaper in **30 of 30** repo × combination cells, Pass@1 up in **4 of 5** [PAPER]. Web-Bench 16–21% lower; two security-auditing benchmarks (every task a new repo) 6–20% lower [PAPER] |
| Reproducibility | Internal benchmarks | Harness, launchers, workloads and aggregation published in `experiments/`. SWE-Bench and Web-Bench reruns supported; **the security sets' adapters ship but the harness does not run or grade them** [SRC]. No independent replication known |

**Per-combination results (paper Table 1, as quoted by the repository)** [PAPER, SRC `docs/assets/results_overview.py`]

| Agent | Model | Mean cost vs default | ± 1 s.d. | Pass@1 default → TE | Δ points |
|---|---|---|---|---|---|
| Claude Code | Sonnet 5 | −23.6% | 1.8 | 74.7 → 76.0 | +1.3 |
| Claude Code | Opus 4.8 | −30.9% | 2.1 | 74.0 → 73.0 | **−1.0** |
| Codex | GPT-5.6 Terra | −24.7% | 1.2 | 73.8 → 74.7 | +0.9 |
| Codex | GPT-5.6 Sol | −26.1% | 2.1 | 80.1 → 81.7 | +1.6 |
| Copilot CLI | Sonnet 5 | −18.4% | 1.5 | 75.0 → 77.6 | +2.6 |

**TokenEconomizer's mechanisms, as implemented** [SRC]

1. **Micro-skills** (off by default). Two proposers:
   - *envfacts* — no model calls; five fact families from repeated evidence: form repair (a failing spelling followed by a working one, e.g. `pytest` → `python -m pytest`), boundary (a host always refused), empty probe, arg repair, env value (`microskills/envfacts.py`).
   - *distiller* — an LLM proposer that reads digests of recent sessions every *k* = 3 sessions (first run after session 2) and writes facts with host-checked detectors (`microskills/distiller.py`).
   - Lifecycle (`microskills/lifecycle.py`): a candidate becomes ACTIVE after **3 confirmations** (for a fact about the code, on sessions that edited *different* files); it retires when a recency-weighted score (decay γ = 0.8, contradictions weighted λ = 5×) drops below 0, or at once on one clean contradiction for a high-stakes fact (command-spelling facts: two contradicting sessions in a row). An ACTIVE fact is injected only while it clears a **firing gate**: the waste it prevents occurred, or kept being prevented, in ≥40% of sessions that exercised it, over ≥4 such sessions.
   - Injection: `SessionStart` `additionalContext`, rendered at fold time, capped at 4 facts / 150 tokens for envfacts (the proposer's facts are not capped); a lint refuses completion or verification advice.
   - Modes: `off` · `envfacts` · `distiller` · `on` (= envfacts+distiller).
2. **Tool pruning** (on by default, `auto`). A loaded tool unused over *p* = 10 prompts spanning ≥ *m* = 2 sessions, or over *s* = 5 sessions of any length, is left out of later sessions through the agent's own settings (3× as long for `NotebookEdit` and Codex `image_media`). The work surface (`Bash`, `Read`, `Edit`, `Write`, `Grep`, `Glob`, …) is protected, as are a person's interactive tools (questions, plan mode, sub-agents, web); a tool ever seen in use is never proposed; `toolprune_protect` adds patterns. The deny list is written only after the repository's sessions have been idle *q* = 60 min (at the latest a day after it formed). `toolprune restore` returns tools, which are then never re-pruned unless named in `toolprune apply NAME`. Modes: `off` · `propose` · `auto`. **The plugin never calls a model to prune.**
3. **Loop guard** (on by default, `nudge`). Idle until *e* = 5 turns in the repository have *ended green* (agent stopped on its own, no failing check after the last edit). After ≥ 20 tool calls (`min_turns`) with a sign of trouble (repeated failures, dense errors), a judge model — run as a detached job, never in the hook — decides; only a "stuck" verdict at confidence ≥ *c* = 0.7 earns a judge-written redirect, at most *r* = 1 per turn, attached to a tool result. A turn past *b* = 2.5× the usual length that also churns the same edits through the same failures is redirected **without** the judge (the backstop). Still stuck afterwards: a one-line notice telling the person how to stop it. Never stops a turn. Presets: `conservative` · `default` · `aggressive`.
4. **Repository map** (opt-in, off). aider's ranked map of key files and symbols, ≤ 2,048 tokens, once per session; no model calls.

Cross-cutting: `shadow=true` computes and logs every decision but acts on none; `TE_DISABLE=1` turns the plugin off for one session; every hook fails open into `~/.tokeneconomizer/errors.log`.

## 2.3 Integration thesis

The two systems act on **different cost components**, so they stack rather than overlap, provided four conflicts are designed out. [SYNTH]

| Conflict | Why it matters | Resolution in this PDD |
|---|---|---|
| **Cache invalidation** | A prompt cache matches the prefix *tools → system → conversation*. TokenEconomizer verified that a running session keeps its tools, but a **resumed** conversation sends the new tool list and must re-write its whole history to cache [SRC `docs/PLUGIN.md` §3] | Compress offline only; freeze and hash-pin each compiled prefix (R1); record the hash per session so a resume gets byte-identical text; leave TokenEconomizer's idle-gated deny-list writer in charge of all tool changes |
| **Lossy compression vs exact strings** | A compressed command, flag or tool parameter silently breaks a call. TokenEconomizer's facts are exact command forms (`python -m pytest`) | Verbatim zone (R2). TO never touches TokenEconomizer's injected block. Tool-description compression applies only to tools **we own** (our MCP servers, ARK HARNESS tools) — the agents' built-in tool descriptions are outside any black-box plugin's reach |
| **Wrong quality metric** | Embedding similarity can stay above 95% while the agent behaves differently | End-task behavioral gate (R3) is pass/fail; similarity stays diagnostic |
| **Learning-signal pollution (new)** | ZPOS-compressed instructions change agent behavior, which changes what TokenEconomizer learns (tool usage evidence, ended-green turns, fact confirmations). A compiled-prompt change mid-stream can make old evidence stale | Treat a new compiled-prefix hash as an **epoch**: log it in the ledger; Phase 4 never compares across epochs; a static-layer release into a live repository is followed by a shadow window before runtime evidence is trusted again |

The order **prune → compress → inject → guard → learn** follows. Pruning goes first because removing a tool is a larger and safer saving than shrinking its description (TokenEconomizer's live check: Claude Code 20 → 10 tools; Codex rendered prompt input 10,397 → 7,021 characters [SRC `docs/PLUGIN.md` §3]). Compression comes second so it only works on what survives.

## 2.4 What the integrated engine adds beyond either parent

v1.0 claimed neither parent had a cost ledger. **That was wrong:** TokenEconomizer has a canonical one (`ledger.py` + `pricing.py`: cache-aware effective tokens per billed model id, plus six mechanism-overhead fields, one priced record per session in `ledger.jsonl`). What it does *not* do is attribute cost by **content component**. TO adds:

1. **Component Ledger (TO-LDG).** Extends TokenEconomizer's ledger, reusing its pricing table unchanged, with attribution by component — static prompt (authored), tool schemas, injected facts, history re-read, tool output, generated output — and by the compiled-prefix epoch. Inputs already available: per-message usage from the agents' own logs (TokenEconomizer's `trace/` readers), tool-definition size (`toolprune measure`; Copilot's `toolDefinitionsTokens`), and TO's own compiled-prompt token counts. [SPEC]
2. **ICMS-L (learning selector).** ICMS 2.0 picks a methodology by keyword rules. ICMS-L records which methodology held up per document class (by behavioral regression results) and shifts defaults only after confirmation, reusing TokenEconomizer's lifecycle discipline (K = 3 confirmations, asymmetric retirement). [SPEC]
3. **Static layer for the runtime's blind spot.** TokenEconomizer explicitly leaves the agent's instructions alone. TO compresses the instruction files and system prompts a team authors and the agent re-reads every turn. [SPEC]
4. **A shared validation harness** that adds a static-only arm and a combined arm to TokenEconomizer's own `experiments/` arms, so the Junglenomics token-economics claim is checked on a public, rerunnable harness. [SPEC]

## 2.5 Value model (no invented totals)

The total saving is not a sum or product of the parents' headlines: they use different bases (share of prompt tokens vs share of total effective agent cost). [SYNTH]

```
Effective cost (TE pricing.py)  =  Σ_calls ( in·w_in + cache_read·w_cr + cache_write·w_cw + out·w_out )
                                   + optimizer overhead (TE ledger.OVERHEAD_KEYS + TO compile/judge calls)

Component view (TO-LDG)         =  C_static + C_schemas + C_facts + C_history + C_toolout + C_output + C_overhead
Saving                          =  Σ_i (baseline_i − engine_i)        — per component, same task set, same agent/model
```

Example weights from TokenEconomizer's table: Claude models input 1.00, cache read 0.10, cache write 1.25, output 5.00 (fresh-input equivalents) [SRC `pricing.py`]. Two consequences for ZPOS+5:

- A token removed from a **cached** static prefix saves ~0.10 of a fresh token per re-read, not 1.00. ZPOS+5's "% of prompt tokens" overstates its billed effect on agent workloads with warm caches. This must be shown in Phase 1 before Phase 2 is sized.
- A compiled-prefix change that **breaks** the cache costs 1.25× on the whole prefix once. One mid-session recompression can erase many turns of savings (R1).

Layer mapping: ZPOS+5 acts on `C_static` (and on `C_schemas` only for tools we own). TokenEconomizer acts on `C_schemas` (pruning), `C_facts` (adds a little), and `C_history`/`C_toolout`/`C_output` (fewer wasted calls and turns), and adds `C_overhead` (judge, proposer).

## 2.6 Disclosed limits and open items (Honesty Gate G3)

1. **Combination is untested.** TokenEconomizer was not evaluated with prompt-compressed instructions; ZPOS+5 was not evaluated on agent trajectories. Every integration claim here is [SYNTH] or [SPEC].
2. **The paper's numbers come from the experiment harness, not the shipped plugin.** The repository is explicit that the plugin differs [SRC `docs/PLUGIN.md` §2]: unit of work is a session (micro-skills, pruning) or prompt-turn (loop guard) instead of a benchmark task; the plugin's redirect is a mid-turn reminder whereas the experiments killed and resumed the agent ("its effect is not inherited from the experiments"); the plugin proposer runs every 3 sessions vs every 5 tasks (10 on Codex) in the experiments; the plugin uses a cheap model (`haiku` on Claude Code) for its own calls vs the agent's own model in the experiments; micro-skills is off by default in the plugin; the plugin has no answer-key guard and keeps web tools. **Our replication must therefore measure the plugin, not just rerun the paper.**
3. **TokenEconomizer's own disclosed limits carry over** [SRC]:
   - Savings depend on repeated work in the same repository; each agent learns separately (Claude Code's learning is not used by Codex in the same repo).
   - Narrow supported versions: Claude Code 2.1.300–2.1.310, Codex 0.153.1–0.164.0, Copilot CLI 1.0.83–1.0.102, Pi 1.2.0–1.3.0; opencode and Cursor experimental; Cursor reports no session cost.
   - Pruning unit varies by agent: single tools (Claude Code, `.claude/settings.local.json`), capability packages (Codex, `.codex/config.toml`, **trusted projects only**), whole MCP servers (Copilot, `.github/copilot/settings.local.json`), nothing (Pi, opencode, Cursor).
   - Loop guard: both rungs exercised only on *provoked* loops; "whether a nudge changes a looping model's course is not established". A turn with no recognised verification counts as ended green and can loosen the envelope.
   - Micro-skills: facts derive from command output a repository controls (a prompt-injection surface); on an untrusted repository leave it off or use `envfacts`.
   - One setting regressed Pass@1 by 1.0 point (Claude Code + Opus 4.8, 74.0 → 73.0) [PAPER].
   - `refresh-catalog` is manual; until it runs, pruning on Claude Code sees too few tools.
4. **ZPOS+5 internal inconsistencies** (carried from v1.0, still unresolved — REG-CHK-006):
   - AEOS expanded three ways (*Adaptive Evolutionary / Efficiency / Entropy Optimization System*); NEXUS and QUANTUM also vary.
   - Per-methodology figures differ across documents (SPHINX SDD: AEOS 35–45% / 96.8% / 180 ms; ICMS 2.0: ZPOS 37.2% / 96.8% / 180 ms and AEOS 42.9% / 94.3% / 2,340 ms).
   - ICMS 2.0's headline $750K–$3.5M/yr is not derived from its own §9.1 table, which tops out at $756,000/yr (5B tokens/month at $30/M). The table arithmetic checks; the headline does not. Note also that $30/M treats every token as fresh input; under cache-aware pricing (§2.5) the figure for agent workloads would be lower.
   - JCSE appears at two incompatible scales (0–50 total; 0–10 per methodology).
5. **No independent assessor, no SOLVA GAUNTLET, no JCSE re-score.** See §4.8.
6. **Repository tests not run.** This session read the source but did not execute `python -m pytest tests`. ADP-PIN-007 does.

---

# PART 3 — WORKSHEET

## 3.1 System architecture

```
┌──────────────────────────────── ZPOS+5 TO ENGINE ─────────────────────────────────┐
│  ORCHESTRATOR (TO-ORC) — order: prune → compress → inject → guard → learn          │
│                                                                                    │
│  ┌─ STATIC LAYER (TO-owned, offline) ──┐   ┌─ RUNTIME LAYER = TokenEconomizer (pinned) ─┐
│  │ ICMS-L selector                     │   │ hooks: SessionStart · UserPromptSubmit ·   │
│  │ ZPOS · AEOS · NEXUS · PRISM ·       │   │        PostToolUse(+Failure) · Stop ·       │
│  │ QUANTUM · SYNTHESIS                 │   │        SessionEnd → detached `te fold`      │
│  │ Verbatim-zone detector              │   │ microskills  (PrefixStage)                 │
│  │ Freeze + hash-pin compiler          │   │ toolprune    (ToolingStage + denylist.py)  │
│  │ Behavioral regression gate          │   │ loopguard    (TurnGuard + detached judge)  │
│  └───────────────┬─────────────────────┘   │ repomap      (opt-in)                      │
│                  │ compiled, frozen         └──────────────┬─────────────────────────────┘
│                  ▼ instruction files                       │ ledger.jsonl · events.jsonl
│  DEPLOY: write compiled files at session boundary only     ▼
│                                                                                    │
│  TO EXTENSIONS (through TE's published seams, no fork):                            │
│    TO-LDG Component Ledger  — reads TE ledger + trace readers; adds component +    │
│                               epoch attribution; reuses TE pricing.py              │
│    TO-ADP DomainAdapter     — TE `DomainAdapter` for our harness/domains           │
│    TO-STG CompiledPrefixStage — TE `Stage` for the benchmark facade (experiments)  │
│                                                                                    │
│  STATE: ~/.tokeneconomizer (TE, untouched)   ~/.to-engine (TO: compiled, epochs)   │
│  VALIDATION: TE experiments/ arms + TO arms (static-only, combined, shadow)        │
│  GOVERNANCE: audit log · reversibility · claim tiering · version pin               │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**Seams used** [SRC `docs/ARCHITECTURE.md`]: `tokeneconomizer.stage.Stage` (`before_task`, `build_tools`, `run_kwargs`, `after_task`); `tokeneconomizer.domain.DomainAdapter`; `tokeneconomizer.ledger` / `pricing` ("the one definition of cost"); `tokeneconomizer.trace.load_trajectory`. TO code lives in its own package; it may import `tokeneconomizer`, never the reverse, and never patches its modules. On-disk TE state names are a compatibility surface — TO reads them, never writes them.

## 3.2 Components

| ID | Component | Responsibility | Source lineage |
|---|---|---|---|
| TO-ORC | Orchestrator | Enforces order of operations, session lifecycle, freeze and epoch rules; deploys compiled files only at session boundaries | New [SPEC] |
| TO-RT | Runtime layer | TokenEconomizer plugin, pinned to a tested commit; configured, not modified | TokenEconomizer [SRC] |
| TO-LDG | Component Ledger | Component + epoch attribution on top of TE's ledger; same pricing | Extends TE `ledger.py` [SRC] + new [SPEC] |
| TO-ADP | Domain adapter | TE `DomainAdapter` for Junglenomics domains (ARK HARNESS / F9 floor), incl. `protected_tools` for our own tools | TE `domain.py` [SRC] |
| TO-STG | CompiledPrefixStage | TE `Stage` that supplies the compiled static prefix in benchmark arms (`before_task`) | TE `stage.py` [SRC] |
| TO-CMP | Static Compressor | Runs ZPOS+5 methods through ICMS-L; outputs frozen, hash-pinned compiled prompts | ZPOS+5 ICMS 2.0 [INT]† |
| TO-VZ | Verbatim-zone detector | Spans never lossy-compressed | New [SPEC] |
| TO-ICL | ICMS-L selector | Learns methodology defaults per document class from regression outcomes | ICMS 2.0 §4 + TE lifecycle discipline [SPEC] |
| TO-VAL | Validation Harness | TE `experiments/` + TO arms; end-task gate; per-component attribution | TE [SRC] + new [SPEC] |
| TO-GOV | Governance | Audit log, reversibility, claim tags, version-pin register | Junglenomics conventions |

## 3.3 Default configuration

TokenEconomizer keys are written exactly as TokenEconomizer names them, so this block can be applied with `te config set` unchanged. TO keys are under `to_engine`.

```yaml
# ── Runtime layer: TokenEconomizer (repo-level config.json or `te config set`) ── [SRC defaults]
tokeneconomizer:
  pin: "microsoft/TokenEconomizer@d58755d"   # TO decision; re-pin only via ADP-PIN-007
  enabled: true
  shadow: true                  # R9: TO rollout starts in shadow; flip to false per mechanism after Phase 3 gate
  microskills: envfacts         # TE default is off; TO starts with the no-model-call layer
  microskills_max_facts: 4
  microskills_max_tokens: 150
  microskills_propose_every: 3  # k (applies only if microskills=on|distiller)
  toolprune: propose            # TE default is auto; TO starts with human approval, then auto
  toolprune_evidence: 10        # p
  toolprune_min_sessions: 2     # m
  toolprune_sessions: 5         # s
  toolprune_quiet_minutes: 60   # q
  toolprune_protect: ""         # add our own MCP tool patterns here, e.g. "mcp__ark__*"
  loopguard: nudge              # TE default; "stop" is read as nudge
  loopguard_aggr: default
  loopguard_nudges: 1           # r
  loopguard.envelope_min_samples: 5   # e
  loopguard.judge_confidence: 0.7     # c
  loopguard.backstop_mult: 2.5        # b
  loopguard.min_turns: 20
  loopguard.gate_min_turns: 18
  repomap: off
  paid_model: ""                # empty: haiku on Claude Code; agent default elsewhere

# ── TO Engine ── [SPEC]
to_engine:
  orchestrator:
    order: [prune, compress, inject, guard, learn]
    recompress_mid_session: false        # R1
    deploy_compiled_at: session_boundary # never while any session in the repo is live
    deploy_idle_minutes: 60              # same reasoning as TE toolprune_quiet_minutes
    prefix_hash: sha256
    resume_policy: replay_recorded_text  # byte-identical on resume
    new_hash_starts_epoch: true

  static_layer:
    targets: [instruction_files, system_prompts, spc, pdd]   # NOT agent built-in tool descriptions
    default_methodology: AUTO            # ICMS-L
    preservation_floor: 0.95             # ZPOS+5 §7.2 (diagnostic)
    behavioral_regression_required: true # R3 (gate)
    verbatim_zone: [tool_names, tool_param_schemas, te_injected_block, code_blocks,
                    shell_commands, file_paths, env_vars, quoted_literals]
    owned_tool_description_methods: [PRISM]   # only for tools we own, after tool-selection test
    hybrid_presets: [enterprise_production, high_volume, segment_based, research]  # ZPOS+5 §6

  ledger:
    pricing: tokeneconomizer.pricing     # reuse; never a second price table
    include_optimizer_overhead: true     # R4: TE OVERHEAD_KEYS + TO compile/regression calls
    components: [static, schemas, facts, history, toolout, output, overhead]
    attribution_per_component: true

  learning:
    icml_confirm_runs: 3                 # mirrors TE K_PROMOTE
    icml_contradiction_weight: 5.0       # mirrors TE LAMBDA
    method_default_shift_requires: regression_pass

  governance:
    audit_log: true
    report_claim_tags: true
```

`microskills: envfacts`, `toolprune: propose`, `shadow: true`, `deploy_idle_minutes` and the ICMS-L values are TO design choices, not TokenEconomizer defaults. All other runtime values are TokenEconomizer's shipped defaults at the pinned commit.

## 3.4 Data model

**TokenEconomizer state — read-only for TO** [SRC `docs/PLUGIN.md` §4]: per repository and host under `$TOKENECONOMIZER_HOME` (default `~/.tokeneconomizer`): mechanism state (`injection.json`, `untool.json`, `untool_applied.json`, `envelope.json`), `ledger.jsonl` (one priced record per session), `events.jsonl` (every injection, redirect, still-stuck notice), per-session guard state, `sessions/<host>/<id>.json` pins. Repository key = `origin` remote (clones and worktrees share), else main checkout path.

**TO state — `~/.to-engine`, same permissions (`0700`/`0600`)** [SPEC]

| Table | Key fields | Purpose |
|---|---|---|
| `compiled_prompts` | prompt_id, source_hash, method, output_hash, tokens_in, tokens_out, regression_result, verbatim_spans | Frozen static layer |
| `epochs` | repo_key, epoch_id, output_hashes[], deployed_at, shadow_until | Separates runtime evidence across static releases |
| `session_prefix` | host, session_id, repo_key, epoch_id, recorded_text_ref | Byte-identical replay on resume |
| `component_ledger` | host, session_id, epoch_id, component, input, cache_read, cache_write, output, effective | Per-component attribution (joins TE `ledger.jsonl` by session) |
| `methodology_outcomes` | doc_class, method, runs, regression_pass, mean_reduction, state | ICMS-L learning |
| `audit_log` | ts, action, target, reason, reversible_by | Governance (TO actions; TE actions stay in `events.jsonl`) |

v1.0's `tool_usage`, `failure_events` and `fact_candidates` tables are **dropped**: TokenEconomizer already keeps that evidence.

## 3.5 Interfaces

**Runtime layer — use TokenEconomizer's CLI** (from a terminal it costs no model call [SRC]):

| Need | Command |
|---|---|
| What was learned/done, session cost | `te status` |
| Facts injected and forming | `te microskills` |
| Pruned tools and evidence | `te toolprune` |
| Apply / roll back pruning | `te toolprune apply [ITEM…]` / `te toolprune restore [ITEM…]` |
| Schema size with/without proposal | `te toolprune measure` |
| Record a session's loaded tools | `te toolprune refresh-catalog` |
| Settings / health | `te config` · `te config set K=V [--global]` · `te doctor` |

**TO additions** [SPEC]

| Endpoint / command | Purpose |
|---|---|
| `POST /api/to/compile` · `to compile` | Compress and freeze a prompt set; returns output hashes and regression result |
| `POST /api/to/deploy` · `to deploy` | Write compiled files at the next idle session boundary; opens a new epoch |
| `GET /api/to/ledger?session=` · `to ledger` | Component attribution, joined to TE `ledger.jsonl` |
| `POST /api/to/restore` · `to restore` | Roll back to the previous epoch's source files; with `--runtime`, also runs `te toolprune restore` and sets `microskills=off` |
| `GET /api/to/report` · `to report` | Claim-tagged report, per layer and per component |

Existing endpoints (`/api/zpos/compress`, `/api/icms/recommend`, `/api/zpos/database`, `/api/zpos/batch`) are retained unchanged. v1.0's `/api/to/session/start|end` are dropped: session lifecycle belongs to the agents' hooks, which TokenEconomizer already handles.

## 3.6 ATLAS Worksheet — prompt index

Phase colors follow the Atomic Prompt ordering rule: foundation before construction, validation never before construction. Prompt IDs use COMPONENT-TYPE-###. Priority: P0 must, P1 should, P2 could. "Dep" lists prerequisites. **30 prompts across 5 phases** (design-level; see §4.8).

### 🔴 Phase 1 — Foundation

| ID | Pri | Component | Task | Dep |
|---|---|---|---|---|
| ORC-SPEC-001 | P0 | Orchestrator | Specify lifecycle, the prune→compress→inject→guard→learn order, freeze, epoch and resume rules | — |
| LDG-DB-002 | P0 | Ledger | Create TO state schema (§3.4) with `0700`/`0600` permissions | ORC-SPEC-001 |
| LDG-API-003 | P0 | Ledger | Component attribution: join TE `ledger.jsonl` + `trace` readers; reuse `tokeneconomizer.pricing`; include `OVERHEAD_KEYS` | LDG-DB-002, ADP-PIN-007 |
| GOV-LOG-004 | P0 | Governance | Append-only TO audit log and `to restore` | LDG-DB-002 |
| BASE-MEAS-005 | P0 | Validation | Baseline effective cost **per component** on a fixed task set, before any optimization; report cache-read share | LDG-API-003 |
| REG-CHK-006 | P1 | Registry | Resolve ZPOS+5 inconsistencies (§2.6 item 4); publish one reconciled methodology table | — |
| ADP-PIN-007 | P0 | Runtime | Pin TokenEconomizer; run `python -m pytest tests experiments/tests` at the pin; record results and agent versions in the version register | — |

### 🟡 Phase 2 — Static layer refit (ZPOS+5 → frozen compiler)

| ID | Pri | Component | Task | Dep |
|---|---|---|---|---|
| CMP-BLD-008 | P0 | Compressor | Wrap ZPOS+5 methods behind `compile`; output hashing and freeze | BASE-MEAS-005, REG-CHK-006 |
| CMP-RULE-009 | P0 | Verbatim zone | Detector for tool names, schemas, code, commands, paths, env vars, quoted literals and the TE `<tokeneconomizer>` / `additionalContext` block | CMP-BLD-008 |
| CMP-TST-010 | P0 | Compressor | Behavioral regression suite: scripted scenarios, compiled vs uncompressed | CMP-BLD-008 |
| CMP-TOOL-011 | P1 | Compressor | PRISM description compression for **tools we own only**, gated by a tool-selection accuracy test | CMP-RULE-009, CMP-TST-010 |
| CMP-GATE-012 | P0 | Compressor | Gate = regression pass **and** similarity floor | CMP-TST-010 |
| CMP-DEP-013 | P0 | Orchestrator | Idle-gated deploy of compiled files; new epoch; per-session recorded text for resume | CMP-GATE-012, ORC-SPEC-001 |

### 🟢 Phase 3 — Runtime layer (adopt and configure TokenEconomizer)

| ID | Pri | Component | Task | Dep |
|---|---|---|---|---|
| RT-INS-014 | P0 | Runtime | Install the pinned plugin on each target agent; `refresh-catalog` on Claude Code and Copilot; `te doctor` clean | ADP-PIN-007 |
| RT-SHD-015 | P0 | Runtime | Run all mechanisms in `shadow=true` over ≥ 20 real sessions per repo; review `events.jsonl` decisions by hand | RT-INS-014 |
| PRN-CFG-016 | P0 | Pruner | Set `toolprune_protect` for our tools; `propose` → review with `toolprune measure` → `auto` | RT-SHD-015 |
| PRN-HOST-017 | P1 | Pruner | Per-host capability register (what each agent can prune, trust requirements); re-verify on every agent upgrade | PRN-CFG-016 |
| FCT-CFG-018 | P1 | Fact Injector | Enable `envfacts`; trust policy for repos; decide `on` (proposer) per repo with cost check | RT-SHD-015 |
| FCT-VZ-019 | P0 | Fact Injector | Prove the static compiler never alters TE's injected block (CMP-RULE-009 against recorded sessions) | CMP-RULE-009, FCT-CFG-018 |
| LPG-CFG-020 | P1 | Loop Guard | Choose preset per repo; confirm judge spend appears in the ledger | RT-SHD-015 |
| LPG-SAFE-021 | P0 | Loop Guard | Adversarial check that the guard cannot stop or alter a turn beyond its redirect/notice (exemplar §3.7) | LPG-CFG-020 |
| ADP-DOM-022 | P1 | Adapter | TE `DomainAdapter` for ARK HARNESS / F9 floor sessions | ADP-PIN-007 |
| ORC-BLD-023 | P0 | Orchestrator | Assemble layers in order; reject mid-session recompression; epoch logging | CMP-DEP-013, PRN-CFG-016, FCT-VZ-019 |

### 🔵 Phase 4 — Validation

| ID | Pri | Component | Task | Dep |
|---|---|---|---|---|
| VAL-REPL-024 | P0 | Validation | Rerun one TE `experiments/` lane (e.g. Claude Code + Sonnet 5, `baseline` vs `trimech`, seeds 0–2) and compare to Table 1; then measure the **plugin** on the same tasks | ADP-PIN-007 |
| VAL-AB-025 | P0 | Validation | Arms: baseline · static-only (TO-STG) · runtime-only (`trimech`) · combined; plus TE ablations `inject-only` · `untool-only` · `spiral-only`. Same tasks, agent, model, seeds | ORC-BLD-023, VAL-REPL-024 |
| VAL-ATTR-026 | P0 | Validation | Per-component attribution; savings per layer; no multiplied percentages | VAL-AB-025 |
| VAL-QUAL-027 | P0 | Validation | End-task success (official grader) non-inferior within −2 points, per agent–model combination | VAL-AB-025 |
| VAL-CACHE-028 | P1 | Validation | Cache read/write before/after, including a resume test; confirm R1 and byte-identical resume | VAL-AB-025 |

### 🟣 Phase 5 — Innovation

| ID | Pri | Component | Task | Dep |
|---|---|---|---|---|
| ICL-BLD-029 | P2 | ICMS-L | Learn methodology defaults per document class; confirm-before-shift | VAL-QUAL-027 |
| FAB-FUT-030 | P2 | Fleet | Cross-workspace anonymized priors for cold start (explicit opt-in; local-first). Note TE keeps learning strictly local and per agent; any sharing is TO's responsibility and privacy burden | ICL-BLD-029 |

## 3.7 Exemplar atomic prompts (7-element ATLAS format)

### CMP-RULE-009 — Verbatim-zone detector

```
PROMPT ID:     CMP-RULE-009
PHASE:         🟡   PRIORITY: P0
COMPONENT:     Static Compressor / Verbatim zone
CAMELOT PAIR:  ZPOS Token Optimization Expert × SPARTAN SI   [roster proposed, SPEC]
ZPOS TARGET:   n/a (defines what is NOT compressed)
JCSE TARGET:   not scored
DEPENDENCIES:  CMP-BLD-008

1. ROLE:        Compression safety engineer. Your job is to prevent silent breakage.
2. INSTRUCTION: Given an input prompt, return the character spans that must be preserved
                byte-for-byte. Classify each as TOOL_NAME, TOOL_PARAM_SCHEMA, CODE_BLOCK,
                SHELL_COMMAND, FILE_PATH, ENV_VAR, INJECTED_FACT, or QUOTED_LITERAL.
                Any text inside a TokenEconomizer-injected block is INJECTED_FACT in full.
3. EXAMPLE:     Input: "run `python -m pytest tests/unit` before committing".
                Output: one SHELL_COMMAND span covering the backticked text.
4. CONSTRAINT:  Prefer false positives to false negatives. No overlapping spans. If
                uncertain, protect. Do not rewrite the text.
5. FORMAT:      JSON array: [{start, end, class, reason}]
6. DATA:        Compressor receives only unprotected spans; protected spans are re-inserted
                verbatim. Regression corpus: TE-recorded sessions' injected blocks.
7. SYSTEM:      Deterministic; temperature 0; fail closed (protect everything) on error.
```

### FCT-CFG-018 — Enabling micro-skills safely

```
PROMPT ID:     FCT-CFG-018
PHASE:         🟢   PRIORITY: P1
COMPONENT:     Fact Injector (TokenEconomizer micro-skills)
DEPENDENCIES:  RT-SHD-015

1. ROLE:        Runtime configuration reviewer for one repository.
2. INSTRUCTION: Decide this repository's micro-skills mode: off, envfacts, or on
                (envfacts + proposer). Base it on: whether the repo is trusted; the facts
                shadow mode would have injected (`te microskills`); and the proposer's
                charged cost in `te status` vs the sessions it would save.
3. EXAMPLE:     Trusted internal repo; shadow shows "use `python -m pytest`" confirmed in
                3 sessions and firing in 5 of 8 → set envfacts; revisit `on` after 10 sessions.
4. CONSTRAINT:  Untrusted repo → off or envfacts only (facts derive from command output the
                repo controls). Never edit TE state files. Never compress the injected block.
5. FORMAT:      Decision record: {repo_key, mode, evidence, cost_check, review_date}
6. DATA:        `te microskills`, `te status`, TE events.jsonl, TO audit log.
7. SYSTEM:      Apply with `te config set microskills=<mode>`; log to TO audit log.
```

### LPG-SAFE-021 — Loop guard safety check

```
PROMPT ID:     LPG-SAFE-021
PHASE:         🟢   PRIORITY: P0
COMPONENT:     Loop Guard (TokenEconomizer)
DEPENDENCIES:  LPG-CFG-020

1. ROLE:        Adversarial tester for an advisory-only supervisor you did not write.
2. INSTRUCTION: Show that under the pinned TE version and our config, the guard's only
                effects on a running turn are (a) at most `loopguard_nudges` redirects
                attached to tool results and (b) one still-stuck notice — never a stop.
3. EXAMPLE:     Judge returns stuck@0.99 on every call → exactly one redirect, then one
                notice; the turn continues to its natural end.
4. CONSTRAINT:  Cover: repeated high-confidence verdicts; judge timeout or no reply; paused
                side calls (signed-out CLI: only the backstop acts); malformed verdict;
                headless mode; a turn past backstop_mult × envelope that churns (redirect
                WITHOUT the judge); `loopguard=stop` (must act as nudge, with a warning).
5. FORMAT:      Test log plus a one-page path/verdict table; cite TE tests that already
                cover a path instead of duplicating them.
6. DATA:        Config §3.3; TE `loopguard/` and `tests/`; TE events.jsonl.
7. SYSTEM:      Run against recorded or provoked sessions only; fail open is expected.
```

---

# PART 4 — IMPLEMENTATION

## 4.1 Timeline

[SPEC] planning figures. Phase 3 is shorter than in v1.0 because the runtime layer is adopted, not built.

| Phase | Weeks | Output |
|---|---|---|
| 1 Foundation | 1–2 | TE pinned and tested; component ledger; audit log; **baseline per component** |
| 2 Static refit | 3–5 | Frozen compiler, verbatim zone, regression gate, idle-gated deploy |
| 3 Runtime adoption | 5–7 | TE installed, shadowed, configured per repo; safety check; adapter |
| 4 Validation | 8–11 | TE replication; plugin measurement; four-arm A/B + ablations; attribution |
| 5 Innovation | 12+ | ICMS-L; optional |

**Decision gate after Phase 1.** If the baseline shows `C_static` is a small share of *effective* (cache-weighted) cost, the static layer is not where the money is and Phase 2 should shrink. Given TE's note that ~90%+ of raw tokens in an agent loop are cache reads billed at ~0.10, this outcome is plausible and must be checked, not assumed. [SYNTH]

## 4.2 Proposed SPC roster [SPEC, not registry-verified]

| Role | Proposed SPC |
|---|---|
| PDD architect and convener | SPHINX ULTRA SI |
| Architecture authority | ADA ULTRA SI |
| Static compression domain | ZPOS Token Optimization Expert SPC; SPARTAN SI |
| Runtime integration / loop analysis | BUGMXT SI |
| Independent assessor | To be assigned. Not the author or the compression SPCs (separation of duties) |

## 4.3 Risks

| # | Risk | Mitigation |
|---|---|---|
| 1 | Compression changes behavior while similarity stays high | R3 behavioral gate; verbatim zone |
| 2 | Cache busting erases savings | R1 freeze; idle-gated deploy; byte-identical resume; VAL-CACHE-028 |
| 3 | Pruned tool needed later | TE: evidence over prompts *and* sessions, protected tools, `toolprune_protect`, `restore` (never re-pruned); TO: start in `propose` |
| 4 | Injected fact stale or wrong | TE lifecycle: 3 confirmations, recency score with 5× contradiction weight, 40% firing gate, 150-token cap; audit via `te microskills` |
| 5 | **Fact injection as prompt-injection vector** (new) | Facts come from command output a repository controls; untrusted repos → off or `envfacts` (fixed templates only); review before `on` |
| 6 | Loop guard misjudges productive work | Advisory only; judge decides all but the backstop; one redirect; presets; shadow first |
| 7 | **Version drift** in agents or TE (new) | Pin TE; TE supports narrow agent version ranges and its transcript readers are pinned to observed formats; re-verify (ADP-PIN-007, PRN-HOST-017) on every upgrade |
| 8 | **Static changes pollute runtime learning** (new) | Epochs; shadow window after each static deploy; never compare across epochs |
| 9 | Learning from too little data | Confirm-before-act everywhere; TE cold start does nothing until evidence forms |
| 10 | Savings overclaimed | R4 cache-aware accounting with overhead; attribution; replication; claim tags |
| 11 | Plugin ≠ paper | VAL-REPL-024 measures both the harness and the plugin; report them separately |
| 12 | Privacy of session data | TE local-only state (`0700`/`0600`), digests only in model calls; TO mirrors this; fleet sharing opt-in only |
| 13 | Savings small for fresh-repo workloads | Disclosed by TE (security benchmarks 6–20%); report by workload type |
| 14 | Licensing | TE is Apache-2.0: keep its LICENSE/NOTICE with any redistributed copy; no trademark use of "Microsoft" in TO branding |

## 4.4 Validation protocol (Phase 4)

1. Fix a task set, agent, model and seeds. Record the baseline per component (BASE-MEAS-005), with TE `pricing.py` effective tokens.
2. **Replicate first** (VAL-REPL-024): rerun one TE lane via `experiments/scripts/launch/main_sweep.sh` (`baseline` vs `trimech`) and compare cost and Pass@1 to Table 1 within its reported s.d. If it does not reproduce, the runtime layer is unproven on our setup; stop and report.
3. **Then measure the plugin** on the same tasks, because the plugin's mechanisms differ from the experiment arms (§2.6 item 2).
4. Run arms: baseline · static-only · runtime-only · combined, plus TE ablations. Same tasks, repeated seeds; pair by instance with `experiments/scripts/compare_arms.py`. Never pool trees across agents or models (TE's own rule: both are part of the cost basis).
5. Report per component, with end-task success beside every cost figure, and optimizer overhead shown separately.
6. Only then may a combined claim be written, tagged [INT]† unless an independent assessor reproduces it.

## 4.5 Acceptance criteria

| # | Criterion |
|---|---|
| A1 | Component ledger accounts for ≥95% of billed effective tokens and reconciles with TE `ledger.jsonl` per session |
| A2 | No mid-session recompression; a resumed session receives byte-identical compiled text (test-proven) |
| A3 | Loop guard cannot stop a turn (LPG-SAFE-021 passes) |
| A4 | Every optimization reversible by one command (`to restore`, `te toolprune restore`); audit log complete |
| A5 | End-task success non-inferior within −2 points in the combined arm, per agent–model combination |
| A6 | Runtime layer ≥15% mean effective-cost reduction on our benchmark *including overhead*, or the shortfall reported plainly |
| A7 | Static layer passes behavioral regression for every compiled prompt shipped |
| A8 | TE replication lane within Table 1's reported range, or the discrepancy reported |

## 4.6 Relationship to existing systems

| System | Relationship |
|---|---|
| ZPOS+5 ICMS Engine 2.0 | Parent. Its methods become the Static Compressor; its API is retained |
| microsoft/TokenEconomizer | Parent and **dependency**. Runtime layer, pinned and configured; extended only via `Stage`, `DomainAdapter`, `ledger`/`pricing`, `trace` |
| ARK HARNESS / F9 Machine Floor | Host for our own tools and sessions; needs a TE `DomainAdapter` (ADP-DOM-022); its tools are the only ones whose descriptions TO may compress |
| SPHINX Marketplace | Static compile could be exposed at the ZPOS free tier; runtime is TE's open-source plugin and is not ours to sell — any paid tier must be TO's own value-add (ledger, static layer, reporting) |
| ATLAS PDD pipeline | Static compile at the SDD-to-PDD step; runtime layer after deployment |
| 4J.BONSAI token manager | Consumes the component ledger and report |
| rtk (shell-output compression) | Compatible with TE per its README; an optional extra arm (`trimech-rtk`) already exists in TE's harness |

## 4.7 Expected blocking conditions for any future SOLVA GAUNTLET

Stated as conditions, not a verdict.

1. No integrated implementation exists.
2. ~~TokenEconomizer behavior has not been verified against its paper or code.~~ **Partly cleared in v1.1:** behavior verified against code and docs at `d58755d`. Still open: paper not read directly; tests not run; results not replicated.
3. ZPOS+5 methodology data is internally inconsistent across project documents (§2.6).
4. No behavioral (end-task) evidence exists for ZPOS+5; only similarity-based and self-scored figures.
5. No independent assessor assigned.

## 4.8 FORGE Certification Block

```
FORGE STATUS:          NOT SUBMITTED
JCSE:                  NOT SCORED (no self-assigned score; a new pipeline run is required)
SOLVA GAUNTLET:        NOT RUN
INDEPENDENT ASSESSOR:  NONE
DOCUMENT SCOPE:        Design-level PDD. 30 prompts across 5 phases; not a full-depth build
                       worksheet. Expansion to build depth is a follow-on task.
TOKENECONOMIZER FACTS: Verified against source at microsoft/TokenEconomizer@d58755d
                       (2026-11-02). Paper figures as quoted by the repo. Tests not run.
CERTIFICATION CLAIMS:  None made by this document.
```

---

# APPENDIX A — Source map

| Topic | Source |
|---|---|
| ZPOS+5 methods, ICMS, API, hybrids, ROI, conformance | ZPOS_ICMS_ENGINE2 (project) |
| Alternate ZPOS+5 descriptions | SPHINX_ENGINE_SOFTWARE_DESIGN_DOCUMENT; MARKET_MATRIX_SPC_Designer_ATLAS_Summary; CRAFT_BOX_Software_Design_Document; THE_ANT_QUEEN_Features_Sheet |
| ATLAS worksheet format, phase colors | THE_ATOMIC_PROMPT_WHITEPAPER; ATLAS_ULTRA_SI_SPC_v1.0; SPARTAN_MAX_ATLAS_PDD |
| TokenEconomizer overview, settings, results, supported versions | [`README.md`](https://github.com/microsoft/TokenEconomizer/blob/d58755d849cd9157d29aef539f2beab3d5fee0fc/README.md) |
| Hook-by-hook design, plugin vs experiments, verified agent behavior, prompt caching, state, limitations | [`docs/PLUGIN.md`](https://github.com/microsoft/TokenEconomizer/blob/d58755d849cd9157d29aef539f2beab3d5fee0fc/docs/PLUGIN.md) |
| Seams (`Stage`, `DomainAdapter`, ledger, trace) | [`docs/ARCHITECTURE.md`](https://github.com/microsoft/TokenEconomizer/blob/d58755d849cd9157d29aef539f2beab3d5fee0fc/docs/ARCHITECTURE.md), `tokeneconomizer/stage.py`, `tokeneconomizer/domain.py` |
| Cost definition, price ratios | `tokeneconomizer/ledger.py`, `tokeneconomizer/pricing.py` |
| Fact lifecycle constants | `tokeneconomizer/microskills/lifecycle.py`, `microskills/envfacts.py` |
| Settings defaults and modes | `tokeneconomizer/plugin/config.py`, `plugin/README.md` |
| Protected / inapplicable tools | `tokeneconomizer/toolprune/tools.py` |
| Loop guard presets | `tokeneconomizer/loopguard/rules.py` (`SPIRAL_LEVELS`), `plugin/README.md` |
| Paper Table 1 rows | `docs/assets/results_overview.py` (`ROWS`) |
| Reproduction harness, arms, workloads | [`experiments/README.md`](https://github.com/microsoft/TokenEconomizer/blob/d58755d849cd9157d29aef539f2beab3d5fee0fc/experiments/README.md) |
| Paper | Li, Xu, Singh, Chen, Inala, Siska, Tamersoy, Yu, Stoica, Galley, Gao. *TokenEconomizer: Experience-Driven Black-Box Cost Control for Coding Agents.* arXiv:2610.22732 (2026) — not read directly |
| Secondary write-ups (v1.0, superseded where they conflict with the repo) | MarkTechPost; explainx.ai; AlphaSignal; Fluid Attacks (99% input claim, search snippet) |

# APPENDIX B — Glossary

| Term | Meaning |
|---|---|
| TO | Token Optimization (reading adopted in this PDD; see 0.1) |
| TE | TokenEconomizer |
| ICMS / ICMS-L | Intelligent Compression Methodology Selector / learning variant (proposed) |
| Verbatim zone | Content never lossy-compressed |
| Freeze | Compiled prompt fixed and hash-pinned for the life of a session, replayed byte-identical on resume |
| Epoch | Period during which one set of compiled-prefix hashes is deployed in a repository |
| Micro-skill | One-line, confirmed repository fact injected at session start (TE) |
| envfacts / distiller | TE's model-free fact families / LLM fact proposer |
| Firing gate | TE rule: inject an active fact only while its waste occurs (or is prevented) in ≥40% of ≥4 exercising sessions |
| Deny list | TE's tool-pruning output, written into the agent's own settings file |
| Envelope | TE's learned usual length of a well-ended turn in a repository |
| Ended green | Turn where the agent stopped on its own and no check after the last edit failed |
| Backstop | TE rule: a churning turn past `backstop_mult` × envelope is redirected without the judge |
| Shadow mode | TE `shadow=true`: decide and log, act on nothing |
| Effective tokens | TE cache-aware cost in fresh-input-token equivalents, per billed model id |
| Pass@1 | Share of tasks solved on the first attempt (official grader) |
| Component Ledger | TO's per-component and per-epoch attribution layered on TE's ledger |

# APPENDIX C — v1.0 → v1.1 correction ledger

Each row is a v1.0 statement checked against `microsoft/TokenEconomizer@d58755d`.

| # | v1.0 said | Repository shows | Action in v1.1 |
|---|---|---|---|
| C1 | TE paper and repository not read; all facts second-hand | Repository read in full for the cited files | Provenance rewritten; [SRC] tag added |
| C2 | Micro-skill: "injected only after 3 sessions confirm it; a contradicting session drops it" | 3 confirmations (code facts: on sessions that edited different files); retirement by recency score with 5× contradiction weight, instant only for high-stakes facts; command-spelling facts need two contradicting sessions in a row; **plus** a 40%/≥4 firing gate | §2.2 rewritten |
| C3 | Facts learned from failures (one source) | Two sources: envfacts (five model-free families) and an LLM proposer every k = 3 sessions | §2.2; FCT prompts re-cut |
| C4 | Pruning thresholds 10 prompts / 2 sessions / 5 sessions; idle 60 min | Confirmed; also: 3× for `NotebookEdit`/`image_media`, protected work surface and interactive tools, `toolprune_protect`, a one-day write deadline, `propose` mode, evidence from a 61-session / 893-prompt replay | §2.2, §3.3 |
| C5 | "Restored tools are not pruned again" | Confirmed, unless re-applied by name (`toolprune apply NAME`) | §2.2 |
| C6 | Loop guard "skip judge if turn exceeds 2.5× usual" | Inverse: past 2.5× the envelope *and* churning, the turn is redirected **without** the judge (backstop) | §2.2, §3.3, LPG-SAFE-021 |
| C7 | Loop guard thresholds: 70% confidence, 1 redirect, 5 good turns | Confirmed; plus `min_turns` 20, `gate_min_turns` 18, three presets, detached judge job, notice after redirect | §2.2, §3.3 |
| C8 | "Neither parent publishes cost attribution as a first-class artifact" | TE has a canonical cache-aware ledger with optimizer overhead; it lacks *content-component* attribution | §2.4 corrected; TO-LDG now extends TE's ledger |
| C9 | Pass@1 dip "74% to 73%" (secondary) | Confirmed: Claude Code + Opus 4.8, 74.0 → 73.0; full Table 1 added | §2.2 table |
| C10 | Three mechanisms | Three plus an opt-in fourth (repository map via aider) and a `shadow` mode | §2.2; R9 added |
| C11 | Pruning varies by agent (single tools / packages / MCP servers / nothing) | Confirmed, with settings-file locations and trust requirements; opencode and Cursor also prune nothing | §2.6 |
| C12 | Runtime layer to be built (TO-PRN, TO-FCT, TO-LPG, TO-WRT) | A complete, tested, Apache-2.0 implementation with stable seams exists | **Architecture changed: adopt + pin + extend.** Build prompts replaced by adopt/configure/verify prompts; v1.0 data tables for tool usage, failures and fact candidates dropped |
| C13 | Replicate "on its own harness" | Harness supports SWE-Bench and Web-Bench reruns; security sets are not run or graded by it; **the plugin differs from the experiment arms** in the twelve ways its `docs/PLUGIN.md` §2 table lists | VAL-REPL-024 now measures harness and plugin separately |
| C14 | Static layer may compress tool descriptions (PRISM) | TE is black-box: it never alters agent tool descriptions, and neither can TO for built-in tools | Scope limited to tools we own |

*Junglenomics FORGE · JNGL-PDD-ZPOSTO-2026-001 · v1.1 DRAFT · 2026-11-03*
