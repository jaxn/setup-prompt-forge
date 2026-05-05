# setup-prompt-forge — User Guide

**setup-prompt-forge** is a Claude skill that turns a description of a desired workspace end state into a fully self-driving setup prompt. The generated prompt can be pasted into Claude Code, Claude Desktop, Codex, or any Claude-powered workspace, and it will autonomously drive the entire setup — using parallel subagents to do work simultaneously and a state file to survive interruption.

-----

## Table of Contents

1. [What It Does](#what-it-does)
1. [When To Use It](#when-to-use-it)
1. [How To Invoke It](#how-to-invoke-it)
1. [The Conversation Flow](#the-conversation-flow)
- [Step 1: Intent Capture](#step-1-intent-capture)
- [Step 2: Phase Map Review](#step-2-phase-map-review)
- [Step 3: Generated Prompt Delivery](#step-3-generated-prompt-delivery)
1. [Understanding the Generated Prompt](#understanding-the-generated-prompt)
- [Anatomy of a Setup Prompt](#anatomy-of-a-setup-prompt)
- [The State File](#the-state-file)
- [Sequential vs. Parallel Phases](#sequential-vs-parallel-phases)
- [Subagents Explained](#subagents-explained)
1. [Running a Generated Prompt](#running-a-generated-prompt)
1. [Resuming an Interrupted Setup](#resuming-an-interrupted-setup)
1. [Environment Reference](#environment-reference)
1. [Tips & Common Patterns](#tips--common-patterns)
1. [Troubleshooting](#troubleshooting)

-----

## What It Does

The skill takes two things as input: **what you want to be true when setup is done** and **where the prompt will be run**. It outputs a complete, copy-pasteable setup prompt.

The generated prompt, when pasted into Claude Code or another AI workspace:

- Checks which steps have already been completed (via a state file)
- Verifies prerequisites before starting
- Runs independent steps in parallel using subagents
- Writes progress to disk after each phase so setup can survive a crash or interruption
- Validates the final end state and prints a clear success summary

You don’t write any code or config. You describe what you want; the skill writes the instructions that Claude will follow to get there.

-----

## When To Use It

Use this skill when you want to automate the process of setting up a workspace for yourself or someone else — and you want that setup to be hands-off, resumable, and reliable.

Good fit:

- Bootstrapping a new project environment (dependencies, config files, env vars, directory structure)
- Onboarding a new team member into your AI toolchain
- Setting up a fresh machine with your standard dev environment
- Automating a multi-step configuration workflow someone would otherwise follow manually
- Creating a repeatable setup for a specific tool, framework, or workflow

Not a great fit:

- Things that require browser-based OAuth flows or GUI interaction (the generated prompt will call these out as manual steps)
- Setups where every step depends on a human decision — the skill works best when the path is knowable in advance

-----

## How To Invoke It

Just describe what you want in plain language. Any of these will trigger the skill:

- “Create a setup prompt that configures my jax-brain workspace from scratch”
- “Generate an onboarding prompt for new engineers joining my team”
- “I want to automate workspace setup for Claude Code with my standard tools”
- “Write a prompt that sets up a fresh Node.js project with my preferred config”
- “Help me bootstrap my AI workspace — I want to create a setup guide that runs itself”

The more detail you give upfront, the fewer clarifying questions you’ll be asked. You don’t need to have the exact steps figured out — describing the end state is enough. The skill will derive the phases.

-----

## The Conversation Flow

When you invoke the skill, Claude walks you through three steps before delivering the generated prompt.

### Step 1: Intent Capture

Claude will ask you several questions — often in a single message:

**End state** (required): What should be true when setup finishes?

> “Node 20 installed, repo cloned, `.env` populated with my API keys, `npm test` passing”

**Target environment** (required): Where will the generated prompt be run?

> Claude Code / Claude Desktop / Codex / Generic (paste-anywhere)

**Prerequisites** (required): What must already be true before setup runs?

> “Docker running, `GITHUB_TOKEN` in environment, SSH key configured for GitHub”

**Known steps** (optional): Do you have a rough sequence in mind?

> If yes, Claude uses it as the phase skeleton. If no, Claude derives phases from the end state.

**Parallel workstreams** (optional): Are there independent pieces that could run at the same time?

> “Installing npm packages and fetching remote config don’t depend on each other”

Defaults are sensible for anything you skip: resumability is always on, state file goes in `.setup/state.json`, and Claude derives a phase map if you don’t provide one.

### Step 2: Phase Map Review

Before writing the full prompt, Claude sketches the planned structure and asks for confirmation:

```
Phase 0: Preflight checks             [sequential]
Phase 1: Clone repo & install deps    [sequential]
Phase 2: Configure environment        [parallel]
  └─ Subagent A: Write .env file
  └─ Subagent B: Initialize git hooks
Phase 3: Validate and smoke test      [sequential]
```

This is your chance to reorder phases, add missing steps, or say “phase 2 should actually be sequential — B depends on A.” The full prompt is only written after you confirm the map.

### Step 3: Generated Prompt Delivery

Claude delivers the complete setup prompt in a copyable markdown block, along with:

- A usage note explaining how to run it
- Any manual steps that couldn’t be automated (auth flows, browser actions, etc.)
- A note on how to resume if it’s interrupted

After reviewing, you can ask for changes: “Add a phase to configure Prettier” or “Remove the git hooks step.” Claude will iterate until the prompt matches your intent.

-----

## Understanding the Generated Prompt

### Anatomy of a Setup Prompt

Every generated prompt has the same skeleton, regardless of what it’s setting up:

```
[HEADER COMMENT]     Human-readable metadata: goal, target, prerequisites, resume instructions
[STATE INIT]         Read or create .setup/state.json; load completed_phases
[PREFLIGHT]          Verify prerequisites; halt clearly if any fail
[PHASE 1..N]         The actual setup work; each phase checks state before running
[VALIDATION]         Confirm the end state is fully achieved
[SUMMARY]            Print what was done and any manual next steps
```

A concrete excerpt showing what a phase looks like:

```markdown
## Phase 2: Install Dependencies [ID: phase-2]

Skip if "phase-2" is in completed_phases.

1. Run `npm ci` to install pinned dependencies
2. Verify: `node_modules/.bin/jest --version` exits 0

After all steps succeed:
- Store: state.values["jest_version"] = discovered version string
- Append "phase-2" to completed_phases in .setup/state.json
- Print: "✓ Phase 2 complete"
```

Notice the pattern: **check → work → verify → write state**. State is always written last, after verification. If something crashes mid-phase, the state file won’t show the phase as complete, so re-running the prompt will safely redo it.

### The State File

The state file lives at `.setup/state.json` in your workspace root. It’s created on first run and updated after every phase. Its purpose is to make the prompt **idempotent** — running it twice should produce the same result as running it once.

```json
{
  "goal": "jax-brain workspace setup",
  "started_at": "2026-05-04T09:00:00Z",
  "last_updated": "2026-05-04T09:12:33Z",
  "completed_phases": ["preflight", "phase-1", "phase-2"],
  "values": {
    "repo_path": "/Users/jackson/jax-brain",
    "node_version": "20.12.0",
    "obsidian_plugin_path": "/Users/jackson/.obsidian/plugins"
  },
  "errors": []
}
```

`completed_phases` is the skip list — any phase whose ID appears here is skipped on the next run. `values` carries discovered information (paths, versions, tokens) from early phases forward to later ones, so you don’t hardcode machine-specific values.

**You should commit this file to `.gitignore`.** It’s a local run record, not part of your project source.

### Sequential vs. Parallel Phases

Sequential phases run one after another, in order. Use sequential when each step depends on the previous one — e.g., you can’t configure a tool before installing it.

Parallel phases run independent workstreams at the same time using subagents. Use parallel when two or more pieces of work don’t depend on each other — e.g., “install npm packages” and “fetch remote config from an API” can both happen simultaneously.

The generated prompt makes this explicit with a `[PARALLEL]` label in the phase heading, and then defines self-contained subagent blocks for each workstream.

### Subagents Explained

A subagent is a separate Claude agent — it has its own context window and runs its task independently. The main agent spawns subagents, waits for all of them to finish, and then collects their outputs.

Because subagents don’t share memory with each other or with the main agent, every subagent block in the generated prompt is fully self-contained: it receives everything it needs in its instruction block and reports its outputs when done. You’ll see this pattern in every parallel phase:

```markdown
### Subagent A: Configure .env

You are a subagent performing one part of a workspace setup.

Your task: Write the .env file with the following keys...

Context you have:
- Repo path: /Users/jackson/my-project
- ANTHROPIC_API_KEY is already set in the environment

Steps:
1. Read $ANTHROPIC_API_KEY from environment
2. Write .env with those values
3. Verify: cat .env | grep ANTHROPIC_API_KEY is non-empty

When complete: print the .env path and confirm each key was written.
```

Subagent availability depends on the target environment. Claude Code supports them fully. Claude Desktop and Codex do not; the skill generates sequential prompts for those environments with a note explaining where parallelism would have helped.

-----

## Running a Generated Prompt

**Claude Code**: Paste the entire prompt into a new conversation. Claude will start executing immediately. You’ll see the TodoList populate at the start and tick off as phases complete.

**Claude Desktop**: Paste the prompt into a new conversation. Claude will guide you through the phases conversationally. If you have a filesystem MCP configured, it will manage the state file automatically. If not, Claude will track progress in the conversation and note where you’d need to take manual steps.

**Codex (CLI)**: Save the prompt to a `.md` file, then pass it as input: `codex < setup-prompt.md`. Or paste it into an interactive Codex session.

**Generic / paste-anywhere**: Start a new conversation with Claude and paste the prompt. Because no specific tool access is assumed, Claude will be more conversational and may ask you to take some steps manually.

In all cases, you don’t need to intervene during the run. If Claude gets stuck on a phase (tool permission issue, missing env var, etc.), it will print a clear error describing what failed and what you need to fix before re-running.

-----

## Resuming an Interrupted Setup

If a setup is interrupted — context window limit, network issue, you had to close the laptop — just re-paste the same prompt in a new conversation.

Claude will immediately read `.setup/state.json`, see which phases are already complete, and skip them. It will pick up at the first incomplete phase and continue.

Nothing from the completed phases is re-done. Files that were already created won’t be overwritten. Packages that were already installed won’t be reinstalled. The setup is designed to be re-run safely.

The only case where re-running re-does work is if a phase crashed in the middle — because state is only written after a phase fully completes and verifies, a partial phase won’t appear in `completed_phases`, so it will run again from the start of that phase. This is intentional: it’s safer to redo a phase than to silently skip it when it may have only half-completed.

-----

## Environment Reference

|Feature                         |Claude Code    |Claude Desktop          |Codex           |Generic          |
|--------------------------------|---------------|------------------------|----------------|-----------------|
|Shell / bash access             |✅ Full         |⚠️ MCP only              |✅ Sandboxed     |❌                |
|State file (`.setup/state.json`)|✅              |⚠️ If filesystem MCP     |✅               |Manual           |
|Subagents (parallel phases)     |✅              |❌                       |❌               |❌                |
|TodoList                        |✅              |❌                       |❌               |❌                |
|Resume support                  |✅              |⚠️ Fragile               |✅               |Manual           |
|Best for                        |Full automation|Config/instruction flows|Script execution|Guided checklists|

**Claude Code** is the richest target. If you’re unsure which environment the prompt will be run in, generating for Claude Code with inline notes for other environments is a safe default.

**Claude Desktop** setups depend heavily on which MCP servers are installed. The skill will ask which MCPs are available and adjust accordingly. Without a filesystem MCP, state persistence isn’t possible and the prompt becomes more of a guided interactive flow.

**Codex** runs prompts more literally and conservatively than Claude. Generated prompts for Codex are simpler and more explicit, with all package versions pinned and fewer agentic assumptions.

-----

## Tips & Common Patterns

**Describe the end state, not the steps.** You’ll get better results saying “I want a repo with Node 20, ESLint configured, and `npm test` passing” than listing every command. The skill derives steps from desired outcomes.

**Separate installation from configuration.** Installation (getting tools onto disk) and configuration (setting up credentials, writing config files) are usually independent and benefit from being in separate phases. Mention them separately when describing your end state.

**Name your prerequisites precisely.** “API key configured” is vague. “ANTHROPIC_API_KEY set in the shell environment” is precise and can be verified programmatically. The more concrete your prerequisite descriptions, the more reliable the preflight phase will be.

**Iterating on an existing prompt.** You don’t have to start from scratch if something changes. Describe the modification: “Add a phase after phase 2 that configures ESLint” or “The git hooks step isn’t needed — remove it.” Claude will update the prompt directly.

**Sharing setups with a team.** Generated prompts are plain markdown files — commit them to your repo (e.g., `docs/setup-prompt.md`). Anyone on the team can paste the contents into Claude Code and get the same environment. State files (`.setup/state.json`) should be gitignored.

**Running in a CI environment.** Generated prompts are designed for interactive use, but the `.setup/state.json` pattern maps cleanly to CI step caching. If you want a CI-adapted version, mention this during the interview: “This should also work as a reference for a GitHub Actions workflow.”

-----

## Troubleshooting

**The preflight phase fails and Claude stops.**  
This is intentional. Read the error message — it will name the specific prerequisite that failed and what it expected. Fix that prerequisite (install the tool, set the env var, start the service) and re-paste the prompt. Since preflight didn’t write to the state file, it will re-run from scratch when you restart.

**A phase partially completed but the state file shows it’s done.**  
This shouldn’t happen by design — state is only written after verification. If you believe a phase is incorrectly marked complete, edit `.setup/state.json` directly: remove the phase ID from `completed_phases`, save the file, and re-paste the prompt. That phase will re-run.

**A subagent task failed but the parent phase marked it complete.**  
A well-generated prompt checks subagent outputs before writing state. If this happens, the generated prompt has a bug. Re-open the session where you generated the prompt and describe the problem — Claude can fix the affected phase.

**The generated prompt references a tool I don’t have.**  
Tell Claude which tools are available during the interview, or after delivery: “I don’t have a filesystem MCP — adjust phase 2 to use in-conversation state instead.” Claude will revise.

**The setup prompt is too long for my context window.**  
Long setups with many phases can hit context limits during a long run. The state file pattern mitigates this — since each resumed run starts fresh, you’re only holding the current phase in context at a time. If a single phase is too complex, ask Claude to break it into two smaller phases.

**I want to reset and start over completely.**  
Delete `.setup/state.json` (or the entire `.setup/` directory) and re-paste the prompt. Claude will treat this as a fresh run.
