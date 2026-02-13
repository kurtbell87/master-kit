# Claude Mathematics Kit

Lean4/Mathlib construction workflow for Claude Code with staged formalization and proof guardrails.

## Monorepo Context

This kit can run standalone, but in this repository it is usually orchestrated through Master-Kit.

- Repository overview: `../README.md`
- Canonical PRD: `../docs/PRD_MASTER_KIT.md`

## Phase Model

| Phase | Command | Purpose |
|---|---|---|
| SURVEY | `./math.sh survey <spec-file>` | Survey Mathlib and domain context |
| SPECIFY | `./math.sh specify <spec-file>` | Define precise requirements |
| CONSTRUCT | `./math.sh construct <spec-file>` | Produce informal construction/proof sketch |
| FORMALIZE | `./math.sh formalize <spec-file>` | Write Lean declarations with placeholders |
| PROVE | `./math.sh prove <spec-file>` | Resolve proofs through build loops |
| AUDIT | `./math.sh audit <spec-file>` | Verify coverage and policy compliance |
| LOG | `./math.sh log <spec-file>` | Commit and PR |
| FULL | `./math.sh full <spec-file>` | Run end-to-end with revision handling |

Additional commands:

- `./math.sh status`
- `./math.sh program [--max-cycles N] [--resume]`
- `./math.sh watch [phase] [--resolve]`

## Quick Start

```bash
# Configure
$EDITOR math.sh

# One full cycle
./math.sh survey specs/my-construction.md
./math.sh specify specs/my-construction.md
./math.sh construct specs/my-construction.md
./math.sh formalize specs/my-construction.md
./math.sh prove specs/my-construction.md
./math.sh audit specs/my-construction.md
./math.sh log specs/my-construction.md
```

Or run:

```bash
./math.sh full specs/my-construction.md
```

## Watch and Logs

```bash
./math.sh watch prove
./math.sh watch prove --resolve
```

- Phase logs: `$MATH_LOG_DIR/{phase}.log` (default `/tmp/math-<project>/`)
- Stdout remains compact; detailed Lean/build traces are stored in logs

## Guardrails

Defense in depth:

1. Phase-specific prompts
2. OS locking of specs during PROVE and `.lean` files during AUDIT
3. Hook policies blocking unsafe or non-auditable patterns

Universal blocks include:

- `axiom`
- `unsafe`
- `native_decide`
- `admit`
- destructive git and permission bypass commands

## Program Mode

Program mode executes queued constructions from `CONSTRUCTIONS.md` and respects dependency ordering.

```bash
./math.sh program
./math.sh program --max-cycles 10
./math.sh program --resume
```

## Configuration

Set in `math.sh`:

```bash
LEAN_DIR="."
SPEC_DIR="specs"
LAKE_BUILD="lake build"
MAX_REVISIONS="3"
MAX_PROGRAM_CYCLES="20"
```

Optional PR controls:

- `MATH_AUTO_MERGE`
- `MATH_DELETE_BRANCH`
- `MATH_BASE_BRANCH`

## Aliases (Optional)

```bash
source math-aliases.sh

math-survey specs/my-construction.md
math-specify specs/my-construction.md
math-construct specs/my-construction.md
math-formalize specs/my-construction.md
math-prove specs/my-construction.md
math-audit specs/my-construction.md
math-log specs/my-construction.md
math-full specs/my-construction.md
math-program
math-status
math-sorrys
math-axioms
math-unlock
```

## Standalone Install

For non-monorepo projects only:

```bash
/path/to/claude-mathematics-kit/install.sh
```

In this monorepo, use `tools/bootstrap` at repo root instead.

## Troubleshooting

- Lean toolchain issues: verify `elan`, `lake`, and project toolchain compatibility.
- Locked files after interruption: run `math-unlock`.
- Repeated proof failures: inspect phase log and `REVISION.md`, then restart at recommended phase.
