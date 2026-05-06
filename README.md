# setup-prompt-forge

A Claude skill that turns an end-state description into a self-driving setup prompt. You describe what you want your workspace to look like when it's done, and the skill generates a complete prompt you paste into Claude Code. Claude handles the rest autonomously.

## Quick Example

**You say:**

> "Set up a Node.js project with ESLint, Prettier, and Jest configured, npm test passing"

**You get:** A complete prompt that, when pasted into `claude code`, will install dependencies, write config files, wire up lint and test scripts, validate everything works, and track progress to a state file so it can resume if interrupted. No hand-holding required.

## What It Does

- Takes a description of your desired end state and generates a copy-paste setup prompt
- The generated prompt verifies prerequisites before doing any work
- Independent steps run in parallel using subagents (in Claude Code)
- Progress is written to disk after each phase, so setup survives crashes and interruptions
- The prompt is idempotent. Re-running it skips completed phases and picks up where it left off
- Works with Claude Code, Claude Desktop, Codex, or any Claude-powered workspace

## How To Invoke

Just describe what you want. Any of these will trigger the skill:

- "Create a setup prompt that configures my workspace from scratch"
- "Generate an onboarding prompt for new engineers joining my team"
- "Write a prompt that sets up a fresh Node.js project with my preferred config"
- "Help me bootstrap my AI workspace with a setup guide that runs itself"

The more detail you give upfront, the fewer clarifying questions you'll get. Describing the end state is enough. The skill derives the phases.

## The Conversation Flow

1. **Describe your end state.** What should be true when setup is done? What environment will it run in? What prerequisites does the runner need? Claude asks these upfront and fills in sensible defaults for anything you skip.
2. **Review the phase map.** Claude sketches the planned structure (which phases, what order, what runs in parallel) and gets your sign-off before writing the full prompt.
3. **Get your prompt.** Claude delivers the complete setup prompt in a copyable block, along with usage notes, any manual steps that couldn't be automated, and resume instructions. You can iterate from here ("add a phase for ESLint", "remove the git hooks step") until it matches.

## Running a Generated Prompt

**Claude Code**: Run the prompt using `claude code` (the autonomous REPL mode). Don't use `claude chat` (conversational, won't self-drive) or `claude cowork` (designed for skill management, not setup execution). If your goal is to install a skill rather than run a setup prompt, use `claude cowork` instead.

**Claude Desktop**: Paste into a new conversation. Works best with a filesystem MCP configured for state persistence.

**Codex**: Save to a `.md` file and pass as input: `codex < setup-prompt.md`.

**Generic**: Paste into any Claude conversation. Claude will be more conversational and may ask you to take some steps manually.

## Resuming

If a setup is interrupted, just re-paste the same prompt in a new conversation. Claude reads the state file, skips completed phases, and picks up at the first incomplete one. Partial phases re-run from scratch (by design, since state is only written after verification).

## Tips

**Describe the end state, not the steps.** "I want a repo with Node 20, ESLint configured, and `npm test` passing" works better than listing every command.

**Name prerequisites precisely.** "ANTHROPIC_API_KEY set in the shell environment" is verifiable. "API key configured" is not.

**Share setup prompts with your team.** They're plain markdown. Commit them to your repo (e.g., `docs/setup-prompt.md`) and anyone can paste them into Claude Code to get the same environment.

---

## Reference

<details>
<summary><strong>Understanding the Generated Prompt</strong></summary>

### Anatomy of a Setup Prompt

Every generated prompt has the same skeleton:

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

The pattern is always **check → work → verify → write state**. State is written last, after verification. If something crashes mid-phase, the state file won't show the phase as complete, so re-running the prompt safely redoes it.

### The State File

The state file lives at `.setup/state.json` in your workspace root. It's created on first run and updated after every phase. Its purpose is to make the prompt **idempotent**.

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

`completed_phases` is the skip list. `values` carries discovered information (paths, versions, tokens) forward to later phases so you don't hardcode machine-specific values.

Add `.setup/state.json` to your `.gitignore`. It's a local run record, not part of your project source.

### Sequential vs. Parallel Phases

Sequential phases run one after another, in order. Use sequential when each step depends on the previous one.

Parallel phases run independent workstreams at the same time using subagents. Use parallel when two or more pieces of work don't depend on each other (e.g., "install npm packages" and "fetch remote config" can happen simultaneously).

The generated prompt marks this with a `[PARALLEL]` label and defines self-contained subagent blocks for each workstream.

### Subagents

A subagent is a separate Claude agent with its own context window. The main agent spawns subagents, waits for all of them to finish, and collects their outputs.

Because subagents don't share memory with each other, every subagent block in the generated prompt is fully self-contained: it receives everything it needs in its instruction block and reports its outputs when done.

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

</details>

<details>
<summary><strong>Environment Reference</strong></summary>

|Feature                         |Claude Code    |Claude Desktop          |Codex           |Generic          |
|--------------------------------|---------------|------------------------|----------------|-----------------|
|Shell / bash access             |✅ Full         |⚠️ MCP only              |✅ Sandboxed     |❌                |
|State file (`.setup/state.json`)|✅              |⚠️ If filesystem MCP     |✅               |Manual           |
|Subagents (parallel phases)     |✅              |❌                       |❌               |❌                |
|TodoList                        |✅              |❌                       |❌               |❌                |
|Resume support                  |✅              |⚠️ Fragile               |✅               |Manual           |
|Best for                        |Full automation|Config/instruction flows|Script execution|Guided checklists|

**Claude Code** is the richest target. If you're unsure which environment the prompt will be run in, generating for Claude Code with inline notes for other environments is a safe default.

**Claude Desktop** setups depend heavily on which MCP servers are installed. The skill will ask which MCPs are available and adjust accordingly. Without a filesystem MCP, state persistence isn't possible and the prompt becomes more of a guided interactive flow.

**Codex** runs prompts more literally and conservatively than Claude. Generated prompts for Codex are simpler and more explicit, with all package versions pinned and fewer agentic assumptions.

</details>

<details>
<summary><strong>Tips & Common Patterns</strong></summary>

**Separate installation from configuration.** Installation (getting tools onto disk) and configuration (setting up credentials, writing config files) are usually independent and benefit from being in separate phases. Mention them separately when describing your end state.

**Iterating on an existing prompt.** You don't have to start from scratch if something changes. Describe the modification: "Add a phase after phase 2 that configures ESLint" or "The git hooks step isn't needed, remove it." Claude will update the prompt directly.

**Running in a CI environment.** Generated prompts are designed for interactive use, but the `.setup/state.json` pattern maps cleanly to CI step caching. If you want a CI-adapted version, mention this during the interview: "This should also work as a reference for a GitHub Actions workflow."

</details>

<details>
<summary><strong>Troubleshooting</strong></summary>

**The preflight phase fails and Claude stops.**
This is intentional. Read the error message, fix the prerequisite (install the tool, set the env var, start the service), and re-paste the prompt.

**A phase is incorrectly marked complete.**
This shouldn't happen by design (state is only written after verification). If it does, edit `.setup/state.json` directly: remove the phase ID from `completed_phases`, save, and re-paste the prompt.

**A subagent task failed but the parent phase marked it complete.**
The generated prompt has a bug. Re-open the session where you generated it and describe the problem. Claude can fix the affected phase.

**The generated prompt references a tool I don't have.**
Tell Claude during the interview, or after delivery: "I don't have a filesystem MCP, adjust phase 2 to use in-conversation state instead."

**The setup prompt is too long for my context window.**
The state file pattern mitigates this (each resumed run starts fresh). If a single phase is too complex, ask Claude to break it into two smaller phases.

**Reset and start over.**
Delete `.setup/state.json` (or the entire `.setup/` directory) and re-paste the prompt.

</details>
