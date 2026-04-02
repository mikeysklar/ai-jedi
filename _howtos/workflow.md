---
title: "JEDI AI Workflow — Claude Power User Guide"
description: "A phased approach to building with AI: architecture, modular builds, guardrails, testing, and agent delegation."
tags: [workflow, claude, architecture, agents, testing]
---

## ⚡ TL;DR — The JEDI Workflow at a Glance

```
┌─────────────────────────────────────────────────────────┐
│                    THE JEDI PIPELINE                     │
│                                                         │
│  🗺️ Plan ──▶ 🧱 Build ──▶ 🛡️ Guard ──▶ 🔗 Integrate   │
│                                                         │
│  💾 Persist ──▶ 🔍 Review ──▶ 📡 Monitor ──▶ 🤖 Delegate│
│                                                         │
│  ♻️  Versioning & drift prevention runs continuously     │
└─────────────────────────────────────────────────────────┘
```

> **The old way:** Dump everything into one massive prompt. Pray.
> **The JEDI way:** Treat AI like a production crew — each specialist gets one job, tight context, and clear deliverables.

---

## 🗺️ Phase 1 — Architecture & Planning

**Analogy:** You wouldn't build a house by handing a contractor a stream-of-consciousness voicemail. You'd give them blueprints.

Before touching any code, brain dump the full system architecture. Then have Claude organize the chaos:

- Have Claude build a **module hierarchy** from your brain dump
- Have Claude propose a **layered module structure**
- Write a **Project Brief** — a living markdown doc that becomes your single source of truth

**What goes in the Project Brief:**

| Section | Purpose |
|---|---|
| Architecture decisions | The "why" behind structural choices |
| Module contracts | Inputs, outputs, and possible errors per module |
| Constraints & assumptions | Hard limits and things you're betting on |
| Open questions | Unknowns to resolve before they become bugs |

### 📋 Example: Planning a CLI Todo App

**Bad prompt (the old way):**
> *"Build me a todo app in Python with add, delete, list, and due dates"*

**JEDI prompt:**
> *"I'm building a CLI todo app in Python. Here's the full architecture:*
> - *Storage layer: JSON file on disk*
> - *CLI layer: argparse with subcommands (add, delete, list, done)*
> - *Model layer: Todo dataclass with id, title, done, due_date*
> - *Display layer: rich library for table output*
>
> *Propose a module hierarchy and interface contracts for each layer. Flag any design decisions I should make before we start building."*

```
  📐 ARCHITECTURE PHASE

  Brain Dump ──▶ Claude Organizes ──▶ Project Brief
      │                                     │
      │  "Here's everything                 │  Living doc:
      │   in my head"                       │  decisions, contracts,
      │                                     │  constraints, questions
      ▼                                     ▼
  ┌──────────┐                      ┌──────────────┐
  │ Raw ideas │  ───────────────▶   │  Blueprints   │
  │ scattered │                     │  organized    │
  └──────────┘                      └──────────────┘
```

### 🚀 Quick Start: Scaffold a Project Brief

Create your brief right from the terminal:

```bash
mkdir -p docs && cat > docs/PROJECT_BRIEF.md << 'EOF'
# Project Brief — [Your Project Name]
> Last updated: $(date +%Y-%m-%d)

## Architecture Decisions
- [ ] Decision 1: ...

## Module Contracts
| Module | Input | Output | Errors |
|--------|-------|--------|--------|
| ...    | ...   | ...    | ...    |

## Constraints & Assumptions
- ...

## Open Questions
- [ ] ...

## Changelog
| Date | Change | Reason |
|------|--------|--------|
EOF
echo "✅ Project brief created at docs/PROJECT_BRIEF.md"
```

---

## 🧱 Phase 2 — Modular Build

**Analogy:** A restaurant kitchen doesn't have one person making the entire meal. The saucier makes sauce. The grill cook grills. Each station has a ticket telling them exactly what to produce.

Build **one module at a time** — never all at once. Each module gets an interface contract first:

| Contract field | Description | Example (Todo app) |
|---|---|---|
| **Input** | What goes in | `title: str, due_date: Optional[date]` |
| **Output** | What comes out | `Todo` object with generated `id` |
| **Errors** | What can go wrong | `DuplicateTitleError`, `InvalidDateError` |

### 🔑 Key Rules

- The contract is the **source of truth** for both build and test prompts
- Write **unit tests alongside each module** — not after
- Never let Claude build two modules in the same prompt

### 📋 Example: Building the Storage Module

> *"Build the storage module per this contract:*
> - *Input: `Todo` object*
> - *Output: `bool` (success), raises `StorageError` on failure*
> - *Persists to `todos.json`, creates file if missing*
> - *Must handle concurrent writes gracefully*
>
> *Write the module and its unit tests together."*

```
  🧱 MODULAR BUILD

  Module A          Module B          Module C
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ Contract │     │ Contract │     │ Contract │
  │    ↓     │     │    ↓     │     │    ↓     │
  │  Build   │     │  Build   │     │  Build   │
  │    ↓     │     │    ↓     │     │    ↓     │
  │  Test ✅  │     │  Test ✅  │     │  Test ✅  │
  └──────────┘     └──────────┘     └──────────┘
       │                │                │
       └────────────────┼────────────────┘
                        ▼
                  Integration ──▶ Phase 4
```

---

## 🛡️ Phase 3 — Guardrails & Adversarial Review

**Analogy:** In martial arts, you don't just practice forms — you spar. Guardrails are your forms. Adversarial review is the sparring.

Every module prompt should include explicit constraints:

### ✅ Do This / ❌ Not That

| ❌ Vague prompt | ✅ JEDI prompt |
|---|---|
| "Handle errors properly" | "Must raise `StorageError` on write failure, never silently swallow exceptions" |
| "Make it robust" | "Must handle empty input, None values, and strings exceeding 500 chars" |
| "Write good code" | "Carefully review the full module before implementing. Fully implement — do not stub or skip edge cases" |

### 🥊 Adversarial Testing

After building, flip Claude's role from builder to attacker:

> *"You just built the storage module. Now try to break it:*
> - *What inputs could crash it?*
> - *What edge cases aren't tested?*
> - *What assumptions are you making that might be wrong?*
> - *Can you corrupt the JSON file through normal usage?"*

This is where Claude catches its own blind spots — things it "assumed would be fine" during building.

### 🚀 Quick Start: Guardrail Template

Paste this at the top of any build prompt to enforce guardrails:

```markdown
## Constraints (non-negotiable)
- [ ] Carefully review the full module before implementing
- [ ] Fully implement — do not stub, placeholder, or skip edge cases
- [ ] Must raise explicit errors, never silently swallow exceptions
- [ ] Must handle: empty input, None/null, boundary values
- [ ] All public functions must have type hints
- [ ] No dependencies beyond what's listed below

## Allowed dependencies
- standard library only (or list specific packages)
```

### 🚀 Quick Start: Adversarial Test Prompt

Copy-paste this after any module build:

```markdown
Switch roles. You are now a QA adversary. Your goal is to BREAK
the module you just built. For each attack:

1. Describe the attack vector
2. Show the exact input that triggers it
3. Show what happens (crash, wrong output, silent corruption)
4. Rate severity: 🔴 critical / 🟡 warning / 🟢 minor

Try at minimum:
- Null/empty/missing inputs
- Extremely large inputs
- Concurrent access (if applicable)
- Malformed data types
- Boundary values (0, -1, maxint)
```

---

## 🔗 Phase 4 — Integration Testing

**Analogy:** Each musician in an orchestra can play their part perfectly in isolation. Integration testing is the first full rehearsal — where you find out the tempo doesn't match.

```
  🔗 INTEGRATION FLOW

  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  CLI      │───▶│  Model   │───▶│ Storage  │
  │  Module   │    │  Module  │    │  Module  │
  └──────────┘    └──────────┘    └──────────┘
       │               │               │
       └───────┬───────┘───────┬───────┘
               ▼               ▼
         Data flows?      Contracts match?
         Errors bubble?   Types align?
               │               │
               ▼               ▼
            ✅ Pass          ❌ Fail
                              │
                              ▼
                    Update the contract
                    (it was wrong, not the code)
```

- Run integration tests **only after** all modules pass unit tests individually
- Validate data flow and contracts across module boundaries
- When integration fails, **trace the failure back to the interface contract** — update the contract if it was wrong, fix the module if it violated the contract

### 🚀 Quick Start: Integration Test Runner

Set up a basic integration test script:

```bash
cat > tests/run_integration.sh << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

echo "🔗 Running integration tests..."
echo "================================"

# Run unit tests first — integration is pointless if units fail
echo "Step 1: Unit tests..."
python -m pytest tests/unit/ -v --tb=short 2>&1 | tee /tmp/unit_results.txt
if [ ${PIPESTATUS[0]} -ne 0 ]; then
    echo "❌ Unit tests failed — fix these before running integration"
    exit 1
fi
echo "✅ Unit tests passed"

# Now run integration
echo ""
echo "Step 2: Integration tests..."
python -m pytest tests/integration/ -v --tb=long 2>&1 | tee /tmp/integration_results.txt
if [ ${PIPESTATUS[0]} -ne 0 ]; then
    echo "❌ Integration failed — check contract boundaries"
    echo "💡 Tip: trace failures back to the interface contract"
    exit 1
fi

echo ""
echo "✅ All integration tests passed"
SCRIPT
chmod +x tests/run_integration.sh
```

---

## 💾 Phase 5 — Context Persistence

**Analogy:** Claude's memory between sessions is like a whiteboard that gets erased every night. The Project Brief is the notebook you photograph before leaving.

This is the phase most people skip — and it's the one that costs them the most time.

### 🧠 The Hard Truth

Claude has **zero memory between sessions**. Every new chat starts from scratch. If you don't persist context, you'll waste the first 20% of every session re-explaining what you already built.

### 📝 End-of-Session Ritual

After every meaningful chat, extract and save to the Project Brief:

| What to capture | Why |
|---|---|
| Decisions made | So you don't relitigate them |
| Modules completed | So you know what's done |
| Issues discovered | So they don't get rediscovered |
| Next steps | So the next session starts running |

### 💡 Pro Tip: Context Injection

Don't paste the entire Project Brief into each new chat. Paste only the **relevant section** for the current task.

### 🚀 Quick Start: End-of-Session Snapshot

Run this prompt at the end of every working session:

```markdown
Summarize this session for the Project Brief. Output ONLY
a markdown block I can paste directly, with these sections:

## Session Snapshot — [date]
### Decisions Made
### Modules Completed
### Issues Discovered
### Next Steps (prioritized)
### Contract Changes (if any)
```

Or automate it — save a shell alias:

```bash
# Add to ~/.bashrc or ~/.zshrc
alias jedi-snapshot='echo "
---
Summarize this session for the Project Brief. Output ONLY a markdown
block with: Decisions Made, Modules Completed, Issues Discovered,
Next Steps (prioritized), Contract Changes (if any).
Format as a dated session snapshot I can paste into docs/PROJECT_BRIEF.md
---
" | pbcopy && echo "📋 Snapshot prompt copied to clipboard"'
```

```
  💾 CONTEXT LIFECYCLE

  Session 1                    Session 2
  ┌──────────────┐            ┌──────────────┐
  │  Work + learn │            │  Paste brief  │
  │       │       │            │  section      │
  │       ▼       │            │       │       │
  │  Update Brief │──────────▶│  Hit the      │
  │  (decisions,  │  overnight │  ground       │
  │   next steps) │  ════════  │  running      │
  └──────────────┘  whiteboard └──────────────┘
                     erased
```

---

## 🔍 Phase 6 — Reviewer Agent

**Analogy:** In film production, the director doesn't also do quality control. There's a separate QA screening before release. Your reviewer agent is that screening room.

Spin up a **dedicated agent** whose only job is code review. Don't just say "review this" — give it a concrete rubric:

### 📊 Review Rubric

| Check | Question |
|---|---|
| 🏷️ Naming | Are conventions consistent across modules? |
| 🚨 Errors | Is every error path handled (not just the happy path)? |
| 🔌 Coupling | Can modules be swapped without breaking others? |
| 🧪 Coverage | Are edge cases tested, not just golden paths? |
| 🔒 Security | Is the attack surface minimized? |

### 📋 Example: Reviewer Agent Prompt

> *"You are a code reviewer. Your only job is to review — not fix, not refactor, just review. Use this rubric: [rubric above]. Here's the module: [paste code]. Rate each check as PASS/WARN/FAIL with a one-line explanation."*

Reviewer output feeds back into the **Project Brief** — it's not a one-off, it's a loop.

### 🚀 Quick Start: Spin Up a Reviewer Agent

Copy-paste this prompt to launch a reviewer agent in a fresh Claude session:

```markdown
You are a CODE REVIEWER AGENT. You do NOT write code, fix code,
or refactor code. You ONLY review and report.

## Your Rubric
Rate each as ✅ PASS | ⚠️ WARN | ❌ FAIL with a one-line reason.

| # | Check            | Question                                      |
|---|------------------|-----------------------------------------------|
| 1 | 🏷️ Naming        | Are naming conventions consistent?             |
| 2 | 🚨 Error paths   | Is every error handled (not just happy path)?  |
| 3 | 🔌 Coupling      | Can this module be swapped without breakage?   |
| 4 | 🧪 Test coverage | Are edge cases tested, not just golden paths?  |
| 5 | 🔒 Security      | Is the attack surface minimized?               |
| 6 | 📏 Complexity    | Any function > 30 lines that should be split?  |
| 7 | 📝 Contracts     | Does impl match the interface contract?        |

## Output Format
For each file reviewed, produce:
- File name
- Rubric scores (table)
- Top 3 findings (severity-ranked)
- Suggested items for the Project Brief changelog

## Code to Review
[paste module here]
```

### 🚀 Quick Start: Automated Review via CLI

If you're using Claude Code (CLI), run a review with one command:

```bash
# Review a specific file
claude "Review this file using the JEDI rubric (naming, errors, \
coupling, coverage, security, complexity, contracts). \
Rate each ✅/⚠️/❌. Top 3 findings." < src/storage.py

# Review all changed files since last commit
git diff --name-only HEAD~1 | xargs -I {} claude \
  "Review {} using the JEDI rubric. Rate each ✅/⚠️/❌." < {}
```

---

## 📡 Phase 7 — Monitoring Agent

**Analogy:** A smoke detector doesn't fight fires. It watches for smoke and sounds the alarm. Your monitoring agent is the same — clearly defined triggers, clearly defined responses.

Before deploying a monitoring agent, define its scope explicitly:

| Scope question | Example answer |
|---|---|
| What triggers it? | Log errors matching `ERROR:storage`, test failures, spec drift |
| What can it fix alone? | Retry transient failures, restart crashed processes |
| What needs a human? | Schema changes, security alerts, contract violations |
| Where does it report? | Slack channel + append to Project Brief |

```
  📡 MONITORING AGENT SCOPE

  ┌──────────────────────────────────────────┐
  │               MONITORS                    │
  │  Log errors │ Test failures │ Spec drift  │
  └──────┬──────┴───────┬───────┴──────┬─────┘
         ▼              ▼              ▼
    ┌─────────┐   ┌──────────┐   ┌─────────┐
    │ Auto-fix │   │  Alert   │   │  human   │
    │ (retry,  │   │  Log to  │   │  Alert   │
    │ restart) │   │  Brief   │   │          │
    └─────────┘   └──────────┘   └─────────┘
    Autonomous     Record for     Needs human
    actions only   future ref     judgment
```

### 🚀 Quick Start: Enable Structured Logging

Add this to any Python project so your monitoring agent has clean signals to watch:

```python
# logging_config.py — drop this into any project
import logging
import json
from datetime import datetime, timezone

class JEDIFormatter(logging.Formatter):
    """Structured JSON logging for monitoring agents to parse."""
    def format(self, record):
        return json.dumps({
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname,
            "module": record.module,
            "function": record.funcName,
            "message": record.getMessage(),
            "extra": getattr(record, "extra", {}),
        })

def setup_logging(level=logging.INFO, log_file="app.log"):
    """Call once at app startup."""
    handler = logging.FileHandler(log_file)
    handler.setFormatter(JEDIFormatter())

    console = logging.StreamHandler()
    console.setFormatter(logging.Formatter(
        "%(asctime)s [%(levelname)s] %(module)s: %(message)s"
    ))

    logging.basicConfig(level=level, handlers=[handler, console])

# Usage in any module:
# import logging
# logger = logging.getLogger(__name__)
# logger.info("Todo created", extra={"extra": {"todo_id": 42}})
```

### 🚀 Quick Start: Log Watcher Script

A lightweight monitor that watches your structured logs and flags errors:

```bash
cat > scripts/log_monitor.sh << 'MONITOR'
#!/usr/bin/env bash
# 📡 JEDI Log Monitor — watches for errors and summarizes them
set -euo pipefail

LOG_FILE="${1:-app.log}"
ALERT_THRESHOLD="${2:-5}"  # alert after N errors in window
WINDOW_SECONDS=300         # 5-minute rolling window

echo "📡 Monitoring $LOG_FILE (alert threshold: $ALERT_THRESHOLD errors / ${WINDOW_SECONDS}s)"
echo "   Press Ctrl+C to stop"
echo "─────────────────────────────────────────"

tail -f "$LOG_FILE" | while read -r line; do
    level=$(echo "$line" | python3 -c "import sys,json; print(json.load(sys.stdin).get('level',''))" 2>/dev/null || echo "")

    case "$level" in
        ERROR|CRITICAL)
            echo "🔴 $(date +%H:%M:%S) $line"
            # Count recent errors
            recent=$(grep -c '"level": "ERROR"\|"level": "CRITICAL"' "$LOG_FILE" | tail -1)
            if [ "$recent" -ge "$ALERT_THRESHOLD" ]; then
                echo "🚨 ALERT: $recent errors detected — review needed"
                # Uncomment to send a notification:
                # osascript -e "display notification \"$recent errors in $LOG_FILE\" with title \"JEDI Monitor\""
            fi
            ;;
        WARNING)
            echo "🟡 $(date +%H:%M:%S) $(echo "$line" | python3 -c "import sys,json; print(json.load(sys.stdin).get('message',''))" 2>/dev/null)"
            ;;
    esac
done
MONITOR
chmod +x scripts/log_monitor.sh
echo "📡 Monitor script created. Run: ./scripts/log_monitor.sh app.log"
```

### 🚀 Quick Start: Monitoring Agent Prompt

Spin this up in a Claude session alongside your running app:

```markdown
You are a MONITORING AGENT. You watch — you do not build or refactor.

## Your Scope
- WATCH: the log file output I'll paste below
- DETECT: errors, warnings, and patterns that suggest drift
- CLASSIFY each finding:
  🟢 AUTO-FIX — you can suggest a fix (retry, config change)
  🟡 INVESTIGATE — needs a human to look but isn't urgent
  🔴 ESCALATE — needs immediate human attention

## Your Output Format
For each finding:
1. Timestamp & log line
2. Classification (🟢/🟡/🔴)
3. Root cause hypothesis
4. Suggested action
5. Should this update the Project Brief? (yes/no + what to add)

## Log Output
[paste recent logs here]
```

---

## 🤖 Phase 8 — Nested Agent Delegation

**Analogy:** A general doesn't personally scout terrain, decode messages, AND lead the charge. They delegate to specialists, each reporting up the chain. Same principle — smaller scope = better results.

```
  🤖 AGENT HIERARCHY

                  ┌─────────────┐
                  │  Orchestrator │
                  │  (you + main │
                  │   Claude)    │
                  └──────┬──────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
      ┌──────────┐ ┌──────────┐ ┌──────────┐
      │ Builder  │ │ Reviewer │ │ Monitor  │
      │ Agent    │ │ Agent    │ │ Agent    │
      └────┬─────┘ └──────────┘ └──────────┘
           │
     ┌─────┼─────┐
     ▼     ▼     ▼
   ┌────┐┌────┐┌────┐
   │ DB ││ API││ UI │   ◀── Sub-agents for
   └────┘└────┘└────┘       atomic tasks
```

### 🔑 Rules for Agent Delegation

Each agent gets:
- **One clearly scoped job** — "test the storage module," not "help with testing"
- **Relevant context only** — the section of the Project Brief it needs, not the whole thing
- **Its own guardrails** — constraints specific to its task
- Sub-agents can spawn **further nested agents** for atomic sub-tasks
- Each layer **reports results upward** to keep the parent informed

### 📋 Example: Delegating a Build Task

> *"You are the Storage Builder Agent. Your one job: build the storage module per this contract: [contract]. Constraints: must use JSON, must handle concurrent writes, must not exceed 200 lines. When done, output: the module code, its tests, and a summary of decisions you made."*

### 🚀 Quick Start: Agent Orchestration Script

Automate spawning multiple agents with Claude Code CLI:

```bash
#!/usr/bin/env bash
# orchestrate.sh — run builder, reviewer, and monitor agents in sequence
set -euo pipefail

MODULE="$1"
CONTRACT_FILE="docs/contracts/${MODULE}.md"
SOURCE_FILE="src/${MODULE}.py"

echo "🤖 JEDI Agent Orchestration for: $MODULE"
echo "==========================================="

# Agent 1: Builder
echo ""
echo "🧱 Phase: Builder Agent..."
claude "You are a Builder Agent. Build the $MODULE module per this \
contract. Write the module AND its unit tests. Output both files." \
  < "$CONTRACT_FILE" > "/tmp/build_output.md"
echo "✅ Builder done"

# Agent 2: Adversarial Tester
echo ""
echo "🥊 Phase: Adversarial Tester..."
claude "You are an Adversarial Test Agent. Try to break this module. \
Show exact inputs that cause failures. Rate each: 🔴/🟡/🟢" \
  < "$SOURCE_FILE" > "/tmp/adversarial_output.md"
echo "✅ Adversarial testing done"

# Agent 3: Reviewer
echo ""
echo "🔍 Phase: Reviewer Agent..."
claude "You are a Reviewer Agent. Review this module using the JEDI \
rubric (naming, errors, coupling, coverage, security, complexity, \
contracts). Rate each ✅/⚠️/❌. Top 3 findings." \
  < "$SOURCE_FILE" > "/tmp/review_output.md"
echo "✅ Review done"

# Summary
echo ""
echo "==========================================="
echo "📋 Results saved to /tmp/"
echo "   build_output.md"
echo "   adversarial_output.md"
echo "   review_output.md"
echo ""
echo "💡 Next: merge findings into docs/PROJECT_BRIEF.md"
```

---

## ♻️ Versioning & Drift Prevention

**Analogy:** A codebase without version tracking is like a game of telephone — by the time the message reaches the end, it's unrecognizable.

| Practice | Why it matters |
|---|---|
| Dated context snapshots | Know what was true *when* a module was built |
| Versioned contracts (v1 → v2) | When a contract changes, old assumptions don't silently break |
| Periodic full-codebase review | Catch drift before it compounds |
| Changelog in the Project Brief | Single place to see what evolved and why |

```
  ♻️  DRIFT PREVENTION

  v1 Contract ──▶ Build ──▶ Works ✅
       │
       │  Requirements change...
       ▼
  v2 Contract ──▶ Rebuild ──▶ Works ✅
       │
       │  Run reviewer across ALL modules
       ▼
  Drift detected? ──▶ Update contracts ──▶ Rebuild affected modules
```

### 🚀 Quick Start: Contract Versioning

Keep contracts in version-controlled markdown files:

```bash
# Create a contracts directory
mkdir -p docs/contracts

# Template for a new contract
cat > docs/contracts/TEMPLATE.md << 'EOF'
# Module Contract: [module_name]
> Version: v1 | Created: [date] | Last updated: [date]

## Input
| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| ...       | ...  | ...      | ...   |

## Output
| Field | Type | Notes |
|-------|------|-------|
| ...   | ...  | ...   |

## Errors
| Error | Trigger | Handling |
|-------|---------|----------|
| ...   | ...     | ...      |

## Changelog
| Version | Date | Change | Reason |
|---------|------|--------|--------|
| v1      | ...  | Initial contract | — |
EOF
```

### 🚀 Quick Start: Drift Detection

Run this periodically to catch when code drifts from contracts:

```bash
# drift_check.sh — ask Claude to compare contracts vs implementation
#!/usr/bin/env bash
set -euo pipefail

echo "♻️  JEDI Drift Detection"
echo "========================"

for contract in docs/contracts/*.md; do
    module=$(basename "$contract" .md)
    source_file="src/${module}.py"

    if [ ! -f "$source_file" ]; then
        echo "⚠️  $module — contract exists but no source file found"
        continue
    fi

    echo ""
    echo "Checking $module..."
    # Combine contract + source and ask Claude to compare
    {
        echo "## Contract:"
        cat "$contract"
        echo ""
        echo "## Implementation:"
        cat "$source_file"
    } | claude "Compare this contract to its implementation. Report \
ONLY drifts — where the code doesn't match the contract. Format: \
🔴 BREAKING / 🟡 MINOR / 🟢 COSMETIC per finding. If no drift, \
say '✅ No drift detected.'"
done
```

---

## 💡 Core Philosophy

> **Treat Claude like a team of specialists, not a single assistant.**
>
> Keep context tight. Modules small. Agents focused on exactly one job.
>
> The Project Brief is your single source of truth — maintain it religiously.

### 🥋 The Old Way vs The JEDI Way

| | 😰 Old Way | ⚔️ JEDI Way |
|---|---|---|
| **Prompting** | One giant prompt, hope for the best | Phased, scoped, contracted |
| **Context** | Copy-paste everything every time | Curated brief, inject only what's relevant |
| **Testing** | "It looks right" | Unit tests per module, integration across boundaries |
| **Review** | You eyeball it at midnight | Dedicated reviewer agent with a rubric |
| **Errors** | Find them in production | Adversarial testing catches them at build time |
| **Scale** | Falls apart after 3 modules | Agent hierarchy scales to any complexity |
| **Memory** | "Wait, what did we decide last time?" | Project Brief captures every decision |
