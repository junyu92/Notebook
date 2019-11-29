# Time management

## Tiemr

Linux提供**software tiemr**使得kernel functions能够在将来被调用。

Linux提供两种类型的**timer**:

* dynamic timers

* interval timers

### 初始化

在``start_kernel``初始化时，内核会调用``init_timers``函数。

```c
void __init init_timers(void)
{
        init_timer_cpus();
        open_softirq(TIMER_SOFTIRQ, run_timer_softirq);
}
```

我们看一下每个函数的实现。

```c
static void __init init_timer_cpus(void)
{
        int cpu;

        for_each_possible_cpu(cpu)
                init_timer_cpu(cpu);
}
```

它为每个possible_cpu调用``init_timer_cpu``函数。

```c
static void __init init_timer_cpu(int cpu)
{
        struct timer_base *base;
        int i;

        for (i = 0; i < NR_BASES; i++) {
                base = per_cpu_ptr(&timer_bases[i], cpu);
                base->cpu = cpu;
                raw_spin_lock_init(&base->lock);
                base->clk = jiffies;
        }
}
```

``init_timer_cpu``函数会初始化每个cpu的``timer_base``。``timer_base``的结构如下：

```c
struct timer_base {
        raw_spinlock_t          lock;
        struct timer_list       *running_timer;
        unsigned long           clk;
        unsigned long           next_expiry;
        unsigned int            cpu;
        bool                    is_idle;
        bool                    must_forward_clk;
        DECLARE_BITMAP(pending_map, WHEEL_SIZE);
        struct hlist_head       vectors[WHEEL_SIZE];
} ____cacheline_aligned;

static DEFINE_PER_CPU(struct timer_base, timer_bases[NR_BASES]);
```

* ``lock``很显然
* ``running_tiemr``表示运行在该cpu上的timer
* ``cpu``表示timer属于哪个cpu
* ``clk`` the clk fields represents the earliest expiration time (it will be used by the Linux kernel to find already expired timers)

``init_timers``最后一个调用的函数是``open_softirq``.

```c
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
        softirq_vec[nr].action = action;
}
```

它给中断向量号``TIMER_SOFTIRQ``注册了deferred interrupt handler函数``run_timer_softirq``.

它会在do_IRQ中被调用，这里不展开讲了。

```c
/* arch/x86/kernel/irq.c */
/*
 * do_IRQ handles all normal device IRQ's (the special
 * SMP cross-CPU interrupts have their own specific
 * handlers).
 */
__visible unsigned int __irq_entry do_IRQ(struct pt_regs *regs)
{
        struct pt_regs *old_regs = set_irq_regs(regs);
        struct irq_desc * desc;
        /* high bit used in ret_from_ code  */
        unsigned vector = ~regs->orig_ax;

        entering_irq();

        /* entering_irq() tells RCU that we're not quiescent.  Check it. */
        RCU_LOCKDEP_WARN(!rcu_is_watching(), "IRQ failed to wake up RCU");

        desc = __this_cpu_read(vector_irq[vector]);

        if (!handle_irq(desc, regs)) {
                ack_APIC_irq();

                if (desc != VECTOR_RETRIGGERED && desc != VECTOR_SHUTDOWN) {
                        pr_emerg_ratelimited("%s: %d.%d No irq handler for vector\n",
                                             __func__, smp_processor_id(),
                                             vector);
                } else {
                        __this_cpu_write(vector_irq[vector], VECTOR_UNUSED);
                }
        }

        exiting_irq();

        set_irq_regs(old_regs);
        return 1;
}
```

``run_timer_softirq``是一个中断处理程序的handler

```c
/*
 * This function runs timers and the timer-tq in bottom half context.
 */
static __latent_entropy void run_timer_softirq(struct softirq_action *h)
{
        struct timer_base *base = this_cpu_ptr(&timer_bases[BASE_STD]);

        __run_timers(base);
        if (IS_ENABLED(CONFIG_NO_HZ_COMMON))
                __run_timers(this_cpu_ptr(&timer_bases[BASE_DEF]));
}

TODO

```c
struct timer_list {
        /*
         * All fields that change during normal runtime grouped to the
         * same cacheline
         */
        struct hlist_node       entry;
        unsigned long           expires;
        void                    (*function)(struct timer_list *);
        u32                     flags;

#ifdef CONFIG_LOCKDEP
        struct lockdep_map      lockdep_map;
#endif
};
```
