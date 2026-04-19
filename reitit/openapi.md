# Reitit OpenAPI and Swagger

## Contents

- OpenAPI 3 setup
- Swagger 2 setup
- Route data keys for docs
- Swagger UI
- Per-content-type coercion
- Excluding routes from docs

---

## OpenAPI 3 Setup

```clj
(require '[reitit.openapi :as openapi])
(require '[reitit.swagger-ui :as swagger-ui])
(require '[reitit.ring :as ring])
(require '[reitit.coercion.malli])
(require '[reitit.ring.coercion :as rrc])
(require '[reitit.ring.middleware.muuntaja :as muuntaja])
(require '[muuntaja.core :as m])

(def app
  (ring/ring-handler
    (ring/router
      [["/openapi.json"
        {:get {:handler (openapi/create-openapi-handler)
               :openapi {:info {:title "My API" :version "0.1.0"}}
               :no-doc true}}]

       ["/swagger-ui/*"
        {:get (swagger-ui/create-swagger-ui-handler
                {:url "/openapi.json"})}]

       ["/api"
        ["/users"
         {:get  {:summary    "List users"
                 :tags       ["users"]
                 :responses  {200 {:body [:vector [:map [:id int?] [:name string?]]]}}
                 :handler    list-users}
          :post {:summary    "Create user"
                 :tags       ["users"]
                 :parameters {:body [:map [:name string?]]}
                 :responses  {201 {:body [:map [:id int?] [:name string?]]}}
                 :handler    create-user}}]

        ["/users/:id"
         {:get {:summary    "Get user"
                :tags       ["users"]
                :parameters {:path [:map [:id int?]]}
                :responses  {200 {:body [:map [:id int?] [:name string?]]}
                             404 {:body [:map [:error string?]]}}
                :handler    get-user}}]]]

      {:data {:coercion   reitit.coercion.malli/coercion
              :muuntaja   m/instance
              :middleware [muuntaja/format-middleware
                           rrc/coerce-request-middleware
                           rrc/coerce-response-middleware]}})

    (ring/create-default-handler)))
```

`create-openapi-handler` collects route data at request time and returns an OpenAPI 3.1.0 spec map. Encode it to JSON via Muuntaja.

---

## Swagger 2 Setup

```clj
(require '[reitit.swagger :as swagger])
(require '[reitit.swagger-ui :as swagger-ui])

(ring/router
  [["/swagger.json"
    {:get {:handler (swagger/create-swagger-handler)
           :swagger {:info {:title "My API" :version "1.0"}}
           :no-doc  true}}]

   ["/swagger-ui/*"
    {:get (swagger-ui/create-swagger-ui-handler
            {:url "/swagger.json"})}]

   ;; ... your routes ...
   ])
```

Swagger 2 and OpenAPI 3 work identically; only the handler and top-level spec key differ.

---

## Route Data Keys for Docs

These keys on any route or method map contribute to the generated spec:

| Key | Description |
|-----|-------------|
| `:summary` | Short one-line description (string) |
| `:description` | Longer description, supports CommonMark |
| `:tags` | Set or vector of string/keyword tags |
| `:no-doc` | `true` — exclude this route from docs entirely |
| `:deprecated` | `true` — mark endpoint as deprecated |
| `:openapi` | Arbitrary OpenAPI data merged into the operation object |
| `:parameters` | Input coercion schemas (`:path`, `:query`, `:body`, …) — also used for docs |
| `:responses` | Response coercion schemas by status code — also used for docs |
| `:request` | Per-content-type body coercion (see below) |

### Tags

```clj
{:get {:tags    ["users" "admin"]
       :summary "List all users"
       :handler list-users}}
```

### Deprecation

```clj
{:get {:deprecated true
       :summary    "Old endpoint — use /v2/users"
       :handler    legacy-list}}
```

---

## Excluding Routes from Docs

```clj
;; Exclude a single route
["/internal/health" {:no-doc true, :get health-handler}]

;; Exclude a group using a route fragment
[["" {:no-doc true}
  ["/swagger.json" {:get swagger-handler}]
  ["/swagger-ui/*" {:get swagger-ui-handler}]]]
```

---

## Per-Content-Type Coercion (OpenAPI 3)

Use `:request` instead of `:body` in `:parameters` to get per-content-type coercion and named examples:

```clj
["/pizza"
 {:post {:summary   "Order a pizza"
         :tags      ["pizza"]
         :request   {:content
                     {"application/json"
                      {:schema   [:map [:flavor string?] [:size keyword?]]
                       :examples {:margherita {:value {:flavor "margherita" :size :large}}
                                  :pepperoni  {:value {:flavor "pepperoni"  :size :small}}}}}}
         :responses {200 {:body [:map [:order-id int?]]}}
         :handler   create-pizza-order}}]
```

Content types default to what the Muuntaja instance supports. Override per-route with:

```clj
:openapi/request-content-types  ["application/json" "application/edn"]
:openapi/response-content-types ["application/json"]
```

---

## Serving Swagger UI

`reitit-swagger-ui` embeds a pre-packaged Swagger UI:

```clj
(require '[reitit.swagger-ui :as swagger-ui])

["/swagger-ui/*"
 {:get (swagger-ui/create-swagger-ui-handler
         {:url          "/openapi.json"   ; or /swagger.json
          :config       {:validatorUrl nil}
          :no-doc       true})}]
```

The `*` catch-all serves static assets. The path prefix must end with `/*`.

---

## Post-processing the Spec

Wrap `create-openapi-handler` (or `create-swagger-handler`) with middleware to modify the spec:

```clj
(defn add-security-middleware [handler]
  (fn [request]
    (-> (handler request)
        (update :body assoc-in [:components :securitySchemes]
                {:bearerAuth {:type "http" :scheme "bearer"}}))))

["/openapi.json"
 {:get {:handler    (-> (openapi/create-openapi-handler)
                        add-security-middleware)
        :no-doc     true}}]
```
