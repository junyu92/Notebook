# Introduction

## Atomicity vs Conditional synchronization

## Spining vs Blocking

Just as synchronization patterns tend to fall into two main camps
(atomicity and condition synchronization), so too do their implementations:
they all employ ``spinning`` or ``blocking``.

For condition synchronization, it takes the form of a trivial loop:

```ocaml
while not condition
  // do nothing(spin)
```

For mutual exclusion, the simplest implementation employs a special
``hardware instruction`` known as ``test_and_set``(TAS). The TAS instuction
sets a specified Boolean variable to true and returns the previous value.

Using TAS, we can implement a trivial spin lock:

```ocaml
type lock = bool := false

L.acquire():
  while not TAS(&L)
    // spin

L.release():
  L := false
```

Pros:
* Blockking yields the processor core to some other, runnable thread.

Cons:
* Spinning wastes processor cycles.
* Blocking wastes on fruitless re-checks of a confition or lock, it spends
cycles on the context switching overhead required to change the running thread.

## Safety and Liveness

Safety means that bad things never happen: we never have two threads in a
critical section for the same lock at the same time; we never have all of
the threads in the system blocked.

Liveness means that good things eventually happen: if lock L is free and at
least one thread is waiting for it, some thread eventually acquire it.