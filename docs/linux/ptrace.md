# Ptrace

The implementaion of gdb depends on ptrace. Linux provides a syscall
``ptrace``.

```c
SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr,
                unsigned long, data)
```

## child

Child should invoke ``ptrace`` with request ``PTRACE_TRACEME`` so that
father can use ``ptrace`` to track child process.

## parent

### attach

TODO

### 

``ptrace_check_attach`` checks whether ptracee is ready for ptrace operation.
If ``ignore_state`` is true, child can be in any state. If ``ignore_state``
is false, child should have type TASK_TRACED.
```c
static int ptrace_check_attach(struct task_struct *child, bool ignore_state)
```