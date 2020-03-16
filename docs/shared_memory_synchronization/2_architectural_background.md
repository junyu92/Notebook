# Architectural Background

## Cores and Caches

* UMA: uniform meory access
Symmetric machine, all memory backs are equally distant from every process core.

* NUMA: nonuniform memoty access

Each memory back is associated with a processor and can be accessed by cores of the
local processor more quickly than by cores of other processes.

## Memory Consistency

Most real machines implement a more relaxed(inconsistent) memory model, in
which accesses by different threads, or to different locations by the same
thread, may appear to occur "out of order" from the perspective of threads
on other cores.

When consistency is required, programmers must employ special ``synchronizing
instructions`` that are more strongly ordered than other, "oridinary"
instructions, forcing the local core to wait for various classes of potentially
inflight event.

### Sources of inconsistency

* reorder buffer

a write must be held in the reorder buffer until all instuctions that precede
it in program order have completed.

* store buffer

When executing a ``load`` instruction, a core checks the contents of its
reorder and store buffers before forwarding a request to the memory system.

### Special instructions to order memory access



## Atomic Primitives