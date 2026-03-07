---
title: "Anatomy of Agentic Coding Systems: What Claude Code, Cursor, and OpenCode Actually Share"
date: 2026-03-06T18:00:00-08:00
draft: false
weight: 2
tags:
  - AI
  - LLM
  - Agents
  - ML Engineering
categories:
  - AI
description: "A grounded teardown of production agentic coding systems—Claude Code, Cursor, OpenCode, OpenHands, Manus, and Kon—revealing that the architecture is a while loop, and the real differentiators are tool design, context management, and knowing when to stop."
showToc: true
TocOpen: true
---

Everyone is building agentic coding systems now. Claude Code, Cursor, OpenCode (117K GitHub stars), OpenHands, Manus—the list grows weekly. But strip away the marketing and the framework abstractions, and they all share the same skeleton.

This post tears down six production systems to extract the **bare minimum components** that make an agentic coding system work, with references to actual source code and architectural decisions.

---

## The Core: It's Just a While Loop

Every production agentic system converges on the same pattern. As [Braintrust documented](https://braintrust.dev/blog/agent-while-loop):

```typescript
while (!done) {
  const response = await callLLM(systemPrompt, messages, tools);
  messages.push(response);
  if (response.toolCalls) {
    messages.push(
      ...(await Promise.all(response.toolCalls.map(tc => executeTool(tc))))
    );
  } else {
    done = true;
  }
}
```

The agent is a system prompt, a handful of tools, and a loop. The LLM decides what to do next. When it stops calling tools and emits plain text, the task is done.

Claude Code calls this the ["master agent loop"](https://blog.promptlayer.com/claude-code-behind-the-scenes-of-the-master-agent-loop/)—a single-threaded loop named `nO` that runs gather context → take action → verify results. [Kon](https://github.com/kuutsav/kon), a minimal open-source agent with only 112 files, implements it as a two-layer design in [`loop.py`](https://github.com/kuutsav/kon/blob/main/src/kon/loop.py) and [`turn.py`](https://github.com/kuutsav/kon/blob/main/src/kon/turn.py): an outer loop manages turns and compaction, while the inner turn streams one LLM call and executes tools.

This pattern wins for the same reason UNIX pipes win: it's simple, composable, and flexible enough to handle complexity without becoming complex itself. As Braintrust puts it, *"many teams discover this pattern through experience—they start with sophisticated frameworks or graph-based structures, but often find simpler approaches more reliable in production."*

---

## The 6 Essential Components

### 1. The Agent Loop

Every system implements the same loop, but with slightly different termination semantics:

| System | Implementation | Termination |
|--------|---------------|-------------|
| **Claude Code** | Single-threaded `while` loop ([source](https://blog.promptlayer.com/claude-code-behind-the-scenes-of-the-master-agent-loop/)) | LLM returns text without tool calls |
| **Cursor** | Same pattern ([docs](https://cursor.com/docs/agent/overview)) | Same |
| **OpenCode** | Agentic loop in `prompt.ts` ([analysis](https://gist.github.com/rmk40/cde7a98c1c90614a27478216cc01551f)) | Same, plus `max-steps.txt` injection |
| **OpenHands** | Event-driven controller ([source](https://github.com/All-Hands-AI/OpenHands/blob/main/openhands/controller/agent_controller.py)) | `AgentFinishAction` event |
| **Kon** | Turn-based async loop ([source](https://github.com/kuutsav/kon/blob/main/src/kon/loop.py)) | `StopReason.STOP` or max turns |
| **Manus** | 6-stage circular loop ([blog](https://manus.im/hi/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)) | Task completion or user handoff |

The loop is always synchronous per session. Parallelism happens through **subagents** (separate loops), never within the main loop.

---

### 2. Tools: The Real Architecture

This is the most consequential design decision. From [Braintrust's analysis](https://braintrust.dev/blog/agent-while-loop): in a typical agent conversation, **tool responses make up 67.6% of total tokens**, while the system prompt accounts for just 3.4%. Tool definitions add another 10.7%. Tools comprise nearly 80% of what the agent sees.

Tool design *is* context engineering.

**Claude Code** uses [18 builtin tools](https://github.com/Piebald-AI/claude-code-system-prompts) across five categories:

| Category | Tools |
|----------|-------|
| File ops | Read, Write, Edit |
| Search | Glob, Grep |
| Execution | Bash, BashOutput, KillShell |
| Web | WebSearch, WebFetch |
| Delegation | Task, Explore |

**Kon** uses 6 tools—the practical minimum:

| Tool | Purpose |
|------|---------|
| `read` | Read files (pagination, images) |
| `edit` | Surgical find-and-replace |
| `write` | Create/overwrite files |
| `bash` | Shell execution |
| `grep` | Content search (ripgrep) |
| `find` | File discovery (fd) |

A [Medium analysis of OpenClaw](https://medium.com/@shivamagarwal7/agentic-ai-pi-anatomy-of-a-minimal-coding-agent-powering-openclaw-5ecd4dd6b440) goes further: their "Agent Pi" core runs on just **4 tools**: `read`, `write`, `bash`, `web_fetch`.

**Key design principle from Braintrust:** purpose-built tools with minimal parameters beat generic tools with many arguments. Don't expose raw APIs—wrap them into the agent's mental model:

```python
# Bad: generic communication tool with 10+ parameters
send_message(channel, recipient, content, subject, template,
             variables, priority, scheduling, tracking, metadata)

# Good: purpose-built for the agent's actual job
notify_customer(customer_email, message)
```

**Tool output formatting matters equally.** Claude Code and [OpenCode](https://github.com/sst/opencode) both cap tool output at **2,000 lines / 50KB**. When truncated, the full content is written to a temp file and the model is told to use Grep or delegate to an explore agent. Manus takes this further—[all verbose tool outputs are offloaded to the sandbox filesystem](https://manus.im/hi/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) and referenced by path.

---

### 3. System Prompt: Layered and Dynamic

No production system uses a single monolithic prompt. Claude Code has [110+ conditionally-added prompt strings](https://github.com/Piebald-AI/claude-code-system-prompts) that vary by environment, model, and configuration.

The universal layered structure:

```
┌─────────────────────────────────┐
│  Base identity / role           │  Static per provider
├─────────────────────────────────┤
│  Tool descriptions              │  ~10-15% of context
├─────────────────────────────────┤
│  Rules & constraints            │  Permissions, safety
├─────────────────────────────────┤
│  Project-specific context       │  AGENTS.md, CLAUDE.md
├─────────────────────────────────┤
│  Dynamic context (injected)     │  Open files, git status
└─────────────────────────────────┘
```

**OpenCode** selects different base prompts per model family: Claude gets `anthropic.txt`, GPT gets `beast.txt`, Gemini gets `gemini.txt` ([analysis](https://gist.github.com/rmk40/cde7a98c1c90614a27478216cc01551f)). If an agent (explore, plan) defines its own prompt, it replaces the provider prompt entirely.

**Instruction file discovery** is a shared pattern. All of Claude Code, OpenCode, Kon, and Cursor walk the filesystem from the working directory up to the git root, looking for `AGENTS.md` or `CLAUDE.md`. OpenCode also discovers instruction files *during tool execution*—when the `read` tool accesses a file in a subdirectory, it walks up from that directory looking for instruction files not yet loaded, injecting them as `<system-reminder>` blocks.

**Kon's** approach is the most minimal: system prompt (~215 tokens) + tool definitions (~600 tokens) = under 1K tokens before conversation context. Everything else comes from discovered `AGENTS.md` files and skills.

---

### 4. Context Management: Where Systems Diverge

This is the hardest engineering problem. [Research from Zylos](https://zylos.ai/research/2026-02-28-ai-agent-context-compression-strategies) estimates that context drift—not raw exhaustion—causes ~65% of enterprise AI agent failures, with 2% early misalignment compounding to 40% failure rates by task end.

A typical Manus task involves ~50 tool calls. At a 100:1 input-to-output token ratio, context grows fast.

**Each system's approach:**

| System | Strategy | Trigger |
|--------|----------|---------|
| **Claude Code** | Auto-compaction via summarizer component | ~92% of context window |
| **Cursor** | Automatic summarization of older messages ([docs](https://cursor.com/docs/agent/overview)) | Context window fills |
| **OpenCode** | Dedicated `compaction` agent with own prompt ([analysis](https://gist.github.com/rmk40/cde7a98c1c90614a27478216cc01551f)) | Token threshold |
| **Kon** | Configurable compaction with `continue` or `pause` modes ([source](https://github.com/kuutsav/kon/blob/main/src/kon/loop.py)) | `buffer_tokens` setting |
| **Manus** | Three-strategy: isolate, offload, reduce ([blog](https://manus.im/hi/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)) | KV-cache aware |

Kon's implementation is the clearest to study. From [`loop.py`](https://github.com/kuutsav/kon/blob/main/src/kon/loop.py):

```python
# After every turn, check for context overflow
if is_overflow(last_usage, context_window, max_output, buffer_tokens):
    summary = await generate_summary(
        session.all_messages, provider, system_prompt
    )
    session.append_compaction(summary=summary, ...)

    if config.on_overflow == "continue":
        # Inject synthetic message to keep going
        session.append_message(UserMessage(
            content="Continue if you have next steps, or stop and ask "
                    "for clarification if you are unsure."
        ))
    elif config.on_overflow == "pause":
        break  # Let user decide
```

The compaction prompt asks for structured output: Goal, Instructions, Discoveries, Accomplished, Relevant files—preserving what matters and discarding verbose tool outputs.

**Manus's three rules** are worth internalizing:
1. **Isolate context** — keep prompt prefixes stable for KV-cache hits (cached tokens cost [10x less](https://manus.im/hi/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus): $0.30 vs $3.00/M tokens on Claude Sonnet)
2. **Offload context** — store full results in the sandbox filesystem, reference by compact paths
3. **Reduce context** — compact stale tool results first, then summarize if needed

---

### 5. Sandbox / Execution Environment

Every system needs a safe place to run code. The trade-off is speed vs. safety:

| System | Sandbox | Philosophy |
|--------|---------|------------|
| **Claude Code** | Local machine + permission gates ([docs](https://code.claude.com/docs/en/permissions)) | Trust user's machine, gate dangerous ops |
| **Cursor** | Local machine + permission gates | Same |
| **OpenCode** | Local machine + configurable permissions ([analysis](https://gist.github.com/rmk40/cde7a98c1c90614a27478216cc01551f)) | Bash parses commands with tree-sitter for path checking |
| **OpenHands** | Docker container by default ([docs](https://docs.openhands.dev/sdk/arch/overview)) | Full isolation |
| **Kon** | Local, no sandbox ([README](https://github.com/kuutsav/kon)) | Simplicity over isolation |
| **Manus** | Virtual machine per session ([blog](https://manus.im/hi/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)) | Full isolation |

OpenCode's approach is interesting: the `bash` tool uses **tree-sitter** to parse shell commands, extract individual commands, and resolve file paths via `realpath`. Paths outside the project trigger external directory permission checks. This is more sophisticated than a simple allowlist.

The `edit` tool across systems also reveals different philosophies. OpenCode runs the model's `oldString` through a chain of **9 increasingly fuzzy replacer strategies**—from exact match through whitespace-normalized, indentation-flexible, and Levenshtein distance matching. Claude Code reportedly uses similar fuzzy matching. Kon keeps it simple: exact match only.

---

### 6. Subagent System

Subagents solve the biggest production problem: **context pollution**. A single `grep` over a large codebase can return thousands of lines that pollute the main context with irrelevant information.

The solution: run the search in a separate agent loop with its own context window. Only the final result enters the parent.

| System | Subagents | Key Design |
|--------|-----------|------------|
| **Claude Code** | Task, Explore, Plan ([source](https://github.com/Piebald-AI/claude-code-system-prompts)) | Explore is read-only with faster model |
| **Cursor** | Explore, Bash, Browser ([docs](https://cursor.com/docs/agent/subagents)) | Each isolates noisy output |
| **OpenCode** | Build, Plan, General, Explore + 3 hidden ([analysis](https://gist.github.com/rmk40/cde7a98c1c90614a27478216cc01551f)) | Hidden agents for compaction, title, summary |
| **OpenHands** | MonologueAgent, CodeActAgent ([docs](https://docs.openhands.dev/sdk/arch/agent)) | Event-driven delegation |
| **Kon** | None (by design) | Simplicity trade-off |
| **Manus** | None (planner module instead) | Modular architecture |

From the parent agent's perspective, a subagent is just another tool:

```json
{
  "name": "task",
  "description": "Launch a subagent for complex tasks. Gets its own context.",
  "input_schema": {
    "type": {"enum": ["explore", "task", "plan"]},
    "prompt": "Detailed task description with all necessary context"
  }
}
```

OpenCode's `task` tool creates a child session with restricted permissions—no todo tools, no recursive task calls unless explicitly allowed. The [Cursor docs](https://cursor.com/docs/agent/subagents) explain the three benefits: context isolation, parallel execution, and cost efficiency (cheaper models for simple searches).

Kon deliberately omits subagents. From its [README](https://github.com/kuutsav/kon): the philosophy is to stay minimal and give preference to running local LLMs. For a 112-file codebase, that's a reasonable trade-off.

---

## The Minimum Viable Agent

Based on what actually ships in production, here's the stack in tiers:

**Tier 1 — MVP (can solve real tasks):**
- Agent while loop (terminate when LLM returns text)
- 4-6 tools: `read`, `write`, `edit`, `bash`, `grep`, `glob`
- System prompt with role + tool descriptions
- Basic token counting

**Tier 2 — Production-ready:**
- Context compaction (auto-summarize at threshold)
- Subagent system (parallel search, context isolation)
- Permission/safety gates
- Streaming output
- Tool output truncation (2K lines / 50KB)

**Tier 3 — Scale:**
- KV-cache optimization (stable prompt prefixes)
- Dynamic context discovery (pull vs. push)
- Skills/rules system (project-specific knowledge)
- MCP integration (extensible tool ecosystem)
- Provider-specific prompt tuning
- Fuzzy edit matching

---

## What Actually Differentiates These Systems

The architecture is the same across all six systems. The differentiation is in:

**1. Tool design quality (40% of impact).** Clear tool descriptions with "when to use" and "when not to use" matter more than the number of tools. [Bloomberg's ACL 2025 research](https://arxiv.org/abs/2505.00000) found that jointly optimizing tool descriptions reduced unnecessary tool calls by 70%.

**2. Context engineering (30%).** What the agent sees at each step matters more than the loop structure. Manus's 100:1 input-to-output ratio means most of the compute is spent processing accumulated context, not generating new tokens.

**3. Knowing when to stop (20%).** When to ask the user vs. keep going vs. give up. OpenCode injects a `max-steps.txt` as a fake assistant message when the model exceeds its step limit, forcing a summary. Kon offers `pause` mode on compaction, handing control back to the user.

**4. Model-specific tuning (10%).** OpenCode maintains separate base prompts per model family. Claude Code's [reverse-engineered prompts](https://github.com/Piebald-AI/claude-code-system-prompts) show ~40 system reminders that are conditionally injected based on the environment.

---

## Putting It All Together

Here's the complete minimal implementation—a working coding agent in ~80 lines:

```python
import anthropic, subprocess, os

client = anthropic.Anthropic()

TOOLS = [
    {"name": "read", "description": "Read a file. Returns line-numbered content.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}}, "required": ["path"]}},
    {"name": "write", "description": "Write content to a file.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "content": {"type": "string"}}, "required": ["path", "content"]}},
    {"name": "edit", "description": "Replace old_string with new_string in a file. old_string must match exactly once.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "old_string": {"type": "string"}, "new_string": {"type": "string"}}, "required": ["path", "old_string", "new_string"]}},
    {"name": "bash", "description": "Run a shell command. Use for git, tests, builds, grep, find.",
     "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}},
]

SYSTEM = """You are a coding agent. You have tools to read, write, edit files and run commands.
Always read a file before editing it. Verify changes by running tests when possible."""

def execute(name, args):
    if name == "read":
        lines = open(args["path"]).readlines()
        return "\n".join(f"{i+1:6}|{l.rstrip()}" for i, l in enumerate(lines))
    elif name == "write":
        os.makedirs(os.path.dirname(args["path"]) or ".", exist_ok=True)
        open(args["path"], "w").write(args["content"])
        return f"Wrote {args['path']}"
    elif name == "edit":
        content = open(args["path"]).read()
        count = content.count(args["old_string"])
        if count == 0: return "ERROR: old_string not found"
        if count > 1: return f"ERROR: {count} matches. Be more specific."
        open(args["path"], "w").write(content.replace(args["old_string"], args["new_string"], 1))
        return f"Edited {args['path']}"
    elif name == "bash":
        r = subprocess.run(args["command"], shell=True, capture_output=True, text=True, timeout=120)
        output = (r.stdout + r.stderr)[:50_000]
        return output or "(no output)"

def agent(query):
    messages = [{"role": "user", "content": query}]
    for _ in range(50):  # max turns
        response = client.messages.create(
            model="claude-sonnet-4-20250514", system=SYSTEM,
            messages=messages, tools=TOOLS, max_tokens=8096,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return next(b.text for b in response.content if hasattr(b, "text"))
        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = execute(block.name, block.input)
                print(f"  [{block.name}] {str(output)[:120]}")
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})
        messages.append({"role": "user", "content": results})
```

This can read code, write files, make surgical edits, run tests, and fix bugs. The same loop structure powers Claude Code with 110+ prompt fragments, Cursor with subagents and browser control, and OpenCode with 117K stars and 75+ LLM providers.

The architecture is trivially simple. The hard problems are tool design, context management, and knowing when to stop.

---

## References

### Systems Analyzed
1. **Claude Code** — [How it works](https://code.claude.com/docs/en/how-claude-code-works) | [Reverse-engineered prompts](https://github.com/Piebald-AI/claude-code-system-prompts) | [Agent loop analysis](https://blog.promptlayer.com/claude-code-behind-the-scenes-of-the-master-agent-loop/)
2. **Cursor** — [Agent overview](https://cursor.com/docs/agent/overview) | [Subagents](https://cursor.com/docs/agent/subagents) | [Dynamic context discovery](https://cursor.com/blog/dynamic-context-discovery)
3. **OpenCode** — [GitHub (117K stars)](https://github.com/sst/opencode) | [Prompt construction analysis](https://gist.github.com/rmk40/cde7a98c1c90614a27478216cc01551f) | [opencode.ai](https://opencode.ai/)
4. **OpenHands** — [GitHub](https://github.com/all-hands-ai/openhands) | [Architecture docs](https://docs.openhands.dev/sdk/arch/overview) | [Agent docs](https://docs.openhands.dev/sdk/arch/agent)
5. **Kon** — [GitHub (174 stars)](https://github.com/kuutsav/kon) | [loop.py](https://github.com/kuutsav/kon/blob/main/src/kon/loop.py) | [turn.py](https://github.com/kuutsav/kon/blob/main/src/kon/turn.py)
6. **Manus** — [Context engineering blog](https://manus.im/hi/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)

### Analysis & Patterns
7. **Braintrust** — [The canonical agent architecture: A while loop with tools](https://braintrust.dev/blog/agent-while-loop)
8. **Zylos Research** — [AI Agent Context Compression Strategies](https://zylos.ai/research/2026-02-28-ai-agent-context-compression-strategies)
9. **arXiv:2601.07190** — Active Context Compression: Autonomous Memory Management in LLM Agents

### SDK & Tools
10. **Anthropic Claude Agent SDK** — [GitHub](https://github.com/anthropics/claude-agent-sdk-python)
11. **OpenClaw** — [System architecture](https://openclawlab.com/en/docs/concepts/system-architecture/) | [Agent Pi analysis](https://medium.com/@shivamagarwal7/agentic-ai-pi-anatomy-of-a-minimal-coding-agent-powering-openclaw-5ecd4dd6b440)

*All source code links verified as of March 6, 2026.*
