# Claude TDD Kit

A drop-in red-green-refactor TDD workflow for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Three separate AI agents -- a test author, an implementer, and a refactorer -- each with enforced guardrails that prevent them from stepping outside their role.

## How It Works

```
You write a spec ──► tdd.sh red      ──► Test Author agent writes failing tests
                     tdd.sh green    ──► Implementation agent makes them pass (tests are OS-locked)
                     tdd.sh refactor ──► Refactoring agent improves quality (tests stay green)
                     tdd.sh ship     ──► Commit, PR, archive spec (preserved in git history)
```

**Key idea:** Each phase spawns a separate Claude agent with a dedicated system prompt. During the GREEN phase, test files are `chmod 444` (read-only) and a pre-tool-use hook blocks any attempt to modify, revert, or unlock them. The implementation agent *must* write code that satisfies the tests -- it cannot cheat.

### Defense in Depth

The kit uses three layers of enforcement:

| Layer | Mechanism | What it prevents |
|-------|-----------|-----------------|
| **System prompt** | Phase-specific agent identity | Agent "wants" to stay in role |
| **File permissions** | `chmod 444` on test files | OS blocks writes even if agent tries |
| **Pre-tool-use hook** | Blocks `chmod`, `git restore`, test file edits | Prevents workarounds to bypass permissions |

## Quick Start

```bash
# Clone or copy the kit into your project
cd your-project
/path/to/claude-tdd-kit/install.sh

# Configure for your project
$EDITOR tdd.sh          # Set BUILD_CMD, TEST_CMD, TEST_DIRS
$EDITOR PRD.md           # Define what you're building
$EDITOR LAST_TOUCH.md    # Set current project state

# Write your first spec
cat > docs/my-feature.md << 'EOF'
# My Feature

## Requirements
- The `add(a, b)` function returns the sum of two numbers
- Negative numbers are supported
- Non-numeric inputs raise TypeError

## Acceptance Criteria
- All happy-path cases pass
- Edge case: very large numbers
- Error case: string inputs
EOF

# Run the TDD cycle
./tdd.sh red docs/my-feature.md    # Agent writes failing tests
./tdd.sh green                      # Agent implements to pass
./tdd.sh refactor                   # Agent cleans up
./tdd.sh ship docs/my-feature.md   # Commit, PR, archive spec

# Or run all four in sequence
./tdd.sh full docs/my-feature.md
```

## Monitoring a Running Phase

Each phase streams structured JSON logs to `$TDD_LOG_DIR/{phase}.log` (defaults to `/tmp/tdd-<project-name>/`). The built-in `watch` command parses these into a live dashboard:

```bash
./tdd.sh watch green              # Live-tail the green phase
./tdd.sh watch refactor           # Live-tail the refactor phase
./tdd.sh watch red --resolve      # One-shot summary of a completed phase
./tdd.sh watch                    # Auto-detect the most recent phase
```

The dashboard shows elapsed time, model, tool call counts, files read/written/edited, agent narration, and test results.

Phase runners redirect all sub-agent output to disk only (`$TDD_LOG_DIR/{phase}.log`). Stdout receives a compact summary — the last agent message, exit code, and log path — so the orchestrator's context window stays clean. If you need more detail, grep or read the log file directly.

### What the Orchestrator Sees

Each `./tdd.sh` phase returns **only** a compact summary on stdout:

- **Phase output** goes to `$TDD_LOG_DIR/{phase}.log` (not stdout)
- **Stdout receives:** last agent message (≤500 chars) + exit code + log path
- **To get more detail:** grep or read the log file (pull-based)
- **The watch script** reads logs directly and is unaffected by this

This is intentional — it prevents sub-agent verbosity from flooding the orchestrator's context window. The orchestrator should treat the summary as the primary signal and only pull from the log when diagnosing a failure.

## What Gets Installed

```
your-project/
├── tdd.sh                          # Orchestrator script (configure this)
├── tdd-aliases.sh                  # Shell aliases (optional, source in .bashrc)
├── CLAUDE.md                       # Instructions for Claude (TDD workflow rules)
├── LAST_TOUCH.md                   # Project state tracking (handoff document)
├── PRD.md                          # Product requirements template
├── docs/                           # Spec files go here
│   └── my-feature.md
├── scripts/
│   └── tdd-watch.py                # Live dashboard for monitoring phases
└── .claude/
    ├── settings.json               # Hook registration
    ├── hooks/
    │   └── pre-tool-use.sh         # GREEN phase enforcement hook
    └── prompts/
        ├── tdd-red.md              # Test Author agent prompt
        ├── tdd-green.md            # Implementation agent prompt
        └── tdd-refactor.md         # Refactoring agent prompt
```

## Configuration

Edit the top of `tdd.sh` to match your project:

```bash
# Test directories (space-separated)
TEST_DIRS="tests"                    # e.g., "tests python/tests src/__tests__"

# Source directory
SRC_DIR="src"                        # Where implementation code lives

# Build and test commands (injected into agent context)
BUILD_CMD="npm run build"            # e.g., "make", "cargo build", "cmake --build build"
TEST_CMD="npm test"                  # e.g., "pytest", "cargo test", "ctest --test-dir build"
```

### Log Directory

Logs are written to a per-project directory under `/tmp/`, derived from the git repo name:

```bash
# Project "my-app" writes to /tmp/tdd-my-app/
# Project "api-service" writes to /tmp/tdd-api-service/

# Override if needed:
TDD_LOG_DIR="/tmp/my-custom-dir" ./tdd.sh green
```

This prevents log collisions when multiple projects use the kit simultaneously.

### Post-Cycle PR Settings

```bash
TDD_AUTO_MERGE="false"               # Set "true" to auto-merge PRs after creation
TDD_DELETE_BRANCH="false"             # Set "true" to delete feature branch after merge
TDD_BASE_BRANCH="main"               # Base branch for PRs (default: main)
```

Or set them as environment variables:

```bash
BUILD_CMD="make" TEST_CMD="make test" ./tdd.sh red docs/feature.md

# Auto-merge and cleanup for a fully autonomous cycle:
TDD_AUTO_MERGE=true TDD_DELETE_BRANCH=true ./tdd.sh full docs/feature.md
```

## The Three Documents

The workflow revolves around three living documents:

| Document | Purpose | Who updates it |
|----------|---------|---------------|
| **PRD.md** | Project requirements, architecture, build order | You (once, then occasionally) |
| **LAST_TOUCH.md** | What's been built, what's next, current test count | You or Claude (after each cycle) |
| **docs/\<feature\>.md** | Spec for a single TDD cycle (archived to git after ship) | You or Claude (Step 1 of each cycle) |

## Shell Aliases (Optional)

Source the aliases file for shorter commands:

```bash
source tdd-aliases.sh

tdd-red docs/feature.md    # Same as ./tdd.sh red docs/feature.md
tdd-green                   # Same as ./tdd.sh green
tdd-refactor                # Same as ./tdd.sh refactor
tdd-ship docs/feature.md   # Same as ./tdd.sh ship docs/feature.md
tdd-full docs/feature.md   # Same as ./tdd.sh full docs/feature.md
tdd-status                  # Show which test files are locked/unlocked
tdd-unlock                  # Emergency: restore write permissions on all test files
```

## Integrating with an Existing CLAUDE.md

If you already have a `CLAUDE.md`, the installer won't overwrite it. Append the TDD workflow rules manually:

```bash
cat templates/CLAUDE.md.snippet >> CLAUDE.md
```

The snippet tells Claude to orchestrate the TDD pipeline instead of writing code directly.

## How the Hook Works

During the GREEN phase, `.claude/hooks/pre-tool-use.sh` fires before every Edit, Write, MultiEdit, and Bash tool call. It blocks:

1. **Permission escalation**: `chmod`, `chown`, `sudo`, `doas`
2. **Git reverts**: `git checkout`, `git restore`, `git stash`, `git reset`
3. **Test file writes**: redirect (`>`), `tee`, `sed -i`, `mv`, `cp`, `rm` targeting test files
4. **Direct edits**: Edit/Write/MultiEdit targeting files matching test patterns

Outside the GREEN phase (RED, REFACTOR, or normal use), the hook does nothing.

## Customizing Agent Prompts

The agent prompts in `.claude/prompts/` are designed to be language-agnostic. You can customize them for your project:

- **Add project-specific conventions** (e.g., "use fixtures from conftest.py")
- **Add language-specific guidance** (e.g., "use GTest TEST_F for stateful tests")
- **Add build-specific instructions** (e.g., "run `npm run lint` after changes")

The `## Context` section is appended dynamically by `tdd.sh` at runtime -- you don't need to hardcode paths.

## Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed and authenticated
- [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated (for `ship` phase)
- Bash 4+
- A project with a test framework already set up

## Troubleshooting

**Tests are stuck as read-only after a crash:**
```bash
tdd-unlock          # If you sourced tdd-aliases.sh
# or
./tdd.sh green      # The EXIT trap will unlock on completion
# or manually
find . -name "test_*" -exec chmod 644 {} \;
```

**Hook isn't firing:**
- Check `.claude/settings.json` exists and has the PreToolUse hook registered
- Run `chmod +x .claude/hooks/pre-tool-use.sh`

**Agent ignores locked tests and errors out:**
- This is expected! The GREEN agent should see "permission denied" and implement code instead of modifying tests. If it keeps retrying, the prompt may need strengthening for your use case.

## License

MIT -- use it however you like.
