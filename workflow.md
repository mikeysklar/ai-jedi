# 🧠 AI JEDI Workflow — Claude Power User Guide

---

## 🗺️ Phase 1 — Architecture & Planning

- Brain dump the full system architecture before touching any code
- Have Claude build a **module hierarchy** from that dump
- Have Claude propose a **layered module structure**
- Write a **Project Brief** (living markdown doc) containing:
  - Architecture decisions
  - Module contracts (inputs, outputs, possible errors)
  - Known constraints and assumptions
  - Open questions

---

## 🧱 Phase 2 — Modular Build

- Build **one module at a time** — never all at once
- Before building each module, write a short **interface contract**:
  - What goes in
  - What comes out
  - What errors are possible
- Use the contract as the source of truth for both build and test prompts
- Write **unit tests alongside each module** as it's built

---

## 🛡️ Phase 3 — Guardrails & Adversarial Review

- Add explicit constraints to every module prompt:
  - *"Must do X"*
  - *"Must never do Y"*
- Use strong directive language in prompts:
  - *"Carefully review the full module before implementing"*
  - *"Fully implement — do not stub or skip edge cases"*
- After building, **prompt Claude to adversarially test its own module**:
  - Try to break it
  - Identify untested edge cases
  - Challenge its own assumptions

---

## 🔗 Phase 4 — Integration Testing

- Once all modules pass unit tests individually, run **integration tests**
- Validate data flow and contracts across module boundaries
- Any failures trace back to the interface contract — update it if needed

---

## 💾 Phase 5 — Context Persistence

- Claude has no memory between sessions — treat this as a hard constraint
- Maintain a **living Project Brief** updated after every meaningful chat
- At the end of each session, extract and save:
  - Decisions made
  - Modules completed
  - Issues discovered
  - Next steps
- Paste only the **relevant section** of the brief into each new chat — not everything

---

## 🔍 Phase 6 — Reviewer Agent

- Spin up a dedicated agent whose **only job is code review**
- Give it a concrete rubric — not just "look for patterns":
  - [ ] Naming conventions consistent?
  - [ ] Error handling complete?
  - [ ] Modules loosely coupled?
  - [ ] Test coverage adequate?
  - [ ] Security surface minimized?
- Reviewer agent output feeds back into the Project Brief

---

## 📡 Phase 7 — Monitoring Agent

- Define scope explicitly before deploying:
  - What triggers it? (log errors, test failures, spec drift)
  - What can it fix autonomously vs. what needs a human?
  - What does it report and where?
- Operational fixes discovered here get logged back to the Project Brief

---

## 🤖 Phase 8 — Nested Agent Delegation

- Spin up **independent focused agents** for specialized tasks
- Each agent gets:
  - One clearly scoped job
  - Relevant context from the Project Brief (not the whole thing)
  - Its own guardrails
- Sub-agents can spawn further nested agents for atomic sub-tasks
- Each layer reports results upward to keep the parent agent informed

---

## 📌 Versioning & Drift Prevention

- Use **dated context snapshots** as modules evolve
- When a module's interface contract changes, version it (v1 → v2)
- Periodically run the reviewer agent across the full codebase to catch drift
- Keep a changelog section in the Project Brief

---

## 💡 Core Philosophy

> Treat Claude like a **team of specialists**, not a single assistant.  
> Keep context tight. Modules small. Agents focused on exactly one job.  
> The Project Brief is your single source of truth — maintain it religiously.
