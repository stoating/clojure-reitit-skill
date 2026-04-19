---
name: reitit
description: Use when working with Reitit — a fast data-driven router for Clojure/ClojureScript. Activate when the user asks about route syntax, routers, Ring handlers, middleware, coercion (malli/spec/schema), OpenAPI/Swagger docs, frontend browser routing, interceptors, path/name-based routing, route conflicts, or building HTTP APIs with Reitit.
version: 1.0.0
---

# Reitit

Reitit is a fast, data-driven router for Clojure and ClojureScript. Routes are plain data — vectors of `[path route-data children*]`. Every routing component (middleware, coercion, docs) reads from that data map.

## Quick Decision: What do you need?

| Task | Go to |
|------|-------|
| Understand route syntax, router creation, path/name-based routing, route data, route conflicts | [core-concepts.md](core-concepts.md) |
| Ring handler, request-method routing, middleware, content negotiation, exception handling | [ring.md](ring.md) |
| Coercing request params / response bodies with malli, spec, or schema | [coercion.md](coercion.md) |
| Generate OpenAPI 3 or Swagger 2 docs | [openapi.md](openapi.md) |
| Frontend/browser routing (ClojureScript, hash/HTML5 history, controllers) | [frontend.md](frontend.md) |
| Something isn't working / weird behavior / things to avoid | [anti-patterns.md](anti-patterns.md) |

## Core Mental Model

Routes are **pure data** — a nested vector of `[path ?data & children]`. A `router` compiles that tree once at startup: it flattens paths, merges nested route data (via meta-merge), resolves conflicts, and selects the fastest matching algorithm. At request time, `match-by-path` or `match-by-name` returns a `Match` record containing `:path-params`, `:data`, and a compiled `:result`.

**Ring layer** (`reitit-ring`) extends the core router with request-method dispatch (`:get`, `:post`, …), `:handler` functions, and a `:middleware` chain that is compiled per-endpoint — middleware that declares `:compile` can inspect route data and return `nil` to skip itself entirely, yielding zero overhead for routes that don't need it.

**Coercion** is pluggable: attach a `Coercion` implementation (malli, spec, or schema) under `:coercion` in route data, declare `:parameters` / `:responses` schemas, then mount the coercion middleware. Coerced values arrive in `request :parameters`.

## Modules

| Artifact | Purpose |
|----------|---------|
| `metosin/reitit` | All bundled |
| `metosin/reitit-core` | Routing core only |
| `metosin/reitit-ring` | Ring router + middleware support |
| `metosin/reitit-middleware` | Common data-driven middleware (params, exceptions, muuntaja) |
| `metosin/reitit-coercion` | Coercion protocol |
| `metosin/reitit-schema` | Plumatic Schema coercion |
| `metosin/reitit-spec` | clojure.spec / data-specs coercion |
| `metosin/reitit-malli` (via `reitit`) | Malli coercion |
| `metosin/reitit-swagger` | Swagger 2 docs |
| `fi.metosin/reitit-openapi` | OpenAPI 3 docs |
| `metosin/reitit-swagger-ui` | Integrated Swagger UI |
| `metosin/reitit-frontend` | ClojureScript frontend routing |
| `metosin/reitit-http` | Pedestal-style interceptor routing |
| `metosin/reitit-interceptors` | Common interceptors for reitit-http |
