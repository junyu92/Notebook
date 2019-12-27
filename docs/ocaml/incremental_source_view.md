# Learn Incremental from source code

## module Var

```ocaml
val create : ?use_current_scope:bool -> 'a -> 'a t
```

We invoke ``Var.create`` to create a new var whose type is ``'a t``, the type of the Var's value is ``'a``.

Let's look at the signature of ``Var``.

```ocaml

type 'a t = 'a Types.Var.t =
  { mutable value : 'a
  ; (* [value_set_during_stabilization] is only set to [Uopt.some] if the user calls
       [Var.set] during stabilization, in which case it holds the (last) value set.  At
       the end of stabilization, all such variables are processed to do [t.value <-
       t.value_set_during_stabilization]. *)
    mutable value_set_during_stabilization : 'a Uopt.t
  ; (* [set_at] the stabilization number in effect the most recent time [t.value] changed.
       This is not necessarily the same as the stabilization number in effect the most
       recent time [Var.set t] was called, due to the effect of [Var.set] during
       stabilization being delayed until after the stabilization. *)
    mutable set_at : Stabilization_num.t
  ; watch : 'a Node.t
  }

```

```ocaml
let create_var t ?(use_current_scope = false) value =
  let scope = if use_current_scope then t.current_scope else Scope.top in
  let watch = create_node_in t scope Uninitialized in
  let var =
    { Var.value
    ; value_set_during_stabilization = Uopt.none
    ; set_at = t.stabilization_num
    ; watch
    }
  in
  Node.set_kind watch (Var var);
  var
;;
```

**The first field of ``Var`` is ``value``** which specifies the value of ``Var``.
When we create ``Var`` by invoking ``create_var``,  this function will initiate
``value``(field of Var) with ``value``(input argument).

```ocaml
module Var = struct
  ..
  let set = State.set_var
  ..
end

let incr_state t = t.watch.state

let set_var var value =
  (* t has type State *)
  let t = Var.incr_state var in
  match t.status with
  | Running_on_update_handlers | Not_stabilizing ->
    set_var_while_not_stabilizing var value
  | Stabilize_previously_raised raised_exn ->
    failwiths
      "cannot set var -- stabilization previously raised"
      raised_exn
      [%sexp_of: Raised_exn.t]
  | Stabilizing ->
    if Uopt.is_none var.value_set_during_stabilization
    then Stack.push t.set_during_stabilization (T var);
    var.value_set_during_stabilization <- Uopt.some value
;;
```

**The second field is ``value_set_during_stabilization``** and **the third field  is ``set_at``**.
When we set the value of var by ``Var.set``, this function will check the status of ``Var``'s
state ``t``.

If it is ``Running_on_update_handlers`` or ``Not_stabilizing``, then we can immediately set var
by ``set_var_while_not_stabilizing``.

If the state of state is ``Stabilizing``, and the field ``value_set_during_stabilization`` equals
``Uopt.none``, then we push this var ``T var`` into stack ``t.set_during_stabilization``.
Then we set ``value_set_during_stabilization`` to ``Uopt.some value``.

```ocaml
let set_var_while_not_stabilizing var value =
  let t = Var.incr_state var in
  t.num_var_sets <- t.num_var_sets + 1;
  var.value <- value;
  if Stabilization_num.compare var.set_at t.stabilization_num < 0
  then (
    var.set_at <- t.stabilization_num;
    let watch = var.watch in
    if debug then assert (Node.is_stale watch);
    if Node.is_necessary watch && not (Node.is_in_recompute_heap watch)
    then Recompute_heap.add t.recompute_heap watch)
;;
```

If the status of state is not ``Stabilizing``, we will set the value of Var immediately.
First, we will modify some fields of state ``t``, and set ``value``(field) to ``value``(input argument).
If ``set_at`` is old, we will update ``set_at`` and insert the node(``watch``) of this var into ``t``'s
``recompute_heap``.

The last field of ``Var`` is ``watch``. The type of ``watch`` is ``Node``, We will introduce it
in the next section.

## Node & Watch

In the previous section, we didn't mention the field ``watch``. Since ``watch`` is a Node, we will
explain Node in this section.

Let's recall the signature of ``Var``.

```ocaml
let create_var t ?(use_current_scope = false) value =
  let scope = if use_current_scope then t.current_scope else Scope.top in
  let watch = create_node_in t scope Uninitialized in
  let var =
    { Var.value
    ; value_set_during_stabilization = Uopt.none
    ; set_at = t.stabilization_num
    ; watch
    }
  in
  Node.set_kind watch (Var var);
  var
;;
```

Once we have create a new ``Var``, the function creates a node ``watch`` whose kind is ``Uninitialized`` by
invoking ``create_node_in`` and immediately modifies the type to ``Var var``.

```ocaml
let create_node_in t created_in kind =
  t.num_nodes_created <- t.num_nodes_created + 1;
  Node.create t created_in kind
;;
```

It's time to look ``Node``.

```ocaml
module Node : sig
  type ('a, 'w) t [@@deriving sexp_of]
  (** [let t = create ?on_observability_change callback] creates a new expert node.
      [on_observability_change], if given, is called whenever the node becomes
      observable or unobservable (with alternating value for [is_now_observable],
      starting at [true] the first time the node becomes observable).  This callback
      could run multiple times per stabilization.  It should not change the
      incremental graph.
      [callback] is called if any dependency of [t] has changed since it was last
      called, or if the set of dependencies has changed.  The callback will only run
      when all the dependencies are up-to-date, and inside the callback, you can
      safely call [Dependency.value] on your dependencies, as well as call all the
      functions below on the parent nodes.  Any behavior that works on all incremental
      nodes (cutoff, invalidation, debug info etc) also work on [t]. *)
  val create
    :  'w State.t
    -> ?on_observability_change:(is_now_observable:bool -> unit)
    -> (unit -> 'a)
    -> ('a, 'w) t
  (** [watch t] allows you to plug [t] in the rest of the incremental graph, but it's
      also useful to set a cutoff function, debug info etc. *)
  val watch : ('a, 'w) t -> ('a, 'w) incremental
  (** Calling [make_stale t] ensures that incremental will recompute [t] before
      anyone reads its value.  [t] may not fire though, if it never becomes
      necessary.  This is intended to be called only from a child of [t].  Along with
      a well chosen cutoff function, it allows to choose which parents should
      fire. *)
  val make_stale : _ t -> unit
  (** [invalidate t] makes [t] invalid, as if its surrounding bind had changed.  This
      is intended to be called only from a child of [t]. *)
  val invalidate : _ t -> unit
  (** [add_dependency t dep] makes [t] depend on the child incremental in the [dep].
      If [dep] is already used to link the child incremental to another parent, an
      exception is raised.
      This is intended to be called either outside of stabilization, or right after
      creating [t], or from a child of [t] (and in that case, as a consequence [t]
      must be necessary).
      The [on_change] callback of [dep] will be fired when [t] becomes observable,
      or immediately, or whenever the child changes as long as [t] is observable.
      When this function is called due to observability changes, the callback may
      fire several times in the same stabilization, so it should be idempotent. The
      callback must not change the incremental graph, particularly not the
      dependencies of [t].
      All the [on_change] callbacks are guaranteed to be run before the callback of
      [create] is run. *)
  val add_dependency : (_, 'w) t -> (_, 'w) Dependency.t -> unit
  (** [remove_dependency t dep] can only be called from a child of [t]. *)
  val remove_dependency : (_, 'w) t -> (_, 'w) Dependency.t -> unit
end
```

### fields of Node
* **id** unique id for the node
* state
* recomputed_at
* value_opt
* **old_value_opt**: It holds the pre-stabilization value of ``t``. It is cleared when running ``t```s
on-update handlers, and sos is always ``Uopt.none`` between stabilizations.
* **kind**: the kind of DAG node ``t`` is. e.g. Array_fold, Const, Map, Invalid, MAP.
* **observers**: is the head of the double-linked list of **observers** of ``t``, or ``Uopt.none`` if there
are no observers.
* cutoff
* **change_at**: the last stabilization when this node was computed and not cut off.
It is used to detect when ``t```s parents are stale and need to be recomputed.
* num_on_update_handlers
* **num_parents**: the parents of ``t`` are the nodes that depend on it, and should be
computed when ``t`` changes, once all of their other children are up to date.
* parent1_and_beyond
* parent0
* **created_in**: the scope that the node is created in.
* next_node_in_same_scope
* **height**: height is used to visit nodes in topological order. It ``is_necessary t``,
then ``height > c.height`` for all children ``c`` of ``t``, and ``height > Scope.height t.created_in``.
If ``not (is_necessary t)``, then ``height = -1``.
* height_in_recompute_heap
* prev_in_recompute_heap
* height_in_adjust_heights_heap
* next_in_adjust_heights_heap
* is_in_handle_after_stabilization
* on_update_handlers
* my_parent_index_in_child_at_index
* my_child_index_in_parent_at_index
* force_necessary
* user_info
* creation_backtrace

### Create Node and the kind of Node

```ocaml
let create state created_in kind =
  let t =
    { id = Node_id.next ()
    ; state
    ; recomputed_at = Stabilization_num.none
    ; value_opt = Uopt.none
    ; kind
    ; cutoff = Cutoff.phys_equal
    ; changed_at = Stabilization_num.none
    ; num_on_update_handlers = 0
    ; num_parents = 0
    ; parent1_and_beyond = [||]
    ; parent0 = Uopt.none
    ; created_in
    ; next_node_in_same_scope = Uopt.none
    ; height = -1
    ; height_in_recompute_heap = -1
    ; prev_in_recompute_heap = Uopt.none
    ; next_in_recompute_heap = Uopt.none
    ; height_in_adjust_heights_heap = -1
    ; next_in_adjust_heights_heap = Uopt.none
    ; old_value_opt = Uopt.none
    ; observers = Uopt.none
    ; is_in_handle_after_stabilization = false
    ; on_update_handlers = []
    ; my_parent_index_in_child_at_index =
        Array.create ~len:(Kind.initial_num_children kind) (-1)
    (* [my_child_index_in_parent_at_index] has one element because it may need to hold
       the child index of [parent0]. *)
    ; my_child_index_in_parent_at_index = [| -1 |]
    ; force_necessary = false
    ; user_info = None
    ; creation_backtrace =
        (if state.keep_node_creation_backtrace then Some (Backtrace.get ()) else None)
    }
  in
```

#### Var

```ocaml
let create_node_in t created_in kind =
  t.num_nodes_created <- t.num_nodes_created + 1;
  Node.create t created_in kind
;;

let create_var t ?(use_current_scope = false) value =
  let scope = if use_current_scope then t.current_scope else Scope.top in
  let watch = create_node_in t scope Uninitialized in
  let var =
    { Var.value
    ; value_set_during_stabilization = Uopt.none
    ; set_at = t.stabilization_num
    ; watch
    }
  in
  Node.set_kind watch (Var var);
  var
;;
```

Invoking ``Var.create`` will create a Node whose type is ``Var``, and the field ``value_opt`` will be set to ``Uopt.none``.

#### Map

```ocaml
let create_node t kind = create_node_in t t.current_scope kind

let map (n : _ Node.t) ~f = create_node n.state (Map (f, n))
let map2 (n1 : _ Node.t) n2 ~f = create_node n1.state (Map2 (f, n1, n2))
let map3 (n1 : _ Node.t) n2 n3 ~f = create_node n1.state (Map3 (f, n1, n2, n3))
let map4 (n1 : _ Node.t) n2 n3 n4 ~f = create_node n1.state (Map4 (f, n1, n2, n3, n4))
...
let map15 ...
```

Let's look an example.

```ocaml
let width_v  = Var.create 3.
let height_v  = Var.create 5.

let width  = Var.watch width_v
let height  = Var.watch height_v

let base_area =
  Inc.map2 width height ~f:( *. )
```

The previous code will create three nodes.

```
  Var
+---------+
|  width  | ----\        Map2
+---------+      \    +-----------+
  Var             +-->| base_area |
+---------+      /    +-----------+
|  height |-----/
+---------+
```

#### Bind

Given a node(``lhs``), bind will create two new nodes(``lhs_change`` and ``main``) whose type are
``Bind_lhs_change`` and ``Bind_main``.

```ocaml
let bind (lhs : _ Node.t) ~f =
  let t = lhs.state in
  let lhs_change = create_node t Uninitialized in
  let main = create_node t Uninitialized in
  let bind =
    { Bind.main
    ; f
    ; lhs
    ; lhs_change
    ; rhs = Uopt.none
    ; rhs_scope = Scope.top
    ; all_nodes_created_on_rhs = Uopt.none
    }
  in
  (* We set [lhs_change] to never cutoff so that whenever [lhs] changes, [main] is
     recomputed.  This is necessary to handle cases where [f] returns an existing stable
     node, in which case the [lhs_change] would be the only thing causing [main] to be
     stale. *)
  Node.set_cutoff lhs_change Cutoff.never;
  bind.rhs_scope <- Bind bind;
  Node.set_kind lhs_change (Bind_lhs_change bind);
  Node.set_kind main (Bind_main bind);
  main
;;

let bind2 n1 n2 ~f =
  bind (map2 n1 n2 ~f:(fun v1 v2 -> v1, v2)) ~f:(fun (v1, v2) -> f v1 v2)
;;

let bind3 n1 n2 n3 ~f =
  bind
    (map3 n1 n2 n3 ~f:(fun v1 v2 v3 -> v1, v2, v3))
    ~f:(fun (v1, v2, v3) -> f v1 v2 v3)
;;
...
```

## Observe

### Concept

Say that a node is **observed** if there is an observer for it.(created via ``observe``).

A node is **necessary** if there is a path from the node to an observed node. **Stabilize** ensures
that all necessary nodes have correct values; it will not compute unnecessary nodes.

An unobserved node becomes necessary by a call to ``observe`` or by being used to compute an observed
node; this will cause the appropriate DAG edges to be added.

A necessary node will become unnecessary if its observer (if any) becomes unused and if the node is no longer used to compute any observed nodes. This will cause the appropriate DAG edges to be removed.

### Create observer

```ocaml
type 'a t = 'a Types.Internal_observer.t =
  { (* State transitions:

       {v
         Created --> In_use --> Disallowed --> Unlinked
           |                                     ^
           \-------------------------------------/
       v} *)
    mutable state : State.t
  ; observing : 'a Node.t
  ; mutable on_update_handlers : 'a On_update_handler.t list
  ; (* [{prev,next}_in_all] doubly link all observers in [state.all_observers]. *)
    mutable prev_in_all : Packed_.t Uopt.t
  ; mutable next_in_all : Packed_.t Uopt.t
  ; (* [{prev,next}_in_observing] doubly link all observers of [observing]. *)
    mutable prev_in_observing : ('a t[@sexp.opaque]) Uopt.t
  ; mutable next_in_observing : ('a t[@sexp.opaque]) Uopt.t
  }
```

The initial state of observer is ``Created``, and ``observing`` is the correspondding node. Other
fields will be set to empty or none.

```ocaml
let create_observer ?(should_finalize = true) (observing : _ Node.t) =
  let t = observing.state in
  let internal_observer : _ Internal_observer.t =
    { state = Created
    ; observing
    ; on_update_handlers = []
    ; prev_in_all = Uopt.none
    ; next_in_all = Uopt.none
    ; prev_in_observing = Uopt.none
    ; next_in_observing = Uopt.none
    }
  in
  Stack.push t.new_observers (T internal_observer);
  let observer = ref internal_observer in
  if should_finalize
  then Gc.Expert.add_finalizer_exn observer (unstage (observer_finalizer t));
  t.num_active_observers <- t.num_active_observers + 1;
  observer
;;
```

## State

Before we dive into ``stabilization``, let's analyze the last but important structure ``State``.

```ocaml
type t = Types.State.t =
  { mutable status : status
  ; bind_lhs_change_should_invalidate_rhs : bool
  ; (* [stabilization_num] starts at zero, and is incremented at the end of each
       stabilization. *)
    mutable stabilization_num : Stabilization_num.t
  ; mutable current_scope : Scope.t
  ; recompute_heap : Recompute_heap.t
  ; adjust_heights_heap : Adjust_heights_heap.t
  ; (* [propagate_invalidity] holds nodes that have invalid children that should be
       considered for invalidation.  It is only used during graph restructuring:
       [invalidate_node] and [add_parent].  Once an element is added to the stack, we then
       iterate until invalidity has propagated to all ancestors as necessary, according to
       [Node.should_be_invalidated]. *)
    propagate_invalidity : Node.Packed.t Stack.t
  ; (* [num_active_observers] is the number of observers whose state is [Created] or
       [In_use]. *)
    mutable num_active_observers : int
  ; (* [all_observers] is the doubly-linked list of all observers in effect, or that have
       been disallowed since the most recent start of a stabilization -- these have
       [state] as [In_use] or [Disallowed]. *)
    mutable all_observers : Internal_observer.Packed.t Uopt.t
  ; (* We enqueue finalized observers in a thread-safe queue, for handling during
       stabilization.  We use a thread-safe queue because OCaml finalizers can run in any
       thread. *)
    finalized_observers : Internal_observer.Packed.t Thread_safe_queue.t
  ; (* [new_observers] holds observers created since the most recent start of a
       stabilization -- these have [state] as [Created] or [Unlinked].  At the start of
       stabilization, we link into [all_observers] all observers in [new_observers] whose
       state is [Created] and add them to the [observers] of the node they are observing.
       We structure things this way to allow observers to be created during stabilization
       while running user code ([map], [bind], etc), but to not have to deal with nodes
       becoming necessary and the the graph changing during such code. *)
    new_observers : Internal_observer.Packed.t Stack.t
  ; (* [disallowed_observers] holds all observers that have been disallowed since the most
       recent start of a stabilization -- these have [state = Disallowed].  At the start
       of stabilization, these are unlinked from [all_observers] and their state is
       changed to [Unlinked].  We structure things this way to allow user code running
       during stabilization to call [disallow_future_use], but to not have to deal with
       nodes becoming unnecessary and the graph changing during such code. *)
    disallowed_observers : Internal_observer.Packed.t Stack.t
  ; (* We delay all [Var.set] calls that happen during stabilization so that they take
       effect after stabilization.  All variables set during stabilization are pushed on
       [set_during_stabilization] rather than setting them.  Then, after the graph has
       stabilized, we do all the sets, so that they take effect at the start of the next
       stabilization. *)
    set_during_stabilization : Var.Packed.t Stack.t
  ; (* [handle_after_stabilization] has all nodes with handlers to consider running at the
       end of the next stabilization.  At the end of stabilization, we consider each node
       in [handle_after_stabilization], and if we decide to run its on-update handlers,
       push it on [run_on_update_handlers].  Then, once we've considered all nodes in
       [handle_after_stabilization], we iterate through [run_on_update_handlers] and
       actually run the handlers.

       These two passes are essential for correctness.  During the first pass, we haven't
       run any user handlers, so we know that the state is exactly as it was when
       stabilization finished.  In particular, we know that if a node is necessary, then
       it has a stable value; once user handlers run, we don't know this.  During the
       second pass, user handlers can make calls to any incremental function except for
       [stabilize].  In particular, some functions push nodes on
       [handle_after_stabilization].  But no functions (except for [stabilize]) modify
       [run_on_update_handlers]. *)
    handle_after_stabilization : Node.Packed.t Stack.t
  ; run_on_update_handlers : Run_on_update_handlers.t Stack.t
  ; mutable only_in_debug : Only_in_debug.t
  ; weak_hashtbls : Packed_weak_hashtbl.t Thread_safe_queue.t
  ; mutable keep_node_creation_backtrace : bool
  ; (* Stats.  These are all incremented at the appropriate place, and never decremented. *)
    mutable num_nodes_became_necessary : int
  ; mutable num_nodes_became_unnecessary : int
  ; mutable num_nodes_changed : int
  ; mutable num_nodes_created : int
  ; mutable num_nodes_invalidated : int
  ; mutable num_nodes_recomputed : int
  ; mutable num_nodes_recomputed_directly_because_one_child : int
  ; mutable num_nodes_recomputed_directly_because_min_height : int
  ; mutable num_var_sets : int
  }

```

## Stabilize

## Recompute_heap
