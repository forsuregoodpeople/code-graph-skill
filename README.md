# Code Graph Skill

Skill untuk review codebase secara visual dengan tracing alur eksekusi, data flow, dependency graph, dan complexity review.

## Install

`npx skill` hanya menerima paket dari `vercel-labs/agent-skills`. Untuk repo ini, install dengan clone/copy ke folder skills agent yang dipakai.

Untuk opencode:

```bash
git clone git@github.com:forsuregoodpeople/code-graph-skill.git ~/.config/opencode/skills/code-graph
```

Atau dari folder lokal:

```bash
mkdir -p ~/.config/opencode/skills/code-graph
cp -R ./* ~/.config/opencode/skills/code-graph/
```

## Usage

Gunakan saat ingin memahami alur kode:

```txt
trace this request
where does data go?
show execution path from login()
analyze codebase
review my code
```

Skill ini akan memprioritaskan output graph terlebih dahulu, lalu ringkasan singkat.

## Features

- Trace request/data/function flow dari entry point ke output
- Render graph visual + ASCII flowchart
- Deteksi architecture pattern
- Support zoom level Function → Statement → Expression → Variable → Property
- Support test trace untuk Cypress dan Playwright
- Deteksi circular dependency, god file, dead export, deep nesting, dan async error risk

## References

- `SKILL.md` — instruksi utama skill
- `references/mental-model.md` — schema symbol table dan zoom rules
- `references/trace-rendering.md` — aturan rendering graph
- `references/problem-pattern.md` — katalog anti-pattern
