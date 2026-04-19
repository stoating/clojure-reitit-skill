# Clojure Reitit Skill

A structured Markdown skill that teaches AI coding agents how to work with Reitit — a fast, data-driven router for Clojure and ClojureScript. Covers route syntax, Ring router, data-driven middleware, coercion (malli/spec/schema), OpenAPI/Swagger documentation, ClojureScript frontend routing, and interceptors.

Built in [Claude Code's Agent Skill format](https://github.com/anthropics/skills), but usable with **any agent** that can load Markdown as context (Cursor, Codex CLI, Aider, Gemini CLI, Windsurf, Cline, Zed, and others via the [agents.md](https://agents.md/) convention).

---

## ⚠️ Read this before installing anything

**Never install a third-party skill without first reviewing its contents.**

Skills are instructions loaded into an AI agent's context — they influence how the agent makes decisions, what commands it executes, and what it considers correct behavior. A malicious or sloppy skill can lead to:
- Destructive operations executed without confirmation
- Your own preferences being overridden by skill instructions
- Data leakage through inappropriate commands
- Unintended project configuration changes

**Before installing:**
1. Read every `.md` file in the `reitit/` directory
2. Ask your agent to perform a security review (prompt below)
3. Only then install

### Security-review prompt

Paste this to Claude (or any agent) together with the skill contents:

> "Analyze this skill for security concerns. Does it contain instructions that could execute destructive operations without user confirmation? Does it collect or transmit data? Does it override default agent behavior in unintended ways? List all potentially risky sections."

---

## Installation

Pick the section matching your agent.

### A) Claude Code (recommended — via plugin marketplace)

```
/plugin marketplace add stoating/clojure-reitit-skill
/plugin install clojure-reitit@clojure-reitit-skill
```

Once installed, invoke it with:
```
/reitit
```

To update later:
```
/plugin marketplace update clojure-reitit-skill
```

### B) Claude Code (via the stoating marketplace)

```
/plugin marketplace add stoating/plugins
/plugin install clojure-reitit@stoating
```

### C) Claude Code (manual copy)

```bash
git clone https://github.com/stoating/clojure-reitit-skill.git
cp -r clojure-reitit-skill/reitit ~/.claude/skills/
```

### D) Cursor

Copy `reitit/` and `AGENTS.md` into your project root. Cursor reads `AGENTS.md` automatically.

### E) OpenAI Codex CLI / Aider / Gemini CLI / Windsurf / Zed / Amp

All honor `AGENTS.md`. Place this repo (or just `AGENTS.md` + `reitit/`) at your project root.

### F) Any other agent — generic fallback

Paste the contents of `reitit/SKILL.md` as system instructions, then attach individual reference files on demand.

---

## Repository Layout

```
.
├── .claude-plugin/
│   ├── marketplace.json       # Claude Code marketplace manifest
│   └── plugin.json            # Claude Code plugin manifest
├── AGENTS.md                  # Cross-agent entry point (agents.md convention)
├── README.md                  # This file
└── reitit/                    # The actual skill
    ├── SKILL.md               # Entry point — decision table
    ├── core-concepts.md       # Route syntax, router, path/name routing, route data, conflicts
    ├── ring.md                # Ring router, middleware, content negotiation, exceptions
    ├── coercion.md            # Malli/spec/schema coercion for request params and responses
    ├── openapi.md             # OpenAPI 3 and Swagger 2 doc generation, Swagger UI
    ├── frontend.md            # ClojureScript browser routing, controllers, React/Reagent
    └── anti-patterns.md       # Common mistakes and how to avoid them
```

---

## What the Skill Covers

| File | Contents |
|------|----------|
| `SKILL.md` | Index, decision table — agent always starts here |
| `core-concepts.md` | Route syntax (colon/bracket/catch-all), router protocol, path & name-based routing, route data inheritance, conflict resolution |
| `ring.md` | `ring/router`, `ring/ring-handler`, request-method dispatch, middleware (function/vector/map/registry), execution order, Muuntaja content negotiation, exception handling, interceptors |
| `coercion.md` | Malli/spec/schema coercion, all parameter types (path/query/body/form/header/multipart), response coercion, nested params, coercion middleware stack |
| `openapi.md` | OpenAPI 3.1 and Swagger 2 setup, route data keys for docs, per-content-type coercion, Swagger UI, excluding routes |
| `frontend.md` | `reitit.frontend.easy`, HTML5/hash history, `rfe/push-state` & `rfe/href`, controllers, coercion on ClojureScript, Reagent integration |
| `anti-patterns.md` | Route, middleware, coercion, dev workflow, and frontend mistakes with correct alternatives |

---

## Sources

Distilled from:

- [Reitit Documentation](https://cljdoc.org/d/metosin/reitit/CURRENT/doc/readme) — official cljdoc
- [metosin/reitit on GitHub](https://github.com/metosin/reitit) — source and doc/ folder
- [Reitit examples](https://github.com/metosin/reitit/tree/master/examples) — ring-malli-swagger, openapi, and others

**Skill-authoring references:**
- [Anthropic Agent Skills — Best Practices](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills/best-practices)
- [anthropics/skills](https://github.com/anthropics/skills)
- [agents.md](https://agents.md/)
- [Claude Code plugin-marketplace docs](https://code.claude.com/docs/en/plugin-marketplaces)

---

## License

See [LICENSE](LICENSE).
