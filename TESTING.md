# Testing Philosophy

## Intellectual Lineage

This testing methodology is derived from two posts by Aleksey Kladov (matklad):

- [How to Test](https://matklad.github.io/2021/05/31/how-to-test.html)
- [Unit and Integration Tests](https://matklad.github.io/2022/07/04/unit-and-integration-tests.html)

and adapted for composable shell tools that wrap impure operations (LLM calls, network IO, filesystem state) behind swappable backends.

---

## Core Principles

### 1. Forget "unit" vs "integration" — think purity and extent

The unit/integration distinction is confused and unproductive. Instead, classify every test on two independent axes:

**Purity** — how much generalized IO does the test perform? Pure tests (single-threaded computation, no disk, no network, no subprocesses) are fast, deterministic, and impossible to make flaky. Each layer of impurity — threads, filesystem, network, external processes — adds half an order of magnitude to runtime and a step function of flakiness. Purity is the axis worth ruthlessly optimizing.

**Extent** — how much of the codebase does the test exercise? A test can drive your entire system through a single entry point and still be pure. Extent is mostly diagnostic (it tells you about your architecture) but is not itself a metric to minimize. Artificially limiting extent by mocking your own code reduces fidelity and makes the code brittle to refactors.

**The sweet spot is maximum extent at maximum purity:** exercise the whole system through a pure path.

### 2. Test features, not code

Apply the **neural network test**: could you reuse this test suite if the entire implementation were replaced by an opaque neural network (or rewritten in a different language)? If not, your tests are coupled to implementation details.

Test at the boundary the user sees. For CLI tools, the boundary is: given this input and these flags, what appears on stdout, stderr, and what is the exit code? For a toolkit of composable tools, the boundary includes pipelines.

### 3. Use a `check` function

Every test should pass through a single entry point that encapsulates the API under test. This gives you three things:

- **Refactor resilience.** When the interface changes, you fix one function, not a hundred assertions.
- **Low friction.** Adding a test becomes a one-liner. If adding a test is more effort than the fix, people skip the test.
- **Artisanal error messages.** Invest once in a detailed failure output that shows input, expected, actual, and diff. Every test benefits.

### 4. Make tests data-driven

If the `check` function takes structured input and expected output, test cases become *data*. Data can be externalized into fixture files. At the limit, a test suite is a single loop over a directory of fixtures, and adding a test means adding a file — zero code.

### 5. Use expect tests for output-heavy assertions

When the expected output is large or changes frequently, support an **update mode**: run the tests with `UPDATE=1`, and mismatches overwrite the `.expected` file with the actual output. Review with `git diff`, commit if correct. This turns output changes from a chore into a one-command operation.

---

## Architecture for Testability

The critical architectural decision that makes all of the above work: **the impure operation must be swappable**.

For tools that call an LLM, hit an API, or touch the network, provide a **stub backend** that returns deterministic output with no IO. This is the equivalent of an in-memory database in a web application — it lets you drive the entire system as a pure function.

```
┌──────────────────────────────────────────────┐
│              Your Tool                       │
│                                              │
│   args ──→ parse ──→ build request ──→ ...   │
│                          │                   │
│                    ┌─────┴──────┐            │
│                    │  backend   │            │
│                    ├────────────┤            │
│                    │ real (API) │ ← impure   │
│                    │ stub       │ ← pure     │
│                    └────────────┘            │
│                          │                   │
│              ... ──→ format ──→ stdout       │
└──────────────────────────────────────────────┘
```

With the stub backend, every test achieves:
- **Maximum purity**: no network, no API keys, no latency, no flakiness
- **Maximum extent**: the entire tool is exercised end-to-end
- **Millisecond runtime**: a thousand tests finish before you lose focus

### Stub design guidelines

The stub should be **predictable but not trivial**. Returning a fixed string for every input hides bugs. Good patterns:

- **Incrementing counter**: `"LLM return 1"`, `"LLM return 2"`, etc. Proves the tool actually called the backend N times.
- **Echo mode**: return the input back, possibly transformed. Proves the tool passed the right data downstream.
- **Fixture-based**: read a response from a file keyed by some hash or name. Allows scenario-specific testing without real calls.

The stub counter file should be configurable (e.g., via `TOOL_STUB_FILE`) so parallel test runs don't collide.

---

## Concrete Test Structure

### Directory layout

```
repo/
├── tool-name           # The tool itself
├── test.sh             # Test runner
├── test_data/          # Externalized fixtures (optional)
│   ├── basic.input
│   ├── basic.expected
│   ├── pipeline.input
│   └── pipeline.expected
└── ...
```

### The test harness

A minimal harness that implements the `check` pattern for shell tools:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
TOOL="$SCRIPT_DIR/<tool-name>"

TESTS_RUN=0
TESTS_PASSED=0
TESTS_FAILED=0

# Colors (if terminal)
if [[ -t 1 ]]; then
  GREEN='\033[0;32m'; RED='\033[0;31m'; DIM='\033[0;90m'; NC='\033[0m'
else
  GREEN=''; RED=''; DIM=''; NC=''
fi

# ─── Core check functions ───────────────────────────────────

# Check exit code only
check_exit() {
  local name="$1" expected_exit="$2"
  shift 2
  TESTS_RUN=$((TESTS_RUN + 1))
  set +e; "$@" >/dev/null 2>&1; local actual_exit=$?; set -e
  if [[ $actual_exit -eq $expected_exit ]]; then
    echo -e "${GREEN}✓${NC} $name"
    TESTS_PASSED=$((TESTS_PASSED + 1))
  else
    echo -e "${RED}✗${NC} $name ${DIM}(exit: expected $expected_exit, got $actual_exit)${NC}"
    TESTS_FAILED=$((TESTS_FAILED + 1))
  fi
}

# Check stdout matches expected string exactly
check_output() {
  local name="$1" expected="$2"
  shift 2
  TESTS_RUN=$((TESTS_RUN + 1))
  set +e; local actual; actual=$("$@" 2>/dev/null); set -e
  if [[ "$actual" == "$expected" ]]; then
    echo -e "${GREEN}✓${NC} $name"
    TESTS_PASSED=$((TESTS_PASSED + 1))
  else
    echo -e "${RED}✗${NC} $name"
    echo -e "  ${DIM}expected:${NC} $(echo "$expected" | head -3)"
    echo -e "  ${DIM}actual:${NC}   $(echo "$actual" | head -3)"
    TESTS_FAILED=$((TESTS_FAILED + 1))
  fi
}

# Check stdout contains a pattern (grep -q)
check_contains() {
  local name="$1" pattern="$2"
  shift 2
  TESTS_RUN=$((TESTS_RUN + 1))
  set +e; local output; output=$("$@" 2>&1); set -e
  if echo "$output" | grep -q "$pattern"; then
    echo -e "${GREEN}✓${NC} $name"
    TESTS_PASSED=$((TESTS_PASSED + 1))
  else
    echo -e "${RED}✗${NC} $name ${DIM}(output missing: '$pattern')${NC}"
    TESTS_FAILED=$((TESTS_FAILED + 1))
  fi
}

# Check piped input → tool → stdout
check_pipe() {
  local name="$1" input="$2" expected="$3"
  shift 3
  TESTS_RUN=$((TESTS_RUN + 1))
  set +e; local actual; actual=$(echo "$input" | "$@" 2>/dev/null); set -e
  if [[ "$actual" == "$expected" ]]; then
    echo -e "${GREEN}✓${NC} $name"
    TESTS_PASSED=$((TESTS_PASSED + 1))
  else
    echo -e "${RED}✗${NC} $name"
    echo -e "  ${DIM}expected:${NC} $(echo "$expected" | head -3)"
    echo -e "  ${DIM}actual:${NC}   $(echo "$actual" | head -3)"
    TESTS_FAILED=$((TESTS_FAILED + 1))
  fi
}

# ─── Data-driven fixture runner ─────────────────────────────

# Run all .input/.expected pairs from a directory
check_fixtures() {
  local fixture_dir="$1"
  shift  # remaining args are the command template; {} is replaced by input
  if [[ ! -d "$fixture_dir" ]]; then return; fi
  for input_file in "$fixture_dir"/*.input; do
    [[ -f "$input_file" ]] || continue
    local base="${input_file%.input}"
    local name="$(basename "$base")"
    local expected_file="$base.expected"
    local actual

    TESTS_RUN=$((TESTS_RUN + 1))
    set +e
    actual=$(cat "$input_file" | "$@" 2>/dev/null)
    set -e

    if [[ "${UPDATE:-}" == "1" ]]; then
      echo "$actual" > "$expected_file"
      echo -e "${GREEN}✓${NC} $name ${DIM}(updated)${NC}"
      TESTS_PASSED=$((TESTS_PASSED + 1))
    elif [[ ! -f "$expected_file" ]]; then
      echo -e "${RED}✗${NC} $name ${DIM}(missing $expected_file — run with UPDATE=1)${NC}"
      TESTS_FAILED=$((TESTS_FAILED + 1))
    elif diff -q <(echo "$actual") "$expected_file" >/dev/null 2>&1; then
      echo -e "${GREEN}✓${NC} $name"
      TESTS_PASSED=$((TESTS_PASSED + 1))
    else
      echo -e "${RED}✗${NC} $name"
      diff --color=auto <(echo "$actual") "$expected_file" | head -10
      TESTS_FAILED=$((TESTS_FAILED + 1))
    fi
  done
}

# ─── Summary ────────────────────────────────────────────────

summary() {
  echo ""
  echo "════════════════════════════════"
  echo "Tests: $TESTS_PASSED/$TESTS_RUN passed"
  if [[ $TESTS_FAILED -eq 0 ]]; then
    echo -e "${GREEN}All tests passed.${NC}"
    exit 0
  else
    echo -e "${RED}$TESTS_FAILED failed.${NC}"
    exit 1
  fi
}
```

### Writing tests using the harness

Source the harness and write tests as one-liners:

```bash
source "$(dirname "$0")/test_harness.sh"

# ─── Setup ──────────────────────────────────────────────────
STUB_FILE=$(mktemp)
rm -f "$STUB_FILE"
export AGEN_BACKEND=stub AGEN_STUB_FILE="$STUB_FILE"
cleanup() { rm -f "$STUB_FILE"; }
trap cleanup EXIT

# ─── Interface contract ─────────────────────────────────────
echo "=== Interface ==="
check_exit     "--help exits 0"              0  "$TOOL" --help
check_exit     "--version exits 0"           0  "$TOOL" --version
check_exit     "no args exits 1"             1  "$TOOL"
check_contains "--help shows SYNOPSIS"       "SYNOPSIS"  "$TOOL" --help

# ─── Stub backend ───────────────────────────────────────────
echo ""
echo "=== Stub Backend ==="
rm -f "$STUB_FILE"
check_output   "first call returns 1"   "LLM return 1"  "$TOOL" "test"
check_output   "second call returns 2"  "LLM return 2"  "$TOOL" "test"

# ─── Piped input ────────────────────────────────────────────
echo ""
echo "=== Piped Input ==="
rm -f "$STUB_FILE"
check_pipe     "pipe passes through"  "hello world"  "LLM return 1"  "$TOOL" "summarize"

# ─── Data-driven fixtures ───────────────────────────────────
echo ""
echo "=== Fixtures ==="
rm -f "$STUB_FILE"
check_fixtures "$SCRIPT_DIR/test_data"  "$TOOL" "process"

# ─── Done ───────────────────────────────────────────────────
summary
```

---

## Cross-Tool Composition Tests

The highest-value tests for a toolkit of composable tools are **pipeline tests**. These exercise maximum extent while remaining pure.

```bash
echo "=== Pipeline: agen | agen-log ==="
rm -f "$STUB_FILE"
TMPLOG=$(mktemp)

# Full pipeline: generate → log → audit
actual=$(echo "input data" \
  | AGEN_BACKEND=stub AGEN_STUB_FILE="$STUB_FILE" agen "summarize" \
  | agen-log --agent=test --log-file="$TMPLOG" \
  && agen-audit --log-file="$TMPLOG" --last=1 --field=output)

check_output "pipeline produces logged output" "LLM return 1" echo "$actual"
rm -f "$TMPLOG"
```

Pipeline tests should live in the overview repo (or a dedicated `tests/` directory) that depends on all tools being installed. Each tool's own `test.sh` tests that tool in isolation; the composition tests verify the contract between tools.

### What to test in pipelines

- **Data flows end-to-end.** Stdin reaches the final tool; stdout contains the expected result.
- **Exit codes propagate.** A failure mid-pipeline should produce a non-zero exit from the whole pipeline (use `set -o pipefail`).
- **Side effects are correct.** If a tool writes to a file (e.g., agen-log writes JSONL), verify the file structure.
- **Tools are order-independent where claimed.** If `agen | agen-log` and `agen-log` accept input in the same format, test both orderings.

---

## What NOT to Test

- **LLM output quality.** The stub backend ensures you never test the LLM itself. Your tests verify the *plumbing*, not the *oracle*.
- **Internal function boundaries.** Don't extract a bash function just to test it in isolation. If it's reachable through the CLI, test it through the CLI.
- **Exact error message wording.** Use `check_contains` with a stable substring, not `check_output` with the full message. Error messages change; the presence of an error should not.

---

## Running Tests

```bash
# Run all tests (fast — stub backend, no IO)
./test.sh

# Update expected outputs after intentional changes
UPDATE=1 ./test.sh

# Review what changed
git diff test_data/

# Run slow tests that hit real backends (CI only)
RUN_SLOW_TESTS=1 ./test.sh
```

### Slow tests

Some tests must hit real backends (e.g., verify that the API backend actually authenticates). Gate these behind an environment variable and skip them by default:

```bash
if [[ "${RUN_SLOW_TESTS:-}" != "1" ]]; then
  echo "=== Skipping slow tests (set RUN_SLOW_TESTS=1) ==="
else
  echo "=== Slow: Real Backend ==="
  check_exit "api backend returns 0" 0 "$TOOL" --backend=api "hello"
fi
```

These exist as a safety net. They run in CI, not during development. They should never be the *only* test for a behavior that can also be tested purely.

---

## Checklist for Adding a New Test

1. Can it be tested with the stub backend? → **Yes**: write a pure test.
2. Is there already a `check_*` function that fits? → Use it. One line.
3. Is the expected output complex or likely to change? → Use a fixture file with `check_fixtures`. Run `UPDATE=1` to generate the `.expected` file.
4. Does it test a pipeline across tools? → Put it in the composition test suite.
5. Does it *require* real IO? → Gate behind `RUN_SLOW_TESTS`. Keep it minimal.

## Checklist for Adding a New Tool

1. Implement a `stub` backend (or equivalent) from day one.
2. Copy the test harness. Source it from `test.sh`.
3. Write interface contract tests: `--help`, `--version`, no-args, unknown flags.
4. Write feature tests using `check_pipe` and `check_output` against the stub.
5. Add pipeline tests to the composition suite in the overview repo.
6. Print test timing. If any single test exceeds 100ms, investigate.
