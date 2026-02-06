# TODO: Update all references in `shell-agentics` thesis

The shell-agentics repo contains the thesis/manifesto README. It references all toolkit
tools extensively. All references need updating for the toolkit-wide rename.

## Rename reference table

| Old | New |
|-----|-----|
| `agen` (tool) | `agent` |
| `agen-memory` | `amem` |
| `agen-log` | `alog` |
| `agen-audit` | `aaud` |
| `agen-skills` | `ascr` |
| "skills" (concept) | "scripts" |
| `AGEN_BACKEND` | `AGENT_BACKEND` |
| `AGEN_LOG_DIR` | `AGENT_LOG_DIR` |
| `AGEN_MEMORY_DIR` | `AGENT_MEMORY_DIR` |
| `shellagentics/agen` | `shellagentics/agent` |
| `shellagentics/agen-memory` | `shellagentics/amem` |
| `shellagentics/agen-log` | `shellagentics/alog` |
| `shellagentics/agen-audit` | `shellagentics/aaud` |
| `shellagentics/agen-skills` | `shellagentics/ascr` |

---

## File-by-file instructions

### 1. Edit `README.md` (the thesis document)

This is a large document (~600+ lines). Changes needed throughout:

#### The Toolkit section
This section explicitly describes each tool. Update all names:
- **`agen`** → **`agent`** — "curl for LLM inference" (update heading, description, code examples)
- **`agen-skills`** → **`ascr`** — Also rename the concept: "Composable workflow skills" → "Composable workflow scripts". Update all text that says "skill" to say "script" in this context.
- **`agen-memory`** → **`amem`** — update heading, description, code examples
- **`agen-log`** → **`alog`** — update heading, description, code examples
- **`agen-audit`** → **`aaud`** — update heading, description, code examples
- **`shellclaw`** — name stays, but update any references to tools it uses

#### Code examples throughout
- All `agen "..."` → `agent "..."`
- All `agen --system-file` → `agent --system-file`
- All `agen-log --agent` → `alog --agent`
- All `agen-memory read/write` → `amem read/write`
- All `agen-audit --today` → `aaud --today`
- Pipeline examples: `cat ... | agen` → `cat ... | agent`

#### Environment variables
- `AGEN_BACKEND` → `AGENT_BACKEND`
- `AGEN_LOG_DIR` → `AGENT_LOG_DIR`
- `AGEN_MEMORY_DIR` → `AGENT_MEMORY_DIR`

#### GitHub URLs
- `shellagentics/agen` → `shellagentics/agent`
- `shellagentics/agen-memory` → `shellagentics/amem`
- `shellagentics/agen-log` → `shellagentics/alog`
- `shellagentics/agen-audit` → `shellagentics/aaud`
- `shellagentics/agen-skills` → `shellagentics/ascr`

#### Conceptual references to "skills"
- "Agent skills" → "Agent scripts"
- "skill scripts" → "agent scripts" or just "scripts"
- The `agen-skills` description section should explain that these are scripts, not "skills"
- Any reference to the "skills" concept as a named thing in the architecture → "scripts"

#### Specific sections to check
- **The Six Primitives**: May reference tool names in examples
- **The Shell as Control Plane**: May have code examples
- **The Oracle Model**: References to how tools work — update names
- **The Trust Gradient**: May reference tool names
- **Substrate A vs B**: May have code examples with `agen`
- **The Derivative Stack**: Unlikely to have tool names, but check
- **Three-Tier Architecture**: May reference toolkit tools
- **Seven Proposed Principles**: May reference tools
- **Roadmap**: References tools and repos
- **References section**: URLs may need updating

#### Important: Do NOT change
- The repo name `shell-agentics` itself (it's the thesis, not a tool)
- The project name "Shell Agentics" (it's the overall project name)
- Any academic/paper citations
- External URLs (like fly.io, anthropic.com, etc.)

---

## Verification

After all changes, run:

```bash
# Search for ANY remaining old references
grep -n "agen-memory" README.md
grep -n "agen-log" README.md
grep -n "agen-audit" README.md
grep -n "agen-skills" README.md
grep -n "AGEN_" README.md

# Check for standalone "agen" (not "agent", not "shellagentics")
# This needs careful regex — "agen" not followed by "t" and not preceded by "shell"
grep -n "[^l]agen[^t]" README.md
grep -n "^agen[^t]" README.md

# Also check for backtick-quoted agen
grep -n '`agen`' README.md
grep -n '`agen ' README.md
```

## Delete this file when done

Remove this TODO.md after completing all changes.
