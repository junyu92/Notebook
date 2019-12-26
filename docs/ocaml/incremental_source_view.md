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

In the previous section, we didn't mention the field ``watch``.

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

## Recompute_heap