# Domestic

![Tests](https://github.com/oconn/domestic/workflows/Tests/badge.svg)
[![Clojars Project](https://img.shields.io/clojars/v/oconn/domestic.svg)](https://clojars.org/oconn/domestic)

Time to bring your state home.

## About

Global state management is great! Then again, sometimes it's not... Domestic is a tiny library that simplifies local state management in clojurescript applications.

If you're fimilar with [re-frame](https://github.com/day8/re-frame), domestic will feel similar. It's APIs we're inspired by `re-frame`, but are designed to be much more simplistic without features like middleware and interceptors. In the end, the overarching ideas are the same;

1) Define state
1) Trigger an event
1) Update state
1) Read state

### Goals for domestic

- Develop a clean and consistent pattern for managing local state.
- Ensure ease of testing  local state.
- Flexiblility to work with all clojurescript librarys that support reactive atoms (reagent, helix, etc...)

### When is local state the right choice?

I've typcailly been on team "put everything in global state" and still heavily leverage it. Over time, while building larger clojurescript applications, I've noticed that there are instances when putting all state into a global database can be less than favorable and quite verbose. Most of the time it comes down to answering this question - **Do any other components care about this state?**. I've also started to view the global database as a one-to-one local representation of data stored on the server. So if it's not stored on the server, it may very well be a canidate for local state. At the end of the day, these are opinions and demostic is there for you when you decided to reach for local state management and want a clean and consistent solution.

## Install

[![Clojars Project](https://img.shields.io/clojars/v/oconn/domestic.svg)](https://clojars.org/oconn/domestic)

Install the latest version from clojars in your project.

## Basic Usage

### defdispatcher

The first step to using domestic is to define a dispatcher to process events.

```clojure
(ns core
  (:require [domestic.core :as d])

(d/defdispatcher my-dispatcher {})
```

### defevent

Next, define then events responsible for updating state / triggering side-effects.

```clojure
(d/defevent my-dispatcher :my-event
  [state]
  ;; update your state here
  )
```

### Dispatching an event

Dispatching an event is simple. Using the above example you trigger `:my-event` by;

```clojure
(my-dispatcher :my-event your-state)
```

### Passing data to an event

If you want to pass additional data to an event, dispatch your event, passing additional arguments.

```clojure
(d/defevent my-dispatcher :my-event
  [state user]
  ;; update your state here
  )

(my-dispatcher :my-event your-state {:user "foo"})
```

Just like functions, you can pass as many arguments as needed and the event will have access to them

### bind-dispatcher

`bind-dispatcher` is a helper function that can cut down on boilerplate code when passing around the dispatcher. It's first argument is your state atom and then any number of additional arguments that you want to partial to all events.

```clojure
(d/defdispatcher my-dispatcher {})

(d/defevent my-dispatcher :event-one
  [state current-user]
  )

(d/defevent my-dispatcher :event-two
  [state current-user contact]
  )

(let [state (reagent/atom {})
      dispatch (d/bind-dispatcher state {:user/id "1"})]
  (dispatch [:event-one])
  (dispatch [:event-two {:user "2"}]))
```

The above example show how you can leverage `bind-dispatcher` to partial in the state atom and any other number of additional arguments, proxying them to each dispatch event.

### reagent Example

```clojure
(d/defdispatcher reagent-dispatcher {})

(d/defevent reagent-dispatcher :add-user
  [state {:user/keys [id] :as user}]
  (swap! state assoc-in [:users id] user))

(defn reagent-component
  []
  (let [local-state (reagent/atom {:users {}})
        dispatch (d/bind-dispatcher reagent-dispatcher local-state)]

    (dispatch [:add-user {:user/id 1 :user/age 34}])
    (dispatch [:add-user {:user/id 2 :user/age 27}])

    (fn []
      [:div
       [:button {:id "addUserBtn"
                 :on-click #(dispatch [:add-user {:user/id (str (random-uuid)) :user/age 67}])}
         "Add a random 67 year old"]
       [:ul {:id "reagentUserList"}
        (for [{:user/keys [id age]} (vals (:users @local-state))]
          [:li {:key id} [:p age]])]])))
```

### helix Example

```clojure
(d/defdispatcher helix-dispatcher {})

(d/defevent helix-dispatcher :add-user
  [state update-state {:user/keys [id] :as user}]
  (update-state (assoc-in state [:users id] user)))

(defnc helix-component
  []
  (let [[local-state update-state] (helix-hooks/use-reducer #(merge-with merge %1 %2) {:users {}})
        dispatch (d/bind-dispatcher helix-dispatcher local-state update-state)]

    (helix-hooks/use-effect
     :once
     (dispatch [:add-user {:user/id 3 :user/age 19}])
     (dispatch [:add-user {:user/id 4 :user/age 65}]))

    ($ :div
       ($ :button {:id "addUserBtn2"
                   :on-click #(dispatch [:add-user {:user/id (str (random-uuid)) :user/age 67}])}
          "Add a random 67 year old")
       ($ :ul {:id "helixUserList2"}
          (for [{:user/keys [id age]} (vals (:users local-state))]
            ($ :li {:key id} ($ :p age)))))))
```

## Testing

domestic makes it easy to test state changes. Each event will return the derefed value of state, reguardless of what is actually returned from the body of an event handler.

```clojure
(d/defdispatcher dispatcher {})

(d/defevent dispatcher :inc
  [state]
  (swap! state inc)
  nil))

(let [state (atom 0)
      dispatch (d/bind-dispatcher dispatcher state)]
  (t/is (= 1 (dispatch [:inc]))
  (t/is (= 1 @state)))
```

### clj-kondo support

To get proper linting when using clj-kondo, add the following to your `config.edn`

```clojure
{:lint-as {domestic.core/defdispatcher clojure.core/defmulti
           domestic.core/defevent clojure.core/defmethod}}
```
