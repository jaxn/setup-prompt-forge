---
name: setup-prompt-forge
description: >
  Generate a runnable "workspace setup prompt" from a description of the desired end state.
  The generated prompt can be pasted into Claude Desktop, Claude Code, Codex, or any Claude-powered
  workspace — and it will autonomously configure the environment, using subagents for parallel work
  and state artifacts to survive interruption and maintain progress across sessions.
  Use this skill whenever the user says "create a setup prompt", "generate an onboarding script",
  "I want to automate workspace setup", "help me bootstrap my AI workspace", "write a prompt that
  sets up X for someone", "create a setup guide that runs itself", or any variation of wanting a
  self-guided, stateful setup automation for an AI coding or agent environment.
---

# Setup Prompt Forge

A skill for generating self-driving workspace setup prompts. The user describes the **desired end state**; this skill produces a **runnable prompt** that a person pastes into their workspace and Claude drives the entire setup — using subagents for parallelism and a state artifact file for resumability.

---

## Phase 1 — Capture Intent

Before writing anything, interview the user. Ask the following (can be a single message with multiple questions):

### Required information

1. **End state** — What should be true when setup is complete? Be concrete:
   - Tools installed / configured
   - Files, directories, or config scaffolded
   - Services connected or authenticated
   - Tests passing, scripts runnable

2. **Target environment** — Where will the generated prompt be run?
   - `claude-desktop` — Claude Desktop with MCP tool access
   - `claude-code` — Claude Code (has bash, filesystem, subagents, TodoList)
   - `codex` — OpenAI Codex CLI
   - `generic` — Paste-anywhere (no special tool assumptions)
   - Multiple / unknown → generate for `claude-code` with notes for others

3. **Prerequisite knowledge** — What should the runner already have?
   - e.g., API keys in environment, a GitHub account, Docker running, Node.js installed
   - The prompt will verify these exist; list them as hard stops if missing

4. **Steps already known** — Does the user have a rough sequence in mind?
   - If yes: use it as the phase skeleton
   - If no: derive from the end state

5. **Resumability** — Should the prompt support being re-run after interruption?
   - Default: yes (always recommended)
   - State artifact location: default `.setup/state.json` at workspace root

6. **Parallel workstreams** — Are there independent pieces that could run simultaneously?
   - e.g., "install npm packages" and "fetch config from API" don't depend on each other
   - If yes, generate subagent calls for those phases

### When to proceed

Once you have answers to 1, 2, and 3 (others have good defaults), proceed to Phase 2.

---

## Phase 2 — Design the Setup Architecture

Before writing the prompt, sketch the **phase map** for the user to review:

```
Phase 0: Preflight checks          [sequential]
Phase 1: <first major step>        [sequential | parallel]
Phase 2: <second major step>       [sequential | parallel]
  └─ Subagent A: <task>
  └─ Subagent B: <task>
Phase N: Validation & summary      [sequential]
```

Label each phase as sequential or parallel. Parallel phases will be implemented using subagents in the generated prompt.

Get confirmation before writing the full prompt. Adjust based on user feedback.

---

## Phase 3 — Generate the Setup Prompt

Generate the prompt using the **canonical setup prompt pattern** described in `references/prompt-architecture.md`.

The generated prompt must:

1. **Open with a state check** — read `.setup/state.json` if it exists; resume from last completed phase
2. **Maintain a TodoList** (if in Claude Code) reflecting phases
3. **Write state after each phase** — mark phase complete, store any discovered values (paths, versions, tokens found, etc.)
4. **Use subagents for parallel phases** — spawn independent Claude calls per workstream
5. **Include a preflight phase** — verify prerequisites before doing any work
6. **Close with a validation phase** — confirm end state is achieved, print summary
7. **Be self-contained** — no references to external docs unless explicitly needed

Read `references/prompt-architecture.md` for the exact structural template and code patterns.

Read `references/env-notes.md` for environment-specific notes (tool availability, subagent syntax, state file location conventions).

---

## Phase 4 — Present and Refine

Deliver the generated prompt in a code block (markdown) so the user can copy it. Add:

- A short **usage note**: how to run it, what to expect, what "success" looks like
- Any **manual steps** that couldn't be automated (auth flows, browser actions, etc.)
- The **resume instruction**: "If it's interrupted, just re-paste this prompt — it will pick up where it left off"

Ask: "Does this cover the full end state? Any phases missing or out of order?" Iterate until satisfied.

---

## Quality bar for generated prompts

A good generated prompt:

- [ ] Reads state at top, skips completed phases
- [ ] Has clearly named phases with a phase-N label in state
- [ ] Validates prerequisites before phase 1 begins
- [ ] Uses subagents (or clearly notes when they'd help but env doesn't support them)
- [ ] Writes state as the *last action* of each phase (so if it crashes mid-phase, phase re-runs)
- [ ] Ends with a concrete success validation (not just "done!")
- [ ] Is readable by a human — someone should be able to understand what it will do before running it
- [ ] Uses conditional logic — if a thing already exists, skip it rather than failing

---

## Reference files

- `references/prompt-architecture.md` — Canonical structure and code patterns for generated prompts
- `references/env-notes.md` — Per-environment tool availability and syntax notes
- `templates/minimal-setup.md` — A minimal single-phase example prompt
- `templates/parallel-setup.md` — A multi-phase, multi-subagent example prompt
