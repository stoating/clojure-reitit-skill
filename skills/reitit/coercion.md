# Reitit Coercion

## Contents

- Coercion concepts
- Malli coercion
- Spec coercion
- Schema coercion
- Parameter types
- Response coercion
- Nested parameter definitions
- Coercion middleware

---

## Coercion Concepts

Coercion converts and validates raw string path/query params and parsed body maps into typed Clojure values. It is **pluggable** — attach a `Coercion` implementation under `:coercion` in route data and declare schemas under `:parameters` and/or `:responses`.

Three built-in coercion implementations:

| Artifact | Require |
|----------|---------|
| `metosin/reitit` (malli) | `reitit.coercion.malli/coercion` |
| `metosin/reitit-spec` | `reitit.coercion.spec/coercion` |
| `metosin/reitit-schema` | `reitit.coercion.schema/coercion` |

Multiple coercions can coexist in a single router — scoping follows normal [route data inheritance](core-concepts.md#route-data-and-nesting).

---

## Malli Coercion

```clj
(require '[reitit.coercion.malli])
(require '[reitit.ring.coercion :as rrc])
(require '[reitit.ring :as ring])

(def app
  (ring/ring-handler
    (ring/router
      ["/users/:id"
       {:get {:coercion   reitit.coercion.malli/coercion
              :parameters {:path {:id int?}}
              :responses  {200 {:body [:map [:name string?] [:age int?]]}}
              :handler    (fn [{:keys [parameters]}]
                            (let [id (-> parameters :path :id)]
                              {:status 200
                               :body   (get-user id)}))}}]
      {:data {:middleware [rrc/coerce-request-middleware
                           rrc/coerce-response-middleware]}})))
```

### Malli Vector vs Lite Syntax

```clj
;; Vector syntax (explicit :map)
:parameters {:path [:map [:id int?] [:org string?]]}

;; Lite syntax (plain map — malli infers :map)
:parameters {:path {:id int? :org string?}}
```

### Configuring Malli Coercion

```clj
(require '[malli.util :as mu])

(reitit.coercion.malli/create
  {:transformers {:body   {:default reitit.coercion.malli/default-transformer-provider
                           :formats {"application/json" reitit.coercion.malli/json-transformer-provider}}
                  :string {:default reitit.coercion.malli/string-transformer-provider}
                  :response {:default reitit.coercion.malli/default-transformer-provider}}
   :error-keys         #{:type :coercion :in :value :humanized}
   :lite               true          ; enable lite map syntax
   :compile            mu/closed-schema  ; reject extra keys
   :validate           true          ; validate response bodies too
   :strip-extra-keys   true
   :default-values     true          ; apply malli default values
   :options            nil})         ; malli options passed through
```

Custom error messages:

```clj
(reitit.coercion.malli/create
  {:options {:errors (assoc malli.error/default-errors
                            :malli.core/missing-key {:error/message {:en "required"}})}})
```

Custom registry:

```clj
(require '[malli.core :as m])

(reitit.coercion.malli/create
  {:options {:registry {:registry (merge (m/default-schemas) {:UserId :int})}}})
```

---

## Spec Coercion

```clj
(require '[reitit.coercion.spec])
(require '[clojure.spec.alpha :as s])

(s/def ::id pos-int?)
(s/def ::name string?)

(def app
  (ring/ring-handler
    (ring/router
      ["/users/:id"
       {:get {:coercion   reitit.coercion.spec/coercion
              :parameters {:path {:id ::id}}
              :handler    (fn [{:keys [parameters]}]
                            {:status 200 :body {:id (-> parameters :path :id)}})}}]
      {:data {:middleware [rrc/coerce-request-middleware]}})))
```

Data-specs (inline maps without `s/def`) also work with `reitit.coercion.spec/coercion`.

---

## Schema Coercion

```clj
(require '[reitit.coercion.schema])
(require '[schema.core :as s])

(def PositiveInt (s/constrained s/Int pos? 'PositiveInt))

(def plus-endpoint
  {:coercion   reitit.coercion.schema/coercion
   :parameters {:query {:x s/Int}
                :body  {:y s/Int}
                :path  {:z s/Int}}
   :responses  {200      {:body {:total PositiveInt}}
                :default {:body {:error s/Str}}}
   :handler    (fn [{:keys [parameters]}]
                 (let [{:keys [x y z]} (merge (:query parameters)
                                              (:body  parameters)
                                              (:path  parameters))]
                   {:status 200 :body {:total (+ x y z)}}))})
```

---

## Parameter Types

| Key | Source in request |
|-----|-------------------|
| `:path` | `:path-params` |
| `:query` | `:query-params` |
| `:body` | `:body-params` |
| `:request` | `:body-params` — allows per-content-type coercion |
| `:form` | `:form-params` |
| `:header` | `:header-params` |
| `:multipart` | `:multipart-params` |

Accessing coerced values in a handler:

```clj
(fn [{:keys [parameters]}]
  (let [{:keys [path query body]} parameters]
    ...))
```

### Key Handling Differences

- `:path`, `:query`, `:header`, `:form` — keys are **keywordized** before coercion; schema is opened to allow extra keys.
- `:body` — keys are **not** keywordized before coercion (Muuntaja may do so depending on format); schema strictness depends on coercion config.

---

## Response Coercion

```clj
:responses {200      {:body {:name string? :age int?}}
            :default {:body {:error string?}}}
```

Mount `coerce-response-middleware` to validate and encode response bodies. Response coercion errors return HTTP 500.

---

## Nested Parameter Definitions

Parameters accumulate along the route tree like other route data. Malli `:map` schemas are merged intelligently:

```clj
(ring/router
  ["/api" {:get {:parameters {:query [:map [:api-key :string]]}}}
   ["/project/:project-id" {:get {:parameters {:path [:map [:project-id :int]]}}}
    ["/task/:task-id"
     {:get {:parameters {:path  [:map [:task-id :int]]
                          :query [:map [:details :boolean]]}
            :handler (fn [req] (prn req))}}]]]
  {:data {:coercion reitit.coercion.malli/coercion}})

;; Resolved for GET /api/project/1/task/2:
;; {:query [:map [:api-key :string] [:details :boolean]]
;;  :path  [:map [:project-id :int] [:task-id :int]]}
```

---

## Coercion Middleware

Three middleware from `reitit.ring.coercion`:

```clj
(require '[reitit.ring.coercion :as rrc])

;; Coerce request params → throws 400 on failure
rrc/coerce-request-middleware

;; Coerce response body → throws 500 on failure
rrc/coerce-response-middleware

;; Handle coercion exceptions (alternative to exception/exception-middleware)
rrc/coerce-exceptions-middleware
```

Typical setup in router `:data`:

```clj
{:data {:coercion   reitit.coercion.malli/coercion
        :middleware [parameters/parameters-middleware
                     muuntaja/format-middleware
                     rrc/coerce-exceptions-middleware
                     rrc/coerce-request-middleware
                     rrc/coerce-response-middleware]}}
```

`coerce-request-middleware` and `coerce-response-middleware` short-circuit to no-op when no `:coercion` or `:parameters`/`:responses` are defined on a route — zero overhead for uncoerced routes.
