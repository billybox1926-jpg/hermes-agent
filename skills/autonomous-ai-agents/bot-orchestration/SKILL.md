---
name: bot-orchestration
description: "Orchestrate multi-agent bot workflows — spawn, delegate, monitor, and coordinate parallel AI coding agents (Claude Code, Codex, OpenCode) and Hermes subagents for complex tasks."
version: 1.0.0
author: OWL
license: MIT
platforms: [linux, macos, windows, android]
environments: [termux, claude-code, codex, opencode, kanban]
metadata:
  hermes:
    tags: [orchestration, multi-agent, delegation, parallel, claude-code, codex, opencode, kanban, workflow]
    related_skills: [claude-code, codex, opencode, kanban-orchestrator, kanban-worker, subagent-driven-development]
---

# Bot Orchestration — Multi-Agent Workflow Coordinator

Orchestrate multiple AI coding agents working in parallel on complex tasks. Covers agent selection, spawning patterns, monitoring, result aggregation, and failure recovery.

## When to Use

- **Multi-component builds**: frontend + backend + tests in parallel
- **Multi-repo work**: changes spanning several git repositories
- **Research + implement**: one agent researches while another codes
- **Batch operations**: fixing issues across many files or repos
- **Long-running tasks**: background agents you monitor asynchronously
- **Human-in-the-loop workflows**: Kanban-gated review cycles
- **Fallback chains**: try Codex first, fall back to Claude Code on failure

Don't use for: single-file edits (just use the coding agent directly), or tasks that fit in one conversation turn.

## Agent Roster

| Agent | Skill | Best For | Mode |
|-------|-------|----------|------|
| **Claude Code** | `claude-code` | Complex multi-step reasoning, large codebases, print mode one-shots | PTY (interactive) or print (`-p`) |
| **OpenAI Codex** | `codex` | Fast single-repo tasks, batch issue-fixing, PR reviews | PTY (interactive) |
| **OpenCode** | `opencode` | Open-source coding agent, PR review, Go/Python focused | Interactive |
| **Hermes Subagents** | `subagent-driven-development` | Isolated reasoning subtasks within current session | `delegate_task` tool |
| **Kanban Workers** | `kanban-orchestrator` | Persistent crash-resistant multi-profile workflows | Kanban board |

Load the relevant agent skill before orchestrating that agent. Each skill has CLI flags, auth setup, and monitoring patterns.

## Orchestration Patterns

### Pattern 1: Fan-Out (Parallel Independent Tasks)

Run multiple agents simultaneously on independent workstreams.

```
# Spawn 3 Claude Code instances in tmux
terminal(command="tmux new-session -d -s agent1 -x 140 -y 40 && tmux send-keys -t agent1 'cd ~/project-a && claude -p \"Build the auth module\" --allowedTools \"Read,Edit,Bash\" --max-turns 15' Enter")
terminal(command="tmux new-session -d -s agent2 -x 140 -y 40 && tmux send-keys -t agent2 'cd ~/project-b && claude -p \"Add dark mode toggle\" --allowedTools \"Read,Edit,Bash\" --max-turns 10' Enter")
terminal(command="tmux new-session -d -s agent3 -x 140 -y 40 && tmux send-keys -t agent3 'cd ~/project-a && claude -p \"Write API tests\" --allowedTools \"Read,Write,Bash\" --max-turns 10' Enter")

# Monitor all
sleep(30)
for s in agent1 agent2 agent3; do echo "=== $s ==="; tmux capture-pane -t $s -p -S -50; done
```

**Rules:**
- Each agent gets its own tmux session (name = descriptive)
- Set `workdir` per agent
- Set `--max-turns` to prevent runaway costs
- Monitor periodically, don't block-wait

### Pattern 2: Pipeline (Sequential Dependencies)

Agent B depends on Agent A's output.

```
# Stage 1: Research
terminal(command="claude -p 'Research the database schema and document all tables and relationships' --allowedTools Read --max-turns 5 --output-format json", workdir="~/project", timeout=120)

# Stage 2: Use research output as input
# (capture output from stage 1, pass to stage 2)
terminal(command="claude -p 'Given this schema documentation: <output>, generate an ORM model for each table' --allowedTools \"Read,Write,Edit\" --max-turns 15", workdir="~/project")
```

**Rules:**
- Capture stdout from upstream agents before spawning downstream
- Use `--output-format json` for machine-parseable handoffs
- Fail fast if upstream produces empty output

### Pattern 3: Background + Monitor

Spawn long-running agents in background, check progress asynchronously.

```
# Launch in background
terminal(command="codex exec --full-auto 'Refactor all error handling in src/'", workdir="~/project", background=true, pty=true)

# Returns session_id — save it
# Later, check progress:
process(action="poll", session_id="<id>")
process(action="log", session_id="<id>", limit=50)

# If stuck:
process(action="submit", session_id="<id>", data="What's your status?")

# When done:
process(action="log", session_id="<id>", offset=0, limit=200)
```

### Pattern 4: Fallback Chain

Try primary agent, fall back to alternative on failure.

```
# Try Codex first (faster, cheaper)
result = terminal(command="codex exec 'Fix the type errors in src/models/'", workdir="~/project", pty=true, timeout=300)
# If it failed or timed out, try Claude Code
if result.exit_code != 0:
    terminal(command="claude -p 'Fix the type errors in src/models/' --allowedTools \"Read,Edit,Bash\" --max-turns 10", workdir="~/project", timeout=180)
```

### Pattern 5: Kanban Orchestration (Persistent Multi-Agent)

For complex workflows that need crash resistance, review gates, and audit trails.

```
# Load kanban-orchestrator skill first
# 1. Discover profiles: hermes profile list
# 2. Create task graph with kanban_create()
# 3. Link dependencies with parents=[...]
# 4. Workers auto-spawn, report back via kanban_complete()

# Example: Research → Implement → Review pipeline
t1 = kanban_create(title="Research: evaluate auth libs", assignee="default", body="Compare 3 auth libraries for our use case. Output: recommendation with pros/cons.")["task_id"]
t2 = kanban_create(title="Implement chosen auth library", assignee="default", body="Based on research from T1, implement the recommended auth library. Include tests.", parents=[t1])["task_id"]
t3 = kanban_create(title: "Review auth implementation", assignee="default", body="Review T2 output for security issues and code quality.", parents=[t2])["task_id"]
```

### Pattern 6: Batch Parallel (Many Similar Tasks)

Run the same operation across many repos or files.

```
repos = ["frontend", "backend", "api-gateway", "auth-service"]
sessions = []
for repo in repos:
    sid = f"batch-{repo}"
    terminal(command=f"tmux new-session -d -d {sid} -x 140 -y 40 && tmux send-keys -t {sid} 'cd ~/{repo} && claude -p \"Add rate limiting to all API endpoints\" --allowedTools \"Read,Edit,Bash\" --max-turns 10' Enter")
    sessions.append(sid)

# Monitor all
sleep(60)
for sid in sessions:
    terminal(command=f"=== {sid ==="; tmux capture-pane -t {sid} -p -S -30")
```

## Agent Selection Guide

| Scenario | Recommended Agent | Why |
|----------|------------------|-----|
| Quick one-shot fix | Claude Code ` -p ` | Fastest for single tasks, no dialog handling |
| Multi-turn exploration | Claude Code (interactive + tmux) | Full TUI with slash commands |
| Batch issue fixing | Codex | Fast, parallel-friendly with worktrees |
| Open-source projects | OpenCode | Open-source, Go/Python focused |
| Crash-resistant workflow | Kanban workers | Survives restarts, audit trail |
| Internal subtask | `delegate_task` | No agent overhead, same session |
| Cost-sensitive work | Codex `--yolo` or Claude Code `--model haiku` | Lower per-token cost |
| Security-critical review | Claude Code `--model opus` + review mode | Deepest reasoning |

## Monitoring Patterns

### Check if Agent is Done

```bash
# Look for ❯ prompt (waiting = done or needs input)
tmux capture-pane -t <session> -p | tail -5

# Check for error indicators
tmux capture-pane -t <session> -p | grep -iE "error|failed|exception|timeout"
```

### Cost Monitoring (Claude Code)

```bash
# Print mode returns JSON with cost
claude -p "task" --output-format json 2>&1 | python3 -c "import json,sys; d=json.load(sys.stdin); print(f'Cost: ${d[\"total_cost_usd\"]:.4f}, Turns: {d[\"num_turns\"]}')"
```

### Session Management

```bash
# List all agent sessions
tmux list-sessions

# Kill a specific agent
tmux kill-session -t <session>

# Kill all agents matching pattern
tmux list-sessions -F '#{session_name}' | grep '^agent-' | xargs -I{} tmux kill-session -t {}
```

## Error Handling

### Agent Timeout
```
# If agent exceeds expected time, check if it's stuck or just slow
tmux capture-pane -t <session> -p -S -5
# If stuck: kill and retry with fewer max-turns
# If slow: give more time
```

### Agent Crash
```
# Check log for error
process(action="log", session_id="<id>", offset=0, limit=100)
# Restart with same task + context from crash log
```

### Agent Producing Bad Output
```
# For Kanban: reviewer blocks with feedback → new task spawned
# For tmux agents: kill session, adjust prompt, restart
# Common fixes:
#   - Be more specific in task description
#   - Add file paths explicitly
#   - Reduce max-turns to prevent tunnel vision
#   - Use --allowedTools to limit scope
#   - Add acceptance criteria to task description
```

## Hermes Subagent Delegation

For lightweight subtasks that don't need a full coding agent, use `delegate_task`:

```
# Spawn isolated subagent
result = delegate_task(
    goal="Analyze the auth flow in src/auth.py and list all security concerns",
    context="Project uses FastAPI + JWT. Focus on token handling, injection, and session management.",
    toolsets=["file", "terminal"]
)
# Returns summary — subagent did the work
```

**Rules:**
- Subagents are leaf nodes — they can't delegate further
- They have no memory of the parent conversation
- Pass all needed context explicitly
- Use for reasoning subtasks, not implementation

## Cross-Repository Orchestration

For work spanning multiple repos:

```
# Each agent works in its own repo
agents = [
    {"session": "fe", "repo": "frontend", "task": "Update API endpoint URLs to v2"},
    {"session": "be", "repo": "backend",  "task": "Add v2 API endpoints"},
    {"session": "docs", "repo": "docs",  "task": "Update API reference for v2"},
]
for a in agents:
    terminal(command=f"tmux new-session -d -s {a['session']} && tmux send-keys -t {a['session']} 'cd ~/{a['repo']} && claude -p \"{a['task']}\" --max-turns 10' Enter")

# After all done: integration test
terminal(command="cd ~/frontend && npm test -- --grep 'API v2'")
```

## Verification Checklist

- [ ] Agent skill loaded before spawning that agent type
- [ ] Each agent has its own named tmux session
- [ ] `--max-turns` set for print mode agents
- [ ] `workdir` set correctly per agent
- [ ] Monitoring frequency appropriate (not too aggressive)
- [ ] Sessions cleaned up after completion
- [ ] Costs logged if tracking spend
- [ ] Output captured before sessions are killed

## Common Pitfalls

1. **Not enough context in task description.** Agents can't see your screen. Include: file paths, function names, expected behavior, acceptance criteria.
2. **Too many agents at once.** 3-4 parallel agents is usually the sweet spot. More than that and monitoring overhead exceeds gains.
3. **Forgetting `--max-turns`** in print mode. An agent can loop 50+ turns on a confusing task. Set 5-15 for most tasks.
4. **Killing slow agents too early.** Some tasks (large refactors, complex debugging) take 10+ minutes. Check before killing.
5. **Not capturing output before killing.** Always `tmux capture-pane` or `process(action="log")` before cleanup.
6. **Reusing tmux session names.** Always use unique names or check `tmux list-sessions` first.
7. **Mixing agent output.** Keep sessions separate so you can attribute results.
8. **Ignoring agent errors.** If an agent reports "I can't do X", don't just retry the same prompt. Adjust or escalate.
