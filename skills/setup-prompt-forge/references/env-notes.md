# Environment Notes

Per-environment notes on tool availability, subagent syntax, and conventions.

---

## claude-code (Claude Code CLI)

**Available tools**: bash, file read/write, TodoList, subagents (Task tool), web_search  
**Subagent syntax**: Use the `Task` tool — spawn a new Claude agent with a self-contained prompt  
**State file location**: `.setup/state.json` at repo root  
**TodoList**: Fully supported — always include it  
**Parallel execution**: Full subagent support; agents run in parallel when spawned together  
**Resume**: Works well — re-run the same prompt, state file is read on startup  

**CLI mode**: Generated prompts target `claude code` (the autonomous REPL mode). Do not run generated prompts in `claude chat` (lacks autonomous execution) or `claude cowork` (designed for skill management, not setup execution). The prompt expects to self-drive through phases without user interaction, which only `claude code` supports.

**Notes**:
- Bash tool gives full shell access; can run arbitrary commands
- Subagents have their own context — pass all needed info in the Task prompt
- TodoList items should map 1:1 to phases
- If a phase needs a secret (env var), instruct Claude to read from env, never hardcode

**Template choice**: Use `templates/parallel-setup.md` for most setups

---

## claude-desktop (Claude Desktop with MCP)

**Available tools**: Depends on configured MCP servers — do NOT assume bash access  
**Subagent syntax**: Not natively supported; use sequential phases only  
**State file**: Use an MCP filesystem server if available, otherwise use memory notes  
**TodoList**: Not available natively  
**Parallel execution**: Not supported without subagents  
**Resume**: Fragile — advise user to keep the conversation open  

**Notes**:
- Must check which MCP servers are configured (filesystem, git, etc.)
- If no filesystem MCP: state artifact isn't persistable — warn user, use in-conversation state tracking instead
- Guide the user to take manual steps where tool access is missing
- Works best for configuration/instruction workflows, not file-heavy setups
- Generated prompt should include explicit "ask Claude to do X" language vs. "Claude will do X automatically"

**Template choice**: Use sequential pattern only; simplify to a checklist-style prompt

---

## codex (OpenAI Codex CLI)

**Available tools**: bash (sandboxed), file read/write  
**Subagent syntax**: No native subagent support  
**State file location**: `.setup/state.json` at repo root  
**TodoList**: Not available — use markdown checklist in state file instead  
**Parallel execution**: Not supported  
**Resume**: Supported via state file  

**Notes**:
- Bash is sandboxed — some system-level commands may be blocked
- No access to external APIs or MCP tools unless configured
- Simpler prompts work better — fewer moving parts
- Pin all package versions since there's no interactive resolution
- Generated prompts should be more explicit and less agentic — Codex follows instructions more literally

**Template choice**: Use sequential pattern; avoid subagent references

---

## generic (Paste-anywhere / unknown)

**Available tools**: Unknown — assume minimal  
**Subagent syntax**: Not available  
**State file**: Include instructions, but note the user may need to handle manually  
**TodoList**: Not available  
**Parallel execution**: Not available  

**Notes**:
- Generate a conservative, sequential, human-readable prompt
- Include "you may need to do this manually" callouts where tool access is uncertain
- Err toward being a guided checklist rather than fully autonomous
- Include expected outputs so the runner can verify manually

**Template choice**: Use `templates/minimal-setup.md` as base

---

## Multi-environment targeting

If generating for multiple environments or unknown environment:

1. Generate for `claude-code` as primary (most capable)
2. Add a note block at the top: "Running in Claude Desktop or Codex? See the ⚠️ notes below"
3. Add environment-specific fallback notes inline where subagents or bash are used

Example inline note:
```
> ⚠️ Claude Desktop users: If you don't have a filesystem MCP, do this step manually:
> 1. Create the file `.setup/state.json` with the content above
> 2. Continue to Phase 1
```
