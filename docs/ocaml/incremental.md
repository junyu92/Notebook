# Incremental

## introduction

Incremental si an optimization tool.

### The computational model

Incremental operates a lot like a build system: it lets you organize
your computation as a graph with explicit dependencies, and it caches
previous computations in the nodes of the graph.

That way, when one of the inputs to your computation is updated, only
the part of the graph that depends on that input needs to be re-evaluated.

There are three basic parts to an increpemtal graph:

* variables

which constitute the inputs to the computation.

* incrementals

which correspond to the intermediate stages of the computation.

* observers

which are nodes from which you can read the output of a computation.


In practice, you always need all three. You **initialize a variable just like you would a ref**; you
**create an incremental from it**, which can be used in as many different incremental computations
you can dream up; as your data changes, you update the value of your variable with ~Incr.Var.set~;
and then those changes **propagate through your incremental computations to the observers**,
one for each computation that you want to use the output from.

Let's look at an example.

```ocaml
open Core

module Incr = Incremental.Make ()

# now we can use Incr.Var.create to declare a few variables.
let (x_v,y_v,z_v) = Incr.Var.(create 3., create 0.5, create 1.5)
```

To be able to compute with these incremental variables, we need to grab the
incremental variables corresponding to those values, using ``Incr.Var.watch``.

```ocaml
let (x,y,z) = Incr.Var.(watch x_v, watch y_v, watch z_v)

# Let's sum these up

let sum =
  Incr.map2
    z
    (Incr.map2 x y ~f:(fun x y -> x +. y))
    ~f:(fun z x_and_y -> z +. x_and_y)
```

The given incremental computation can be represented by a graph:

```
  +---+
  | x |-.
  +---+  \   +---------+
  +---+   '->| x_and_y |-.
  | y |----->+---------+  \
  +---+                    '->+------+
  +---+                       | sum  |
  | z |---------------------->+------+
  +---+
```

We can also rewrite this using let syntax, as follows.

```ocaml
open Incr.Let_syntax

let sum =
  let%map x_and_y =
    let%map x = x and y = y in
    x +. y
  and z = z in
  z +. x_and_y
```

In order to read the value out, we have to create an **observer**, at which
point we can grab the current value.

However, before reading the value, we need to call ``stabilize``, which makes
sure that the output of the computation is up-to-date.

The function of getting the value from an observer is called ``~value_exn~`` because
it will throw in the case that the observer in question has not been made
valid by a stabilize.

```ocaml
let sum_o = Incr.observe sum

let show x =
  Incr.stabilize()
  print_s [%sexp (Incr.Observer.value_exn x : float)];;

let () =
  show sum_o
```

We can update the value by changing the inputs.

```ocaml
let (:=) = Incr.Var.set
let (!) = Incr.Var.value

let () =
  z_v := 100.;
  show sum_o;
```

It would probably be a little more efficient to have written ~sum~ as

```ocaml
let sum =
  let%map x = x and y = y and z = z in
  x +. y +. z

val sum : float Incr.t = <abstr>
```

### Projections and Cutoffs

Another useful trick when building an incremental computation is to break
up a complex data-type by projecting out individual components of that
data type, and then doing further incremental work on those components
directly.

Let's start with a data type with two components.

```ocaml
type z =
  { a: int list
  ; b: (int * int)
  }
```

Now, we are going to write a function that multiplies together the integers
in ~a~ and ~b~ respectively, and then sums them up.

```ocaml
let sumproduct z =
  let a_prod =
    let%map a = z >>| a in
    printf "a\n";
    List.fold ~init:1 ~f:( * ) a
  in
  let b_prod =
    let%map (b1, b2) = z >>| b in
    printf "b\n";
    b1 * b2
  in
  let%map a_prod = a_prod and b_prod = b_prod in
  a_prod + b_prod
```

The shape of the implied graph is something like this:

```
            +---+   +--------+
         .->| a |-->| a_prod |-.
        /   +---+   +--------+  \
  +---+/                         '->+--------+
  | z |                             | result |
  +---+\                         .->+--------+
        \   +---+   +--------+  /
         '->| b |-->| b_prod |-'
            +---+   +--------+
```

Let's run it.

```ocaml
let z = Incr.Var.create { a = [3;2]; b = (1,4) }
let result = Incr.observe (sumproduct (Incr.Var.watch z))
let show () =
  Incr.stabilize ();
  printf "result: %d\n" (Incr.Observer.value_exn result)

let () = show ()
```

The result should be 10.

The **incremental's change-propagation algorithm** cuts off when
the value of a given node doesn't change.

WARNINGS: incremental nodes by default cut off change propagation
on physical equality of inputs. I.e., if the value in question is
exactly the same (for an immediate) or the same pointer (for heap-allocated
objects), then propagation stops.

The use of physical equality can have surprising results.  For
example, if I set ~b~ again to the same value, the computation will
again be rerun, since I'll be setting it to a newly allocated tuple.

```ocaml
let () =
  z := { !z with b = (1,1) };
  show ()

(*
 * b
 * result: 7
 *)
```

Be careful, if we don't create that intermediate map node.

```c
let sumproduct z =
  let a_prod =
    let%map { a; _ } = z in
    printf "a\n";
    List.fold ~init:1 ~f:( * ) a
  in
  let b_prod =
    let%map {b = (b1,b2); _ } = z in
    printf "b\n";
    b1 * b2
  in
  let%map a_prod = a_prod and b_prod = b_prod in
  a_prod + b_prod
```

This function computes the same thing, but the graph is very different.

```
            +--------+
         .->| a_prod |-.
        /   +--------+  \
  +---+/                 '->+--------+
  | z |                     | result |
  +---+\                 .->+--------+
        \   +--------+  /
         '->| b_prod |-'
            +--------+
```

We don't have the ``a`` and ``b`` nodes to short-circuit the computation,
so the computation of ``a_prod`` and ``b_prod`` will always happen
whenever ``z`` changes. In this situation, ``a`` will be recomputed even though we only
change ``b``.

```ocaml
let () =
  z := { !z with b = (1,1) };
  show ()
[%%expect{|
a
b
result: 7
|}]
```


It often makes sense to set a **custom cutoff function**, which you can do
using this function:

```ocaml
let _ = Incr.set_cutoff

let int_cutoff = Incr.Cutoff.of_equal Int.equal
```

## dynamic computations with bind

## map

## pitfalls

## time

## patterns

## performance and optimazation

## sharing