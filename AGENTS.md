# Agent instructions

This repository contains the **`reitit`** skill — a structured set of Markdown files that teach an AI coding agent how to work with Reitit, the fast data-driven router for Clojure and ClojureScript.

**Any agent that understands this `AGENTS.md` convention should:**

1. Treat `reitit/SKILL.md` as the entry point — it contains a decision table pointing to the right reference file for the task.
2. Load reference files on demand based on that table:
   - `core-concepts.md` — route syntax, router creation, path/name-based routing, route data, route conflicts
   - `ring.md` — Ring router, request-method routing, middleware, content negotiation, exception handling, interceptors
   - `coercion.md` — coercion with malli, spec, and schema; parameter types; response coercion; nested params
   - `openapi.md` — OpenAPI 3 and Swagger 2 doc generation, Swagger UI, per-content-type coercion
   - `frontend.md` — ClojureScript browser routing, HTML5/hash history, controllers, React/Reagent patterns
   - `anti-patterns.md` — common mistakes and things to avoid
3. Never mount coercion middleware before Muuntaja format middleware — body params must be parsed before coercion runs.
4. Always add an exception middleware — `reitit-ring` does not catch exceptions by default.
5. For dev workflows, use var-quoted handlers (`#'my-ns/handler`) or function-returning routers to support hot reloading without rebuilding the full router.

This file follows the [agents.md](https://agents.md/) convention and is honored by OpenAI Codex CLI, Cursor, Aider, Zed, Amp, Gemini CLI, Google Jules, Windsurf, Factory, RooCode, and many others.

For Claude Code, the richer native format is `.claude-plugin/` + `reitit/SKILL.md`.
