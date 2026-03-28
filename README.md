# wukong

> **Memory Efficient Coding Agent (MECA)** — reference implementation.

---

## What Is MECA

MECA (Memory Efficient Coding Agent) is a class of coding agent defined by the following properties:

1. **VRAM budget is a first-class constraint** — the agent's tool schema, context management, and task decomposition are designed around a defined VRAM ceiling, not retrofitted to one
2. **Tool schema surface is minimized for reliable small model parsing** — flat, typed, no nested objects
3. **Context is actively managed per step** — token spend is tracked and pruned explicitly, not truncated as a fallback
4. **Task decomposition happens before model invocation** — tasks are broken into subtasks bounded by small model working memory limits before the model sees anything
5. **Reliability is instrumented and published** — benchmark scores per task class are part of the artifact, not anecdotal

An agent that supports local models but was not designed around these constraints is not MECA-class — it is a large-model agent with local inference shimmed in.

---

## What This Is

wukong is the first MECA-class implementation. Primary target: 4–8GB VRAM via ollama.

Aider, opencode, goose — all built for large models. Run them on a 7b model and they degrade silently: context assumptions are wrong, tool schemas are too complex for reliable small model parsing, no budget awareness.

wukong is built the other way. The 7b model is the design target. Everything follows from that.

---

## Why The Architecture Looks This Way

| Problem | What existing agents do | What wukong does |
|---|---|---|
| 7b models hallucinate complex tool call formats | Simplify nothing, degrade silently | Flat tool schema with minimal surface area |
| Context bloat tanks small model output quality | Truncate or ignore | Active token budget tracking with explicit pruning per step |
| Full tasks exceed small model working memory | Pass full task, let model figure it out | Rule-based decomposition into bounded subtasks before model sees anything |
| No reliability numbers for small model agents | Anecdotal | statma-native logging on every tool call, published benchmark scores |

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                   wukong CLI                    │
│              (typer, single entry)              │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│              Task Decomposer                    │
│       rule-based → bounded subtask queue        │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│               Agent Loop                        │
│         Think → Tool Call → Observe             │
│              (ReAct pattern)                    │
└──────┬──────────────────────────┬───────────────┘
       │                          │
┌──────▼──────┐          ┌────────▼────────┐
│   Context   │          │  Tool Registry  │
│   Budget    │          │  (4 primitives) │
│   Manager   │          └────────┬────────┘
└─────────────┘                   │
                     ┌────────────┼────────────┐
               ┌─────▼──┐  ┌─────▼──┐  ┌──────▼─────┐  ┌──────────┐
               │  read  │  │ write  │  │    run     │  │  search  │
               └────────┘  └────────┘  └────────────┘  └──────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│              Model Adapter Layer                │
│         OllamaAdapter (v0.1 only)              │
│         --model qwen2.5-coder:7b               │
└─────────────────────────────────────────────────┘
```

---

## Tool Schema (v0.1)

Four tools. No more in v0.1.

Schema is flat and typed — no nested objects, no optional fields with ambiguous behavior. 7b models produce unreliable tool call JSON when the schema surface gets complex. Keeping it minimal is a deliberate reliability decision, not a scope cut.

```
read_file(path: str, lines: tuple[int,int] | None) → str
write_diff(path: str, diff: str)                  → str
run_command(cmd: str, timeout: int)               → {stdout, stderr, exit_code}
search_codebase(pattern: str, path: str | None)   → list[Match]
```

Tool surface expands only when statma scores confirm reliability holds after the change.

---

## Context Budget Manager

7b models produce garbage output when the context window fills — they don't fail cleanly, they hallucinate. The budget manager runs every agent step, not as a fallback.

**Pruning order:**
1. Summarize old conversation turns
2. Drop file context outside the active diff window
3. Hard prune to keep: system prompt + active subtask + last N tool results

Token spend is tracked per subtask and written to the statma log.

---

## Task Decomposer

Tasks come in before the model sees them. The decomposer splits them into subtasks sized for small model working memory using rule-based heuristics — no LLM call to plan the LLM calls.

LLM-driven planning was considered and rejected for v0.1: it adds a model call before every task, compounds failure modes, and makes benchmark results harder to interpret.

**Example:**

Input: `"Refactor AuthService to use dependency injection"`

Decomposed queue:
```
1. read_file(auth_service.py)
2. identify constructor dependencies
3. write_diff: extract interface
4. write_diff: inject via constructor
5. run_command: python -m pytest tests/test_auth.py
```

---

## Repo Structure

```
wukong/
├── README.md
├── pyproject.toml
├── CHANGELOG.md
├── LICENSE
│
├── wukong/
│   ├── __init__.py
│   ├── cli.py                  # typer entry point
│   ├── agent.py                # ReAct loop
│   │
│   ├── tools/
│   │   ├── base.py             # ToolResult, ToolError, shared contract
│   │   ├── read.py
│   │   ├── write.py
│   │   ├── run.py
│   │   └── search.py
│   │
│   ├── adapters/
│   │   └── ollama.py           # OllamaAdapter — model as runtime param
│   │
│   ├── context/
│   │   └── budget.py           # ContextBudget: track, prune, summarize
│   │
│   └── decomposer/
│       └── task.py             # TaskDecomposer: rule-based subtask queue
│
├── benchmarks/
│   ├── suite.py                # WukongSuite: statma-compatible runner
│   └── tasks/
│       ├── refactor_single.py
│       ├── test_gen.py
│       └── docstring_pass.py
│
├── tests/
│   ├── unit/
│   │   ├── test_tools.py
│   │   ├── test_budget.py
│   │   └── test_decomposer.py
│   └── integration/
│       └── test_agent_loop.py
│
└── docs/
    ├── architecture.md         # ADRs
    ├── tool-schema.md          # formal tool spec
    └── statma-contract.md      # instrumentation output contract
```

---

## Target Models (v0.1)

| Model | VRAM | Role |
|---|---|---|
| `qwen2.5-coder:7b` | ~4.5GB | primary target |
| `deepseek-r1:8b` | ~5GB | secondary / reasoning-heavy tasks |

Model is a runtime flag. The agent does not hardcode a model — `--model` sets it at session start.

---

## statma Integration

wukong is the primary benchmark target for [statma](https://github.com/davidgracemann/statma).

Every tool call writes a structured log entry:

```json
{
  "tool": "write_diff",
  "task_class": "refactor_single",
  "model": "qwen2.5-coder:7b",
  "success": true,
  "latency_ms": 312,
  "retry_count": 0,
  "goal_faithful": true,
  "context_tokens_used": 2841
}
```

**Tracked metrics:**
- Tool call success rate per task class
- Goal faithfulness score
- Context token efficiency
- Failure recovery rate
- End-to-end latency per subtask

Benchmark scores published in this README once v0.1 loop closes.

---

## v0.1 Scope

**In scope:**
- Single model, single session, single task
- Four tool primitives
- Ollama adapter only
- Rule-based decomposer
- statma instrumentation on every tool call
- qwen2.5-coder:7b as primary target

**Out of scope until benchmarks justify adding it:**
- Multi-model routing
- Parallel tool calls
- Persistent session memory
- Web or external API tools
- TUI or GUI
- Multi-agent orchestration

---

## Architectural Constraints

Fixed for v0.1. Changes require an ADR.

1. Tool schema stays flat — no nested objects in tool signatures
2. Context budget runs every step — not skipped for latency
3. Decomposer is rule-based — no LLM call in the planning phase
4. Model is always a runtime parameter — never hardcoded
5. Every tool call is statma-logged — no silent execution

---

## ADR Index

| # | Decision | Status |
|---|---|---|
| ADR-001 | ReAct as agent loop pattern | Accepted |
| ADR-002 | Rule-based over LLM-driven decomposition in v0.1 | Accepted |
| ADR-003 | Flat tool schema | Accepted |
| ADR-004 | ollama-only adapter in v0.1 | Accepted |
| ADR-005 | statma as native instrumentation layer | Accepted |

Full writeups in `docs/architecture.md`.

---

## Installation

```bash
# requires uv
git clone https://github.com/davidgracemann/wukong
cd wukong
uv sync
uv run wukong --help
```

Requires ollama running locally with at least one target model pulled:

```bash
ollama pull qwen2.5-coder:7b
```

---

## Usage

```bash
# basic task
wukong "add type annotations to utils/parser.py" --model qwen2.5-coder:7b

# with explicit context budget cap
wukong "refactor AuthService to use DI" --model qwen2.5-coder:7b --budget 4096

# run statma benchmark suite
wukong benchmark --suite refactor_single --model qwen2.5-coder:7b
```

---

## Build Status

| Component | Status |
|---|---|
| Repo scaffold | ✅ done |
| OllamaAdapter | 🔧 building |
| Tool primitives (4) | 🔧 building |
| ReAct agent loop | 🔧 building |
| Context budget manager | 🔧 building |
| Task decomposer | 🔧 building |
| statma instrumentation | 🔧 building |
| v0.1 benchmark scores | ⏳ pending loop close |

---

## Related

- [statma](https://github.com/davidgracemann/statma) — CLI benchmarking tool for AI agents. wukong is its primary benchmark target.

---

## License

Apache 2.0
