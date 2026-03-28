# GSD FAQ

Common questions about how GSD works and how it compares to alternatives.

---

## Table of Contents

- [How does GSD choose which model/agent to use for each task?](#how-does-gsd-choose-which-modelagent-to-use-for-each-task)
- [What are the model profiles?](#what-are-the-model-profiles)
- [Can I override model choices per-agent?](#can-i-override-model-choices-per-agent)
- [What specialized agents does GSD use?](#what-specialized-agents-does-gsd-use)
- [What similar tools exist?](#what-similar-tools-exist)

---

## How does GSD choose which model/agent to use for each task?

GSD uses a **model profile system** combined with **per-agent routing** to assign the right model to each task. The resolution works through three layers:

### 1. Model Profiles (project-wide)

Your project's `.planning/config.json` contains a `model_profile` setting (default: `"balanced"`). This maps every agent to a specific model tier:

| Profile | Planning agents | Execution agents | Verification agents |
|---------|----------------|------------------|---------------------|
| **quality** | Opus | Opus | Sonnet |
| **balanced** (default) | Opus | Sonnet | Sonnet |
| **budget** | Sonnet | Sonnet | Haiku |
| **inherit** | Inherit | Inherit | Inherit |

### 2. Per-agent overrides (optional)

You can override the profile for specific agents via `model_overrides` in config:

```json
{
  "model_profile": "balanced",
  "model_overrides": {
    "gsd-executor": "opus",
    "gsd-codebase-mapper": "sonnet"
  }
}
```

### 3. Resolution at runtime

When a workflow runs (e.g., `/gsd:plan-phase 1`), it calls `gsd-tools.cjs init plan-phase 1` which resolves models for every agent role in the workflow:

```
1. Check model_overrides for agent → if found, use override
2. Otherwise, look up agent in MODEL_PROFILES table using active profile
3. Return model alias ("opus", "sonnet", "haiku") or "inherit"
```

The workflow then spawns each agent with its resolved model:

```
Research  → gsd-phase-researcher  → model: "sonnet"  (balanced profile)
Planning  → gsd-planner           → model: "opus"    (balanced profile)
Checking  → gsd-plan-checker      → model: "sonnet"  (balanced profile)
```

### Why different models for different tasks?

- **Opus for planning** — Architecture decisions and task decomposition benefit most from stronger reasoning
- **Sonnet for execution** — Following explicit plan instructions doesn't need Opus-level reasoning
- **Haiku for read-only analysis** — Pattern extraction and codebase mapping are low-reasoning tasks
- **Inherit for non-Claude runtimes** — Lets the runtime (Gemini, Codex, OpenCode) use its own model

Switch profiles with `/gsd:set-profile <profile>` or `/gsd:settings`.

---

## What are the model profiles?

Four built-in profiles control the quality/cost tradeoff:

| Agent | Quality | Balanced | Budget | Inherit |
|-------|---------|----------|--------|---------|
| gsd-planner | Opus | Opus | Sonnet | Inherit |
| gsd-roadmapper | Opus | Sonnet | Sonnet | Inherit |
| gsd-executor | Opus | Sonnet | Sonnet | Inherit |
| gsd-phase-researcher | Opus | Sonnet | Haiku | Inherit |
| gsd-project-researcher | Opus | Sonnet | Haiku | Inherit |
| gsd-research-synthesizer | Sonnet | Sonnet | Haiku | Inherit |
| gsd-debugger | Opus | Sonnet | Sonnet | Inherit |
| gsd-codebase-mapper | Sonnet | Haiku | Haiku | Inherit |
| gsd-verifier | Sonnet | Sonnet | Haiku | Inherit |
| gsd-plan-checker | Sonnet | Sonnet | Haiku | Inherit |
| gsd-integration-checker | Sonnet | Sonnet | Haiku | Inherit |
| gsd-nyquist-auditor | Sonnet | Sonnet | Haiku | Inherit |
| gsd-ui-researcher | Opus | Sonnet | Haiku | Inherit |
| gsd-ui-checker | Sonnet | Sonnet | Haiku | Inherit |
| gsd-ui-auditor | Sonnet | Sonnet | Haiku | Inherit |

**When to use each:**

- **quality** — Critical architecture work where you have API quota to spare
- **balanced** — Default; Opus only where it matters most (planning), Sonnet everywhere else
- **budget** — High-volume prototyping; Sonnet for code-writing, Haiku for analysis
- **inherit** — Non-Anthropic runtimes (OpenCode, Gemini CLI, Codex) or when you want agents to use your session's active model

---

## Can I override model choices per-agent?

Yes. Add `model_overrides` to `.planning/config.json`:

```json
{
  "model_profile": "balanced",
  "model_overrides": {
    "gsd-executor": "opus",
    "gsd-debugger": "opus",
    "gsd-codebase-mapper": "sonnet"
  }
}
```

Overrides take priority over the profile. You can also use fully-qualified model IDs for non-Anthropic providers:

```json
{
  "model_overrides": {
    "gsd-planner": "openai/o3",
    "gsd-executor": "openai/o4-mini"
  }
}
```

Or use `/gsd:settings` for interactive configuration.

---

## What specialized agents does GSD use?

GSD orchestrates 15+ specialized agents, each with focused capabilities:

| Agent | Role | Key tools |
|-------|------|-----------|
| **gsd-planner** | Task decomposition, dependency analysis | Read, Write, Bash, Glob, Grep |
| **gsd-executor** | Code implementation with atomic commits | Read, Write, Edit, Bash, Glob, Grep |
| **gsd-phase-researcher** | Phase-specific technical research | Read, Bash, Glob, Grep, WebSearch |
| **gsd-project-researcher** | Domain research before roadmap | Read, Bash, Glob, Grep, WebSearch |
| **gsd-research-synthesizer** | Combines parallel researcher output | Read, Write |
| **gsd-roadmapper** | Requirements traceability mapping | Read, Write, Bash, Glob, Grep |
| **gsd-verifier** | Post-execution goal verification | Read, Bash, Glob, Grep |
| **gsd-plan-checker** | 8-point plan verification | Read, Bash, Glob, Grep |
| **gsd-debugger** | Bug investigation with persistent state | Read, Write, Edit, Bash, Glob, Grep |
| **gsd-codebase-mapper** | Pattern extraction from existing code | Read, Bash, Glob, Grep |
| **gsd-integration-checker** | Cross-phase integration verification | Read, Bash, Glob, Grep |
| **gsd-nyquist-auditor** | Test coverage gap filling | Read, Write, Edit, Bash, Glob, Grep |
| **gsd-ui-researcher** | UI design contract generation | Read, Bash, Glob, Grep, WebSearch |
| **gsd-ui-checker** | UI-SPEC design validation | Read, Bash, Glob, Grep |
| **gsd-ui-auditor** | 6-pillar visual audit | Read, Bash, Glob, Grep |

Agents are spawned by thin orchestrator workflows that never do heavy lifting themselves — they just route work to the right specialist with a fresh context window.

---

## What similar tools exist?

Several tools occupy a similar space to GSD — spec-driven development, AI workflow orchestration, and context engineering for coding agents. Here's how they compare:

### Spec-Driven / AI Workflow Tools

| Tool | Approach | Best for | Key difference from GSD |
|------|----------|----------|------------------------|
| **[BMAD](https://github.com/bmadcode/BMAD-METHOD)** | Full SDLC methodology with roles (PM, architect, dev) | Teams wanting enterprise process structure | Heavier process — sprint ceremonies, story points, stakeholder syncs |
| **[Roo Code](https://github.com/RooVetGit/Roo-Code)** (formerly Roo Cline) | VS Code extension with multi-model AI agent | VS Code users wanting IDE-integrated AI | IDE-specific, different UX model |
| **[Cline](https://github.com/cline/cline)** | Autonomous coding agent in VS Code | VS Code-first developers | Single-agent model, no multi-agent orchestration |
| **[Aider](https://github.com/paul-gauthier/aider)** | Terminal-based AI pair programming | Developers who want chat-based coding in terminal | Pair programming model vs. spec-driven pipeline |
| **[Claude Engineer](https://github.com/Doriandarko/claude-engineer)** | Interactive Claude coding assistant | Quick interactive coding sessions | No planning pipeline; conversational approach |
| **[Cursor](https://cursor.com)** | AI-native code editor | Developers wanting AI baked into editor | Full IDE, not a workflow system |

### Task/Project Management for AI

| Tool | Approach | Best for | Key difference from GSD |
|------|----------|----------|------------------------|
| **[Taskmaster AI](https://github.com/eyaltoledano/claude-task-master)** | Task management with AI-generated subtasks | Breaking down PRDs into tasks | Task management focus, less context engineering |
| **[Speckit](https://github.com/speckit/speckit)** | Spec generation from natural language | Generating specs/requirements | Spec generation only, no execution pipeline |
| **[OpenSpec](https://github.com/OpenSpecAI/openspec)** | Open standard for AI specifications | Standardizing AI project specs | Spec standard, not an execution system |

### What makes GSD different?

1. **Context engineering** — Structured artifacts sized to avoid quality degradation; fresh 200K context per agent
2. **Multi-agent orchestration** — 15+ specialized agents vs. single-agent approaches
3. **End-to-end pipeline** — Requirements → research → planning → execution → verification in one system
4. **No enterprise theater** — No sprint ceremonies, story points, or Jira workflows
5. **Runtime-agnostic** — Works with Claude Code, Gemini CLI, Codex, Copilot, Cursor, Windsurf, OpenCode, and Antigravity
6. **Automatic model routing** — Right model for each task via profile system (Opus for planning, Sonnet for execution, Haiku for analysis)

GSD's philosophy: *"The complexity is in the system, not in your workflow."*
