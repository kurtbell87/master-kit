# PRD: Master Orchestrator Monorepo for TDD + Research + Lean Math Kits

## 1) Overview

Build a **single monorepo** (“master kit”) that hosts and composes three existing kits:

* **TDD kit** (language-agnostic)
* **Research kit** (language-agnostic)
* **Mathematics kit** (Lean4 + Mathlib)

The system embraces **fractal / nested LLM execution** (each phase uses an LLM as CPU) while preventing:

* **context bloat** (no transcript-passing between LLMs)
* **LLM infinity mirrors** (uncontrolled recursion / repeated re-summarization)
* **loss of observability** (hard-to-debug, no global trace)

It does this by enforcing **artifact-first interop**:

* phases communicate via **bounded capsules** + **pointer manifests**
* the “truth” stays on disk; only addresses cross boundaries

---

## 2) Problem Statement

Each kit already has a disciplined phase model and safety rails (hooks). But when combined:

1. **Hook collision**: each kit installs `.claude/hooks/pre-tool-use.sh` and `.claude/settings.json`. Only one can win.
2. **Interop ambiguity**: research → math → tdd handoffs are not standardized; ad-hoc passing leads to token bloat.
3. **Observability gap**: each kit logs internally; there is no **global run index** to correlate across kits/phases.
4. **Efficiency**: without strict boundaries, nested LLM sessions can recursively consume large logs/specs and balloon context.

---

## 3) Goals

### G1 — Seamless interop between kits

* Any phase can request work from another kit (e.g., Research asks Math to formalize/prove a lemma; Math asks Research to run an experiment; TDD asks Research to evaluate approaches).

### G2 — No transcript passing

* Downstream LLMs receive **capsule + manifest** only (bounded). Everything else is referenced by file path.

### G3 — Maximum observability

* Every run/phase/handoff produces:

  * `capsule.md` (bounded, human-readable)
  * `manifest.json` (lossless pointers + hashes)
  * `events.jsonl` (global append-only run trace)

### G4 — Preserve existing kit behavior

* Kits remain runnable as they are today (direct script usage still works).
* Master orchestration adds interop and observability without breaking single-kit workflows.

### G5 — Guardrails against context bloat

* Default budget limits on reading large artifacts (especially logs).
* Encourage “query by pointer” (tail/grep/classify) instead of reading full files.

---

## 4) Non-Goals (v1)

* Distributed execution, concurrency, remote queues, web UI dashboards
* Multi-user tenancy and permissions
* Perfect artifact diffing of every file in the repo (v1 uses bounded tracking)

---

## 5) Monorepo Layout

You will clone the three repos into the master repo. Recommended structure:

```
master-kit/
  kits/
    claude-tdd-kit/           (cloned)
    claude-research-kit/      (cloned)
    claude-mathematics-kit/   (cloned, Lean)
  .claude/
    hooks/
      pre-tool-use.sh         (MASTER dispatcher)
    settings.json             (MASTER settings)
    prompts/                  (optional: consolidated; v1 can keep kit prompts where they are)
  interop/
    requests/
    responses/
    schemas/
  runs/
    <run_id>/
      events.jsonl
      capsules/
      manifests/
      artifacts/              (optional: copied/linked deliverables)
  tools/
    kit                       (master CLI entrypoint)
    pump                      (request executor)
    observe                   (optional helper: summarize run)
  docs/
    PRD_MASTER_KIT.md         (this PRD)
```

**Key decision (v1):** keep each kit’s scripts intact; master adds wrappers and a hook dispatcher.

---

## 6) Core Concepts & Contracts

### 6.1 Run IDs

Every top-level invocation gets a `run_id` (timestamp + short hash), propagated via env vars:

* `RUN_ID`
* `RUN_ROOT=runs/<run_id>`

### 6.2 Capsule (bounded, cross-LLM payload)

File: `runs/<run_id>/capsules/<kit>_<phase>.md`

Hard requirements:

* ≤ 30 lines
* no code blocks
* no raw logs
* includes **only pointers** to truth artifacts

Template:

* Goal:
* What happened:
* Current status:
* Next action requested (exactly one):
* Evidence pointers:
* If blocked: error signature + where to find full trace:

**How it’s produced (v1):**

* Preferably printed by the LLM at end of the phase using markers:

  * `===CAPSULE===` … `===/CAPSULE===`
* The master wrapper extracts and writes it to the capsule file.
* This avoids violating “read-only” phases (e.g., Math SURVEY).

### 6.3 Manifest (lossless pointers + hashes)

File: `runs/<run_id>/manifests/<kit>_<phase>.json`

Contains:

* metadata: `run_id`, `kit`, `phase`, timestamps, exit_code
* artifact index: list of `{path, kind, bytes, sha256}`
* truth pointers: canonical “source of truth” files for this kit/phase
* log pointers: `{path, kind, hint}` (hint = how to query without reading all)
* optional: metrics summary pointers

**v1 artifact collection approach:**

* Configured allowlist of tracked paths/globs per kit (see §10).
* On phase end: enumerate files under allowlist, record sizes + hashes (bounded cap by count/bytes; overflow recorded as “omitted”).

### 6.4 Interop Request (handoff between kits)

File: `interop/requests/<request_id>.json`

Minimal schema:

* `request_id`
* `from_kit`, `to_kit`
* `action` (e.g., `math.full`, `research.run`, `tdd.red`)
* `args` (CLI-like array)
* `run_id` (parent)
* `inputs` (paths only)
* `must_read` (capsule/manifest/truth pointers)
* `read_budget`:

  * `max_files` (default 8)
  * `max_total_bytes` (default 300k)
  * `allowed_paths` (globs)
* `deliverables_expected` (paths/patterns)
* `priority` (optional)

### 6.5 Interop Response

File: `interop/responses/<request_id>.json`

* `request_id`
* `status` (`ok|blocked|failed`)
* `child_run_id`
* `capsule_path` (child capsule pointer)
* `manifest_path` (child manifest pointer)
* `deliverables` (paths)
* `notes` (short string; no long content)

### 6.6 Global Event Stream

File: `runs/<run_id>/events.jsonl` (append-only)

Each line is JSON with:

* `ts`
* `event` (phase_started, phase_finished, request_enqueued, request_completed, artifact_indexed, etc.)
* `kit`, `phase` (if relevant)
* `pointers` (capsule/manifest paths)
* `exit_code` (if relevant)

This is the **global trace** for observability and replay.

---

## 7) Unified Hook Dispatcher (Critical)

### Problem

All three kits have their own `.claude/hooks/pre-tool-use.sh` with different policies. In a monorepo, only one hook can be active.

### Solution

Create a **master hook** at:

* `.claude/hooks/pre-tool-use.sh`

It will:

1. Detect which kit/phase is active by env vars:

   * `TDD_PHASE`, `EXP_PHASE`, `MATH_PHASE`
2. Apply the corresponding enforcement rules (ported from each kit hook).
3. Add global bloat guardrails:

   * **Read budget**: block `Read` tool on large files unless whitelisted
   * Encourage query-by-pointer (tail/grep/classify scripts)

**v1 approach to porting:**

* Copy each kit hook logic into functions inside the master hook, then call them conditionally.
* Ensure semantics remain unchanged for each kit’s critical restrictions.

### Global Read Guardrail (v1)

When `CLAUDE_TOOL_NAME == "Read"`:

* Extract file path from `CLAUDE_TOOL_INPUT`
* If file size > `MAX_READ_BYTES` (default 200k) and not under allowed globs:

  * BLOCK with message: “Use query approach (tail/grep) or add pointer to must_read allowlist.”

---

## 8) Master CLI & Orchestration

### 8.1 `tools/kit` (entrypoint)

Responsibilities:

* create `run_id`
* set `RUN_ROOT`
* append run start to `events.jsonl`
* invoke the desired kit phase (shelling out to existing scripts)
* capture stdout/stderr to `runs/<run_id>/logs/<kit>_<phase>.log`
* extract capsule markers from stdout
* build manifest and append events

Example commands:

* `tools/kit tdd red docs/my-feature.md`
* `tools/kit research run experiments/exp-001.md`
* `tools/kit math full specs/construction-foo.md`

### 8.2 `tools/pump` (interop executor)

Responsibilities:

* poll `interop/requests/`
* for each request:

  * run child action via `tools/kit ...`
  * write `interop/responses/<id>.json`
  * append events to parent run and/or child run (v1 can just log in child + write response pointer)

### 8.3 Nested LLM calls (supported, efficient)

A phase can request another kit by:

* writing a request file (via `tools/kit request ...` helper or a simple bash utility)
* calling `tools/pump --once --request <id>` (or letting a parent orchestrator pick it up)

**Efficiency guarantee:**

* the child run starts with only:

  * request JSON
  * parent capsule/manifest pointers
  * must_read allowlist
* no transcript is passed.

---

## 9) User Flows

### Flow A: Research → Math proof

1. Research phase concludes it needs a lemma formalized.
2. Research writes `interop/requests/RQ123.json` with:

   * `to_kit=math`, `action=math.full`, args: spec path
   * must_read: research capsule + experiment spec + one results summary
3. Pump runs Math, generating:

   * `runs/<child>/capsules/math_full.md`
   * `runs/<child>/manifests/math_full.json`
4. Response points Research to deliverables:

   * Lean files / results / audit log pointer

### Flow B: Math → Research experiment

Math audit identifies an empirical ambiguity; requests research run:

* `action=research.run` with an experiment spec to validate a conjecture.

### Flow C: TDD → Research

TDD is stuck on architecture choice; requests a structured research survey and synthesis.

---

## 10) Default “Truth” Pointers & Artifact Tracking

### TDD

Truth pointers (default):

* `PRD.md`
* `LAST_TOUCH.md`
* `docs/**` (feature specs)
* `tests/**` (spec-as-tests)
* test output log pointer

Tracked globs:

* `docs/**`
* `src/**`
* `tests/**`
* `PRD.md`, `LAST_TOUCH.md`

### Research

Truth pointers:

* `QUESTIONS.md`
* `experiments/**`
* `results/**/metrics.*`
* `analysis.md`, `SYNTHESIS.md`
* `RESEARCH_LOG.md`

Tracked globs:

* `experiments/**`
* `results/**`
* `analysis.md`, `SYNTHESIS.md`, `RESEARCH_LOG.md`
* `handoffs/**`

### Math (Lean)

Truth pointers:

* `specs/**`
* `CONSTRUCTIONS.md` (if using program mode)
* `CONSTRUCTION_LOG.md`
* `REVISION.md`
* `results/**`
* Lean build log pointer

Tracked globs:

* `specs/**`
* `**/*.lean`
* `CONSTRUCTION_LOG.md`, `REVISION.md`, `CONSTRUCTIONS.md`
* `results/**`

---

## 11) Implementation Plan & Milestones

### Milestone 0 — Monorepo bootstrap

**Tasks**

* [ ] Create `runs/`, `interop/requests`, `interop/responses`, `tools/`, `docs/`
* [ ] Add `docs/PRD_MASTER_KIT.md` (this document)
* [ ] Ensure `.gitignore` covers run artifacts and logs appropriately

**Acceptance**

* Repo has the standard directories and PRD committed.

---

### Milestone 1 — Master hook dispatcher

**Tasks**

* [ ] Implement `.claude/hooks/pre-tool-use.sh` master hook
* [ ] Port rules from:

  * TDD green protections
  * Research run/read/synthesize protections
  * Math universal + phase protections
* [ ] Add global `Read` guardrail (max bytes + allowlist)

**Acceptance**

* Running each kit standalone still enforces the same restrictions as before.
* Attempts to `Read` a large log file directly are blocked unless whitelisted.

---

### Milestone 2 — Run scaffolding + event stream

**Tasks**

* [ ] Implement `tools/kit` that:

  * creates run_id
  * writes `runs/<run_id>/events.jsonl`
  * captures stdout/stderr logs
  * extracts capsule markers into `capsules/`
  * builds manifest into `manifests/`
* [ ] Implement artifact tracking via allowlisted globs

**Acceptance**

* A single `tools/kit <kit> <phase> ...` run produces:

  * `events.jsonl`
  * at least one capsule
  * at least one manifest
  * logs captured under `runs/<run_id>/logs/`

---

### Milestone 3 — Interop requests + pump

**Tasks**

* [ ] Define JSON schemas in `interop/schemas/`
* [ ] Implement `tools/pump`:

  * `--once` executes one request
  * writes response JSON
  * runs child via `tools/kit`
* [ ] Provide helper `tools/kit request ...` (optional) to author requests consistently

**Acceptance**

* Research can enqueue a Math request and receive a response pointing to child capsule/manifest.
* No transcripts are copied into request/response—only pointers.

---

### Milestone 4 — Efficiency policies (anti-bloat)

**Tasks**

* [ ] Enforce read budgets:

  * `must_read` allowlist honored
  * large `Read` blocked unless explicitly allowed
* [ ] Provide recommended log-query helpers:

  * `tail`, `grep`, and Lean error summarizers
* [ ] Add “capsule strictness” validator script (fails CI if capsule too long)

**Acceptance**

* Cross-kit handoffs remain small and stable.
* A run cannot accidentally balloon by reading multi-MB logs into context.

---

### Milestone 5 — CI & regression tests

**Tasks**

* [ ] Add lightweight tests:

  * master hook respects each kit’s rules (golden tests with sample inputs)
  * capsule length validator
  * manifest schema validation
* [ ] Add a “smoke run” workflow (no actual training; just file ops) to verify structure

**Acceptance**

* CI prevents regressions in enforcement and capsule/manifest invariants.

---

## 12) Technical Notes / Design Constraints

* **Read-only phases must remain read-only.** Capsules should be printed to stdout and extracted by wrapper, not written by the LLM in those phases.
* **No changes to Lean cache discipline.** Math rules that forbid `lake clean` remain enforced.
* **One `.claude/settings.json`.** It must point to the master hook; per-kit prompt files can remain wherever scripts expect them.

---

## 13) Risks & Mitigations

**Risk:** Hook dispatcher subtly diverges from kit-specific enforcement.
**Mitigation:** Port rules verbatim; add tests using captured tool inputs.

**Risk:** Artifact tracking is too expensive (hashing big trees).
**Mitigation:** Allowlist + caps (max files / max bytes); record “omitted”.

**Risk:** Agents ignore capsule discipline.
**Mitigation:** Capsule validator + prompts that instruct “capsule only; pointers only”; global `Read` guardrail.

---

## 14) Acceptance Criteria (Definition of Done)

* ✅ Monorepo runs any kit phase via `tools/kit` and produces:

  * capsule + manifest + events + logs
* ✅ Interop request/response works across kits without transcript passing
* ✅ Master hook enforces all three kits’ restrictions correctly
* ✅ Default anti-bloat rules prevent large direct reads and encourage query-by-pointer
* ✅ Observability: given a `run_id`, you can reconstruct what happened and where the truth artifacts are without opening transcripts

---

## 15) Quick Start (for the agent implementing this)

1. Implement master hook dispatcher first (it’s the main collision point).
2. Implement `tools/kit` to create run dirs, capture logs, extract capsule markers, and write manifest + events.
3. Implement `tools/pump` + request/response schemas.
4. Add capsule validator + schema validator + basic CI smoke tests.

---

## 16) Implementation Status (Updated 2026-02-13)

### 16.1 Hook Re-entry Guard Contract

The master and kit hook integration now enforces a strict non-recursive delegation contract:

1. Master hook (`.claude/hooks/pre-tool-use.sh`) sets:
   * `MASTER_HOOK_ACTIVE=1`
   * `MASTER_KIT_ROOT` (defaults to repo root if unset)
2. Master dispatches to kit hooks with `MASTER_HOOK_ACTIVE=1 "$hook_path"`.
3. Kit hooks delegate back to master only when:
   * `MASTER_KIT_ROOT` is set, and
   * `MASTER_HOOK_ACTIVE != 1`
4. After kit-to-master delegation, kit hook exits immediately (`exit $?`) to avoid double enforcement.

This prevents re-entry loops of the form:
`master -> kit -> master -> kit -> ...`

### 16.2 Optional Hook Debug Tracing

Master hook supports opt-in debug tracing:

* env: `MASTER_HOOK_DEBUG=1`
* stderr markers:
  * `MASTER_HOOK: enter`
  * `MASTER_HOOK: dispatch tdd|research|math`

Default behavior is unchanged when `MASTER_HOOK_DEBUG` is unset.

### 16.3 Regression Coverage

Current automated coverage includes:

* `tests/test_hook_reentry_guard.py`
  * `delegation_happens_once` subtest: invoking a kit hook with `MASTER_KIT_ROOT` set yields exactly one block and one master entry/dispatch trace.
  * `master_dispatch_does_not_recurse` subtest: invoking master directly with `MASTER_HOOK_ACTIVE=1` yields exactly one block and one master entry/dispatch trace.
* `tests/test_master_hook.py`
  * read budget large-file block + allowlist bypass
  * unique-file budget overflow block
  * allowlisted must-read pointer does not consume unique-file budget
  * TDD/Research/Math phase enforcement smoke checks
* `tests/test_validators.py`
  * capsule validator + manifest validator regression checks

### 16.4 Validation Commands

Use these commands for hook and anti-bloat regression validation:

* `python3 -m pytest -q tests/test_hook_reentry_guard.py`
* `python3 -m pytest -q tests/test_master_hook.py`
* `python3 -m pytest -q`
* `tools/smoke-run`
