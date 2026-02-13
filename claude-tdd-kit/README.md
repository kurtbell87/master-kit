# Claude TDD Kit

Role-focused red-green-refactor orchestration for Claude Code.

## Monorepo Context

This kit can run standalone, but in this repository it is usually orchestrated by Master-Kit.

- Repository overview: `../README.md`
- Canonical PRD: `../docs/PRD_MASTER_KIT.md`

## Phase Model

| Phase | Command | Purpose |
|---|---|---|
| RED | `./tdd.sh red <spec-file>` | Write failing tests from spec |
| GREEN | `./tdd.sh green` | Implement to make tests pass |
| REFACTOR | `./tdd.sh refactor` | Improve implementation with tests still green |
| SHIP | `./tdd.sh ship <spec-file>` | Commit, PR, archive spec |
| FULL | `./tdd.sh full <spec-file>` | Run RED -> GREEN -> REFACTOR -> SHIP |

## Quick Start

```bash
# Configure for your project
$EDITOR tdd.sh
$EDITOR PRD.md
$EDITOR LAST_TOUCH.md

# Run one cycle
./tdd.sh red docs/my-feature.md
./tdd.sh green
./tdd.sh refactor
./tdd.sh ship docs/my-feature.md
```

## Watch and Logs

```bash
./tdd.sh watch green
./tdd.sh watch refactor
./tdd.sh watch red --resolve
```

- Phase logs: `$TDD_LOG_DIR/{phase}.log` (default `/tmp/tdd-<project>/`)
- Stdout remains compact by design (summary + exit code + log path)

## Guardrails

Defense in depth:

1. Phase-specific prompts
2. OS permissions (`chmod 444` on test files during GREEN)
3. Pre-tool-use hook blocks test modification bypasses (`chmod`, destructive git, test-file writes)

## Configuration

Set in `tdd.sh`:

```bash
TEST_DIRS="tests"
SRC_DIR="src"
BUILD_CMD="npm run build"
TEST_CMD="npm test"
```

Optional env:

- `TDD_LOG_DIR`
- `TDD_AUTO_MERGE`
- `TDD_DELETE_BRANCH`
- `TDD_BASE_BRANCH`

## Aliases (Optional)

```bash
source tdd-aliases.sh

tdd-red docs/feature.md
tdd-green
tdd-refactor
tdd-ship docs/feature.md
tdd-full docs/feature.md
tdd-status
tdd-unlock
```

## Standalone Install

For non-monorepo projects only:

```bash
/path/to/claude-tdd-kit/install.sh
```

In this monorepo, use `tools/bootstrap` at repo root instead.

## Troubleshooting

- Hook not firing: ensure `.claude/settings.json` registers the hook and `chmod +x .claude/hooks/pre-tool-use.sh`.
- Tests left read-only after interruption: run `tdd-unlock` (or restore permissions manually).
- GREEN blocks on test edits: expected behavior; implementation should change source code, not tests.
