# Code Graph Skill

> A visual codebase review skill for tracing execution flow, data flow, dependency graphs, and structural complexity.

## About

Code Graph Skill turns compatible AI coding assistants into visual code reviewers. It helps an agent trace where a request, data object, file operation, function call, or test action starts, what it touches, and where it ends.

It prioritizes graph-first output: directed execution traces, ASCII flowcharts, dependency maps, zoomable symbol views, and concise findings.

- **Execution tracing** — request/data/function flow from entry point to final output
- **Graph-first review** — visual graph before prose explanation
- **5 zoom levels** — Function → Statement → Expression → Variable → Property
- **Happy + error paths** — shows normal flow and failure branches together
- **Architecture review** — dependency graphs, module maps, complexity hotspots
- **Test tracing** — supports Cypress and Playwright files as trace entry points
- **Problem detection** — circular dependencies, god files, dead exports, deep nesting, async error risk

## Installation

### Via `skills` CLI (recommended)

Requires `npx` and Node.js.

**Remote install (from GitHub):**

```bash
npx skills add git@github.com:forsuregoodpeople/code-graph-skill.git
```

**Local install (from cloned directory):**

```bash
git clone git@github.com:forsuregoodpeople/code-graph-skill.git
npx skills add ./code-graph-skill
```

The CLI will prompt you to select which agents to install to, such as OpenCode, Claude Code, Cursor, Cline, and other supported agents.

### Manual install (OpenCode)

Copy `SKILL.md` and `references/` to your OpenCode skills folder:

```bash
git clone git@github.com:forsuregoodpeople/code-graph-skill.git ~/.config/opencode/skills/code-graph
```

Expected structure:

```txt
~/.config/opencode/skills/code-graph/
├── SKILL.md
└── references/
    ├── mental-model.md
    ├── trace-rendering.md
    └── problem-pattern.md
```

### Manual install (Claude Code)

Copy `SKILL.md` and `references/` to your Claude Code skills folder:

```txt
~/.claude/skills/code-graph/
├── SKILL.md
└── references/
    ├── mental-model.md
    ├── trace-rendering.md
    └── problem-pattern.md
```

### Trigger phrases

The skill auto-activates when your input matches phrases like:

- "trace this request"
- "where does data go?"
- "what happens after X?"
- "show execution path from login()"
- "arah data kemana?"
- "trace function ini"
- "analyze codebase"
- "review my code"
- "map the flow"
- "show dependency graph"

## What is this?

This is a reusable AI skill: a structured instruction set that teaches an agent how to inspect code visually through execution traces, dependency graphs, data-flow analysis, and complexity review.

It is **not a software application** with build/run/test steps. It is a prompt framework loaded automatically when your input matches code-tracing or code-review trigger phrases.

## Core philosophy

Code structure is easier to understand when flow is visible first:

```txt
Code understanding = symbols + edges + direction + side effects
Graph first        = less guessing, faster debugging, clearer review
```

The skill builds an internal symbol table before answering, then renders the most useful graph for the user's question.

## How it works

```txt
User asks about code flow
  → detect mode
  → build internal symbol table
  → choose zoom level
  → trace entry point
  → render directed graph
  → show happy path + error path
  → flag complexity/problem patterns
  → summarize only what matters
```

## Modes

The skill auto-detects user intent and selects the right mode:

| Intent | Mode | Action |
|--------|------|--------|
| Trace a request/function/data path | **Trace** | Render directed execution/data-flow graph |
| Shared multiple files or folder tree | **Trace** | Silently pick the primary entry point and deep scan end-to-end |
| Cypress or Playwright test file | **Test Trace** | Extract test actions and trace each path |
| Review/complexity/dependencies | **Review** | Show architecture graph and structural risks |
| Ambiguous input | **Trace or question** | Trace the primary detectable entry point; ask only if no entry point is detectable |

Core rule: **Trace mode wins. Always show a visual graph first. Trace top-to-bottom until every reachable branch reaches a terminal output.**

## Zoom levels

| Level | Focus | Node type |
|-------|-------|-----------|
| A | Function | Function/method calls |
| B | Statement | Assignments, returns, branches, throws |
| C | Expression | Calls, awaits, property access, transforms |
| D | Variable | Bindings, assignments, reads, mutations |
| E | Property | Field access chains like `req.body.email` |

Default starts at **Level A**. The user can ask to "drill into X", "zoom ke X", or "lebih detail" to move deeper.

## Deep scan contract

Trace does not stop at the first handler/controller/service. It follows every reachable in-repo call from the selected entry point through middleware, validators, services, repositories, models, helpers, serializers/resources, side effects, and response builders until each branch reaches a terminal node: response, return, DB/file/cache write, emitted event/job/webhook/email, handled error response, `[PARTIAL]`, or `[EXTERNAL]`.

## Graph conventions

```txt
[ENTRY]      where execution begins
[ROUTER]     route or dispatch layer
[MIDDLEWARE] auth, validation, rate limit, logging
[CONTROLLER] request handler
[SERVICE]    business logic
[REPOSITORY] data access
[DB/STORE]   database, cache, filesystem
[EXTERNAL]   third-party API, queue, email service
[RESPONSE]   final output
[ERROR]      exception or error response
```

Badges:

```txt
★  side effect
⟳  async boundary
⊕  data transform
```

## Project structure

```txt
├── SKILL.md                       # Core skill definition and protocols
├── references/
│   ├── mental-model.md            # Symbol table schema and zoom rules
│   ├── trace-rendering.md         # Execution graph rendering rules
│   └── problem-pattern.md         # Structural anti-pattern catalog
```

## Reference files

- **mental-model.md** — Symbol table schema, zoom levels, graph build rules, and extraction model
- **trace-rendering.md** — Happy/error path rendering, edge labels, node badges, Mermaid/ASCII rules
- **problem-pattern.md** — Circular dependency, god file, deep nesting, dead export, async risk, fan-in/fan-out rules

## Key rules

- **Graph first, prose second.**
- **Always show happy path and error path.**
- **Label every edge with what moves through it.**
- **Never guess missing code; mark partial or unknown.**
- **External library calls are `[EXTERNAL]`, not traced internally.**
- **Use dashed edges for dynamic dispatch and error paths.**
- **If flow is requested, trace from the named entry point.**

## License

MIT
