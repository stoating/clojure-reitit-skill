# Reitit Frontend (ClojureScript)

## Contents

- Setup and browser integration
- HTML5 history vs hash routing
- Matching and navigation
- Controllers
- Coercion on the frontend
- React/Reagent patterns

---

## Setup

```clj
;; deps.edn (shadow-cljs or similar)
metosin/reitit {:mvn/version "0.10.1"}
;; or just the frontend module:
metosin/reitit-frontend {:mvn/version "0.10.1"}
```

```cljs
(require '[reitit.frontend :as rf])
(require '[reitit.frontend.easy :as rfe])
(require '[reitit.coercion.spec :as rss])
(require '[reitit.frontend.controllers :as rfc])
```

---

## Browser Integration (reitit.frontend.easy)

`reitit.frontend.easy` is the high-level API. It wires a reitit router to browser navigation events (hash-change or HTML5 history) and calls a callback on every navigation.

### HTML5 History (pushState)

```cljs
(defonce match (r/atom nil))

(def routes
  [["/"           {:name ::home}]
   ["/users"      {:name ::users}]
   ["/users/:id"  {:name ::user
                   :parameters {:path {:id int?}}}]])

(defn on-navigate [new-match]
  (reset! match new-match))

;; Call once on app startup
(rfe/start!
  (rf/router routes {:data {:coercion rss/coercion}})
  on-navigate
  {:use-fragment false})  ; false = HTML5 history
```

### Hash Routing

```cljs
(rfe/start!
  (rf/router routes)
  on-navigate
  {:use-fragment true})   ; true = hash (#/path)
```

---

## Matching and Navigation

### Reading Current Match

The `match` atom (or whatever you pass to `on-navigate`) holds a `Match` record:

```cljs
@match
; #Match{:template  "/users/:id"
;        :data      {:name ::user}
;        :path-params {:id 42}       ; coerced if coercion enabled
;        :parameters {:path {:id 42}}
;        :path      "/users/42"}
```

### Navigating Programmatically

```cljs
;; Push a new history entry
(rfe/push-state ::user {:id 42})

;; Replace current history entry (no back-button entry)
(rfe/replace-state ::home)

;; With query params
(rfe/push-state ::users {} {:page 2 :sort "name"})
```

### Generating Hrefs

```cljs
(rfe/href ::user {:id 42})
; "/users/42"

(rfe/href ::users {} {:page 2})
; "/users?page=2"
```

Use in Reagent/React components:

```cljs
[:a {:href (rfe/href ::user {:id (:id user)})} (:name user)]
```

### Matching by Path/Name

```cljs
;; Direct match (lower level, via reitit.frontend)
(rf/match-by-path router "/users/42")
(rf/match-by-name  router ::user {:id 42})
```

---

## Controllers

Controllers run side effects in response to navigation — fetching data, starting/stopping subscriptions. They are declared in route data and managed by `rfc/apply-controllers`.

```cljs
(def routes
  [["/"          {:name ::home
                  :controllers [{:start (fn [_] (js/console.log "home start"))
                                 :stop  (fn [_] (js/console.log "home stop"))}]}]

   ["/users/:id" {:name ::user
                  :controllers [{:parameters {:path [:id]}
                                 :start      (fn [{:keys [path]}]
                                               (fetch-user! (:id path)))
                                 :stop       (fn [_]
                                               (clear-user!))}]}]])
```

Wire controllers into your `on-navigate` callback:

```cljs
(defonce controllers (r/atom []))

(defn on-navigate [new-match]
  (let [new-match* (if new-match
                     (assoc new-match :controllers
                            (rfc/apply-controllers @controllers new-match))
                     new-match)]
    (reset! controllers (:controllers new-match*))
    (reset! match new-match*)))
```

`apply-controllers` diffs the old and new controller sets:
- `:stop` is called on controllers whose identity-parameters changed or whose route was left.
- `:start` is called on controllers that are new or whose identity-parameters changed.
- Controllers with no `:parameters` key restart on every navigation to their route.

A controller's `:parameters` map (`{:path [:id]}`) specifies which route params determine identity — the controller restarts only when those params change.

---

## Coercion on the Frontend

Attach a `Coercion` implementation to route data — parameters will be coerced on match:

```cljs
(require '[reitit.coercion.spec :as rss])

(def router
  (rf/router
    [["/users/:id" {:name ::user
                    :parameters {:path {:id int?}}}]]
    {:data {:coercion rss/coercion}}))
```

`rf/router` calls `coercion/compile-request-coercers` by default, so coerced values land in `:parameters`:

```cljs
(:parameters (rf/match-by-path router "/users/42"))
; {:path {:id 42}}   ; int, not "42"
```

Query string parameters are also coerceable:

```cljs
(rf/match-by-name router ::users {} {:page "2"})
;; → :parameters {:query {:page 2}}
```

---

## React/Reagent Integration Pattern

```cljs
(ns my-app.core
  (:require [reagent.core :as r]
            [reitit.frontend :as rf]
            [reitit.frontend.easy :as rfe]
            [reitit.frontend.controllers :as rfc]))

(defonce match (r/atom nil))
(defonce controllers (r/atom []))

(def routes
  [["/"          {:name ::home    :view #'home-page}]
   ["/about"     {:name ::about   :view #'about-page}]
   ["/users/:id" {:name ::user    :view #'user-page
                  :parameters {:path {:id int?}}
                  :controllers [{:parameters {:path [:id]}
                                 :start (fn [{:keys [path]}]
                                          (load-user! (:id path)))}]}]])

(defn on-navigate [new-match]
  (let [new-match* (when new-match
                     (assoc new-match :controllers
                            (rfc/apply-controllers @controllers new-match)))]
    (reset! controllers (or (:controllers new-match*) []))
    (reset! match new-match*)))

(defn app []
  (if-let [m @match]
    [(-> m :data :view)]
    [:div "Loading..."]))

(defn init! []
  (rfe/start!
    (rf/router routes {:data {:coercion rss/coercion}})
    on-navigate
    {:use-fragment false})
  (r/render [app] (js/document.getElementById "app")))
```
