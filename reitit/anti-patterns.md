# Reitit Anti-Patterns

## Contents

- Route definition anti-patterns
- Middleware anti-patterns
- Coercion anti-patterns
- Dev workflow anti-patterns
- Frontend anti-patterns
- Quick diagnostic checklist

---

## Route Definition Anti-Patterns

### Forgetting that route data is inherited and merged

```clj
;; WRONG — expecting :middleware to apply only to /ping
(ring/router
  ["/api"
   {:middleware [[wrap-auth]]}
   ["/ping" handler]      ; wrap-auth IS applied here
   ["/health" handler]])  ; wrap-auth IS applied here too

;; RIGHT — isolate with a sub-tree or apply selectively
(ring/router
  ["/api"
   ["/ping"   {:middleware [[wrap-auth]]
               :get handler}]
   ["/health" {:get handler}]])
```

Route data inherits from all ancestors. Always think about what a parent route's data implies for its children.

---

### Using the same route name twice

```clj
;; WRONG — causes a compile-time exception
[["/ping"       ::ping]
 ["/admin/ping" ::ping]]  ; name conflict!
```

Route names must be globally unique. Name conflicts cannot be suppressed.

---

### Mixing colon and bracket syntax with port numbers

```clj
;; WRONG — colon syntax parses :8080 as a path param
(r/router
  ["http://localhost:8080/api/user/:id" ::user])
;; path-params: {:id "1", :8080 ":8080"}  ← :8080 is a bogus param

;; RIGHT — use bracket-only syntax for URLs with ports
(r/router
  ["http://localhost:8080/api/user/{id}" ::user]
  {:syntax :bracket})
```

---

### Defining handlers as bare values (not vars or functions) in dev

```clj
;; WRONG — router captures the value at creation time; reloading ns won't update it
(def app (ring/ring-handler (ring/router ["/ping" {:get my-handler}])))

;; RIGHT — use #' so the router dispatches through the var on every request
(def app (ring/ring-handler (ring/router ["/ping" {:get #'my-handler}])))
```

---

## Middleware Anti-Patterns

### Wrong middleware order causing NullPointerException or 500s

Common bad ordering:

```clj
;; WRONG — coercion before Muuntaja means :body-params is nil
[rrc/coerce-request-middleware
 muuntaja/format-middleware]   ; too late — body already past coercion

;; RIGHT
[muuntaja/format-middleware         ; 1. parse body → :body-params
 rrc/coerce-request-middleware]     ; 2. coerce :body-params
```

Required order: `parameters → format (muuntaja) → exceptions → coerce-request → coerce-response`.

---

### Applying exception middleware after coercion middleware

```clj
;; WRONG — coercion errors thrown before the exception handler is in place
[rrc/coerce-request-middleware
 exception/exception-middleware]   ; too late — coercion already threw

;; RIGHT
[exception/exception-middleware    ; 1. catches everything below
 rrc/coerce-request-middleware]    ; 2. may throw coercion errors
```

---

### Mounting middleware on `ring-handler` top-level for route-specific logic

```clj
;; WRONG — top-level middleware runs even on the default handler (404 route)
(ring/ring-handler
  router
  default-handler
  {:middleware [[wrap-auth]]})   ; auth runs on 404 requests too

;; RIGHT — mount on routes or via {:data {:middleware ...}} in router
(ring/router routes {:data {:middleware [[wrap-auth]]}})
```

---

### Forgetting that `reitit-ring` does not catch exceptions by default

```clj
;; WRONG — unhandled exceptions return a raw Java stack trace or nil
(def app (ring/ring-handler (ring/router routes)))

;; RIGHT — always add an exception middleware
(ring/router routes {:data {:middleware [exception/exception-middleware]}})
```

---

## Coercion Anti-Patterns

### Defining `:coercion` without mounting the middleware

```clj
;; WRONG — coercion in route data is just data; nothing coerces without the middleware
{:coercion   reitit.coercion.malli/coercion
 :parameters {:path {:id int?}}
 :handler    (fn [{:keys [parameters]}]
               ;; parameters is nil — coercion middleware was never mounted!
               {:status 200 :body {:id (-> parameters :path :id)}})}

;; RIGHT — mount coerce-request-middleware in the router or route
{:data {:middleware [rrc/coerce-request-middleware]}}
```

---

### Expecting `:body` params to be keywordized automatically

```clj
;; WRONG — body keys are NOT keywordized before coercion (unlike :path/:query)
:parameters {:body {:userName string?}}
;; Incoming JSON {"userName":"Alice"} → coercion fails because key is string

;; RIGHT — configure Muuntaja or your JSON decoder to keywordize, or use Malli's json-transformer
```

Use `reitit.coercion.malli/json-transformer-provider` for `:body` to handle JSON key coercion:

```clj
(reitit.coercion.malli/create
  {:transformers {:body {:formats {"application/json"
                                   reitit.coercion.malli/json-transformer-provider}}}})
```

---

### Closed malli schemas rejecting valid requests

```clj
;; By default, mu/closed-schema is applied — extra keys are rejected
{:parameters {:query [:map [:page int?]]}}
;; Request: /items?page=1&sort=asc → 400 because :sort is unexpected

;; RIGHT — open the schema for query/path params (extra keys are allowed by default
;; unless you explicitly close them via compile option)
;; If you do use closed schemas, strip-extra-keys strips unknowns before coercion
```

---

## Dev Workflow Anti-Patterns

### Def-ing a router at namespace load time prevents hot-reloading

```clj
;; WRONG — router is compiled once; ns reloads don't update route data
(def routes [["/api" some-ns/routes]])
(def router (ring/router routes))
(def app    (ring/ring-handler router))

;; RIGHT (dev) — router is a function, rebuilt on every call
(defn make-router [] (ring/router [["/api" (some-ns/routes)]]))
(def app (ring/ring-handler (make-router)))
;; or use var-quoted handlers
```

---

### Missing route conflicts because `:conflicts nil` is set globally

```clj
;; WRONG — silently swallows all ambiguous routes
(ring/router routes {:conflicts nil})

;; BETTER — log conflicts so you can decide intentionally
(ring/router routes
  {:conflicts (fn [c]
                (log/warn "route conflicts" (pr-str c)))})
```

---

## Frontend Anti-Patterns

### Starting `rfe/start!` more than once

Calling `rfe/start!` twice registers duplicate navigation listeners. Call it once, at app init. Use a `defonce` guard:

```cljs
(defonce _started
  (rfe/start! router on-navigate {:use-fragment false}))
```

---

### Not using `:parameters` in controllers, causing restarts on every navigation

```cljs
;; WRONG — controller with no :parameters restarts on EVERY navigation to this route
{:controllers [{:start (fn [_] (fetch-heavy-data!))
                :stop  (fn [_] (cancel-fetch!))}]}

;; RIGHT — declare which params identify this controller instance
{:controllers [{:parameters {:path [:id]}   ; restart only when :id changes
                :start (fn [{:keys [path]}] (fetch-user! (:id path)))
                :stop  (fn [_] (cancel-fetch!))}]}
```

---

### Generating hrefs before `rfe/start!` completes

`rfe/href` returns `nil` before the router is initialized. Ensure `rfe/start!` has been called before rendering components that use `rfe/href`.

---

## Quick Diagnostic Checklist

```
[ ] Is the router being rebuilt after ns reloads? (use fn or #'var-handler)
[ ] Is coercion middleware mounted? (coerce-request-middleware, coerce-response-middleware)
[ ] Is Muuntaja format-middleware before coercion middleware?
[ ] Is exception-middleware before coercion middleware?
[ ] Are route names globally unique?
[ ] Are route conflicts expected, or a sign of a routing design problem?
[ ] For body coercion: are keys keywordized before reaching the coercion schema?
[ ] For frontend: is rfe/start! called exactly once at init?
[ ] For frontend controllers: do controllers declare :parameters to avoid spurious restarts?
[ ] Is the default handler (create-default-handler) in place for clean 404/405 responses?
[ ] Are responses encoded by Muuntaja? (format-middleware or format-response-middleware)
```
