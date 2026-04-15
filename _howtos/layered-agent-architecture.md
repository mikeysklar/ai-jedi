---
title: "Layered Agent Architecture — Multi-Agent Workflows with Claude Code"
description: "A three-layer agentic workflow (Orchestrator → Architect → Implementers) for running complex, autonomous tasks with minimal human prompting."
tags: [agents, architecture, automation, llm, workflow, claude-code]
---

<img src="{{ '/assets/images/layered-agent-architecture.png' | relative_url }}" alt="Layered Agent Architecture" style="max-width: 50%; height: auto;" />

## 🎯 TL;DR

> *"Do not try and prompt the agent. That's impossible. Instead, only try to realize the truth: there is no single agent — there are layers."*

A multi-layer agentic workflow where each layer has a **distinct responsibility**, allowing complex tasks to be executed autonomously with minimal human prompting. Stop treating Claude as one big prompt-hungry brain and start treating it as a small org chart. 🧠➡️🏢

---

## 🧱 The Three Layers

### 🧭 L1 — Orchestration agent
Owns the top-level goal list. Decides priorities, keeps everything moving, and unblocks downstream agents **without needing to prompt the user**. This is the "always-on" driver of the whole system.

### 🏛️ L2 — Architecture agent
Owns the design of each subtask. Keeps the overall system context in mind, watches implementation for mistakes, and provides review and course-correction. Acts as a **senior reviewer** sitting above the code.

### 🛠️ L3 — Implementation agents
Do the actual work — writing code, editing files, running commands. Can be run in **parallel** across subtasks. Receive instructions from the architecture agent and report results back up.

```
┌─────────────────────────────────┐
│       Orchestration agent       │  L1 — priorities · backlog · unblocking
└──────────────┬──────────────────┘
               │
┌──────────────▼──────────────────┐
│       Architecture agent        │  L2 — design · review · course-correct
└──────┬───────────────────┬──────┘
       │                   │
┌──────▼──────┐     ┌──────▼──────┐
│  Impl A     │     │  Impl B     │  L3 — code · files · commands
└─────────────┘     └─────────────┘
```

---

## 💡 Why This Layering Works

- 🧠 The architecture agent keeps **full system context** so implementation agents can stay focused and small
- 🛑 Mistakes get caught at **L2** before propagating up to the orchestrator
- 📋 The orchestrator never needs to understand implementation details — it only tracks goals and progress

> *"A single giant agent has to hold the goal, the design, and the diff all at once. Three small agents each hold one thing well."*

---

## 🧰 Suggested Extensions

### 💾 Memory store (feeds L1)
Persistent cross-session state so the orchestrator remembers what was finished, what was abandoned, and why. Without it every session restarts from scratch.

### 🚦 Approval gate (feeds L1)
Don't interrupt for every action — route only **irreversible or expensive** ones (deploy, delete, large spend) through a quick human confirm. Keeps autonomy high while guarding the exits.

### 💸 Budget guard (feeds L2)
A token/cost cap wired to the architecture agent. If a subtask is burning too much context, it signals back up rather than silently degrading quality.

### 📜 Trace log (feeds L2)
Structured log of every architectural decision and code diff. Cheap to add, **invaluable** when something goes wrong three sessions later.

### ✅ Test / lint runner (feeds L3)
Implementation agents write code, the runner gives pass/fail signal back automatically. Closes the feedback loop without needing L2 to manually review every change.

---

## 🗺️ Extended Topology

Wiring the extensions in around the three core layers:

```
                   ┌──────────────┐
                   │  Memory store│◄────── cross-session state
                   └──────┬───────┘
                          │
                   ┌──────▼───────┐
           ┌──────►│  L1  Orchestr│◄──── Approval gate (human-in-loop)
           │       └──────┬───────┘        (irreversible ops only)
           │              │ goals
           │       ┌──────▼───────┐
           │       │  L2  Architect│◄──── Budget guard (tokens/$)
           │       └──┬────────┬──┘◄──── Trace log (decisions/diffs)
           │          │        │
           │     ┌────▼──┐ ┌───▼────┐
           │     │ Impl A│ │ Impl B │   L3 — parallel workers
           │     └───┬───┘ └───┬────┘
           │         │         │
           │       ┌─▼─────────▼─┐
           └───────┤ Test/Lint   │◄──── auto pass/fail signal
       status up   │ runner      │
                   └─────────────┘
```

**Signal flow:**
- ⬇️ **Down:** goals → designs → tasks
- ⬆️ **Up:** test results → reviewed diffs → progress → memory

---

## ⚡ Optimization Principles

- 🔒 **Context isolation** — each layer only holds what it needs. L3 never sees the backlog; L1 never sees diffs. Smaller context = faster, cheaper, fewer hallucinations.
- 🪢 **Parallelism at L3** — implementation agents fan out. L2 serializes only when dependencies force it.
- 🎯 **Cheap feedback loops** — prefer deterministic signals (tests, lint, types) over LLM review. Save L2 attention for judgment calls.
- 📡 **Fail upward, not sideways** — L3 never retries blindly; it bubbles blockers to L2, which re-plans or escalates to L1.
- 🤝 **Autonomy budget** — L1 only pings the human when: (a) priorities are ambiguous, (b) approval gate fires, (c) budget guard trips.

---

## 🖥️ CLI Usage — Running the Stack with Claude Code

Here's how to actually wire this up using Claude Code, subagents, and a few shell helpers. None of this requires extra frameworks — it's all native harness features.

### 1️⃣ Start the Orchestrator (L1)

The orchestrator runs as a long-lived session. It owns the goal list and drives everything.

```bash
# Start a dedicated orchestrator session in your project
cd ~/my-project
claude

# Inside the session, give it the top-level goal:
> You are the L1 Orchestrator. Keep the backlog in ./backlog.md.
> For each goal, spawn an L2 architect subagent via the Agent tool.
> Only interrupt me for approval gates or budget overruns.
```

> 💡 **Tip:** Put persistent orchestrator instructions into `CLAUDE.md` at the repo root so every new session boots into the right role automatically.

### 2️⃣ Define the Architect (L2) as a Subagent

Subagents live under `~/.claude/agents/` (user-wide) or `.claude/agents/` (project-scoped). Create one for the architect:

```bash
mkdir -p .claude/agents
cat > .claude/agents/architect.md <<'EOF'
---
name: architect
description: L2 design and review agent. Receives a subtask from L1, designs the approach, and dispatches L3 implementers.
tools: Read, Grep, Glob, Agent, Bash
---

You are the L2 Architecture agent. For any subtask you receive:
1. Read enough of the codebase to understand context
2. Write a short plan (3-6 steps)
3. Dispatch L3 implementers via the Agent tool, one per independent step
4. Review their diffs before reporting back to L1
5. If budget or ambiguity trips, bubble up to L1 — never retry blindly
EOF
```

### 3️⃣ Define Implementer (L3) subagents

```bash
cat > .claude/agents/implementer.md <<'EOF'
---
name: implementer
description: L3 worker. Executes a single focused subtask — writes code, runs tests, reports results. No backlog awareness.
tools: Read, Edit, Write, Bash, Grep, Glob
---

You are an L3 Implementation agent. You have ONE job from the architect.
- Do not read outside the files you were told about
- Run tests/lint after each edit
- Report back ONLY: what you changed, what passed, what failed
- Never retry more than once — bubble blockers up
EOF
```

### 4️⃣ Verify the agents are loaded

```bash
# From inside Claude Code:
> /agents

# Should list:
#   architect     (project)
#   implementer   (project)
```

### 5️⃣ Run a goal through the stack

From the orchestrator session, just describe the goal in plain English:

```
> Add rate limiting to the /api/upload endpoint.
> Dispatch to architect.
```

Behind the scenes, the harness will:

```
L1 (you) ──▶ Agent(architect, "rate limit /api/upload")
                │
                ├──▶ Agent(implementer, "add limiter middleware")
                ├──▶ Agent(implementer, "wire limiter into upload route")
                └──▶ Agent(implementer, "add integration tests")
                         │
                         ▼
                   pytest / eslint ─── pass/fail back to architect
                         │
                         ▼
              architect reviews diffs ── reports up to L1
```

### 6️⃣ Add an approval gate via hooks

Wire a `PreToolUse` hook so destructive actions ping you before firing. Edit `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "if echo \"$TOOL_INPUT\" | grep -qE '(rm -rf|git push|deploy)'; then osascript -e 'display dialog \"Approve destructive command?\" buttons {\"No\",\"Yes\"}'; fi"
          }
        ]
      }
    ]
  }
}
```

### 7️⃣ Add a trace log (feeds L2 review)

```bash
# In .claude/settings.json, add a PostToolUse hook that appends every diff to a log
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"[$(date -Iseconds)] $TOOL_NAME $TOOL_INPUT\" >> .claude/trace.log"
          }
        ]
      }
    ]
  }
}
```

### 8️⃣ Schedule the Orchestrator to wake itself (optional)

Use the `/schedule` skill to let L1 reprioritize on a cron without you having to babysit it:

```
> /schedule every 30 minutes "review ./backlog.md, pick the next ready item, dispatch to architect"
```

Or use `/loop` for in-session polling:

```
> /loop 10m /run-next-backlog-item
```

---

## 🔭 Further Extensions

- 🔍 **Research agent (L2-adjacent)** — web/docs lookup on demand so architects don't guess APIs.
- 📝 **Spec generator (L1→L2)** — converts vague goals into testable acceptance criteria before L2 designs.
- 🧑‍⚖️ **Critic agent (L2 shadow)** — a second model reviews L2's plan before L3 starts. Catches design errors cheaply.
- ⏰ **Scheduler / cron** — L1 wakes on a timer to reprioritize, not just on user input.
- 📂 **Artifact store** — shared scratchpad between L3 agents so parallel workers can hand off without L2 mediation.
- 📣 **Notification fanout** — Slack/email/push when approval gate fires; keeps user out of the loop until needed.

---

## 🎓 The Mental Model

> *"The orchestrator doesn't write code. The architect doesn't write code. Only the implementer writes code — and it doesn't know why."*
>
> *"That's the whole trick."*

When you stop trying to cram every concern into one prompt, the model gets better at each individual concern. 🚀

---

## 🔗 Links

- **Claude Code docs:** [docs.anthropic.com/en/docs/claude-code](https://docs.anthropic.com/en/docs/claude-code/overview)
- **Subagents reference:** [docs.anthropic.com/en/docs/claude-code/sub-agents](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- **Hooks reference:** [docs.anthropic.com/en/docs/claude-code/hooks](https://docs.anthropic.com/en/docs/claude-code/hooks)
