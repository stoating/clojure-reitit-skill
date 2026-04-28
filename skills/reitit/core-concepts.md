# Reitit Core Concepts

## Contents

- Route syntax
- Router creation and protocol
- Path-based routing
- Name-based (reverse) routing
- Route data and nesting
- Route conflicts

---

## Route Syntax

Routes are plain Clojure data — vectors of `[path ?data & children]`:

```clj
;; Simple route with a name keyword
["/ping" ::ping]

;; Route with a handler function (expands to {:handler f})
["/pong" identity]

;; Route with explicit data map
["/users" {:get {:roles #{:admin}
                 :handler get-users}}]
```

### Path Parameters

Both `:colon` and `{bracket}` syntaxes are supported by default:

```clj
;; Colon syntax
["/users/:id" ::user]
["/files/file-:number" ::file]

;; Bracket syntax (supports qualified keywords and mid-segment params)
["/users/{id}" ::user]
["/files/{name}.{extension}" ::file]
["/resources/{:resource/id}/activate" ::activate]

;; Catch-all
["/public/*path" ::static]
["/public/{*path}" ::static]
```

### Nested Routes

Children inherit and accumulate parent route data:

```clj
["/api"
 ["/admin" {:middleware [::admin]}
  ["" {:name ::admin}]          ;; path: /api/admin
  ["/db" {:name ::db}]]         ;; path: /api/admin/db
 ["/ping" {:name ::ping}]]      ;; path: /api/ping
```

### Route Fragments

Empty path `""` adds data without adding path segments:

```clj
[["" {:no-doc true}             ;; applies no-doc to swagger.json & api-docs
  ["/swagger.json" ::swagger]
  ["/api-docs" ::api-docs]]
 ["/api/ping" ::ping]]
```

### Generating Routes Programmatically

Routes are data — generate them with plain Clojure:

```clj
(defn cqrs-routes [actions]
  ["/api" {:interceptors [::api ::db]}
   (for [[type interceptor] actions
         :let [path (str "/" (name interceptor))
               method (case type :query :get :command :post)]]
     [path {method {:interceptors [interceptor]}}])])
```

---

## Router Creation

```clj
(require '[reitit.core :as r])

(def router
  (r/router
    [["/api/ping" ::ping]
     ["/api/orders/:id" ::order]]))
```

The `Router` protocol:

```clj
(r/router-name router)    ; :mixed-router
(r/routes router)         ; flattened route tree
(r/options router)        ; router options map
(r/route-names router)    ; [::ping ::order]
```

Router options (passed as second argument):

| Key | Description |
|-----|-------------|
| `:data` | Top-level route data merged into all routes |
| `:syntax` | Path param syntax — `:colon`, `:bracket`, or `#{:colon :bracket}` (default) |
| `:conflicts` | Conflict handler fn or `nil` to ignore |
| `:expand` | Custom route argument expander |
| `:compile` | Route compilation fn (used by Ring, coercion) |
| `:router` | Force a specific router implementation |

Top-level route data example:

```clj
(r/router
  ["/api" {:middleware [::api]}
   ["/ping" ::ping]]
  {:data {:middleware [::session]}})
;; Both routes get [:session :api] middleware
```

### Composing Routers

Routes are data — merge multiple trees freely:

```clj
(def router
  (r/router
    [user-routes
     admin-routes
     ["/health" ::health]]))
```

---

## Path-Based Routing

```clj
(r/match-by-path router "/api/ping")
; #Match{:template "/api/ping"
;        :data {:name ::ping}
;        :result nil
;        :path-params {}
;        :path "/api/ping"}

(r/match-by-path router "/api/orders/42")
; #Match{:template "/api/orders/:id"
;        :data {:name ::order}
;        :result nil
;        :path-params {:id "42"}
;        :path "/api/orders/42"}

(r/match-by-path router "/not-found")
; nil
```

Path params are always strings at the core level. Use coercion (see [coercion.md](coercion.md)) to parse them into typed values.

---

## Name-Based (Reverse) Routing

```clj
(r/match-by-name router ::ping)
; #Match{:template "/api/ping" ...}

;; Route with required params returns a PartialMatch
(r/match-by-name router ::order)
; #PartialMatch{:template "/api/orders/:id" :required #{:id} ...}

(r/partial-match? (r/match-by-name router ::order))
; true

;; Supply params to get a full match
(r/match-by-name router ::order {:id 42})
; #Match{... :path "/api/orders/42"}

;; Extract path string
(-> (r/match-by-name router ::order {:id 42})
    (r/match->path))
; "/api/orders/42"

;; Append query string
(r/match->path match {:extra "param"})
; "/api/orders/42?extra=param"
```

---

## Route Data and Nesting

Route data is **accumulated recursively** from root toward leaves using [meta-merge](https://github.com/weavejester/meta-merge). For collections, the default behavior is `:append` (children add to parent), but you can override per-key using metadata:

```clj
(r/router
  ["/api" {:interceptors [::api]}
   ["/admin" {:roles #{:admin}}
    ["/users" ::users]
    ["/db" {:interceptors [::db]
            :roles ^:replace #{:db-admin}}]]])

;; Resolved:
;; /api/admin/users → {:interceptors [::api] :roles #{:admin}}
;; /api/admin/db    → {:interceptors [::api ::db] :roles #{:db-admin}}
```

Meta-merge override markers: `^:replace`, `^:prepend`, `^:displace`.

---

## Route Conflicts

Reitit detects path and name conflicts at router creation time and throws by default.

```clj
;; These conflict (wildcard ambiguity):
[["/ping"]
 ["/:user-id/orders"]   ; conflicts with /bulk/:bulk-id
 ["/bulk/:bulk-id"]
 ["/public/*path"]
 ["/:version/status"]]
```

Options:

```clj
;; Throw (default)
(r/router routes)

;; Ignore all conflicts
(r/router routes {:conflicts nil})

;; Log conflicts
(require '[reitit.exception :as exception])
(r/router routes
  {:conflicts (fn [c] (println (exception/format-exception :path-conflicts nil c)))})

;; Ignore specific routes
[["/:user-id/orders" {:conflicting true}]
 ["/bulk/:bulk-id"   {:conflicting true}]]
```

**Name conflicts** cannot be disabled — each route name must be unique.

---

## Dev Workflow: Dynamic Routers

Def-ing a router from vars means route changes after namespace reload won't be picked up. Fix: use functions.

```clj
;; Dev: rebuild router on every call
(def dev-router #(r/router (routes)))
(r/match-by-path (dev-router) "/api/ping")

;; Prod: build once
(def prod-router (constantly (r/router (routes))))
(r/match-by-path (prod-router) "/api/ping")
```

Alternatively, use var-quoted handlers so handler changes take effect without rebuilding the router:

```clj
(ring/router
  ["/ping" {:get #'my-ns/handler}])
```
