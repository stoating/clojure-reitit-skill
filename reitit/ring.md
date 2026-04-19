# Reitit Ring

## Contents

- Ring router and handler
- Request-method routing
- Middleware
- Middleware execution order
- Content negotiation (Muuntaja)
- Exception handling
- Default middleware stack

---

## Ring Router and Handler

```clj
(require '[reitit.ring :as ring])
```

`ring/router` wraps the core router with request-method dispatch, handler support, and middleware compilation.

```clj
(defn handler [_]
  {:status 200, :body "ok"})

(def router
  (ring/router
    [["/ping" {:get handler}]
     ["/echo" {:post handler}]]))
```

`ring/ring-handler` turns the router into a standard Ring handler:

```clj
(def app (ring/ring-handler router))

(app {:request-method :get, :uri "/ping"})
; {:status 200, :body "ok"}

(app {:request-method :get, :uri "/not-found"})
; nil
```

### Default and Not-Found Handlers

```clj
(def app
  (ring/ring-handler
    router
    (ring/create-default-handler)))
;; Returns 404/405/406 responses instead of nil
```

Custom not-found:

```clj
(ring/ring-handler
  router
  (fn [_] {:status 404, :body "not found"}))
```

### ring-handler Options

| Key | Description |
|-----|-------------|
| `:middleware` | Top-level middleware wrapping the whole handler (applied before routing) |
| `:inject-match?` | Inject `Match` into request as `:reitit.core/match` (default: `true`) |
| `:inject-router?` | Inject `router` into request as `:reitit.core/router` (default: `true`) |

### Accessing the Router at Runtime

```clj
(require '[reitit.core :as r])

(-> app
    (ring/get-router)
    (r/match-by-name ::ping)
    (r/match->path))
; "/ping"
```

---

## Request-Method Routing

Place handlers at the top level (any method) or under a specific method key:

```clj
(def app
  (ring/ring-handler
    (ring/router
      [["/all"  handler]                            ; matches any method
       ["/item" {:get  get-handler
                 :post post-handler
                 :name ::item}]])))
```

Supported method keys: `:get`, `:head`, `:post`, `:put`, `:patch`, `:delete`, `:options`, `:trace`.

By default, an `:options` endpoint is generated for every path to support CORS preflight. Disable via router option `:reitit.ring/default-options-endpoint`.

---

## Middleware

Middleware is declared under `:middleware` in route data. Four accepted forms:

1. **Function** — `handler -> request -> response`
2. **Vector** — `[middleware-fn arg1 arg2 …]`
3. **Map / Record** — data-driven middleware (see below)
4. **Keyword** — lookup from `:reitit.middleware/registry`

```clj
(defn wrap [handler id]
  (fn [request]
    (handler (update request ::acc (fnil conj []) id))))

(def app
  (ring/ring-handler
    (ring/router
      ["/api" {:middleware [#(wrap % :api)]}
       ["/ping" handler]
       ["/admin" {:middleware [[wrap :admin]]}
        ["/db {:middleware [[wrap :db]]
               :delete {:middleware [[wrap :delete]]
                        :handler handler}}]]])))

(app {:request-method :delete, :uri "/api/admin/db"})
; {:status 200, :body [:api :admin :db :delete :handler]}
```

### Data-Driven Middleware

Prefer map/record middleware — it's inspectable, composable, and can be compiled per-endpoint:

```clj
(require '[reitit.middleware :as middleware])

(def auth-middleware
  (middleware/map->Middleware
    {:name ::auth
     :description "Checks Bearer token"
     :wrap (fn [handler]
             (fn [request]
               (if (valid-token? request)
                 (handler request)
                 {:status 401, :body "Unauthorized"})))}))

;; Use by name via registry
(ring/router routes
  {:reitit.middleware/registry {::auth auth-middleware}})
```

### Middleware Execution Order

```
ring-handler top-level :middleware
└─ router top-level route data :middleware   (via {:data {:middleware [...]}})
   └─ nested parent :middleware
      └─ leaf route :middleware
         └─ handler
```

Full example showing all levels:

```clj
(def app
  (ring/ring-handler
    (ring/router
      ["/api" {:middleware [[wrap :3-parent]]}
       ["/get" {:get handler
                :middleware [[wrap :4-route]]}]]
      {:data {:middleware [[wrap :2-top-level-route-data]]}})
    nil
    {:middleware [[wrap :1-top]]}))

(app {:request-method :get, :uri "/api/get"})
; {:status 200, :body [:1-top :2-top-level-route-data :3-parent :4-route :handler]}
```

### Middleware Placement Guide

| Goal | Where to put it |
|------|----------------|
| Applies to default handler too | Top-level `:middleware` on `ring-handler` |
| Generic, applies to all routes | `{:data {:middleware [...]}}` router option |
| Per-subtree | `:middleware` in parent route data |
| Only a few routes | `:middleware` in leaf route data |
| Needs to read route data | Compiling middleware — see [Compiling Middleware](https://cljdoc.org/d/metosin/reitit/CURRENT/doc/ring/compiling-middleware) |

---

## Content Negotiation (Muuntaja)

```clj
(require '[muuntaja.core :as m])
(require '[reitit.ring.middleware.muuntaja :as muuntaja])

(ring/router
  routes
  {:data {:muuntaja m/instance
          :middleware [muuntaja/format-middleware]}})
```

`format-middleware` handles content negotiation, request body decoding (→ `:body-params`), and response encoding in one step. Split versions: `format-negotiate-middleware`, `format-request-middleware`, `format-response-middleware`.

The `:muuntaja` key in route data controls the Muuntaja instance — omitting it on a route skips negotiation for that route.

---

## Exception Handling

```clj
(require '[reitit.ring.middleware.exception :as exception])
```

### Pre-configured Middleware

```clj
(ring/router
  routes
  {:data {:middleware [exception/exception-middleware]}})
```

Catches: coercion errors (400/500), Muuntaja decode errors, exceptions with `:type :reitit.ring/response` (returns `:response` from ex-data), and all other exceptions (500).

### Custom Exception Handlers

```clj
(derive ::not-found ::exception)
(derive ::unauthorized ::exception)

(def custom-middleware
  (exception/create-exception-middleware
    (merge
      exception/default-handlers
      {::not-found     (fn [_ _ _] {:status 404, :body "not found"})
       ::unauthorized  (fn [_ _ _] {:status 401, :body "unauthorized"})
       ;; wrap all exceptions for logging
       ::exception/wrap (fn [handler e request]
                          (log/error e "unhandled exception")
                          (handler e request))})))
```

Handler lookup order: `:type` of ex-data → class of exception → `:type` ancestors → superclasses → `::exception/default`.

Throw a structured response directly:

```clj
(throw (ex-info "not found" {:type :reitit.ring/response
                              :response {:status 404, :body "item not found"}}))
```

---

## Default Middleware Stack

A typical `reitit-middleware` stack in dependency order:

```clj
(require '[reitit.ring.middleware.parameters :as parameters])
(require '[reitit.ring.middleware.muuntaja   :as muuntaja])
(require '[reitit.ring.middleware.exception  :as exception])
(require '[reitit.ring.coercion             :as coercion])
(require '[reitit.coercion.malli])

(def app
  (ring/ring-handler
    (ring/router
      routes
      {:data {:muuntaja  m/instance
              :coercion  reitit.coercion.malli/coercion
              :middleware [parameters/parameters-middleware
                           muuntaja/format-middleware
                           exception/exception-middleware
                           coercion/coerce-request-middleware
                           coercion/coerce-response-middleware]}})
    (ring/create-default-handler)))
```

---

## Interceptors (reitit-http)

Use `reitit-http` for Pedestal-style interceptors instead of middleware:

```clj
(require '[reitit.http :as http])
(require '[reitit.interceptor.sieppari :as sieppari])

(def app
  (http/ring-handler
    (http/router
      ["/api"
       {:interceptors [(fn [number]
                         {:enter (fn [ctx] (update-in ctx [:request :n] (fnil + 0) number))})
                       10]}
       ["/num" {:get {:interceptors [100]
                      :handler (fn [req] {:status 200, :body {:n (:n req)}})}}]])
    (ring/create-default-handler)
    {:executor sieppari/executor}))
```

The `:interceptors` key replaces `:middleware`. An interceptor is a map with `:enter` and/or `:leave` functions of `ctx → ctx`.
