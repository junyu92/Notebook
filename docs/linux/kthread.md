# Kthread

Kernel thread is a thread running in the kernel mode. If we execute ``ps -ef``,
we can find plenty of threads whose CMD are surrounded by [] are kthread.

## Create

```c
/**
 * kthread_run - create and wake a thread.
 * @threadfn: the function to run until signal_pending(current).
 * @data: data ptr for @threadfn.
 * @namefmt: printf-style name for the thread.
 *
 * Description: Convenient wrapper for kthread_create() followed by
 * wake_up_process().  Returns the kthread or ERR_PTR(-ENOMEM).
 */
#define kthread_run(threadfn, data, namefmt, ...)                          \
({                                                                         \
        struct task_struct *__k                                            \
                = kthread_create(threadfn, data, namefmt, ## __VA_ARGS__); \
        if (!IS_ERR(__k))                                                  \
                wake_up_process(__k);                                      \
        __k;                                                               \
})
```

```c
 __printf(4, 5)
 struct task_struct *kthread_create_on_node(int (*threadfn)(void *data),
                                            void *data,
                                            int node,
                                            const char namefmt[], ...);

 /**
  * kthread_create - create a kthread on the current node
  * @threadfn: the function to run in the thread
  * @data: data pointer for @threadfn()
  * @namefmt: printf-style format string for the thread name
  * @arg...: arguments for @namefmt.
  *
  * This macro will create a kthread on the current node, leaving it in
  * the stopped state.  This is just a helper for kthread_create_on_node();
  * see the documentation there for more details.
  */
 #define kthread_create(threadfn, data, namefmt, arg...) \
         kthread_create_on_node(threadfn, data, NUMA_NO_NODE, namefmt, ##arg)
```

```c
/**
 * kthread_create_on_node - create a kthread.
 * @threadfn: the function to run until signal_pending(current).
 * @data: data ptr for @threadfn.
 * @node: task and thread structures for the thread are allocated on this node
 * @namefmt: printf-style name for the thread.
 *
 * Description: This helper function creates and names a kernel
 * thread.  The thread will be stopped: use wake_up_process() to start
 * it.  See also kthread_run().  The new thread has SCHED_NORMAL policy and
 * is affine to all CPUs.
 *
 * If thread is going to be bound on a particular cpu, give its node
 * in @node, to get NUMA affinity for kthread stack, or else give NUMA_NO_NODE.
 * When woken, the thread will run @threadfn() with @data as its
 * argument. @threadfn() can either call do_exit() directly if it is a
 * standalone thread for which no one will call kthread_stop(), or
 * return when 'kthread_should_stop()' is true (which means
 * kthread_stop() has been called).  The return value should be zero
 * or a negative error number; it will be passed to kthread_stop().
 *
 * Returns a task_struct or ERR_PTR(-ENOMEM) or ERR_PTR(-EINTR).
 */
struct task_struct *kthread_create_on_node(int (*threadfn)(void *data),
					   void *data, int node,
					   const char namefmt[],
					   ...)
{
	struct task_struct *task;
	va_list args;

	va_start(args, namefmt);
	task = __kthread_create_on_node(threadfn, data, node, namefmt, args);
	va_end(args);

	return task;
}
EXPORT_SYMBOL(kthread_create_on_node);
```

The following function is the core of ``kthread_run``.

```c
struct kthread_create_info
{
        /* Information passed to kthread() from kthreadd. */
        int (*threadfn)(void *data);
        void *data;
        int node;

        /* Result passed back to kthread_create() from kthreadd. */
        struct task_struct *result;
        struct completion *done;

        struct list_head list;
};
```

```c
/* Location: kernel/kthread.c */

static DEFINE_SPINLOCK(kthread_create_lock);
static LIST_HEAD(kthread_create_list);
struct task_struct *kthreadd_task;

struct task_struct *__kthread_create_on_node(int (*threadfn)(void *data),
						    void *data, int node,
						    const char namefmt[],
						    va_list args)
{
	DECLARE_COMPLETION_ONSTACK(done);
	struct task_struct *task;

  // alloc a physically sequential memory area.
	struct kthread_create_info *create = kmalloc(sizeof(*create),
						     GFP_KERNEL);

	if (!create)
		return ERR_PTR(-ENOMEM);
	create->threadfn = threadfn;
	create->data = data;
	create->node = node;
	create->done = &done;

  // Insert the create info to create_list
	spin_lock(&kthread_create_lock);
	list_add_tail(&create->list, &kthread_create_list);
	spin_unlock(&kthread_create_lock);

	wake_up_process(kthreadd_task);
	/*
	 * Wait for completion in killable state, for I might be chosen by
	 * the OOM killer while kthreadd is trying to allocate memory for
	 * new kernel thread.
	 */
	if (unlikely(wait_for_completion_killable(&done))) {
		/*
		 * If I was SIGKILLed before kthreadd (or new kernel thread)
		 * calls complete(), leave the cleanup of this structure to
		 * that thread.
		 */
		if (xchg(&create->done, NULL))
			return ERR_PTR(-EINTR);
		/*
		 * kthreadd (or new kernel thread) will call complete()
		 * shortly.
		 */
		wait_for_completion(&done);
	}
	task = create->result;
	if (!IS_ERR(task)) {
    // Fetch name from namefmt and set it to task
		static const struct sched_param param = { .sched_priority = 0 };
		char name[TASK_COMM_LEN];

		/*
		 * task is already visible to other tasks, so updating
		 * COMM must be protected.
		 */
		vsnprintf(name, sizeof(name), namefmt, args);
		set_task_comm(task, name);
		/*
		 * root may have changed our (kthreadd's) priority or CPU mask.
		 * The kernel thread should not inherit these properties.
		 */
		sched_setscheduler_nocheck(task, SCHED_NORMAL, &param);
		set_cpus_allowed_ptr(task, cpu_all_mask);
	}
	kfree(create);
	return task;
}
```

As we can see above that ``__kthread_create_on_node`` won't create
``struct task_struct`` but create ``kthread_create_info`` and insert
it into ``kthread_create_list``.

It leads us to another function ``kthreadd``.

The loop body of this function will pop a ``kthread_create_info`` if
``kthread_create_list`` is not empty, and then invoke ``create_kthread``

Let's bypass ``create_kthread`` for a moment.

```c
int kthreadd(void *unused)
{
        struct task_struct *tsk = current;

        /* Setup a clean context for our children to inherit. */
        set_task_comm(tsk, "kthreadd");
        ignore_signals(tsk);
        set_cpus_allowed_ptr(tsk, cpu_all_mask);
        set_mems_allowed(node_states[N_MEMORY]);

        current->flags |= PF_NOFREEZE;
        cgroup_init_kthreadd();

        for (;;) {
                set_current_state(TASK_INTERRUPTIBLE);
                if (list_empty(&kthread_create_list))
                        schedule();
                __set_current_state(TASK_RUNNING);

                spin_lock(&kthread_create_lock);
                while (!list_empty(&kthread_create_list)) {
                        struct kthread_create_info *create;

                        create = list_entry(kthread_create_list.next,
                                            struct kthread_create_info, list);
                        list_del_init(&create->list);
                        spin_unlock(&kthread_create_lock);

                        create_kthread(create);

                        spin_lock(&kthread_create_lock);
                }
                spin_unlock(&kthread_create_lock);
        }

        return 0;
}
```

```
> ps -ef

root         2     0  0 09:33 ?        00:00:00 [kthreadd]
```

There is a deamon called ``kthreadd`` which was created at initial stage.

```
/* Location: init/main.c */

pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
```

So all creation tasks are handled by ``kthreadd``.

Go back to ``create_kthread``.

```c
static void create_kthread(struct kthread_create_info *create)
{
        int pid;

#ifdef CONFIG_NUMA
        current->pref_node_fork = create->node;
#endif
        /* We want our own signal handler (we take no signals by default). */
        pid = kernel_thread(kthread, create, CLONE_FS | CLONE_FILES | SIGCHLD);
        if (pid < 0) {
                /* If user was SIGKILLed, I release the structure. */
                struct completion *done = xchg(&create->done, NULL);

                if (!done) {
                        kfree(create);
                        return;
                }
                create->result = ERR_PTR(pid);
                complete(done);
        }
}
```

```c
/*
 * Create a kernel thread.
 */
pid_t kernel_thread(int (*fn)(void *), void *arg, unsigned long flags)
{
        struct kernel_clone_args args = {
                .flags          = ((flags | CLONE_VM | CLONE_UNTRACED) & ~CSIGNAL),
                .exit_signal    = (flags & CSIGNAL),
                .stack          = (unsigned long)fn,
                .stack_size     = (unsigned long)arg,
        };

        return _do_fork(&args);
}
```

## Wake up

After creating kthread, ``kthread_run`` invokes ``wake_up_process`` to wake up
the creatied thread. Since the function is belong to schedule module, we don't
dive into it now.

```c
/* Location: kernel/sched/core.c */

/**
 * wake_up_process - Wake up a specific process
 * @p: The process to be woken up.
 *
 * Attempt to wake up the nominated process and move it to the set of runnable
 * processes.
 *
 * Return: 1 if the process was woken up, 0 if it was already running.
 *
 * This function executes a full memory barrier before accessing the task state.
 */
int wake_up_process(struct task_struct *p)
{
        return try_to_wake_up(p, TASK_NORMAL, 0);
}
EXPORT_SYMBOL(wake_up_process);
```

## Exit

The thread will keep running unless it voluntarily exits by invoking ```do_exit``` or
received a signal(``kthread_should_stop``) then stop itselt. 

The signal is send by other thread by invoking ``kthread_stop``.

```c
int kthread_stop(struct task_struct *k);
bool kthread_should_stop(void);
```

```c
/**
 * kthread_stop - stop a thread created by kthread_create().
 * @k: thread created by kthread_create().
 *
 * Sets kthread_should_stop() for @k to return true, wakes it, and
 * waits for it to exit. This can also be called after kthread_create()
 * instead of calling wake_up_process(): the thread will exit without
 * calling threadfn().
 *
 * If threadfn() may call do_exit() itself, the caller must ensure
 * task_struct can't go away.
 *
 * Returns the result of threadfn(), or %-EINTR if wake_up_process()
 * was never called.
 */
int kthread_stop(struct task_struct *k)
{
        struct kthread *kthread;
        int ret;

        trace_sched_kthread_stop(k);

        get_task_struct(k);
        kthread = to_kthread(k);
        set_bit(KTHREAD_SHOULD_STOP, &kthread->flags);
        kthread_unpark(k);
        wake_up_process(k);
        wait_for_completion(&kthread->exited);
        ret = k->exit_code;
        put_task_struct(k);

        trace_sched_kthread_stop_ret(ret);
        return ret;
}
EXPORT_SYMBOL(kthread_stop);
```