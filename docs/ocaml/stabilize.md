## stabilize

In this section we will explain how stabilization works.

At the very beginning, if current state is not ``Not_stabilizing``, then raise an exception.

```ocaml
let stabilize t =
  ensure_not_stabilizing t ~name:"stabilize" ~allow_in_update_handler:false;
```

then try to stabilize the state ``t``. if an exception raised then set the status to
``Stabilize_previously_raised`` and raise the exception.

The first job to stabilize is setting the status of state ``t`` to ``Stabilizing``.

```ocaml
t.status <- Stabilizing;
```

By default, an observer has a **finalizer** that calls ``disallow_future_use`` when the observer
is no longer referenced. If an observer has not finalizer, the observer will live until
``disallow_future_use`` is explicitly called.

TODO

```ocaml
disallow_finalized_observers t;
```

Then handle all new observers(created via ``create_observer``) whose type are ensured to be ``Created``.

```ocaml
add_new_observers t;
```

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

Provided that ``base_area`` node is ``Created``.

The state of ``base_area`` will be set to ``In_use``.

```ocaml
internal_observer.state <- In_use;
```

If there are some ``observers``(``old_all_observers``) in ``t``,  then we set

```ocaml
if Uopt.is_some old_all_observers
then (
    internal_observer.next_in_all <- old_all_observers;
    Packed.set_prev_in_all (Uopt.unsafe_value old_all_observers) (Uopt.some packed));
t.all_observers <- Uopt.some packed;
```

If the node of internal_observer was not necessary, then we set it and its children necessary. and
the node will be added to the recompute_heap.

```ocaml
let observing = internal_observer.observing in
let was_necessary = Node.is_necessary observing in
if not was_necessary then became_necessary observing
```

Then add ``num_on_update_handlers`` of ``observer`` to ``observing``.

```ocaml
observing.num_on_update_handlers
<- observing.num_on_update_handlers
    + List.length internal_observer.on_update_handlers;
      let old_observers = observing.observers in
```

Add the observer to observing's observers.

```ocaml
if Uopt.is_some old_observers
then (
    internal_observer.next_in_observing <- old_observers;
    (Uopt.unsafe_value old_observers).prev_in_observing
    <- Uopt.some internal_observer);
observing.observers <- Uopt.some internal_observer;
```

By adding ``internal_observer`` to ``observing.observers``, we may have added
on-update handlers to ``observing``.  We need to handle ``observing`` after this
stabilization to give those handlers a chance to run.

```ocaml
let handle_after_stabilization : type a. a Node.t -> unit =
  fun node ->
  (* Make sure we will not add a repeat node *)
  if not node.is_in_handle_after_stabilization
  then (
    let t = node.state in
    node.is_in_handle_after_stabilization <- true;
    Stack.push t.handle_after_stabilization (T node))
;;

handle_after_stabilization observing;
```

Now we can go back to ``stabilize``. Next step is ``unlink_disallowed_observers``.

TODO

This function will pop all ``disallowed_observers`` out and the state of them to ``Disallowed``.

```ocaml
let unlink_disallowed_observers t =
  while Stack.length t.disallowed_observers > 0 do
    let packed = Stack.pop_exn t.disallowed_observers in
    let module Packed = Internal_observer.Packed in
    let (T internal_observer) = packed in
    if debug
    then
      assert (
        match internal_observer.state with
        | Disallowed -> true
        | _ -> false);
    internal_observer.state <- Unlinked;
    let (T all_observers) = Uopt.value_exn t.all_observers in
    if Internal_observer.same internal_observer all_observers
    then t.all_observers <- internal_observer.next_in_all;
    Internal_observer.unlink internal_observer;
    check_if_unnecessary internal_observer.observing
  done
;;

unlink_disallowed_observers t;
```

Then compute ``recompute_heap`` via ``recompute_everything_that_is_necessary``.

```ocaml
recompute_everything_that_is_necessary t;
```

and update ``stabilization_num``

```ocaml
t.stabilization_num <- Stabilization_num.add1 t.stabilization_num;
```

then update lazy updated value

```ocaml
while not (Stack.is_empty t.set_during_stabilization) do
  let (T var) = Stack.pop_exn t.set_during_stabilization in
  let value = Uopt.value_exn var.value_set_during_stabilization in
  var.value_set_during_stabilization <- Uopt.none;
  set_var_while_not_stabilizing var value
done;
```

Then push all nodes that in handle after stabilization into the stack ``run_on_update_handlers``
of state ``t``. And them run the registed update function via ``run_on_update_handlers``.

```ocaml
while not (Stack.is_empty t.handle_after_stabilization) do
  let (T node) = Stack.pop_exn t.handle_after_stabilization in
  node.is_in_handle_after_stabilization <- false;
  let old_value = node.old_value_opt in
  node.old_value_opt <- Uopt.none;
  let node_update : _ Node_update.t =
    if not (Node.is_valid node)
    then Invalidated
    else if not (Node.is_necessary node)
    then Unnecessary
    else (
      let new_value = Uopt.value_exn node.value_opt in
      if Uopt.is_none old_value
      then Necessary new_value
      else Changed (Uopt.unsafe_value old_value, new_value))
  in
  Stack.push t.run_on_update_handlers (T (node, node_update))
done;
t.status <- Running_on_update_handlers;
let now = t.stabilization_num in
while not (Stack.is_empty t.run_on_update_handlers) do
  let (T (node, node_update)) = Stack.pop_exn t.run_on_update_handlers in
  Node.run_on_update_handlers node node_update ~now
done;
```

The last step is setting the status of ``state`` to ``Not_stabilizing``.

```ocaml
t.status <- Not_stabilizing;
reclaim_space_in_weak_hashtbls t
```

## Recompute & Recompute_heap

Every state maintains a recompute_heap to compute the value of nodes.
Assuming that ``n`` is a node in the heap, **every node depend on ``n`` has the
higher height than ``n``**.

```ocaml
let recompute_everything_that_is_necessary t =
  let module R = Recompute_heap in
  let r = t.recompute_heap in
  while R.length r > 0 do
    let (T node) = R.remove_min r in
    recompute node
  done;
;;
```

The core function of ``recompute_everything_that_is_necessary`` is ``recompute``.
According to the type of Node, ``recompute`` will compute value differently.

If the type of Node is ``Var``, this function just get the value of var.

```ocaml
| Var var -> maybe_change_value node var.value
```

If the type is ``Map2``, the function will apply ``n1`` and ``n2`` to function ``f``. Since
current node depends on ``n1`` and ``n2``, ``n1`` and ``n2`` have lower height than current node,
they must had already been computed.

```ocaml
| Map2 (f, n1, n2) ->
  maybe_change_value node (f (Node.value_exn n1) (Node.value_exn n2))
```

``Bind`` is much different and powerful than others.

```ocaml
val bind : 'a t -> f:('a -> 'b t) -> 'b t

val if_ : bool t -> a t -> a t -> a t
let if_ a b c = bind a ~f:(fun a -> if a then b else c)
```

```ocaml
| Bind_lhs_change
    ({ main
     ; f
     ; lhs
     ; rhs_scope
     ; rhs = old_rhs
     ; all_nodes_created_on_rhs = old_all_nodes_created_on_rhs
     ; _
     } as bind) ->
```

We clear ``all_nodes_created_on_rhs`` so it will hold just the nodes created by
this call to ``f``(via ``run_with_scope``).

```ocaml
bind.all_nodes_created_on_rhs <- Uopt.none;
let rhs = run_with_scope t rhs_scope ~f:(fun () -> f (Node.value_exn lhs)) in
bind.rhs <- Uopt.some rhs;
```


(* Anticipate what [maybe_change_value] will do, to make sure Bind_main is stale
   right away. This way, if the new child is invalid, we'll satisfy the invariant
   saying that [needs_to_be_computed bind_main] in [propagate_invalidity] *)

```ocaml
  node.changed_at <- t.stabilization_num;
  change_child
    ~parent:main
    ~old_child:old_rhs
    ~new_child:rhs
    ~child_index:Kind.bind_rhs_child_index;
```

We invalidate after ``change_child``, because invalidation changes the ``kind`` of
nodes to ``Invalid``, which means that we can no longer visit their children.
Also, the ``old_rhs`` nodes are typically made unnecessary by ``change_child``, and
so by invalidating afterwards, we will not waste time adding them to the
recompute heap and then removing them.

```ocaml
if Uopt.is_some old_rhs
then (
  if t.bind_lhs_change_should_invalidate_rhs
  then invalidate_nodes_created_on_rhs old_all_nodes_created_on_rhs
  else
    rescope_nodes_created_on_rhs
      t
      old_all_nodes_created_on_rhs
      ~new_scope:main.created_in;
  propagate_invalidity t);
(* [node] was valid at the start of the [Bind_lhs_change] branch, and invalidation
   only visits higher nodes, so [node] is still valid. *)
if debug then assert (Node.is_valid node);
maybe_change_value node ()
| Bind_main { rhs; _ } -> copy_child ~parent:node ~child:(Uopt.value_exn rhs)
```
